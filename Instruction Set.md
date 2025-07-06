Instruction op-codes are encoded in a variable length format. If an op-code byte is between 0 and 63 inclusive, it is the final op-code byte. Otherwise (i.e. if it is between 64 and 127 inclusive) it is followed by another op-code byte. This means that instruction 0 has one op-code byte of `0b000_0000`, instruction 63 has one op-code byte of `0b011_1111`, instruction 64 has two op-code bytes of `0b100_0000`, `0b000_0000`, instruction 65 has two op-code bytes of `0b100_0000`, `0b000_0001`, and so on. Note that all these bytes are only 7 bits longs. That's because the most significant bit of each byte is used for determination of whether or not it is the first byte of the instruction, as described in README.md. In general, each byte of an instruction only contains 7 useful bits of information.

Each instruction may be followed by a specific or variable number of additional bytes. If an invalid number of bytes or excessive number of bytes is provided then an exception is thrown. If a correct number of bytes is provided but contain invalid data, an exception is thrown. If excessive bytes are provided in a valid encoding, i.e. something is specified that can be encoded with fewer bytes, an exception is thrown. The number of provided bytes is determined by how many following bytes have a most significant bit of 0. The first following byte with a most significant bit of 1 is considered the beginning of the next instruction.

0. No operation
    - This instruction may contain up to 15 additional bytes which must all be null bytes.
1. Relative label
    - This instruction may contain up to 10 additional bytes which encode a signed integer in little-endian order. If all 10 bytes are provided, the most significant 6 of the total 70 provided bits must match the sign bit of the encoded 64 bit number. If fewer than 10 bytes are provided, it is automatically sign-extended to full 64 bit length. If no additional bytes are provided, it is considered an indirect label. This instruction does not perform any operation.
2. Absolute label
    - This instruction behaves like a relative label but operates using an unsigned absolute address instead. If no additional bytes are provided it does not behave like an indirect label but instead assumes an absolute address of 0. This instruction does not perform any operation.
3. Function label
    - This instruction does not accept any additional bytes. This instruction does not perform any operation.
4. Relative jump
    - This instruction decodes its target address the same way as a relative label and jumps to it. An exception is thrown if the target address is not a suitable label. A label is suitable if it is either an indirect label or a label with a specified address equal to the location of this jump instruction. If this instruction does not have any additional bytes, the top eight bytes of the stack are popped and taken to be the absolute address to be jumped to.
5. Absolute jump
    - This instruction decodes its target address the same way as an absolute label and jumps to it. Otherwise, behavior is identical to the relative jump instruction. If this instruction does not have any additional bytes, the specified address is assumed to be 0.
6. Relative call
    - This instruction decodes its target address the same way as the relative jump instruction. If this instruction does not have any additional bytes, the top eight bytes of the stack are popped and taken to be the absolute address to be jumped to. Afterwards the current location of this instruction is pushed to the stack as an 8 byte address. Finally, the target address is jumped to. If the target address is not a function label, an exception is thrown.
7. Absolute call
    - This instruction decodes its target address the same way as the absolute jump instruction. If this instruction does not have any additional bytes, the specified address is assumed to be 0. The current location of this instruction is pushed to the stack as an 8 byte address and then the target address is jumped to. If the target address is not a function label, an exception is thrown.
8. Relative return
    - This instruction decodes its target address the same way as the relative label. It then pops the top eight bytes of the stack as an address and jumps to the instruction immediately after the location specified by that address. An exception is thrown if the specified location is not either an indirect call or call to the same location as this instruction's target address. If this instruction does not have any additional bytes, then the condition that the call it is jumping to must be to the specified target address if it is a direct call is relaxed.
9. Absolute return
    - This instruction decodes its target address the same way as the absolute label. It otherwise behaves identical to a relative return instruction, but if no additional bytes are provided then the target address is assumed to be 0 and the condition that the call it is jumping to must be to the target address if it is a direct call is not relaxed.
10. Conditional relative jump
    - This instruction behaves the same as a relative jump, but begins by unconditionally popping the first byte of the stack and reading its least significant bit. If that bit is a 1, it proceeds with the behavior of the relative jump instruction, and if it's a 0 it doesn't do anything.
11. Conditional absolute jump
    - This instruction behaves the same as an absolute jump, but uses the same conditional mechanism as the conditional relative jump.
