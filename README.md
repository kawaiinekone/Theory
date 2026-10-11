# SERV: The Ultra-Minimalist Bit-Serial RISC-V Core

## 1. Introduction

**SERV (SErial RISC-V)**, created by Olof Kindgren, is an open-source, award-winning RISC-V processor engineered to achieve the absolute minimal physical footprint on FPGA and ASIC targets. While conventional 32-bit embedded processors instantiate parallel datapaths to process an entire word per clock cycle, SERV processes data **one bit at a time**.

Its architecture exemplifies a fundamental trade-off in digital design: **hardware resources can be minimized by substituting spatial parallel logic with temporal reuse**. Rather than instantiating 32-bit adders, wide barrel shifters, and multi-ported register files, SERV uses a single 1-bit full adder and dynamic shift registers.

```text
Parallel Datapath (e.g., PicoRV32/Ibex):

  Operand A [31:0] ──┐
                     ├──► [ 32-bit Parallel ALU ] ──► Result [31:0]   (1 cycle)
  Operand B [31:0] ──┘

Bit-Serial Datapath (SERV):

  Operand A (a0 → a1 → ... → a31) ──┐
                                    ├──► [ 1-bit ALU + Flip-Flop ] ──► Result (b0 → ... → b31)   (32 cycles)
  Operand B (b0 → b1 → ... → b31) ──┘
```

---

## 2. Microarchitectural Anatomy & Core Datapath

SERV implements the full uncompressed **RV32I** (and RV32E) base user-level integer ISA using a serialized datapath governed by an internal state counter.

```text
              +----------------------------+
              |       Instruction Bus      |
              |    (32-bit Wishbone Read)  |
              +----------------------------+
                            |
                            v
              +----------------------------+
              |    32-bit Shift Register   |
              |   Instruction / Immediate  |
              +----------------------------+
                            |
          +-----------------+------------------+
          | Decode signals                     | Serial bit streams
          v                                    v
+------------------------+        +-----------------------------+
| Stage / State Counter  |        |    Bit-Serial Register File |
|     (0 to 31 / 63)     |        |  (mapped to LUT shift RAM)  |
+------------------------+        |      rs1_ser   rs2_ser      |
          |                       +-----------------------------+
          | Shift / mux controls            |          |
          +---------------------------------+----------+
                                            |
                                            v
                                [ 1-Bit Execution Unit ]
                                  - 1-bit full adder
                                  - 1-bit carry latch
                                  - 1-bit logic / SLL / SLT
                                            |
                                            v
                                     rd_ser (sum)
                                            |
                                            v
                              [ Recirculated to shift RAM ]
```

### 2.1 The Two-Phase Temporal Pipeline (`INIT` & `RUN`)

SERV does not use spatial pipeline stages (`IF`, `ID`, `EX`, `MEM`, `WB`). Instead, instructions execute across **time**, sequenced by a 6-bit hardware state counter running between 0 and 31 (or 63):

- **`INIT` phase (cycles 0 to 31):**
  - Deconstructs instruction fields serially from the shift buffer.
  - Streams base register `rs1` and immediate values into the 1-bit adder to compute effective load/store addresses or branch targets ($PC + \text{imm}$).
  - Runs bit-serial subtraction ($rs1 - rs2$) to prepare branch condition flags.
- **`RUN` phase (cycles 32 to 63):**
  - Streams primary operands through the 1-bit ALU, runs shifts, or services Wishbone bus read/write transfers.
  - Streams the final 32-bit result bit-by-bit into the destination register (`rd`).

> **Pipelined execution optimization:** For basic register-to-register operations (`ADD`, `SUB`, `AND`, `OR`, `XOR`), SERV overlaps the `INIT` and `RUN` phases. It decodes and executes concurrently, retiring the instruction in **32 clock cycles**. Control transfers, memory load/stores, and shift operations require both phases, taking **64+ clock cycles**.

---

