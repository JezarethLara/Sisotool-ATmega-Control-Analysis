# Phase 2: Microarchitecture and Hardware Mapping of the ATmega328P

## Abstract
To successfully retarget the Spider Toolchain, the compiler backend and linker must possess an exact topographical map of the target silicon. This phase documents the microarchitectural constraints of the Atmel AVR ATmega328P, an 8-bit Advanced Virtual RISC microcontroller. We analyze its Modified Harvard architecture, strict memory boundaries, Register File, and Interrupt Vector Table (IVT), establishing the physical rules the toolchain must obey to prevent fatal faults such as stack overflows or invalid opcodes.

---

## 1. Architectural Paradigm: The Modified Harvard Model
Unlike general-purpose host machines (x86_64) which utilize a von Neumann architecture (shared memory for data and instructions), the ATmega328P operates on a **Modified Harvard Architecture**.
* **Physical Separation:** The CPU accesses Program Memory (Flash) and Data Memory (SRAM) via entirely separate physical buses. 
* **Execution Pipeline:** This separation allows the CPU to fetch the next instruction from Flash while simultaneously reading/writing data to SRAM in a single clock cycle, achieving nearly 1 MIPS (Million Instructions Per Second) per MHz.
* **Compiler Implication:** The Spider Compiler cannot treat memory as a single flat array. Pointers to Flash (constants) must be handled differently than pointers to SRAM (variables), typically requiring special assembly instructions like `LPM` (Load Program Memory).

---

## 2. Memory Topography and Linker Constraints
The Toolchain's Linker Script (`.ld`) must be programmed with absolute boundaries. If the Spider linker places a variable outside these bounds, the system will crash.

### 2.1 Program Memory (Flash)
* **Capacity:** 32 Kbytes (organized as 16K x 16-bit words, as all AVR instructions are 16 or 32 bits wide).
* **Address Space:** `0x0000` to `0x3FFF`.
* **Sections:** Contains the `.text` section (executable code) and the `.rodata` section (read-only constants). 
* **Bootloader Safety:** The upper section of Flash is often reserved for the bootloader. The Spider toolchain must ensure the application code does not overwrite this sector.

### 2.2 Data Memory (SRAM)
* **Capacity:** 2 Kbytes. This is the tightest bottleneck in the system.
* **Address Space:** `0x0100` to `0x08FF`.
* **Sections:** Contains `.data` (initialized variables) and `.bss` (zero-initialized variables).
* **The Stack:** The stack grows downwards from the top of the RAM (`0x08FF`). If the Spider compiler does not aggressively manage memory in the Optimizer stage, the Stack will grow downward and crash into the `.bss` section, causing catastrophic memory corruption.

---

## 3. The Instruction Set Architecture (ISA) & ALU
The ATmega328P implements a highly optimized RISC ISA with 131 instructions. 
* **ALU Operations:** The Arithmetic Logic Unit is directly connected to the 32 general-purpose registers. It performs arithmetic and logical operations in a single clock cycle.
* **Status Register (SREG):** Located at I/O address `0x3F`. It holds mathematical flags such as Carry (C), Zero (Z), and Overflow (V). The Spider Backend must evaluate the SREG immediately after operations to implement conditional branching (e.g., translating a Spider `if` statement using the `BRNE` - Branch if Not Equal opcode).

---

## 4. The Register File (GPRs)
The compiler's Register Allocation algorithm must manage 32 x 8-bit General Purpose Registers ($R0$ to $R31$).
* **Direct Access:** These registers are mapped directly into the lowest addresses of the data space (`0x0000 - 0x001F`), preceding the actual SRAM and I/O registers.
* **Pointer Registers:** The six highest registers ($R26$ to $R31$) are grouped into three 16-bit indirect address registers: **X, Y, and Z pointers**. The Spider compiler will rely heavily on these pointers to iterate through arrays or access memory offsets dynamically.

---

## 5. System Initialization and the IVT
A microcontroller does not have an Operating System to launch programs. The Spider Toolchain itself must generate the critical startup sequence.

### 5.1 The Interrupt Vector Table (IVT)
At the absolute lowest addresses of Flash (`0x0000`), the Linker must place the IVT. 
* **Vector 1 (`0x0000`):** The RESET Vector. This must contain a `JMP` instruction pointing to the initialization code.
* **Vectors 2-26 (`0x0002 - 0x0032`):** Hardware interrupts (Timers, External Interrupts, ADC).

### 5.2 The C-Runtime Equivalent (`crt0`)
Before the Spider `main()` function can execute, the toolchain must inject a tiny, invisible assembly routine that:
1. Initializes the Stack Pointer (SP) to `0x08FF`.
2. Clears the `.bss` section in SRAM to zero.
3. Copies the `.data` section values from Flash into SRAM.
4. Executes a `CALL main` to start the user's program.
Without this step, no Spider program will ever run.

[Detailed Case Study: Compilation Trace](./Compilation_Trace_Case_Study.md)