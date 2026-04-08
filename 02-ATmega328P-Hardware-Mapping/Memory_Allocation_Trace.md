# Case Study: Spider Memory Allocation on ATmega328P

## Objective
To demonstrate how the theoretical boundaries of the ATmega328P Harvard architecture are practically enforced by the toolchain. We will map a hypothetical Spider program to its exact physical hexadecimal addresses.

---

### 1. The Source Code (Spider)
Consider the following Spider program with different types of memory requirements:

```spider
const int ID = 99;           // Constant (Read-Only)
int sensor_val = 25;         // Initialized Global Variable
int error_count = 0;         // Uninitialized Global Variable (Zeroed)

void main() {
    int local_temp = 30;     // Local Variable (Dynamic)
    while(true) {}
}

2. Flash Memory Mapping (Program Space)
The Linker places immutable data and instructions into the 32KB Flash (0x0000 - 0x3FFF).

Section,Address Range,Contents / Purpose,Spider Code Equivalent
.vectors  ,0x0000 - 0x0033  ,Interrupt Vector Table   ,JMP crt0 at 0x0000
.init     ,0x0034 - 0x0050  ,crt0 Startup Code        ,Toolchain injected code
.text     ,0x0052 - 0x0150  ,Executable Machine Code  ,The main() loop instructions
.rodata   ,0x0152 - 0x0154  ,Read-Only Data           ,const int ID = 99;

3. SRAM Memory Mapping (Data Space)
The Linker and the CPU at runtime manage the volatile 2KB SRAM (0x0100 - 0x08FF).

Section,Address,Contents / Purpose,Spider Code Equivalent
.data   ,0x0100                      ,Initialized Variables        ,int sensor_val = 25;
.bss    ,0x0102                      ,Zero-Initialized Variables   ,int error_count = 0;
Heap    ,0x0104 ->                   ,Dynamic Memory (Grows UP)    ,Typically avoided in Embedded
...,... ,Free Memory / Buffer Zone   ,Prevents collisions
Stack   ,<- 0x08FF                   ,LIFO structure (Grows DOWN)  ,int local_temp = 30;

4. The Critical Danger Zone: Stack Collision
If the main() function in the Spider code uses too many nested functions or creates massive arrays of local variables, the Stack will grow downwards past the Free Memory and crash into the .bss section (around address 0x0104).

The Result: local_temp would physically overwrite error_count, causing silent, catastrophic failure.

The Toolchain Solution: The Spider compiler's Semantic Analyzer must calculate maximum Stack depth at compile-time to warn the user if a crash is mathematically imminent.

[Detailed Case Study: Memory Allocation Trace](./Memory_Allocation_Trace.md)