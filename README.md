# 28-Bit CPU in Logisim

A custom **28-bit CPU architecture** implemented in **Logisim**, featuring a ROM-based microprogrammed control unit, a reusable 28-bit ALU, accumulator-based datapath, and a dedicated Booth multiplication unit.

## Highlights

- **28-bit data word**
- **24-bit memory address field**
- **4-bit opcode**
- Accumulator-based datapath
- 28-bit ALU with AND, ADD, OR, and SUB
- INC/DEC implemented by reusing the existing ADD/SUB datapath with a constant-1 source
- ROM-based microprogrammed Control Unit
- **6-bit micro-sequence counter**
- **64 × 7 control ROM**
- Integrated **28 × 28 Booth multiplier**
- 56-bit multiplication result exposed as `product_high : product_low`
- Lower 28 bits of a MUL result are routed back to the CPU accumulator

## Repository Structure

```text
28-BIT-CPU/
├── 28bit_cpu.circ
├── 28bit_alu.circ
├── Booths_multiplication28bit.circ
└── README.md
```

### Circuit Files

| File | Purpose |
|---|---|
| `28bit_cpu.circ` | Main CPU, datapath, RAM interface, registers, and Control Unit |
| `28bit_alu.circ` | Standalone 28-bit ALU and supporting arithmetic/logic subcircuits |
| `Booths_multiplication28bit.circ` | Dedicated Booth multiplication datapath and controller |

> Keep all three `.circ` files in the **same directory** so Logisim can resolve the circuit-library dependencies.

## CPU Architecture

The processor uses the following major registers and datapath elements:

- **PC** — 24-bit Program Counter
- **MAR** — 24-bit Memory Address Register
- **MBR** — 28-bit Memory Buffer Register
- **IR** — 4-bit Instruction Register / opcode
- **AC** — 28-bit Accumulator
- **ALU** — 28-bit arithmetic and logic unit
- **Main Memory** — 28-bit data words addressed through the 24-bit address path
- **Control Unit** — ROM-based microprogrammed sequencer

## Instruction Format

```text
 27                     24 23                               0
+-------------------------+----------------------------------+
|      OPCODE (4 bits)    |      ADDRESS / OPERAND (24)      |
+-------------------------+----------------------------------+
```

Each machine instruction is therefore one **28-bit word**, represented conveniently as **7 hexadecimal digits**.

## Instruction Set

| Opcode | Mnemonic | Operation |
|---:|---|---|
| `0x0` | AND | `AC ← AC AND M[address]` |
| `0x1` | ADD | `AC ← AC + M[address]` |
| `0x2` | STO | `M[address] ← AC` |
| `0x3` | OR | `AC ← AC OR M[address]` |
| `0x4` | SUB | `AC ← AC - M[address]` |
| `0x5` | BUN | `PC ← address` |
| `0x6` | LDA | `AC ← M[address]` |
| `0x7` | HLT | Halt processor execution |
| `0x8` | INC | `AC ← AC + 1` |
| `0x9` | DEC | `AC ← AC - 1` |
| `0xA` | MUL | Multiply `AC` by `M[address]`; write the lower 28 bits back to `AC` |

For INC and DEC, the 24-bit operand field is unused. Typical encodings are:

```text
INC = 8000000
DEC = 9000000
HLT = 7000000
```

## ALU

The 28-bit ALU supports four primary operations selected by a 2-bit operation code:

| Operation bits | Function |
|---|---|
| `00` | AND |
| `01` | ADD |
| `10` | OR |
| `11` | SUB |

INC and DEC reuse the ADD/SUB hardware by selecting a 28-bit constant `1` as the ALU's second operand.

## Booth Multiplication

The multiplier is integrated as a separate Logisim circuit with CPU-facing control and result signals.

Conceptually:

```text
AC ---------------------> M
MBR --------------------> Q

CPU mul_start ----------> Start
Booth Done -------------> Control Unit

product_low ------------> AC write-back path
product_high -----------> Upper 28-bit product output
```

The complete multiplication result is:

```text
[ product_high (28 bits) ][ product_low (28 bits) ]
             = 56-bit product
```

The current CPU write-back path stores **`product_low` in AC**. The upper 28 bits remain available through `product_high` and can be connected to a dedicated HI register in a future extension.

### MUL Microprogram

The MUL opcode is `0xA`, so its execution microprogram starts at `0x2B`.

| µAddress | Action |
|---|---|
| `2B` | `MAR ← MBR[23:0]` |
| `2C` | Read memory operand into MBR |
| `2D` | Assert `mul_start` |
| `2E` | Wait for Booth `Done` |
| `2F` | Assert `mul_load`; write `product_low` into AC |
| `30` | Clear the micro-sequence counter and return to fetch |

## Example Program — 3 × 4

A simple RAM program can be used to demonstrate multiplication:

| Address | Value | Meaning |
|---:|---:|---|
| `0` | `6000004` | `LDA 4` |
| `1` | `A000005` | `MUL 5` |
| `2` | `2000006` | `STO 6` |
| `3` | `7000000` | `HLT` |
| `4` | `0000003` | Data = 3 |
| `5` | `0000004` | Data = 4 |
| `6` | `0000000` | Result location |

Expected lower-word result:

```text
AC = 000000C
M[6] = 000000C
```

## Running the Project

1. Clone or download this repository.
2. Keep the three `.circ` files together in the repository root.
3. Open `28bit_cpu.circ` in **Logisim 2.7.1 or a compatible Logisim version**.
4. Load a program into RAM using Logisim's memory editor.
5. Reset the CPU.
6. Advance the clock manually or enable ticks.
7. Observe the PC, MAR, MBR, IR, AC, control signals, RAM, and Booth multiplier states while the program executes.

## Design Notes

- The CPU is intentionally modular: ALU, CPU, Control Unit, and Booth multiplier are separated into reusable circuit blocks.
- The control unit uses microcode rather than a large hardwired instruction-state network.
- INC and DEC demonstrate hardware reuse instead of introducing separate arithmetic units.
- MUL uses a start/done handshake so the micro-sequencer can wait while the iterative Booth multiplier completes.
- The repository versions use **relative same-directory circuit-library references** for portability.

## Possible Extensions

- Dedicated 28-bit `HI` register for the upper half of multiplication results
- Additional arithmetic instructions
- Conditional branch instructions
- Status/flag register
- Stack and subroutine support
- Interrupt handling
- Cache or additional memory hierarchy
- Improved I/O subsystem

## Purpose

This project is intended for learning and demonstrating **computer architecture, datapath design, microprogrammed control, arithmetic circuits, instruction execution, and Logisim-based CPU construction**.
