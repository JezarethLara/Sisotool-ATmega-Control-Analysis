# Case Study: The Command-Line Pipeline

## Objective
To demonstrate the exact sequence of CLI (Command Line Interface) operations required to transform source code into a physical silicon state using the GNU Toolchain. The Spider project can wrap these commands in an automated `Makefile` or build system.

---

### Step 1: Compilation and Assembly
The frontend parses the code and generates a relocatable object file. We use the `-mmcu` flag to explicitly tell the compiler the hardware constraints.
```bash
# Translates main.c into an object file (main.o) for the ATmega328P
avr-gcc -Os -DF_CPU=16000000UL -mmcu=atmega328p -c -o main.o main.c

-Os: Instructs the optimizer to prioritize Size over Speed (critical for 32KB Flash).

-DF_CPU: Defines the clock speed (16 MHz) for delay calculations.

Step 2: Linking
The linker merges main.o with the avr-libc standard library and the crt0 startup code to produce an ELF file containing absolute memory addresses.

# Links objects into an Executable and Linkable Format (ELF)
avr-gcc -mmcu=atmega328p main.o -o main.elf

Step 3: Hex Extraction (The Payload)
The ELF file contains debugging metadata that the ATmega328P cannot understand. We must extract only the pure machine code into an Intel HEX format.

# Strips metadata and outputs pure Flash memory payload
avr-objcopy -O ihex -R .eeprom main.elf main.hex

-R .eeprom: Tells the utility to ignore the EEPROM data section, focusing only on Flash.

Step 4: Flashing the Silicon
The toolchain's job is technically done. Now, avrdude acts as the courier to deliver the payload.

# Flashes the hex file using an Arduino Uno as the ISP programmer
avrdude -c arduino -p m328p -P COM3 -b 115200 -U flash:w:main.hex:i

-p m328p: Verifies the physical chip signature matches the ATmega328P.

-U flash:w: Instructs the programmer to Write (w) to the Flash memory.

By automating this pipeline, the Spider Toolchain can provide a seamless "One-Click Build" experience for its users.

[Detailed Case Study: GNU Command-Line Pipeline Trace](./GNU_Pipeline_Trace.md)