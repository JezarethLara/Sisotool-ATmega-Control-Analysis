# Phase 4: Spider Integration Logic & Application Binary Interface (ABI)

## Abstract
For the **Spider** programming language to successfully compile for the ATmega328P, it cannot simply generate arbitrary AVR assembly. It must strictly adhere to a defined **Application Binary Interface (ABI)**. This phase dictates the architectural rules for function calls, argument passing, and stack management. Adhering to this ABI is mandatory if Spider intends to use standard C libraries (`avr-libc`) or achieve deterministic execution.

---

## 1. The Spider Calling Convention
When a Spider function calls another function, the CPU must know exactly where the arguments are stored and who is responsible for cleaning up the memory. We propose adopting the standard AVR-GCC calling convention to ensure interoperability.

### 1.1 Argument Passing (Registers vs. Stack)
To maximize speed, Spider must pass function arguments via registers rather than pushing them to the slow SRAM stack.
* **Rule:** Arguments are allocated left to right, starting from register $R25$ downwards to $R8$.
* **Example:** `void set_motor_speed(int pin, int speed);`
  * `pin` (8-bit) is passed in $R24$.
  * `speed` (16-bit) is passed in $R22$ and $R23$.
* If a Spider function has too many arguments and exhausts the available registers, the remaining arguments MUST be pushed to the Stack.

### 1.2 Return Values
When a Spider function finishes executing, it must leave its result in a predictable location for the caller to retrieve.
* 8-bit return values: Placed in $R24$.
* 16-bit return values: Placed in $R24$ (Low Byte) and $R25$ (High Byte).

---

## 2. Register Volatility (Caller-Saved vs. Callee-Saved)
The Spider Backend must manage the preservation of register states during function calls to prevent data corruption.

* **Call-Used Registers (Volatile):** $R18 - R27$, $R30, R31$. 
  * *Logic:* The calling function must assume these registers will be overwritten and destroyed by the child function. If the caller needs the data, it must save it to the Stack before the `CALL` instruction.
* **Call-Saved Registers (Non-Volatile):** $R2 - R17$, $R28, R29$.
  * *Logic:* The child function is allowed to use these, but it MUST save their original values to the Stack upon entry (`PUSH`) and restore them before exiting (`POP`).

---

## 3. Foreign Function Interface (FFI): Calling C from Spider
If the Spider team wants to avoid reinventing the wheel, the language should be able to call pre-compiled C functions (like `_delay_ms()` or `printf()`).

Because we are enforcing the AVR-GCC ABI in Section 1, the Spider compiler only needs to generate an `EXTERN` assembly directive. The GNU Linker will seamlessly connect the Spider object files with the `avr-libc` object files, drastically reducing the Spider development time.

---

## 4. Hardware Interaction: I/O Port Mapping
Spider needs a native way to control hardware pins without relying on C macros. The compiler must map Spider variables to physical memory addresses.

**Spider Syntax Proposal:**
```spider
hardware_port PORTB at 0x25;
PORTB = 0xFF; // Turns on all pins on Port B

Backend Translation Requirement:
The compiler must recognize the hardware_port keyword and translate the assignment into a direct I/O memory write, avoiding the standard SRAM .data section.

LDI R16, 0xFF
OUT 0x05, R16   ; 0x05 is the I/O mapped address for PORTB

[Detailed Case Study: ABI and Function Call Trace](./ABI_and_Calling_Convention_Trace.md)