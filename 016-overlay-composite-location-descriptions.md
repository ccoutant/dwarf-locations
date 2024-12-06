# DWARF Operation to Create Runtime Overlay Composite Location Description

## Problem Description

It is common in SIMD vectorization for the compiler to generate code that
promotes portions of an array into vector registers. For example, if the
hardware has vector registers with 8 elements, and 8 wide SIMD instructions, the
compiler may vectorize a loop so that is executes 8 iterations concurrently for
each vectorized loop iteration.

On the first iteration of the generated vectorized loop, iterations 0 to 7 of
the source language loop will be executed using SIMD instructions. Then on the
next iteration of the generated vectorized loop, iteration 8 to 15 will be
executed, and so on.

If the source language loop accesses an array element based on the loop
iteration index, the compiler may read the element into a register for the
duration of that iteration. Next iteration it will read the next element into
the register, and so on. With SIMD, this generalizes to the compiler reading
array elements 0 to 7 into a vector register on the first vectorized loop
iteration, then array elements 8 to 15 on the next iteration, and so on.

The DWARF location description for the array needs to express that all elements
are in memory, except the slice that has been promoted to the vector register.
The starting position of the slice is a runtime value based on the iteration
index modulo the vectorization size. This cannot be expressed by `DW_OP_piece`
and `DW_OP_bit_piece` which only allow constant offsets to be expressed.

Therefore, a new operator is defined that takes two location descriptions, an
offset and a size, and creates a composite that effectively uses the second
location description as an overlay of the first, positioned according to the
offset and size.

Consider an array that has been partially registerized such that the currently
processed elements are held in registers, whereas the remainder of the array
remains in memory. Consider the loop in this C function, for example:

    extern void foo(uint32_t dst[], uint32_t src[], int len) {
        for (int i = 0; i < len; ++i)
        dst[i] += src[i];
    }

Inside the loop body, the machine code loads src[i] and dst[i] into registers,
adds them, and stores the result back into dst[i].

Considering the location of dst and src in the loop body, the elements dst[i]
and src[i] would be located in registers, all other elements are located in
memory. Let register R0 contain the base address of dst, register R1 contain i,
and register R2 contain the registerized dst[i] element. We can describe the
location of dst as a memory location with a register location overlaid at a
runtime offset involving i:

    // 1. Memory location description of dst elements located in memory:
    `DW_OP_breg0` 0

    // 2. Register location description of element dst[i] is located in R2:
    `DW_OP_reg2`

    // 3. Offset of the register within the memory of dst:
    `DW_OP_breg1` 0
    `DW_OP_lit4`
    `DW_OP_mul`

    // 4. The size of the register element:
    `DW_OP_lit4`

    // 5. Make a composite location description for dst that is the memory #1
    //    with the register #2 positioned as an overlay at offset #3 of size #4:
    `DW_OP_overlay`

## Proposal

In Section 2.5.4.4.6 "Composite Location Description Operations" of [Allow
location description on the DWARF evaluation stack], add the following
operations after `DW_OP_bit_piece` and `DW_OP_piece_end`:

> 4.  `DW_OP_overlay`
>     `DW_OP_overlay` pops four stack entries. The first must be an integral
>     type value that represents the overlay byte size value S. The second
>     must be an integral type value that represents the overlay byte offset
>     value O. The third must be a location description that represents the
>     overlay location description OL. The fourth must be a location
>     description that represents the base location description BL.
>
>     The action is the same as for `DW_OP_bit_overlay`, except that the overlay
>     bit size BS and overlay bit offset BO used are S and O respectively
>     scaled by 8 (the byte size).

### FIXME: Allow concat-like behavior

> 5.  `DW_OP_bit_overlay`
>     `DW_OP_bit_overlay` pops four stack entries. The first must be an integral
>     type value that represents the overlay bit size value BS. The second
>     must be an integral type value that represents the overlay bit offset
>     value BO. The third must be a location description that represents the
>     overlay location description OL. The fourth must be a location
>     description that represents the base location description BL.
>
>     The DWARF expression is ill-formed if BS or BO are negative values.
>
>     rbss(L) is the minimum remaining bit storage size of L which is defined
>     as follows. LS is the location storage and LO is the location bit offset
>     specified by a single location description SL of L. The remaining bit
>     storage size RBSS of SL is the bit size of LS minus LO. rbss(L) is the
>     minimum RBSS of each single location description SL of L.
>
>     The DWARF expression is ill-formed if rbss(BL) is less than BO plus BS.

### FIXME: Restate so not in terms of DW_OP_bit_piece

>     If BS is 0, then the operation pushes BL.
>
>     If BO is 0 and BS equals rbss(BL), then the operation pushes OL.
>
>     Otherwise, the operation is equivalent to performing the following steps
>     to push a composite location description.
>
>     [non-normative] The composite location description is conceptually the
>     base location description BL with the overlay location description OL
>     positioned as an overlay starting at the overlay offset BO and covering
>     overlay bit size BS.
>
>     1.  If BO is not 0 then push BL followed by performing the
>         `DW_OP_bit_piece` BO, 0 operation.
>     2.  Push OL followed by performing the `DW_OP_bit_piece` BS, 0 operation.
>     3.  If rbss(BL) is greater than BO plus BS, push BL followed by
>         performing the `DW_OP_bit_piece` (rbss(BL) - BO - BS), (BO + BS)
>         operation.
>     4.  Perform the `DW_OP_piece_end` operation.

In Section 7.7.1 Operation Expressions of [Allow
location description on the DWARF evaluation stack], add the following
rows to Table 7.9 "DWARF Operation Encodings":

> Table 7.9: DWARF Operation Encodings
>
> | Operation                          | Code  | Number of Operands | Notes |
> | ---------------------------------- | ----- | ------------------ | ----- |
> | `DW_OP_overlay`                    | TBA   | 0                  |       |
> | `DW_OP_bit_overlay`                | TBA   | 0                  |       |
