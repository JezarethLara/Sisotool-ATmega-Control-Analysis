# Case Study: Spider ABI and Function Call Trace

## Objective
To demonstrate how the Spider compiler enforces the AVR-GCC Application Binary Interface (ABI) during function calls. We will trace how arguments are passed via registers and how Spider achieves Foreign Function Interface (FFI) interoperability with standard C libraries.

---

### 1. The Source Code (Spider)
Consider a Spider program that calculates a value and then calls a native C function (`_delay_ms`) from `avr-libc`.

```spider
// Declare an external C function
extern void _delay_ms(int ms);

// Spider function with two 16-bit arguments
int calculate_speed(int distance, int time) {
    return distance / time;
}

void main() {
    // Call Spider function
    int current_speed = calculate_speed(100, 5);
    
    // Call C Library function
    _delay_ms(250);
}

2. Trace: The Caller (main)Before main() jumps to calculate_speed, it must place the arguments exactly where the ABI dictates (Registers $R24$ downwards).Backend Assembly Generation for calculate_speed(100, 5):

; Argument 1: 'distance' = 100 (Passed in R24 and R25)
LDI R24, 100      ; Low byte of 100
LDI R25, 0        ; High byte of 100

; Argument 2: 'time' = 5 (Passed in R22 and R23)
LDI R22, 5        ; Low byte of 5
LDI R23, 0        ; High byte of 5

; Execute the call
CALL calculate_speed

; Argument 1: 'distance' = 100 (Passed in R24 and R25)
LDI R24, 100      ; Low byte of 100
LDI R25, 0        ; High byte of 100

; Argument 2: 'time' = 5 (Passed in R22 and R23)
LDI R22, 5        ; Low byte of 5
LDI R23, 0        ; High byte of 5

; Execute the call
CALL calculate_speed

3. Trace: The Callee (calculate_speed)The child function assumes its data is already waiting in $R24$ and $R22$. It performs the division. According to the ABI, it MUST leave its final answer in $R24$ and $R25$ before returning.Backend Assembly Generation:

; [Division logic executes here, using R24/R25 and R22/R23]
; ...
; Assume the result (20) is calculated and stored in R16

; ABI Compliance: Move result to the mandatory return registers
MOV R24, R16
LDI R25, 0

RET               ; Return to main()

4. Trace: Foreign Function Interface (C Library Call)
Because Spider strictly follows the AVR-GCC ABI, calling _delay_ms(250) is identical to calling a Spider function. The Spider compiler doesn't need to know how _delay_ms works; it just sets up the registers and trusts the Linker.

Backend Assembly Generation for _delay_ms(250):

; Argument 1: 'ms' = 250 (Passed in R24 and R25)
LDI R24, 250      ; Low byte
LDI R25, 0        ; High byte

; Call the external symbol
CALL _delay_ms

[Detailed Case Study: ABI and Function Call Trace](./ABI_and_Calling_Convention_Trace.md)