# Interrupt 3

Whenever a software interrupt exception is triggered, the exception address and the value of the EIP register both point to the next instruction after the one that caused the exception. However, the breakpoint exception is a special case.

When an `EXCEPTION_BREAKPOINT(0x80000003)` exception is triggered, Windows assumes it was caused by a single-byte "`CC`" opcode (i.e., the `Int 3` instruction). Windows decrements the exception address to point to the presumed "`CC`" opcode and then passes the exception to the exception handler. However, the value of the EIP register does not change.

Therefore, if `CD 03` (which is the machine code representation of `Int 03`) is used, when the exception handler receives control, the exception address will point to the position of `03`.