### 2.2 Shift-RAM Register File (`SRL16E` / `CFGLUT5`)

A standard 32-bit RISC-V register file ($32 \times 32 = 1024$ bits) with two read ports and one write port typically requires multiple dual-port distributed RAM slices or block RAM (BRAM).

SERV avoids this overhead by synthesizing the register file directly into **FPGA look-up table shift registers (SRL)**:

- On AMD/Xilinx architectures, LUTs are mapped into `SRL16E` or `SRL32E` primitives.
- Registers are stored as recirculating bit-shift loops. When an operand is read, bit 0 exits the LUT, feeds the 1-bit ALU, and is pushed back into the input stage so state is maintained.
- **Resource impact:** the entire 32-entry register file fits into **~32 to 64 FPGA LUTs**, consuming **0 Block RAMs**.

---

### 2.3 Bit-Serial ALU Implementation

The execution core contains no multi-bit parallel arithmetic units.

- **Addition & subtraction (`ADD`, `SUB`):**
  A single 1-bit full adder coupled with a 1-bit carry flip-flop. At each cycle $k \in [0, 31]$, the adder computes:

  $$\text{Sum}_k = a_k \oplus b_k \oplus C_k$$

  $$C_{k+1} = (a_k \cdot b_k) + (C_k \cdot (a_k \oplus b_k))$$

  The computed sum bit streams into `rd`, and $C_{k+1}$ is latched for the next clock edge.
- **Bitwise logic (`AND`, `OR`, `XOR`):**
  Evaluated through single 2-input boolean primitives on the incoming 1-bit streams.
- **Shift operations (`SLL`, `SRL`, `SRA`):**
  Instead of an area-heavy multiplexer-based barrel shifter, SERV implements shifting through **temporal offset delay**. The datapath pauses or delays the injection of the streaming bits into the destination register by $N$ clock cycles (governed by the 5-bit shift amount), using time rather than hardware multiplexers to position the data.
- **Set less than (`SLT`, `SLTU`):**
  Performs serial subtraction ($rs1 - rs2$) and samples the sign and carry bits at cycle 31. A single `1` or `0` bit is shifted into bit 0 of `rd`, followed by 31 cycles of zeros.

---

### 2.4 External Memory & Bus Interface (Wishbone)

SERV connects to memory systems using an external **Wishbone Classic / B3** bus master:

- **Address generation:** Memory addresses are evaluated bit-serially during the `INIT` phase and assembled into an external 32-bit parallel latch.
- **Bus handshake:** Once assembled, SERV asserts `wb_cyc` and `wb_stb`.
- **Serialization/deserialization:** For write cycles (`SW`, `SH`, `SB`), data shifts from the register file buffer into parallel memory bus lines. For read cycles (`LW`, `LH`, `LB`), incoming bus words deserialize into serial streams targeting the destination register.

---

## 3. Hazard Analysis: Why SERV Has Zero Pipeline Stalls

In conventional pipelined designs, structural, data, and control hazards introduce stalls, bubbles, or forwarding networks. SERV eliminates these hazards by construction:

| Hazard Class | Standard Parallel Pipeline Behavior | SERV Hardware Mechanism |
| :--- | :--- | :--- |
| **RAW data hazard** | Requires bypass forwarding multiplexers from `EX`/`MEM` to `ID`. | **Zero penalty.** Instructions retire across 32/64 cycles; preceding instructions finish writeback before subsequent instructions sample operands. |
| **Load-use hazard** | Demands a 1-cycle pipeline bubble and interlock stall. | **Zero bubble.** Memory latency is absorbed by Wishbone bus wait states prior to subsequent instruction issue. |
| **Control hazard (branches)** | Requires branch prediction and 1- to 3-cycle flush penalty. | **Zero flush loss.** Branch comparison resolves at cycle 31 before the target PC is updated, preventing speculative incorrect fetches. |
| **Structural hazard** | Data memory reads collide with instruction fetches on unified buses. | **Eliminated by sequencing.** Instruction fetch and data load/store execute in non-overlapping state-machine phases. |

