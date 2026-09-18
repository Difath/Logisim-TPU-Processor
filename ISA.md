# Instruction Set Architecture (ISA) & Architecture Design

This document describes the 16-bit ISA used by the custom Tensor Processing Unit (TPU) and details the core architectural decisions made in the single-cycle processor design.

## 1. Instruction Formats (16-bit, Single-Cycle)
All instructions are 16 bits wide and are divided into 5 fixed fields:

| Format  | Bits [15:13] | Bits [12:10] | Bits [9:7] | Bits [6:4] | Bits [3:0] |
|---------|--------------|--------------|------------|------------|------------|
| **R-Type** | `funct3`   | `rs2`        | `rs1`      | `rd`       | `opcode`   |
| **I-Type** | \-           | \-           | \-         | \-         | \-         |
*(Note: I-Type uses `imm` across bits [15:7], followed by `rd` [6:4], and `opcode` [3:0])*

### Field Descriptions
- **opcode** (4 bits, `[3:0]`): 
  - The Most Significant Bit (`[3]`) indicates if the instruction uses the ALU (1 = ALU path, 0 = Non-ALU).
  - Bits `[2:0]` distinguish between different instructions within the same format.
- **funct3** (3 bits, `[15:13]`): Selects the specific R-Type operation. This maps directly to the `ALUop` control signal.
- **imm** (9 bits, `[15:7]`): Used only by the `li` instruction. The immediate is sign-extended to 16 bits (-256 to 255).

### R-Type Sub-formats
R-Type instructions have two logical sub-formats from an assembly syntax perspective:
- **R3-Type** (`dota rd, rs1, rs2`): Utilizes three distinct registers (`rd`, `rs1`, `rs2`).
- **R2-Type** (`add rd, rs` / `dot rd, rs`): Utilizes two registers. `rd` acts as both the destination and the first operand. The assembler duplicates `rd` into the `rs1` field (`rs1=rd`) and places `rs` into `rs2`.

## 2. Opcodes and Operations

| Instruction | Opcode | funct3 | Operation |
|-------------|--------|--------|-----------|
| `li`        | `0001` | \-     | `R[rd] <- imm` |
| `add`       | `1000` | `000`  | `R[rd] <- R[rd] + R[rs1]` |
| `dota`      | `1000` | `001`  | `R[rd] <- R[rd] + R[rs1]*R[rs2] + R[rs1+1]*R[rs2+1]` |
| `dot`       | `1000` | `010`  | `R[rd] <- R[rd]*R[rs1] + R[rd+1]*R[rs1+1]` *(Note: `rd+1` is represented as `rs1+1`)* |

## 3. Example Instruction Encoding

1. **`li R0, 2`**
   - `imm = 000000010`, `rd = 000`, `opcode = 0001`
   - Binary: `0000 0001 0000 0001` (Hex: `0x0101`)
2. **`add R3, R1`**
   - `funct3 = 000`, `rs2 = 001`, `rs1 = 011`, `rd = 011`, `opcode = 1000`
   - Binary: `0000 0101 1011 1000` (Hex: `0x05B8`)
3. **`dot R0, R2`**
   - `funct3 = 010`, `rs2 = 010`, `rs1 = 000`, `rd = 000`, `opcode = 1000`
   - Binary: `0100 1000 0000 1000` (Hex: `0x4808`)
4. **`dota R0, R4, R6`**
   - `funct3 = 001`, `rs2 = 110`, `rs1 = 100`, `rd = 000`, `opcode = 1000`
   - Binary: `0011 1010 0000 1000` (Hex: `0x3A08`)

## 4. Architectural Design Decisions

- **Hardware Simplification**: The redundancy in two-operand instructions (`add`, `dot`) is resolved during encoding (by duplicating `rd` into `rs1`), rather than adding complexity to the hardware multiplexers.
- **Register File Indexing**: The 3-bit adders responsible for calculating adjacent register addresses (`rs1+1` and `rs2+1`) are contained within the `REGISTER_FILE` module. Since `rd` is duplicated into `rs1`, the `rd+1` required by `dot` is achieved via `rs1+1`. This keeps the main datapath clean and free of redundant address adders.
- **ALU Design**: The ALU consists of adders and two multipliers, governed by a final multiplexer driven by `ALUop`. To maintain this clear separation, the immediate path for the `li` instruction bypasses the ALU entirely, routing directly to the Write-Back multiplexer (`WBSel` signal).
- **Opcode Structure**: Bit `[3]` of the opcode serves as a strict "uses ALU" flag. Bits `[2:0]` provide room for future instruction types, while the 3-bit `funct3` field allows for future ALU operations, ensuring the ISA is easily expandable without restructuring the hardware.
- **Control Unit**: The control signals `is_Li` and `is_Alu` drive the datapath. Because `add`, `dot`, and `dota` share similar routing, `is_Alu` groups their activation signals, simplifying the circuit logic and preparing the architecture for future extensions.
