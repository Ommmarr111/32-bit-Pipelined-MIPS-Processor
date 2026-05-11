# 32-Bit Pipelined MIPS Processor

![Architecture](https://img.shields.io/badge/Architecture-MIPS%2032--bit-blue?style=for-the-badge&logo=buffer&logoColor=white)
![Simulator](https://img.shields.io/badge/Simulator-Logisim-orange?style=for-the-badge&logo=probot&logoColor=white)
![Pipeline](https://img.shields.io/badge/Pipeline-5%20Stage-green?style=for-the-badge&logo=githubactions&logoColor=white)
![Hazard](https://img.shields.io/badge/Hazard%20Handling-Forwarding%20%2B%20Stall-red?style=for-the-badge&logo=dependabot&logoColor=white)
![University](https://img.shields.io/badge/Fayoum%20University-ECE%20Dept-purple?style=for-the-badge&logo=academia&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed%20%E2%9C%85-brightgreen?style=for-the-badge)

> A fully functional 32-bit RISC processor based on the MIPS architecture, designed and simulated using **Logisim**. Implements both a single-cycle and a 5-stage pipelined datapath with hazard detection, forwarding, and stall logic.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Instruction Set Architecture](#instruction-set-architecture)
- [Phase 1 — Single-Cycle Processor](#phase-1--single-cycle-processor)
  - [Datapath Components](#datapath-components)
  - [ALU](#alu)
  - [Control Units](#control-units)
  - [Testing](#testing)
- [Phase 2 — Pipelined Processor](#phase-2--pipelined-processor)
  - [Pipeline Stages](#pipeline-stages)
  - [Hazard Handling](#hazard-handling)
  - [PC Control Unit](#pc-control-unit)
- [Project Structure](#project-structure)
- [Tools Used](#tools-used)
- [Team](#team)

---

## Overview

This project was developed as part of the Electronics & Communication Engineering program at **Fayoum University**, Faculty of Engineering. It consists of two phases:

1. **Phase 1** — A fully functional **single-cycle MIPS processor** where every instruction completes in one clock cycle.
2. **Phase 2** — A **5-stage pipelined version** of the same processor with forwarding and hazard detection units, improving throughput significantly.

Both phases were built from scratch in Logisim using modular, reusable components.

---

## Features

- 32-bit RISC MIPS architecture
- 32 general-purpose registers (R0 hardwired to zero)
- Three instruction formats: R-type, I-type, SB-type
- Word-addressable instruction and data memory (2²⁰ locations each)
- 20-bit Program Counter
- Complete ALU supporting arithmetic, logic, shift, rotate, and comparison operations
- Modular control logic: Main CU, ALU CU, Branch CU
- Full 5-stage pipeline: IF → ID → EX → MEM → WB
- Data forwarding (EX/MEM → EX, MEM/WB → EX, WB → EX)
- Load-use hazard detection with automatic stalling
- Control hazard handling via pipeline flushing (Kill signal)

---

## Instruction Set Architecture

### Register File
- **32 registers** (R0–R31), each 32 bits wide
- R0 is hardwired to `0` — reads always return `0`, writes have no effect
- Two read ports (RS1, RS2) and one write port (Rd)
- Clocked write — occurs on the rising edge when `RegWrite` is high

### Instruction Formats

| Format | Layout (MSB → LSB) |
|--------|---------------------|
| **R-Type** | `F[11]` \| `S2[5]` \| `S1[5]` \| `D[5]` \| `Op[6]` |
| **I-Type** | `Imm16[16]` \| `S1[5]` \| `D[5]` \| `Op[6]` |
| **SB-Type** | `ImmU[11]` \| `S2[5]` \| `S1[5]` \| `ImmL[5]` \| `Op[6]` |

### Supported Instructions

| Category | Instructions |
|----------|-------------|
| Arithmetic | `ADD`, `ADDI`, `SUB`, `MUL` |
| Logic | `AND`, `ANDI`, `OR`, `ORI`, `XOR`, `XORI`, `NOR`, `NORI` |
| Shift / Rotate | `SLL`, `SLLI`, `SRL`, `SRLI`, `SRA`, `SRAI`, `ROR`, `RORI` |
| Comparison | `SLT`, `SLTU`, `SEQ`, `SEQI` |
| Memory | `LW`, `SW` |
| Branch | `BEQ`, `BNE`, `BLT`, `BGE`, `BLTU`, `BGEU` |
| Jump | `JALR` (and pseudo-instructions `JR`, `J`) |
| Special | `SET`, `SSET` |

---

## Phase 1 — Single-Cycle Processor

In this phase, every instruction — fetch, decode, execute, memory access, write-back — completes within a **single clock cycle**. All components are purely combinational except the register file and data memory (which are clocked).

### Datapath Components

#### Register File
- Decoder with `RegWrite`-controlled AND gates selects the target register
- Two multiplexers allow flexible output port selection
- Covers R1–R31 (R0 is hardwired to 0)

#### Instruction Memory
- Read-only, word-addressable, 2²⁰ entries
- Addressed directly by the 20-bit PC
- PC increments by **1** per cycle (word-addressable, not byte-addressable)
- Output fields: `Op[6]`, `RS1[5]`, `RS2/Rd[5]`, `ImmL/ImmU/Func[11]`, `Imm16[16]`

#### Data Memory
- Separate from instruction memory, 2²⁰ × 32-bit words
- Controlled by `MemRead` and `MemWrite` signals
- Used by `LW` (read) and `SW` (write) instructions
- Output routed through a MUX to select between ALU result and memory data

#### Program Counter (PC)
- 20-bit special-purpose register
- Updated each cycle to one of three targets:
  - `PC + 1` — sequential execution
  - `PC + sign_extend({ImmU, ImmL})` — branch target
  - `RS1 + sign_extend(Imm16)` — jump target (JALR)
- Two MUXes controlled by `Branch` and `Jump` signals select the next PC

### ALU

#### 1-Bit ALU Module
Each 1-bit ALU cell supports:
- `AND`, `OR`, `XOR`, `NOR`
- `ADD` (full adder with CarryIn/CarryOut)
- `SUB` (via `Binvert = 1`, `CarryIn = 1`)

#### 32-Bit ALU
- 32 cascaded 1-bit ALU modules
- Additional dedicated logic for shift/rotate operations and SLT/SLTU/SEQ
- Outputs: `ALUResult[32]`, `ZeroFlag`, `LessFlag` (signed), `LessUFlag` (unsigned), `OverFlow`, `CarryOut`

| Operation Type | Examples |
|---------------|---------|
| Arithmetic | ADD, SUB, MUL |
| Logic | AND, OR, XOR, NOR |
| Shift | SLL, SRL, SRA, ROR |
| Comparison | SLT (sign bit), SLTU (borrow), SEQ (XOR→NOT) |

### Control Units

#### Main Control Unit
Decodes the 6-bit opcode and drives the datapath:

| Signal | Purpose |
|--------|---------|
| `RegWr` | Enable register file write |
| `MemRd` | Enable data memory read |
| `MemWr` | Enable data memory write |
| `MemToReg` | Select write-back source (memory vs ALU) |
| `ALUSrc` | Select ALU operand B (register vs immediate) |
| `ExtOp` | 0 = zero-extend, 1 = sign-extend immediate |
| `Jump` | Active for JALR |
| `Branch` | Active for branch instructions |
| `R-TypeEnable` | Signals ALU CU to use Func field |
| `SW`, `SSET` | Special signals for store and upper-half write |

> **Special case:** Opcode `000000` covers R-type, SW, and SSET — differentiated by the Func field.

#### ALU Control Unit
Generates fine-grained ALU signals (`CarryIn`, `Ainvert`, `Binvert`, `Operation`, `SLTCtrl`, `BranchCtrl`, `ShftCtrl`, `ResCtrl`) based on:
- `Opcode` + `Func` when `R-TypeEnable = 1` (R-type)
- `Opcode` only when `R-TypeEnable = 0` (I-type)

#### Branch Control Unit
Evaluates branch conditions using ALU flags:

| Instruction | Opcode | Condition |
|-------------|--------|-----------|
| `BEQ` | 18 | `Zero == 1` |
| `BNE` | 19 | `Zero == 0` |
| `BLT` | 20 | `Less == 1` |
| `BGE` | 21 | `Less == 0` |
| `BLTU` | 22 | `LessU == 1` |
| `BGEU` | 23 | `LessU == 0` |

The `BranchTaken` output is ANDed with `Branch` from the Main CU to drive the PC MUX.

### Testing

A comprehensive test program was run in simulation, verifying:
- Register initialization (`SET`, `SSET`)
- Memory store/load (`SW`, `LW`)
- Arithmetic (`ADD`, `ADDI`, `SUB`, `MUL`)
- Logic (`AND`, `OR`, `XOR`)
- Comparisons (`SLT`)
- Branching (`BEQ`, `BGE`, `BNE`, unconditional loop)
- Jumps and return (`JALR`, `JR`)
- Shift/rotate (`SRL`, `SRA`, `RORI`)

All simulation results matched expected outputs.

---

## Phase 2 — Pipelined Processor

The single-cycle processor was extended into a classic **5-stage RISC pipeline**, allowing multiple instructions to be in-flight simultaneously — significantly improving throughput.

### Pipeline Stages

```
IF ──► ID ──► EX ──► MEM ──► WB
     IF/ID  ID/EX  EX/MEM  MEM/WB
```

Each stage is separated by a **pipeline register** that latches all required data and control signals at the end of each clock cycle.

#### Stage 1 — Instruction Fetch (IF)
- Fetches instruction from Instruction Memory using the PC
- Stores `Instruction` and `PC+1` in the **IF/ID** register
- Supports stalling (via `Enable` signal) for hazard handling
- Flushes incorrectly fetched instructions on branches/jumps

#### Stage 2 — Instruction Decode (ID)
- Decodes opcode and generates control signals
- Reads RS1 and RS2 from the register file
- Includes the **Extender** sub-block for:
  - Branch target: `PC + sign_extend({ImmU, ImmL})`
  - Jump address: `RS1 + sign_extend(Imm16)`
  - ALU immediate: sign-extended or zero-extended based on `ExtOp`
- Evaluates branch condition via the Branch Control Unit (early resolution)
- Latches all signals into the **ID/EX** register

**ID/EX stored signals:**

| Signal | Description |
|--------|-------------|
| `A`, `B` | Register operand values |
| `Rd` | Destination register |
| `IMM` / `imm1_ex` | Extended immediate value |
| `PC+1` | Used for JALR return address |
| `Control Signals` | Forwarded to EX, MEM, WB stages |
| `Jump` | Indicates JALR instruction |

#### Stage 3 — Execute (EX)
- ALU performs the computation using forwarded or register values
- MUXes select between register values and forwarded data (from Forwarding Unit)
- Result and metadata stored in the **EX/MEM** register

**EX/MEM stored signals:**

| Signal | Description |
|--------|-------------|
| `ALUres` | ALU computation result or memory address |
| `ALUFwd` | Forwarded source operand (e.g., for SW) |
| `B` | Store data (RS2 value for SW) |
| `Rd` | Destination register |
| `PC+1` | Return address for JALR |
| `IMM` | Immediate (passed through) |
| `Control Signals` | Drives MEM and WB behavior |
| `Jump` | Control flow flag |

#### Stage 4 — Memory Access (MEM)
- Reads from data memory for `LW` (`MemRead = 1`)
- Writes to data memory for `SW` (`MemWrite = 1`)
- Non-memory instructions simply pass ALU result through
- Results stored in the **MEM/WB** register

**MEM/WB stored signals:**

| Signal | Description |
|--------|-------------|
| `DataOut` | Value read from memory (for LW) |
| `Rd` | Destination register |
| `IMM16` | Immediate for SET/SSET |
| `RegWrite` | Whether to write back |
| `SSET` | Triggers special upper-half write behavior |

#### Stage 5 — Write Back (WB)
- Writes the final result into the destination register
- Source selection:
  - ALU result — arithmetic/logic instructions
  - `DataOut` — load instructions (`LW`)
  - `IMM16` — immediate instructions (`SET`, `SSET`)
- Controlled by `RegWrite` and `SSET` signals from MEM/WB

---

### Hazard Handling

#### Data Hazards — Forwarding Unit
Bypasses stale register values by forwarding results directly from later stages back to the ALU inputs.

| Condition | Forward A | Forward B | Source |
|-----------|-----------|-----------|--------|
| No hazard | 00 | 00 | Register file |
| EX hazard (Rs) | 10 | — | EX/MEM stage |
| EX hazard (Rt) | — | 10 | EX/MEM stage |
| MEM hazard (Rs) | 01 | — | MEM/WB stage |
| MEM hazard (Rt) | — | 01 | MEM/WB stage |
| WB hazard (Rs) | 11 | — | WB stage |
| WB hazard (Rt) | — | 11 | WB stage |
| Multiple hazards | Priority to newest result | | EX/MEM > MEM/WB |

#### Load-Use Hazard — Stall Mechanism
Forwarding cannot resolve a load-use hazard (LW result needed in the very next cycle). The **Hazard Detection Unit** detects this and:

1. **Freezes** the PC and IF/ID register for one cycle
2. **Inserts a NOP bubble** into the EX stage
3. Allows the LW to complete its MEM stage, then forwards the result

**Detection condition:**
```
if (ID/EX.MemRead &&
   (ID/EX.RegisterRt == IF/ID.RegisterRs ||
    ID/EX.RegisterRt == IF/ID.RegisterRt))
    → STALL
```

#### Control Hazards — PC Control Unit
Handles branches and jumps by stalling the PC and flushing incorrectly fetched instructions.

| Branch | Jump | PC Control | Kill 1 |
|--------|------|------------|--------|
| 0 | 0 | 1 (stall) | 0 |
| 0 | 1 | 1 (stall) | 1 (flush) |
| 1 | 0 | 1 (stall) | 1 (flush) |
| 1 | 1 | 0 (update) | 1 (flush) |

- `Kill 1 = Branch OR Jump` — flushes the IF-stage instruction
- `PC Control = NOT(Branch AND Jump)` — only allows update when both are asserted

---

## Project Structure

```
/
├── Phase1_SingleCycle/       # Logisim file for single-cycle processor
├── Phase2_Pipelined/         # Logisim file for pipelined processor
├── Documentation.pdf         # Full project report with circuit diagrams
└── README.md
```

---

## Tools Used

- **[Logisim](http://www.cburch.com/logisim/)** — Digital circuit design and simulation
- Custom ISA based on 32-bit MIPS architecture

---

## Team

| Name | Contributions |
|------|--------------|
| **Omar Ahmed Othman** | ALU Control Unit, Register File, PC, Main Control Unit, Pipeline Registers, Extender, full integration of both phases, simulation & testing |
| **Ahmed Osama Abd-Elghaffar** | 1-bit & 32-bit ALU, Branch Control Unit, Main Control Unit, PC Control Unit, Hazard Detect Forward & Stall Unit, integration & bug fixing |
| **Mohamed Nady Mahmoud** | Instruction Memory, Data Memory, pipeline stage assistance, report writing |

**Supervised by:**
- Prof. Dr. Gihan Naguib — Associate Professor, Department of Electrical Engineering
- Eng. Jihad Awad — Teaching Assistant, Department of Electrical Engineering

**Institution:** Fayoum University, Faculty of Engineering — Electronics & Communication Engineering

---

> *This project was a two-phase effort to understand processor design from the ground up — starting with the simplicity of a single-cycle implementation and evolving into the performance-oriented world of pipelining.*
