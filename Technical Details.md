This architecture is currently only defined for user mode. Kernel mode is assumed to exist but not yet defined because I know far too little about how kernel mode works in popular CPU architectures.

The architecture has no general purpose registers, but it does have some special purpose registers:
- Instruction pointer
  - Points to the location in the code segment where the code is currently executing from
- Stack pointer
  - Points to the top of the stack in the stack segment
- Frame pointer
  - Points to the bottom of the current stack frame in the stack segment

