# 16-bit Single-Cycle Processor with DOT Acceleration

A specialized 16-bit single-cycle processor designed as a Tensor Processing Unit (TPU) to accelerate dot product operations for the Mini-Transformer.

## Table of Contents
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Quick Start](#quick-start)
  - [Prerequisites](#prerequisites)
  - [Execution](#execution)
- [Instruction Set Architecture (ISA) Examples](#instruction-set-architecture-isa-examples)
- [Repository Structure](#repository-structure)

## Features
- **Hardware Acceleration** - Custom `dot` and `dota` instructions for rapid vector calculations.
- **Single-Cycle Design** - Fully functional datapath and control unit operating in a single clock cycle.
- **Integrated Register File** - 8 general-purpose 16-bit registers (R0-R7).

## Tech Stack
- **Environment**: Logisim-Evolution
- **Architecture**: 16-bit Single-Cycle CPU

## Quick Start

### Prerequisites
- [Logisim-Evolution](https://github.com/logisim-evolution/logisim-evolution)

### Execution
1. Open Logisim.
2. Load the `projeto3.circ` circuit file.
3. Load an Assembly program encoded in machine language into the ROM memory.
4. Advance the clock cycle (ticks) to observe the datapath in action.

## Instruction Set Architecture (ISA) Examples

> [!NOTE]
> For full technical details on instruction formats, bits layout, opcodes, and architectural design justifications, please read the [ISA.md](./ISA.md) file.

### Code Example
```assembly
li R0, 2
li R1, 3
li R2, 4
li R3, 2
li R4, 1
li R5, 5
li R6, 2
li R7, 2
add R3, R1
dot R0, R2
dota R0, R4, R6
```

### Execution Trace & Output
- `dot R0, R2`: Computes the dot product of vectors at (R0, R1) and (R2, R3).
  Calculation: R0 <- (2 * 4) + (3 * 5) = 23
- `dota R0, R4, R6`: Accumulates the dot product of vectors at (R4, R5) and (R6, R7) into R0.
  Calculation: R0 <- 23 + (1 * 2) + (5 * 2) = 35

Expected Final Register State: `R0 = 35`

## Repository Structure
```text
.
├── projeto3.circ        # Main Logisim simulator file (Datapath, Control Unit, and Registers)
├── README.md            # Project documentation and Quick Start
└── ISA.md               # Instruction Set Architecture and design decisions
```
