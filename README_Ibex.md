# Ibex: The 2-Stage In-Order RISC-V Core

## 1. Introduction

**Ibex** (originally developed as *zero-riscy* by the PULP Platform at ETH Zürich, now maintained and formally verified by lowRISC) is a production-grade 32-bit microcontroller core. It serves as the reference Root-of-Trust CPU in the Google/lowRISC **OpenTitan** silicon security project.

Unlike SERV (bit-serial) or PicoRV32 (multi-cycle FSM), Ibex introduces an overlapped, pipelined datapath, collapsing execution into just two physical stages to maximize efficiency while keeping verification simple.

---

## 2. Microarchitectural Anatomy: The 2-Stage Pipeline

Ibex cuts the traditional 5-stage RISC pipeline down to two balanced stages:

```text
[ Stage 1: IF (Instruction Fetch) ]
   - Program counter generation & fetch FIFO (pre-fetch buffer)
   - Compressed instruction decompressor (RVC expander: 16-bit -> 32-bit)
   - Bus interface (pipelined instruction request / grant / rvalid)
                      │
                      ▼
[ Stage 2: ID / EX (Decode, Execute, Memory, Writeback) ]
   - Register file access (dual-port read, single-port write)
   - RV32IMC control & immediate generation
   - ALU, hardware multiplier / divider (M-extension)
   - Data memory access (LSU: Load-Store Unit)
   - Register writeback & architectural state commit
```

### 2.1 Detailed Stage Operations

- **Instruction Fetch (IF stage):**
  - **Pre-fetch buffer / FIFO:** Uses an internal 2-to-4 entry fetch FIFO to decouple memory bus latency from the execution stage.
  - **Compressed decoder:** Detects 16-bit compressed instructions (C extension) combinationally and expands them into canonical 32-bit RV32I instructions before passing them to the ID/EX boundary.
  - **Branch target application:** When a branch or jump triggers in Stage 2, the new target redirects Stage 1 directly, flushing the prefetch buffer.
- **Decode / Execution (ID/EX stage):**
  - Combines register operand read, ALU arithmetic, branch resolution, data memory load/store generation, and writeback into a single physical pipeline boundary.
  - **ALU operations** (`ADD`, `SUB`, logic, shifts): Retire in 1 clock cycle (CPI = 1 for basic arithmetic).
  - **Memory transactions** (`LW`, `SW`): The stage stalls while the Load-Store Unit (LSU) asserts bus requests and awaits the memory response.
  - **Branches & jumps:** Evaluated directly in this stage. When taken, a redirection signal is sent back to Stage 1, incurring a 1-cycle branch bubble.

---

## 3. Hazard Handling & Interlock Mechanics

Because execution is concentrated in Stage 2, pipeline hazard logic is substantially simpler than in 4- or 5-stage cores:

| Hazard Class | Ibex Behavior |
| :--- | :--- |
| **RAW: ALU-to-ALU** | **Non-existent** between sequential instructions. Stage 2 finishes execution and commits the result before the next instruction from the fetch buffer is decoded, so dependent instructions (`add x1, x2, x3` followed by `sub x4, x1, x5`) execute without stalls. |
| **RAW: load-use** | Stage 2 simply holds in a wait-state until the bus transaction completes, creating an atomic interlock without complex bypass networks. |
| **Structural** | Avoided by prioritizing data bus requests over instruction fetch requests on shared memory buses, using the fetch FIFO to absorb instruction-side stalls. |
| **Control (branches / traps)** | When a branch condition evaluates to true, the target PC is calculated combinationally and fed to the Stage 1 fetch logic. The pre-fetched instruction in the fetch buffer is invalidated, resulting in a fixed **1-cycle** control penalty. |

---

## 4. Physical Subsystems & Verification Hardening

### 4.1 Register File Implementation Options