12. Push immediate
    - This instruction may accept either 1, 2, 3, 4, 5, 6, 7, 8, or 10 additional bytes. If the number of provided bits is not a multiple of 8, it truncates the excess bits and pushes the remaining ones to the stack as a list of bytes. If only one additional byte is provided, it assumes the eighth bit to be a 0. If no additional bytes are provided, an exception is thrown.
13. Push stack 1 byte
    - This instruction accepts 0, 1, or 2 additional bytes. These bytes specify as an unsigned integer how deep into the stack to fetch one byte of data. This byte is then pushed to the top of the stack. If no bytes are provided, the offset is assumed to be 0 and the top most byte of the stack is duplicated.
14. Push stack 2 byte
    - This instruction behaves like the previous one but moves two bytes of data instead. An offset of 0 means the top two bytes of the stack are duplicated.
15. Push stack 4 byte
    - This instruction behaves like the previous one but moves four bytes of data instead. An offset of 0 means the top four bytes of the stack are duplicated.
16. Push stack 8 byte
    - This instruction behaves like the previous one but moves eight byes of data instead. An offset of 0 means the top eight bytes of the stack are duplicated.
17. Push frame 1 byte
    - This instruction behaves like "push stack 1 byte" but offsets from the frame pointer instead. The offset is interpreted as a signed integer instead of an unsigned one. An attempt to read past the top of the stack throws an exception.
18. Push frame 2 byte
    - This instruction behaves like the previous one but moves 2 bytes of data instead.
19. Push frame 4 byte
    - This instruction behaves like the previous one but moves 4 bytes of data instead.
20. Push frame 8 byte
    - This instruction behaves like the previous one but moves 8 bytes of data instead.
21. Push from heap
    - This instruction takes 0 or 1 additional bytes. It pops 8 bytes of the stack and uses it as an address into the heap. It then reads a number of bytes from that location and pushes them onto the stack. The number of bytes is equal to the value of the provided additional byte or otherwise assumed to be equal to 8. A provided value of 0 throws an exception.
22. Push frame pointer
    - This instruction accepts no additional bytes. It pushes the 8 byte frame pointer to the stack and sets the frame pointer register to the location where those 8 bytes were pushed.
23. Pop stack 1 byte
    - This instruction accepts 0, 1, or 2 additional bytes. It pops one byte from the stack and then uses the additional bytes to index how far into the stack the byte should be inserted. Insertion does not expand the stack but instead overwrites data. If the index is 0 then the top byte is simply moved into the stack by one byte.
24. Pop stack 2 byte
    - This instruction behaves the same as the last one but moves 2 bytes of data instead. If the index is 0 then the top two bytes are simply moved down the stack by two bytes.
25. Pop stack 4 byte
    - This instruction behaves the same as the last one but moves 4 bytes of data instead. If the index is 0 then the top four bytes are simply moved down the stack by four bytes.
26. Pop stack 8 byte
    - This instruction behaves the same as the last one but moves 8 bytes of data instead. If the index is 0 then the top eight bytes are simply moved down the stack by eight bytes.
27. Pop frame 1 byte
    - This instruction accepts 0, 1, or 2 additional bytes. It interprets this data as a signed offset from the frame pointer. It pops one byte from the stack and deposits it at the specified location. If this location extends beyond the end of the stack after the pop operation then an exception is thrown.
28. Pop frame 2 byte
    - This instruction behaves the same as the last one but moves 2 bytes of data instead.
29. Pop frame 4 byte
    - This instruction behaves the same as the last one but moves 4 bytes of data instead.
30. Pop frame 8 byte
    - This instruction behaves the same as the last one but moves 8 bytes of data instead.
31. Pop to heap
    - This instruction takes 0 or 1 additional bytes. It pops 8 bytes of the stack and uses it as an address into the heap. It then pops a number of bytes from the stack and writes them to that location. The number of bytes popped is equal to the value of the provided additional byte or otherwise assumed to be equal to 8.
32. Pop frame pointer
    - This instruction accepts no additional bytes. It pops 8 bytes from the stack and overwrites the frame pointer register with that value.
63. Undefined instruction
    - This instruction throws an exception immediately. It does not search for additional bytes before doing so. As such, an unlimited number of additional bytes are allowed to be provided because they will simply be ignored.

