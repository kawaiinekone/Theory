# Comprehensive Microarchitectural Specification

### Pipelining, Hazards, Dynamic Scheduling, and Superscalar Execution

---

## Table of Contents

1. [Pipeline Execution Foundations](#1-pipeline-execution-foundations--the-physics-of-latency-vs-throughput)
2. [Pipeline Hazards](#2-pipeline-hazards-physical-causes-bypassing-networks-and-branch-prediction)
3. [The ILP Wall & Out-of-Order Architecture](#3-the-instruction-level-parallelism-wall--out-of-order-ooo-architecture)
4. [Superscalar Scaling Limits](#4-superscalar-scaling-limits--the-physical-realities-of-wide-issue)
5. [Architectural Comparison Matrix](#5-architectural-comparison-matrix)
6. [Worked Example: Slide 8 Assignment B2](#6-real-world-execution-slide-8-assignment-b2-fully-solved)

---

## 1. Pipeline Execution Foundations & The Physics of Latency vs. Throughput

### 1.1 The Synchronous Instruction Assembly Line

Pipelining decomposes the execution of an Instruction Set Architecture (ISA) into discrete, balanced combinatorial processing stages separated by clocked state-retention elements (**pipeline barrier registers**).

- **Instruction Latency (L):** The absolute wall-clock time required for a single instruction to traverse from Fetch to Writeback.

  ```text
  L = N × T_clk = N × (T_comb_max + T_cq + T_setup + T_skew)
  ```

  > **Pipelining never decreases the latency of an individual instruction.** It strictly *increases* it, due to the finite propagation delay (`T_cq`), setup time margin (`T_setup`), and clock jitter/skew overhead (`T_skew`) of each sequential barrier register.

- **System Throughput (TP):** The number of retired instructions committed per unit time.

  ```text
  TP = Instructions Retired / Time = IPC / T_clk = IPC × F_clk
  ```

  By dividing a monolithic single-cycle datapath of delay `T_monolithic` into **N** roughly equivalent sub-blocks (`T_comb_max ≈ T_monolithic / N`), the achievable clock frequency `F_clk` scales up to **N×**, scaling system throughput by up to **N×** despite the per-instruction latency degradation.

**Concurrent staircase space-time timeline:**

```text
Inst \ Cycle │ C1   C2   C3   C4   C5   C6   C7   C8   C9
─────────────┼─────────────────────────────────────────────
i1: lw       │ IF ─ ID ─ EX ─ MEM ─ WB
i2: add      │      IF ─ ID ─ EX ─ MEM ─ WB
i3: sub      │           IF ─ ID ─ EX ─ MEM ─ WB
i4: or       │                IF ─ ID ─ EX ─ MEM ─ WB
i5: and      │                     IF ─ ID ─ EX ─ MEM ─ WB
                                              ▲
             Steady state: 1 instruction commits per clock edge (IPC = 1.0)
```

### 1.2 The Canonical 5-Stage Classic RISC Datapath

| Stage | Name | Description |
|:-----:|------|-------------|
| **IF** | Instruction Fetch | The Program Counter (PC) outputs the virtual/physical instruction address to Instruction Memory (`imem`) or L1 I-Cache. Simultaneously, a dedicated adder generates the default sequential target `PC + 4`. |
| **ID** | Instruction Decode & Register Fetch | The 32-bit instruction word is partitioned into control fields. The Register File asynchronous read ports are driven by `rs1` (bits `[19:15]`) and `rs2` (bits `[24:20]`). Concurrently, the Immediate Generator extracts and sign-extends the immediate formats (I, S, B, U, J). |
| **EX** | Execute / Address Generation | The ALU operates on operands from the register file or bypass multiplexers. For memory instructions it computes the effective base-plus-offset address (`rs1 + imm`). Branch conditions and targets (`PC + imm`) are evaluated. |
| **MEM** | Memory Access | For loads (`lw`), the computed address drives the Data Memory (`dmem`) read port. For stores (`sw`), data is committed to the memory array. Pure register operations transparently propagate through this stage. |
| **WB** | Writeback | A multiplexer arbitrates between the ALU result and the memory read data, presenting the selected 32-bit word to the Register File write port, where it is synchronously clocked into `rd` on the rising edge. |

### 1.3 Theory vs. Reality: The Silicon Realities of Pipelining

**The Textbook Ideal:** Subdividing a single-cycle datapath into N pipeline stages yields an exact **N×** frequency scaling and **N×** speedup.

**The Physical Reality:**

- **Stage Imbalance** — Combinatorial workloads cannot be sliced into mathematically identical delay slices. The clock period is bound to the slowest single stage:

  ```text
  T_clk = max(T_IF, T_ID, T_EX, T_MEM, T_WB) + T_overhead
  ```

  If the MEM stage requires **2.8 ns** while Decode requires **1.2 ns**, the other 4 stages waste time waiting for the trailing edge of the slowest phase.

- **Pipeline Register Overhead** — Each pipeline register introduces physical setup and clock-to-Q times (`T_cq + T_setup ≈ 0.15–0.3 ns`). In a 30-stage superpipeline, this sequential overhead consumes up to **30–40%** of the entire clock period, imposing a law of diminishing returns on pipeline depth.

- **Clock Skew and Jitter** — Due to physical RC parasitic delays across the die, clock edges do not strike all pipeline registers simultaneously. Designers must pad the clock cycle with safety guardbands, directly penalizing `F_max`.

---

## 2. Pipeline Hazards: Physical Causes, Bypassing Networks, and Branch Prediction

### 2.1 The Three Classical Hazard Taxonomies

A **pipeline hazard** is any architectural or structural condition that prevents the next instruction in the stream from executing in its designated clock cycle.

```text
                    PIPELINE HAZARDS
                           │
      ┌────────────────────┼────────────────────┐
      ▼                    ▼                    ▼
 1. Structural          2. Data             3. Control
 • Resource collisions  • RAW (true flow)   • Branch direction
 • Memory port sharing  • WAR (name/anti)   • Target uncertainty
 • Multi-cycle dividers • WAW (name/output) • Exception traps
```

### 2.2 Structural Hazards & Arbitration

- **Hardware collision:** Occurs when simultaneous instructions require the same physical execution or memory resource.
- **Textbook model:** If instruction and data memory share a single physical port (Von Neumann single-bus model), an Instruction Fetch in Stage 1 and a Data Load (`lw`) in Stage 4 create a port collision.
- **Silicon reality:**
  - Modern processors resolve this at the L1 level with **split Harvard caches** (independent L1 I-Cache and L1 D-Cache arrays, each with dedicated address/data buses).
  - Multi-cycle non-pipelined units (such as an iterative 32×32 radix-4 sequential divider taking **8–16 cycles**) introduce structural stalls. If an incoming division follows an uncompleted division, the decode/dispatch stage must freeze until the functional unit asserts its internal *ready* flag.

### 2.3 Data Hazards: True vs. False Dependencies

```asm
(i1) add x1, x2, x3   # Writes architectural x1 in WB (Cycle 5)
(i2) sub x4, x1, x5   # Reads x1 in ID (Cycle 3)  -> RAW hazard (true flow)
(i3) mul x1, x6, x7   # Writes x1 again           -> WAW with i1, WAR with i2
(i4) or  x8, x1, x9   # Reads updated x1          -> RAW hazard on i3
```

| Hazard | Type | Dependence | Description |
|:------:|:----:|:----------:|-------------|
| **RAW** (Read-After-Write) | True | Flow | `i2` strictly requires the value generated by `i1`. Data flows physically through time and space. **Cannot be removed by renaming**; it represents genuine computation order. |
| **WAR** (Write-After-Read) | False | Anti | `i3` writes a register name that `i2` must read first. If `i3` finishes before `i2` reads the operand, `i2` receives corrupt data. Occurs purely because register names are reused (finite ISA registers). |
| **WAW** (Write-After-Write) | False | Output | `i3` writes the same register name as `i1`. If `i3` commits before `i1`, the architectural register file is left with outdated state. |

> **Note:** In an in-order 5-stage pipeline, WAR and WAW are physically harmless because instructions advance, access registers, and commit in strict program order. **Only RAW hazards pose an operational threat.**

### 2.4 Forwarding (Bypassing) Networks and Silicon Routing Realities

To prevent the pipeline from stalling 2 cycles on every ALU-to-ALU RAW dependency, **forwarding** taps the pipeline barrier registers directly into the inputs of the EX-stage ALU.

```text
 ┌────────────┐     ┌────────────┐     ┌────────────┐
 │ ID/EX Reg  │ ──► │ EX/MEM Reg │ ──► │ MEM/WB Reg │
 └─────┬──────┘     └─────┬──────┘     └─────┬──────┘
       │                  │                  │
       │          EX/MEM Forward Bus   MEM/WB Forward Bus
       │          (distance: 1 cycle)  (distance: 2 cycles)
       ▼                  │                  │
 ┌───────────┐            │                  │
 │  Mux rs1  │ ◄──────────┴──────────────────┘
 └─────┬─────┘
       ▼
 ┌───────────┐
 │    ALU    │
 └───────────┘
```

#### Forwarding Comparator Control Equations

The Forwarding Unit monitors the source register pointers from ID/EX and compares them against destination registers in subsequent stages.

**EX hazard** (forward from EX/MEM to ALU input A):

```verilog
forward_a = (ex_mem_regwrite && (ex_mem_rd != 5'd0) && (ex_mem_rd == id_ex_rs1)) ? 2'b10 : ...
```

**MEM hazard** (forward from MEM/WB to ALU input A):

```verilog
forward_a = (mem_wb_regwrite && (mem_wb_rd != 5'd0) &&
            !(ex_mem_regwrite && (ex_mem_rd != 5'd0) && (ex_mem_rd == id_ex_rs1)) &&
            (mem_wb_rd == id_ex_rs1)) ? 2'b01 : 2'b00;
```

#### The Load-Use Hazard Barrier

When an instruction immediately reads a register targeted by an immediately preceding `lw`, forwarding cannot give a 0-cycle resolution:

```text
         Cycle 1   Cycle 2   Cycle 3   Cycle 4   Cycle 5
lw       [IF] ───► [ID] ───► [EX] ───► [MEM] ──► [WB]
                                        (data ready)
                                           │
                                  CANNOT GO BACK IN TIME!
                                           ▼
add                [IF] ───► [ID] ───► [EX]
                                     (needs data HERE!)
```

The data is physically extracted from SRAM at the **end of Cycle 4 (MEM)**. The dependent ALU operation requires that data at the **start of Cycle 4 (EX)**. A causality violation cannot be solved with copper wire.

- **Hardware Hazard Detection Unit:** Inserts a hardware bubble (NOP control flags: `RegWrite=0`, `MemWrite=0`) into the ID/EX register, while freezing the PC and IF/ID pipeline registers for **1 clock cycle**.

**Theory vs. Reality**

| | |
|---|---|
| **Textbook** | *"Just use forwarding muxes."* |
| **Reality** | Forwarding buses are high-capacitance global routing traces traversing several physical layout blocks. In a 64-bit core, 64-bit-wide dual bypass buses crossing long distances add significant wire-load (RC) delay, frequently making the forwarding multiplexer selection tree the **primary critical timing path of the entire core**. |

### 2.5 Control Hazards & Branch Prediction Dynamics

When a conditional branch (`beq`, `bne`, `blt`) is evaluated in the EX stage (Cycle 3), the processor has already fetched the subsequent two instructions (`PC+4`, `PC+8`) into the IF and ID stages. If the branch evaluates as **taken**, these instructions are incorrect speculative work and must be synchronously flushed by clearing the IF/ID and ID/EX registers with zero-control bubbles (NOPs).

**2-bit saturating counter state machine:**

```text
State  Name                Prediction   On Taken (T)   On Not-Taken (N)
─────  ──────────────────  ───────────  ─────────────  ─────────────────
 00    Strongly Not-Taken  Not-Taken     → 01           stay at 00
 01    Weakly Not-Taken    Not-Taken     → 10           → 00
 10    Weakly Taken        Taken         → 11           → 01
 11    Strongly Taken      Taken         stay at 11     → 10

 00 ◄──N── 01 ◄──N── 10 ◄──N── 11
    ──T──►    ──T──►    ──T──►
```

- **Hysteresis in 2-bit predictors:** A 1-bit predictor toggles its prediction on every unexpected outcome. In a loop of 100 iterations it mispredicts **twice**: once on loop exit (expected taken, but exits), and again on loop re-entry (expected not-taken from the previous exit, but the loop repeats). The 2-bit saturating counter introduces state inertia: an exit transition merely degrades state `11` (Strongly Taken) to `10` (Weakly Taken) without flipping the prediction (MSB) bit, incurring **only 1 mispredict** per loop execution.

- **Branch Target Buffer (BTB):** Predicting direction (taken vs. not-taken) is useless if the fetch stage does not know the target address. A BTB is an SRAM-based cache indexed by the low-order bits of the current fetch PC. It stores the target address computed during previous executions, enabling single-cycle fetch redirection.

**Theory vs. Reality**

| | |
|---|---|
| **Textbook** | *"Flush pipeline on mispredict; costs 1–2 cycles."* |
| **Reality** | In high-frequency, deeply pipelined industrial cores (e.g., Intel Golden Cove in Alder Lake, or AMD Zen 4), the branch resolution path is **16 to 22 stages** deep. A single misprediction incurs a catastrophic **16–22 cycle penalty**. If a core executes a branch every 5 instructions with an 85% accurate predictor, it spends more than half its execution time flushing and refilling wasted pipeline stages. |

---

## 3. The Instruction-Level Parallelism Wall & Out-of-Order (OoO) Architecture

### 3.1 The In-Order Latency Tolerance Failure

An in-order pipeline operates under an inviolable dispatch invariant: **instruction i+1 can never overtake an uncompleted instruction i.**

- If instruction *i* triggers an L2 or LLC cache miss requiring a **150-cycle** main DRAM access, the entire pipeline freezes.
- Hundreds of completely independent arithmetic operations queued behind it sit idle in decode, even though their source operands are available and their target ALUs are vacant.

```text
IN-ORDER SERIAL FREEZE:

[ lw  x1, 0(x2)    ] ──► DRAM access: 150 cycles latency
[ add x3, x1, x4   ] ──► Blocked on RAW x1
[ sub x6, x7, x8   ] ──► STALLED! (independent, but locked behind add)
[ or  x9, x10, x11 ] ──► STALLED! (independent, but locked behind add)
```

To break this bottleneck, **Out-of-Order (OoO) execution** decouples instruction decode from instruction execution, providing architectural *latency tolerance* by maintaining a pool (the **Instruction Window**) of decoded work, dynamically identifying ready instructions, and firing them across execution ports irrespective of original program order.

### 3.2 Register Renaming: Eliminating False Dependences (WAR & WAW)

#### The Problem of Register Starvation

ISAs define a limited set of register names (e.g., 32 architectural registers in RV32I, or 16 GPRs in x86-64). Compilers must reuse identical names across independent code segments, introducing artificial anti-dependencies (WAR) and output dependencies (WAW) that block parallel execution.

```text
ARCHITECTURAL CODE              PHYSICAL REGISTER RENAMING MAP

(i1) add x1, x2, x3   ──────►   P80 <= P10 + P20
(i2) sub x4, x1, x5   ──────►   P81 <= P80 - P30   (true RAW on P80 preserved)
(i3) mul x1, x6, x7   ──────►   P82 <= P40 * P50   (P82 allocated: WAR/WAW eliminated)
(i4) div x8, x1, x9   ──────►   P83 <= P82 / P60   (true RAW on P82 preserved)
```

#### Renaming Infrastructure

| Component | Role |
|-----------|------|
| **Physical Register File (PRF)** | A physically expanded register bank containing **128 to 300+** real hardware registers. |
| **Register Alias Table (RAT)** | An array mapping each architectural register (`x0`–`x31`) to its current physical register (`P0`–`PN`). |
| **Free List** | A FIFO tracking available, unallocated physical registers. |

**Execution flow:** Every instruction with an architectural destination register pulls a brand-new physical register from the Free List and updates the RAT. Subsequent instructions reading that register query the RAT and bind directly to that physical register index. **WAR and WAW hazards cease to exist in silicon; only genuine RAW dataflow remains.**

### 3.3 Dynamic Scheduling: Tomasulo's Algorithm & Reservation Stations

Developed by Robert Tomasulo at IBM, this distributed execution paradigm dynamically tracks dataflow readiness through distributed tagging.

```text
        TOMASULO'S DISTRIBUTED OOO EXECUTION ENGINE

           [ In-Order Decode & Register Renaming ]
                            │
          ┌─────────────────┴─────────────────┐
          ▼                                   ▼
┌────────────────────────────┐     ┌────────────────────────────┐
│ Reservation Stations (ALU) │     │ Reservation Stations (Load)│
├─────┬──────┬─────┬────┬────┤     ├─────┬──────┬─────┬────┬────┤
│ Tag │ Busy │ Vj  │ Vk │ Qj │     │ Tag │ Busy │ Vj  │ Vk │ Qj │
├─────┼──────┼─────┼────┼────┤     ├─────┼──────┼─────┼────┼────┤
│ RS1 │  1   │ [D] │ -  │RS3 │     │ RS4 │  1   │ [D] │[D] │ 0  │
└─────┴──────┴─────┴────┴──┬─┘     └─────┴──────┴─────┴────┴──┬─┘
                           │ Ready                            │ Ready
                           ▼                                  ▼
                     ┌───────────┐                      ┌───────────┐
                     │ Integer   │                      │ Address   │
                     │ Execution │                      │ Gen (AGU) │
                     └─────┬─────┘                      └─────┬─────┘
                           └──────────────┬───────────────────┘
                                          ▼
                              COMMON DATA BUS (CDB)
                     Broadcast: (Tag = RS4, Value = 0xDEADBEEF)
                                          │
                  ┌───────────────────────┴───────────────────────┐
                  ▼                                               ▼
     Snagged by RS1                                  Buffered into the
     (Qj cleared, Vj updated)                        Reorder Buffer (ROB)
```

**Reservation Station tuple structure**

| Field | Meaning |
|:-----:|---------|
| `Tag` | Unique identifier assigned to this reservation station entry. |
| `Busy` | 1-bit flag indicating the station is currently occupied. |
| `Vj`, `Vk` | Value fields holding actual 32/64-bit operand data when available. |
| `Qj`, `Qk` | Tag fields specifying which external reservation station will produce the missing operand. A value of **zero** indicates the operand is present in `Vj`/`Vk`. |

**The Common Data Bus (CDB)**

- When an execution unit computes its result, it drives the CDB with an arbitration-granted broadcast packet: `{Producer_Tag, Computed_Data}`.
- Every reservation station in the core monitors the CDB. If a station's `Q` tag matches the broadcast tag, it latches the data into its `V` field and clears `Q`. When both operands are present (`Qj = 0`, `Qk = 0`), the instruction immediately flags its readiness to dispatch to an execution unit.

### 3.4 In-Order Retirement: The Reorder Buffer (ROB) & Precise Interrupts

#### The Problem of Architectural Scrambling

If instructions execute out-of-order and write directly to the architectural register file or memory:

- An early instruction (Instruction 10) might divide by zero, triggering an OS exception.
- But Instruction 14, an independent addition, has already executed and overwritten its target register.
- The architectural state of the processor is now unrecoverable: the OS cannot cleanly resume or inspect the faulting thread state.

#### The Solution: In-Order Commit via ROB

The Reorder Buffer is a circular FIFO queue that tracks **every instruction in flight in strict original program order**.

```text
REORDER BUFFER (CIRCULAR FIFO)

        HEAD (oldest in-flight instruction)
          │
          ▼
Entry   │ State │ Spec?   │ Arch Dest │ Phys Dest │ Exception
────────┼───────┼─────────┼───────────┼───────────┼───────────
ROB_01  │ DONE  │ No      │ x1        │ P80       │ None        ──► COMMIT / RETIRE
ROB_02  │ WAIT  │ No      │ x4        │ P81       │ None            (architectural state updates)
ROB_03  │ DONE  │ Branch? │ x0        │ -         │ None
ROB_04  │ DONE  │ Spec    │ x6        │ P82       │ DIV_ZERO!   ──► held until it reaches HEAD
          ▲
          │
        TAIL (allocation of youngest decoded instructions)
```

**The commit mechanism**

- Instructions allocate an entry at the **Tail** of the ROB during the in-order decode/rename stage.
- Instructions execute out-of-order across parallel ALUs, routing their results into their assigned ROB entry.
- Only when an instruction reaches the **Head** of the ROB is it permitted to commit (retire) its state to the architectural register file and allow its pending store data to commit to the L1 memory system.

**Handling exceptions and branch mispredictions**

- If a speculative branch mispredicts, or if an instruction faults (e.g., `ROB_04` in the diagram), the ROB does **not** handle it immediately. It waits until that faulting instruction reaches the Head of the FIFO.
- If an older branch mispredicts, the ROB immediately resets its Tail pointer to the branch entry, instantaneously invalidating every younger speculative instruction. The speculative physical registers are recycled back to the Free List, and the RAT restores its checkpointed baseline state.

> **Result:** Execution is fully out-of-order, but the programmer-visible architectural state is 100% in-order and clean (**Precise Exceptions**).

### 3.5 Memory Disambiguation & The Load/Store Queue (LSQ)

Memory hazards cannot be resolved by standard register renaming because memory addresses are only computed dynamically inside the execution stage.

**The problem:** Consider this sequence:

```asm
sw  x1, 0(x2)    # Address computation delayed (pending x2)
lw  x3, 0(x4)    # Address computed immediately: Address = 0x1000
```

Can the load execute immediately? If `x2 + 0 == x4 + 0 == 0x1000`, the load reading memory out-of-order will read stale data (a violation of RAW memory coherence).

**The Load/Store Queue architecture**

| Structure | Behavior |
|-----------|----------|
| **Store Queue (SQ)** | Holds pending memory writes in program order. Stores **never** commit data to cache until the instruction leaves the ROB Head. |
| **Load Queue (LQ)** | Monitors all in-flight loads. |
| **Store-to-Load Forwarding** | When a load computes its address, it queries the Store Queue. If an older store targets the exact same address and its data is already computed, the Store Queue bypasses the byte-sliced data directly into the load's execution register without accessing the L1 Cache array. |
| **Memory Speculation Violation Recovery** | If a load executes speculatively assuming no address collision, and an older store subsequently resolves to the identical address, the Load Queue flags a violation, forcing the core to flush the pipeline and re-execute the load. |

---

## 4. Superscalar Scaling Limits & The Physical Realities of Wide Issue

### 4.1 Multiple-Issue Taxonomy

A scalar core issues at most 1 instruction per cycle (`IPC ≤ 1.0`). A **superscalar processor** duplicates hardware datapaths to fetch, decode, rename, dispatch, execute, and retire **N** instructions simultaneously per cycle (N = 2, 4, 8, …).

```text
4-WIDE SUPERSCALAR UNROLLED FRONTEND PIPELINE

Fetch Block:      [ Fetch 0 ]   [ Fetch 1 ]   [ Fetch 2 ]   [ Fetch 3 ]
                       │             │             │             │
Decode / Rename:  [ Decode 0]   [ Decode 1]   [ Decode 2]   [ Decode 3]
                       │             │             │             │
                ═════════════════════════════════════════════════════
                INTER-INSTRUCTION DEPENDENCY CHECKING MESH  O(N^2)
                Does Inst 1 depend on Inst 0? Does Inst 3 depend on 0, 1, 2?
                ═════════════════════════════════════════════════════
                       │             │             │             │
Dispatch to RS:   [  ALU 0  ]   [  ALU 1  ]   [ Load/Store] [ Branch/ALU]
```

### 4.2 The N² Complexity Explosion & Diminishing Returns

Increasing the issue width N from 2 to 4 to 8 does not yield linear performance scaling. It triggers a steep silicon complexity barrier:

- **Intra-cycle dependency checking (O(N²) comparators):** In a single clock cycle, the rename stage must check every instruction in the incoming packet against all older instructions in that same packet to identify internal RAW hazards.

  ```text
  Comparisons = N × (N − 1) / 2 = O(N^2)
  ```

  For an 8-wide machine, this requires **28** full 5-bit register-equality comparator networks operating combinatorially in a fraction of a clock cycle.

- **Register file port proliferation:** For an N-wide machine with 2 source operands and 1 destination per instruction:

  ```text
  Read Ports = 2N        Write Ports = N
  ```

  An 8-wide design requires **16 read ports** and **8 write ports** on a single physical register file. The physical area of a multi-ported SRAM cell scales roughly with the square of its port count:

  ```text
  Area_cell ∝ (Read Ports + Write Ports)^2 = (2N + N)^2 = 9N^2
  ```

  A 24-port register file cell is massive, introduces immense parasitic capacitance on bitlines, and severely limits operational frequency (`F_max`).

- **CDB broadcast congestion:** If 6 execution units complete simultaneously, they all require access to broadcast on the Common Data Bus. Fabricating 6 independent 64-bit CDBs spanning 64 distributed reservation stations creates an unroutable congestion problem on modern metal layers.

> **Conclusion:** General-purpose OoO cores hit an aggressive efficiency ceiling between **4-wide and 8-wide** issue. Beyond this, power and die area explode quadratically while delivering marginal real-world IPC gains due to memory latency barriers and basic-block branch limits.

---

## 5. Architectural Comparison Matrix

| Architectural Parameter | In-Order 5-Stage Scalar Core | 4-Wide Out-of-Order Superscalar Core |
|---|---|---|
| **Peak Instruction Issue** | 1 instruction per cycle | 4 instructions per cycle |
| **Typical Sustained IPC** | 0.6–0.85 (limited by stalls) | 1.8–2.8 (extracts hidden ILP) |
| **Register Structure** | 32 architectural registers (`x0`–`x31`) | 32 arch names mapped to 128–256 physical registers |
| **Dynamic Renaming (WAR/WAW)** | None. WAR/WAW resolved implicitly by strict order | Fully resolved via Register Alias Table (RAT) & Free List |
| **Hazard Resolution (RAW)** | Pipeline stalls (bubbles) + direct bypass muxes | Out-of-order execution via reservation station matching |
| **Exception Handling** | Imprecise without complex shadow registers | Strictly precise via in-order ROB commit |
| **Memory Hazard Handling** | 1-cycle load-use stall | Speculative Load/Store Queue with store-forwarding |
| **Hardware Complexity** | ≈ 20k–50k gates; tiny silicon footprint | ≈ 2M–10M+ gates; massive die area & power |
| **Dominant Workload Fit** | IoT edge nodes, microcontrollers, low-power sensors | High-performance servers, desktops, mobile flagships |

---

## 6. Real-World Execution: Slide 8 Assignment B2 Fully Solved

Step-by-step timing analysis for the assembly sequence given in **µArch Lab Assignment B2 (Slide 8)**.

### Given Target Program Sequence

```asm
(i1) lw  x1, 0(x2)
(i2) add x3, x1, x4
(i3) sub x5, x6, x7
(i4) or  x8, x3, x9
```

### Task 1: 5-Stage Space-Time Trace WITHOUT Forwarding

**Register File contract:** writes occur in the first half of the cycle; reads occur in the second half.

**Hazards identified**

- `i2` (add) requires `x1` produced by `i1` (lw). `i1` writes `x1` during Cycle 5 (WB), so `i2` cannot read `x1` in ID until Cycle 5. Thus `i2` stalls in ID across Cycles 3 and 4 (**2 bubble cycles**).
- `i4` (or) requires `x3` produced by `i2` (add). `i2` writes `x3` in WB during Cycle 8, so `i4` cannot read `x3` in ID until Cycle 8.

```text
TIMING TRACE (NO FORWARDING)
Cycle:      C1   C2   C3   C4   C5   C6   C7   C8   C9   C10  C11
────────────────────────────────────────────────────────────────────
(i1) lw     IF   ID   EX   MEM  WB
(i2) add         IF   ID*  ID*  ID   EX   MEM  WB
(i3) sub              IF   IF*  IF*  ID   EX   MEM  WB
(i4) or                              IF   ID*  ID   EX   MEM  WB
────────────────────────────────────────────────────────────────────
* = stall bubble

Total Execution Time = 11 clock cycles
```

### Task 2: 5-Stage Space-Time Trace WITH Forwarding

**Bypass capabilities**

- `EX/MEM → EX` (0-cycle delay for ALU-to-ALU).
- `MEM/WB → EX` (0-cycle delay for operations separated by 2 instructions).

**Hazards identified**

- `i1` (lw) produces `x1` at the end of MEM (Cycle 4). `i2` (add) requires `x1` at the start of EX. This causality violation forces **exactly 1 load-use stall cycle** in Cycle 4. In Cycle 5, the loaded value is forwarded directly to ALU Input A.
- `i4` (or) requires `x3` from `i2` (add). Because `i3` (sub) sits between them, `i2` has already advanced far enough by the time `i4` reaches EX, so no further stall bubbles are needed.

```text
TIMING TRACE (WITH FORWARDING)
Cycle:      C1   C2   C3   C4   C5   C6   C7   C8   C9
─────────────────────────────────────────────────────────
(i1) lw     IF   ID   EX   MEM  WB
(i2) add         IF   ID   ID*  EX   MEM  WB
(i3) sub              IF   IF*  ID   EX   MEM  WB
(i4) or                         IF   ID   EX   MEM  WB
─────────────────────────────────────────────────────────
* = stall bubble

Total Execution Time = 9 clock cycles (1 load-use bubble paid)
```

### Task 3: Compiler Reordering for Zero-Stall Optimization

**Hazard target:** the 1-cycle stall bubble between `lw x1` and `add x3, x1, x4`.

**Optimization analysis:** Instruction `i3` (`sub x5, x6, x7`) is entirely independent:

- It does not read `x1` or `x2`.
- It does not write to `x1`, `x2`, `x3`, or `x4`.

**Rescheduled program sequence:**

```asm
lw  x1, 0(x2)      # Cycle 1: fetch initiates
sub x5, x6, x7     # Fills the load-use delay slot with useful work
add x3, x1, x4     # x1 now available; forwarded with ZERO stalls
or  x8, x3, x9     # Receives x3 via EX/MEM forwarding with ZERO stalls
```

> **Performance:** Total execution drops to **8 clock cycles**, the theoretical limit for 4 instructions in a 5-stage pipeline (5 base cycles + 3 additional instructions = 8 cycles).

### Task 4: Effective CPI Calculation

Using the performance-degradation formulation:

```text
Effective CPI = CPI_ideal + Σ (Frequency × Penalty)
```

**Given parameters**

| Parameter | Value |
|-----------|:-----:|
| `CPI_ideal` | 1.0 |
| Load instruction frequency | 30% (0.30) |
| Loads followed immediately by their dependent use | 40% (0.40) |
| Load-use stall penalty | 1 cycle |
| Branch instruction frequency | 15% (0.15) |
| Branch misprediction rate | 8% (0.08) |
| Branch misprediction penalty | 2 cycles |

**Calculations**

```text
Stall_load-use = 0.30 × 0.40 × 1 = 0.120 cycles/instruction
Stall_branch   = 0.15 × 0.08 × 2 = 0.024 cycles/instruction

Effective CPI  = 1.0 + 0.120 + 0.024 = 1.144
Effective IPC  = 1 / 1.144 ≈ 0.874
```

---

<sub>End of specification.</sub>