---

## 4. Hardware Footprint & FPGA Synthesis Metrics

Reported baseline utilization across common FPGA families:

| Target Device | LUTs | Flip-Flops | Block RAMs | DSP Slices | Max Clock ($F_{\max}$) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **AMD/Xilinx Artix-7** | **125** | 164 | 0 | 0 | ~220 MHz |
| **Lattice iCE40 UltraPlus** | **198** | 164 | 0 | 0 | ~130 MHz |
| **Intel Cyclone 10 LP** | **239** | 164 | 0 | 0 | ~180 MHz |
| **Microchip PolarFire** | **~155** | 164 | 0 | 0 | ~195 MHz |

> **Note:** Resource numbers vary with specific CSR configuration, timer integration, and selected bus interfaces.

---

## 5. Architectural Trade-Off Analysis

| Metric | High-Performance Pipelined Core (e.g., Rocket, CV32E) | Minimal Bit-Serial Core (SERV) |
| :--- | :--- | :--- |
| **Datapath width** | 32-bit parallel | 1-bit serial |
| **Average CPI** | ~1.0 – 1.4 cycles/instruction | **~32 – 64 cycles/instruction** |
| **Area impact** | High (1,500 – 5,000+ LUTs) | **Minimal (<200 LUTs)** |
| **Static power** | Higher silicon area, higher leakage | Ultra-low static leakage footprint |
| **$F_{\max}$ capability** | Constrained by carry propagation and bypass muxes | High (short critical path: 1-bit full adder + latch) |

### Key Strengths

- **Ultra-dense multi-core deployments:** Fits thousands of compliant RISC-V cores on a single FPGA fabric (e.g., the *CoreScore* project demonstrating 10,000+ cores on a Xilinx VU9P).
- **Minimalist control planes:** Well-suited for power management controllers, sensor housekeeping, and board-level management.

### Practical Limitations

- **Low single-thread compute throughput:** Inefficient for high-performance integer math, complex signal filtering, or real-time control loops requiring sub-microsecond response deadlines.
- **High memory wait amplification:** The high cycle count per instruction means off-chip memory latencies can degrade overall workload execution times.

---

## 6. Security Analysis & Vulnerability Profile

A minimal gate count does not automatically guarantee a secure processing envelope.

- **Side-channel vulnerability (timing & power):**
  - *Timing channels:* Bit-serial state machines can exhibit data-dependent timing variance during memory alignment and conditional operations, presenting risks for cryptographic implementations.
  - *Power channels:* While overall power consumption is low, persistent 1-bit serial toggling across repeated clock cycles can expose localized power variations measurable via Differential Power Analysis (DPA).
- **Physical fault injection:**
  - Because an operation takes 32 to 64 clock cycles to complete, the temporal vulnerability window for clock glitching or voltage droop fault injection attacks is substantially wider than in single-cycle datapaths.
- **Hardware mitigations:**
  - Implement constant-time instruction execution wrappers for cryptographic routines.
  - Couple with hardware memory bounds checking or access control wrappers on the external Wishbone interconnect to isolate untrusted code.

---

## 7. Primary Target Use Cases

1. **System-on-Chip (SoC) housekeeping:** Handling power sequencing, reset orchestration, and clock frequency management where logic footprint is strictly budgeted.
2. **Slow protocol bridges:** Protocol translation engines for I2C, SPI, and UART sensor buses where execution deadlines are on the order of milliseconds.
3. **Hardware root-of-trust helper:** Ultra-compact controller executing low-speed authentication routines and boot signature verifications in heavily constrained environments.

---

## 8. References

1. **SERV source repository:** <https://github.com/olofk/serv>
2. **SERV architecture documentation:** <https://serv.readthedocs.io>
3. **RISC-V ISA specifications:** <https://docs.riscv.org>