- **Flip-flop based:** Synthesizes registers into standard logic flip-flops, enabling fully asynchronous read access within the execution stage.
- **FPGA / ASIC memory macros:** Supports synchronous dual-port RAM macros, inserting a single-cycle setup state when needed.

### 4.2 Hardware Multiplier / Divider (M-Extension)

- **Fast multiplier:** Uses a single-cycle or 3-cycle parallel multiplier leveraging DSP blocks.
- **Iterative / slow multiplier:** Iterates across 32 cycles for minimal area in deeply constrained configurations.

### 4.3 Security & Fault Hardening (OpenTitan Features)

- **Lockstep redundancy:** Ibex supports instantiation as a dual-core lockstep pair with temporal skew (shadow core running delayed cycles) and parity-checked comparator logic to detect physical fault injection.
- **Dummy instruction insertion:** Injects randomized dummy operations to obscure side-channel power signatures during cryptographic routines.
- **Register file scrambling / integrity:** Adds parity bits to register entries to detect bit-flips caused by laser or voltage fault attacks.

---

## 5. Implementation Metrics (FPGA Synthesis Baseline)

For an RV32IMC configuration without security lockstep:

| Target Device | LUTs | Flip-Flops | Block RAMs | DSP Blocks | Max Frequency ($F_{\max}$) | CoreMark / MHz |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **AMD/Xilinx Artix-7** | ~1,600 – 2,200 | ~1,200 – 1,600 | 0 – 1 | 0 – 3 | ~120 – 150 MHz | ~2.4 – 2.9 |
| **Microchip PolarFire** | ~1,800 – 2,400 | ~1,300 – 1,700 | 0 – 1 | 0 – 3 | ~100 – 130 MHz | ~2.4 – 2.9 |
| **Intel Cyclone 10 LP** | ~2,000 – 2,800 | ~1,400 – 1,800 | 0 – 1 | 0 – 3 | ~100 – 120 MHz | ~2.4 – 2.9 |

---

## 6. Architectural Trade-Off Analysis

| Metric | SERV (Bit-Serial) | PicoRV32 (Multi-Cycle) | Ibex (2-Stage Pipeline) |
| :--- | :--- | :--- | :--- |
| **Datapath width** | 1-bit | 32-bit | 32-bit |
| **Execution model** | Bit-serial iterative | Multi-cycle state machine | Pipelined (2-stage) |
| **Average CPI** | ~32 – 64 | ~3 – 4 | ~1.2 – 1.6 |
| **Branch penalty** | 0 cycles (multi-cycle) | 0 cycles (multi-cycle) | 1 cycle |
| **Logic utilization** | ~125 LUTs | ~850 LUTs | ~1,800 LUTs |
| **Timing closure** | Easiest (short paths) | High $F_{\max}$ | Balanced (moderate critical path in ID/EX) |

### Strengths

- **High IPC for low area:** Achieves a CPI close to 1 for arithmetic code without needing multi-stage forwarding crossbars.
- **Formally verified:** Rigorously checked via the OpenTitan verification testbench suite, making it one of the most bug-free open-source cores available.
- **Extensive security options:** Built-in hooks for physical fault detection, bus integrity, and side-channel mitigation.

### Limitations

- **Stage 2 critical path:** Packing operand read, ALU execution, and data address generation into one stage restricts maximum clock frequency compared to 4- or 5-stage designs.
- **Higher area than pure microcontrollers:** Uses more logic than PicoRV32 due to compressed instruction decoding and prefetch FIFO buffers.

---

## 7. Primary Target Use Cases

1. **Hardware Root-of-Trust (RoT):** Managing key storage, crypto coprocessors, and secure boot validation (e.g., in OpenTitan and secure enclaves).
2. **Main system microcontrollers:** Running real-time control loops, sensor data ingestion, and FreeRTOS tasks where consistent single-cycle integer execution is required.
3. **Safety-critical embedded systems:** Automotive and industrial IoT applications leveraging dual-core lockstep configurations to satisfy functional safety requirements.
