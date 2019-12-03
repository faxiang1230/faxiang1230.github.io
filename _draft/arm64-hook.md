# arm64 LKM hook
arm64 hook,直接ldr x3, xxx,然后blr x3
https://stackoverflow.com/questions/45279810/arm64-assembly-branch-to-function-address/45283531#45283531

echo -e -n "\xf9\x40\x18\x22" > tmp
objdump -D -b binary -m aarch64 kerne.bin

关于劫持中寄存器是否应该保存的
https://stackoverflow.com/questions/9268586/what-are-callee-and-caller-saved-registers
    Caller-saved registers (AKA volatile registers, or call-clobbered) are used to hold temporary quantities that need not be preserved across calls.

For that reason, it is the caller's responsibility to push these registers onto the stack or copy them somewhere else if it wants to restore this value after a procedure call.

It's normal to let a call destroy temporary values in these registers, though.

    Callee-saved registers (AKA non-volatile registers, or call-preserved) are used to hold long-lived values that should be preserved across calls.

When the caller makes a procedure call, it can expect that those registers will hold the same value after the callee returns, making it the responsibility of the callee to save them and restore them before returning to the caller. Or to not touch them.
翻译过来简要的是说:Caller-saved寄存器是调用者需要负责的，一般是作为传参之用的，它并不保证callee不会修改它，所以调用者用到这些寄存器的



Callee vs caller saved is a convention for who is responsible for saving and restoring the value in a register across a call. ALL registers are "global" in that any code anywhere can see (or modify) a register and those modifications will be seen by any later code anywhere. The point of register saving conventions is that code is not supposed to modify certain registers, as other code assumes that the value is not modified.

In your example code, NONE of the registers are callee save, as it makes no attempt to save or restore the register values. However, it would seem to not be an entire procedure, as it contains a branch to an undefined label (l$loop). So it might be a fragment of code from the middle of a procedure that treats some registers as callee save; you're just missing the save/restore instructions.

