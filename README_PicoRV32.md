# PicoRV32: The Multi-Cycle FSM RISC-V Core

## 1. Introduction

**PicoRV32** was developed by Claire Wolf (YosysHQ) as a compact, robust **RV32IMC** core. Instead of chopping an instruction across space (like a 3-stage or 5-stage pipeline) or across 32 bit-serial clock cycles (like SERV), PicoRV32 uses a **multi-cycle, non-pipelined finite-state machine (FSM)**.

It processes full 32-bit words in parallel, but steps through distinct sequential cycles for each phase of an instruction.

---

## 2. The Microarchitecture & FSM Cycle Flow

An instruction in PicoRV32 walks through sequential states without overlapping another instruction:

```text
   +-------------------------------------------------------------+
   |                                                             |
   v                                                             |
[ FETCH ] ──► [ DECODE / REG_READ ] ──► [ EXEC ] ──► [ MEM ] ──► [ WB ]
```

- **`FETCH` (1 to 2 cycles):**
  - Sets `mem_addr = pc`, asserts `mem_valid`, and waits for memory acknowledgment (`mem_ready`).
  - Latches the 32-bit instruction into an internal register once valid.
- **`DECODE / REG_READ` (1 cycle):**
  - Slices the opcode and register indices, and sign-extends the immediate.
  - Reads `rs1` and `rs2` from the register file.
- **`EXEC` (1 to 32 cycles):**
  - Drives the parallel 32-bit ALU.
  - Basic operations (`ADD`, `SUB`, bitwise `AND`/`OR`/`XOR`, and conditional branches) complete in 1 cycle.
  - Shifts (`SLL`, `SRL`, `SRA`) use an iterative shifter by default, taking between 1 and 32 cycles depending on the shift amount.
- **`MEM` (variable cycles, loads and stores only):**
  - Drives the effective target address calculated during execution out to the bus.
  - Holds the core in wait states until the external memory bus asserts `mem_ready`.
- **`WB` (writeback, 1 cycle):**
  - Writes the ALU result or memory load word into `rd`.
  - Updates the `pc` register with sequential `PC + 4` or the branch target, then loops back to `FETCH`.

Because every instruction finishes completely before the next one starts, PicoRV32 achieves an average throughput of **3 to 4 clock cycles per instruction** ($\text{CPI} \approx 3\text{–}4$), corresponding to an effective IPC of 0.25 to 0.33.

---

## 3. Physical Subsystems & RTL Design Choices

### 3.1 The Register File (BRAM vs LUTRAM)

- PicoRV32 can map its register file to dedicated FPGA Block RAM (BRAM). Because BRAM has a synchronous 1-cycle read latency, the FSM uses separate register-address-setup and register-read states.
- Alternatively, it can synthesize registers into distributed LUTRAM/flip-flops for single-cycle combinational read, cutting an FSM state at the cost of FPGA logic slices.
- Register `x0` is hardwired to zero via combinational read gating, and write enables are suppressed when `rd == 5'b00000`.

### 3.2 The Shifter Trade-Off (Iterative vs Barrel Shifter)

- **Default iterative shifter:** Shifts 1 or 2 bits per clock cycle. This saves around 150 to 250 LUTs, but shift-heavy code (like hashing or cryptography) stalls for up to 32 cycles per shift.
- **Single-cycle barrel shifter (`BARREL_SHIFTER = 1`):** Instantiates full multi-stage multiplexers to perform shifts in 1 cycle, increasing area but stabilizing cycle count.

### 3.3 Native Memory Bus Protocol

- Uses a minimal 4-phase handshake interface: `mem_valid`, `mem_ready`, `mem_addr`, `mem_wdata`, `mem_wstrb`, and `mem_rdata`.
- Can be wrapped with optional bridges:
  - `picorv32_axi`: AXI4-Lite master for Vivado/AMBA systems.
  - `picorv32_wb`: Wishbone B4 master for open-source SoC platforms.

---

## 4. Hazard Elimination by Construction

PicoRV32 completely avoids classical pipeline hazards because it is non-pipelined:

| Hazard Class | PicoRV32 Behavior |
| :--- | :--- |
| **RAW data hazard** | **Zero penalty.** Instruction N writes its result into the register file during `WB` before instruction N+1 even enters `FETCH`. |
| **Load-use hazard** | **Zero bubbles.** The FSM pauses in the `MEM` state until data is returned from memory, commits to the register file, and only then starts the dependent instruction. |
| **Control hazard (branches)** | **Zero flush penalty.** Branch conditions and targets are calculated in the `EXEC` state. If taken, the PC is redirected immediately before the next instruction fetch starts. No speculative instructions are ever flushed. |
| **Structural hazard** | **No contention.** Unified memory buses are shared naturally because instruction fetches and data accesses occur on distinct, non-overlapping FSM cycles. |

---

## 5. Implementation Metrics (FPGA Synthesis Baseline)

For a standard RV32IMC configuration with native bus interface on mid-range FPGAs:

| Target Device | LUTs / LEs | Flip-Flops | BRAMs | Max Clock ($F_{\max}$) |
| :--- | :---: | :---: | :---: | :---: |
| **AMD/Xilinx Artix-7** | ~750 – 950 | ~450 – 550 | 0 – 1 | ~250 – 300 MHz |
| **Microchip PolarFire** | ~850 – 1,000 | ~480 | 0 – 1 | ~200 MHz |
| **Intel Cyclone 10 LP** | ~900 – 1,100 (LEs) | ~500 | — | ~180 – 220 MHz |

---

## 6. Architectural Verdict

### Strengths

- High clock speeds ($F_{\max}$) due to very shallow combinational logic paths.
- Small footprint.
- Simple formal verification (no complex forwarding matrices or hazard stalls to deadlock).

### Weaknesses

- Low single-thread throughput ($\text{CPI} \approx 3\text{–}4$).
- Cannot hide off-chip memory access latency, since memory stalls freeze the entire core.

### Best Use Cases

Deterministic FPGA board controllers, peripheral glue logic (DDR/PLL initializers, SPI/UART handlers), and embedded state machines where low resource count and high clock frequency matter more than IPC.
