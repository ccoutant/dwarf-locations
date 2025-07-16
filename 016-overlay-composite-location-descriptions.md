# DWARF Operation to Create Runtime Overlay Composite Location Description

## Problem Description

It is common in SIMD vectorization for the compiler to generate code
that promotes portions of an array into vector registers. For example,
if the hardware has vector registers with 8 elements, and 8-wide SIMD
instructions, the compiler may vectorize a loop so that it executes 8
iterations concurrently for each vectorized loop iteration.

On the first iteration of the generated vectorized loop, iterations 0 to 7 of
the source language loop will be executed using SIMD instructions. Then on the
next iteration of the generated vectorized loop, iterations 8 to 15 will be
executed, and so on.

If the source language loop accesses an array element based on the
loop iteration index, the compiler may read the element into a
register for the duration of that iteration. Then on the next
iteration it will read the next element into the register, and so
on. However with SIMD, this generalizes to the compiler reading array
elements 0 to 7 into a vector register on the first vectorized loop
iteration, then array elements 8 to 15 on the next iteration, and so
on.

The DWARF location description for the array needs to express that all elements
are in memory, except the slice that has been promoted to the vector register.
The starting position of the slice is a runtime value based on the iteration
index modulo the vectorization size. This cannot be expressed by `DW_OP_piece`
and `DW_OP_bit_piece` which only allow constant offsets to be expressed.

Therefore, a new operator is defined that takes two location
descriptions plus an offset and a size, and creates a composite that uses
the second location description as an overlay of the first, positioned
according to the offset and size.

Consider an array that has been partially registerized such that the currently
processed elements are held in registers, whereas the remainder of the array
remains in memory. Consider the loop in this C function, for example:

    extern void foo(uint32_t dst[], uint32_t src[], int len) {
        for (int i = 0; i < len; ++i)
            dst[i] += src[i];
    }

Inside the vectorized loop body, the machine code loads
src[i]..src[i+7] and dst[i]..dst[i+7] into registers, adds them, and
stores the result back into dst[i].dst[i+7].

Considering the location of dst and src in the loop body, the elements
dst[i]..dst[i+7] and src[i]..src[i+7] would be located in vector
registers, all other elements are located in memory. Let register R0
contain the base address of dst, register R1 contain i, and vector
register R2 contain the registerized dst[i]..dst[i+7] elements.

     + dst's address stored in R0
     v
     +------------------------------------------------------+
     | 0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F 10 ...|
     +------------------------------------------------------+

     R1 contains i
     R2 has copy of a portion of dst
     +------------------------+
     + 0  1  2  3  4  5  6  7 |
     +------------------------+

We can describe the location of dst as a memory location with a
register location overlaid at a runtime offset involving i:

    // 1. Memory location description of dst elements located in memory:
    `DW_OP_breg0` 0

    // 2. Register location description of element dst[i] is located in R2:
    `DW_OP_reg2`

    // 3. Offset of the register within the memory of dst:
    `DW_OP_breg1` 0
    `DW_OP_lit4`
    `DW_OP_mul`

    // 4. The size of the vector register:
    `DW_OP_lit8`

    // 5. Make a composite location description for dst that is the memory #1
    //    with the register #2 positioned as an overlay at offset #3 of size #4:
    `DW_OP_overlay`

On the first iteration of the vectorized loop, the overlay would look like:

         +------------------------+
     R2  | 0  1  2  3  4  5  6  7 |
         +-------------------------------------------------------+
    dst  | 0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F 10 ... |
         +-------------------------------------------------------+

A consumer accessing dst[8] would reference the unsigned int R0+8 but
when a consumer accesses dst[2] it would reference the byte located at
dst+2.

Then on the second iteration of the vectorized loop after i had been
incremented by 8:

                                 +------------------------+
     R2                          | 8  9  A  B  C  D  E  F |
         +-------------------------------------------------------+
    dst  | 0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F 10 ... |
         +-------------------------------------------------------+

The situation would be reversed. A consumer accessing dst[8] would
reference the first byte of R0 but when a consumer accesses dst[2], it
would reference the unsigned int located R0+2.

An overlay can also be used to create a composite location without
using `DW_OP_piece`. For example GPUs often store doubles in two
32b parts. An overlay can be used to concatenate the locations.

    DW_OP_addr 0x100
    DW_OP_addr 0x200
    DW_OP_lit4  # offset 4
    DW_OP_lit4  # size 4
    DW_OP_overlay

            +-----------------------------------------+
    overlay |                 200 201 202 203         |
    base    | 100 101 102 103                 108 ... |
            +-----------------------------------------+

A similar construct using the piece operators would be:

    DW_OP_addr 0x100
    DW_OP_piece (4)
    DW_OP_addr 0x200
    DW_OP_piece (4)

However, there is a difference. The piece operator creates a location
referencing composite storage which is 8 bytes long. It is assumed
that the value to be read are those eight bytes. With locations on the
stack, a composite piece location can be offset but offsetting into
that location is only meaningful for those 8 bytes. Offsetting beyond
those 8 bytes is an error.

On the other hand, an overlay creates a location with an intial offset
of zero which extends out to the full extent of the underlying base
storage. Thus in this example, if the base address space is 64b long,
any offset that does not overlow the generic type would be valid. In
this way, composite overlay locations are more similar to an address
where the consumer determines how many bytes to read from the
location.

If producer wants to make a location a bit more like what is created
when using piece operators and ensure that the composite storage that
a location references only contains 8 usable bytes, then two overlay
can be placed over an empty composite location.

    DW_OP_composite
    DW_OP_addr 0x100
    DW_OP_lit0  # offset 0
    DW_OP_lit4  # size 4
    DW_OP_overlay
    DW_OP_addr 0x200
    DW_OP_lit4  # offset 4
    DW_OP_lit4  # size 4
    DW_OP_overlay

                +----------------------------------+
    2nd overlay |                  200 201 202 203 |
                |+--------------------------------+|
    1st overlay || 100 101 102 103                ||
    base        || <empty composite>              ||
                |+--------------------------------+|
                +---------------------------------++

It is currently believed that in most cases the extra step of making
the underlying storage an empty composite is unnecessary.

When a portion of the location is undefined. DW_OP_piece can
concatenate an empty piece of a specific size to a previous piece to
leave a section undefined.

     DW_OP_reg0
     DW_OP_piece (4)
     DW_OP_piece (4)

To do the same thing with overlays, the undefined portion must be
explicit. Either by using it as the underlying base storage:

    DW_OP_undefined
    DW_OP_reg0
    DW_OP_lit0
    DW_OP_lit4
    DW_OP_overlay

or by making the actual bytes undefined:

    DW_OP_reg0
    DW_OP_undefined
    DW_OP_lit0
    DW_OP_lit4
    DW_OP_overlay

In the former case, any offset into that location beyond the first
four bytes up to the size of the generic type is meaningful but will
be undefined. In the latter case, the range of meaningful offsets is
limited to the maximum of the size of the underlying base storage and
overlay itself. Thus if reg0 were 4 bytes, then the meaningful offsets
of the overlay location would be 8 bytes. However, if the size of reg0
were 64 bytes, then the range of meaniful offsets would be 64 bytes.

If an overlay leaves a gap between the maximum size of the underlying
base storage and the overlay, then those bits are inferred to be
undefined and the size of the overlay grows to include the overlaid
section. For example:

    DW_OP_reg0  # assume a normal 64b general purpos register
    DW_OP_reg1  # another 64b register
    DW_OP_lit16 # well beyond the 8 bytes of reg0
    DW_OP_lit8
    DW_OP_overlay

would leave the offsets between 8 and 15 undefined.

Using an overlay this way as opposed to using the piece operators has
two big advantages.

The first being that `DW_OP_piece` has an ABI dependency:

     2.5.4.5 Composite Location Descriptions, point 2.:

        "If the piece is located in a register, but does not occupy the
        entire register, the placement of the piece within that register
        is defined by the ABI. "

While this is not often a problem with normal relatively small CPU
registers. GPUs make much more heavy use of vector registers which are
often thousands of bits long. Furthermore, to work around this ABI
dependency on CPU registers the ABI often defines different names for
sub-registers. For example on the x86, the lowest 16b of the 32b
register known as EAX can be referred to as AX and within those 16b
the lowest 8b can be referred to as AL and the next 8b are AH as if
they were different registers. This workaround is not practical when
there are already 256 vector registers which regularly have 2048b.

These large vector registers can also be spilled to fast GPU memory to
free up registers for a portion of a computation. Thus it is helpful
to have DWARF expressions which refer to these source variables in the
same way whether they are in a vector register or in memory. To allow
the DWARF expressions to be factored the DWARF expressions must be
composable in a way that expressions with the piece opperator are not.

A very common thing done in GPUs is combining two 32b slices of vector
registers to make a 64b double. This can be done with the piece
operators with something like:

    DW_OP_regx vreg0
    DW_OP_offset 8 # This could also be computed using DW_OP_push_lane
                   # but in this example, I just picked a number.
    DW_OP_piece 4
    DW_OP_regx vreg1
    DW_OP_offset 8
    DW_OP_piece 4

*Note that the small 'v' indicates where the offset into the base
  location is.*

    yields:
         +---------------------------+        +---------------------------+
         |                 v         |        |                 v         |
         | ... C  B  A  9  8  7  ... |        | ... C  B  A  9  8  7  ... |
    vreg0| ...   XX XX XX XX  ...    |   vreg1| ...   YY YY YY YY  ...    |
         +---------------------------+        +---------------------------+
                           \                         /
                            \                       /
                             v                     v
                            [XX XX XX XX YY YY YY YY]


This also works if those vector registers were spilled to memory:

    DW_OP_addr 0x100
    DW_OP_offset 8
    DW_OP_piece 4
    DW_OP_addr 0x200
    DW_OP_offset 8
    DW_OP_piece 4

    yields:
    +-----------------------------------------------------------------------------+
    |                   v                                   v                     |
    | ... 100 101 ... 108 109 110 111 112 ... 200 201 ... 208 209 210 211 212 ... |
    |                  XX  XX  XX  XX                      YY  YY  YY  YY         |
    +-----------------------------------------------------------------------------+
                                     \                    /
                                      \                  /
                                       v                v
                                    [XX XX XX XX YY YY YY YY]

This can also be generalized into a DWARF function which ignores the
location. This would allow the compiler to use the same function DWARF
function whether the variable is stored in a register or in some kind
of memory. It should be noted that on some architectures when a
shorter value is placed in a larger register its placement within the
register is defined by the architecture's ABI and may not be the same
as DW_OP_bit_piece with an offset of 0.

Note this example is not particularly useful as is. However
generalizing the function to use `DW_OP_push_lane` rather than having a
fixed offset would make a function that is generally useful and a good
candidate for factorization. However, that general function would
greatly increase the size of the examples below making them less
understandable.

    func: # expects 2 locations to be on the stack when called
      DW_OP_swap
      DW_OP_offset 8
      DW_OP_piece 4
      DW_OP_swap
      DW_OP_offset 8
      DW_OP_piece 4

Then if the variable is in registers

    DW_OO_regx vreg0
    DW_OP_regx vreg1
    DW_OP_call func

    as above yields:
         +---------------------------+        +---------------------------+
         |                 v         |        |                 v         |
         | ... C  B  A  9  8  7  ... |        | ... C  B  A  9  8  7  ... |
    vreg0| ...   XX XX XX XX  ...    |   vreg1| ...   YY YY YY YY  ...    |
         +---------------------------+        +---------------------------+
                           \                         /
                            \                       /
                             v                     v
                            [XX XX XX XX YY YY YY YY]

or if it is in memory:

    DW_OP_addr 0x100
    DW_OP_addr 0x200
    DW_OP_call func

    yields:
    +-----------------------------------------------------------------------------+
    |                   v                                   v                     |
    | ... 100 101 ... 108 109 110 111 112 ... 200 201 ... 208 209 210 211 212 ... |
    |                  XX  XX  XX  XX                      YY  YY  YY  YY         |
    +-----------------------------------------------------------------------------+
                                     \                    /
                                      \                  /
                                       v                v
                                    [XX XX XX XX YY YY YY YY]

However if you already have a composite as your first part of your
value then the DWARF function will not work. For example:

    DW_OP_addr 0x100
    DW_OP_offset 8
    DW_OP_piece 2
    DW_OP_regx AX
    DW_OP_piece 2
    DW_OP_addrx 0x200
    DW_OP_call func

    The first part creates:
    +-----------------------------+
    |                   v         |
    | ... 100 101 ... 108 109 ... |      +-------+
    |                  XX  XX     |   AX | YY YY |
    +-----------------------------+      +-------+
                            \            /
                             \          /
                              v        v
                          [ 108 109 <-AX--> ]

That is because in the function above does an 8 byte offset off of the
piece built composite above which is only 4 bytes long and doing an
offset off the end of a composite is undefined behavior.

If on the other hand the function is defined using overlays:

  func: # expects 2 locations to be on the stack when called
    DW_OP_offset 8
    DW_OP_swap
    DW_OP_offset 8
    DW_OP_lit4
    DW_OP_lit4
    DW_OP_overlay

Then all three scenarios work.

    DW_OP_regx vreg0
    DW_OP_regx vreg1
    DW_OP_call func

    yields:
         +---------------------------+
         |                 v         |
    vreg0| ... C  B  A  9  8  7  ... |
         |       XX XX XX XX         +-----------+
         +-----------+                 v         |
              \ vreg1| ... C  B  A  9  8  7  ... |
               \     |       YY YY YY YY         |
                \    +---------------------------+
                 \                |
                  v               v
                [XX XX XX XX YY YY YY YY]

    DW_OP_addr 0x100
    DW_OP_addr 0x200
    DW_OP_call func

    yields:
    +---------------------------------------------------------------------+
    |                   v                               v                 |
    | ... 100 101 ... 108 109 10A 10B ... 200 201 ... 208 209 20A 20B ... |
    |                  XX  XX  XX  XX                  YY  YY  YY  YY     |
    +---------------------------------------------------------------------+
                              |                               |
                              v                               |
    +-------------------------------------+                   |
    |                 108 109 10A 10B     |                   |
    |                  XX  XX  XX  XX     |                   |
    | 208 209 20A 20B --- --- --- --- 210 |                   |
    |  YY  YY  YY  YY                 ... |<------------------+
    +-------------------------------------+

    DW_OP_addr 0x100
    DW_OP_regx AX
    DW_OP_lit10
    DW_OP_lit2
    DW_OP_overlay # This creates the first overlay
    DW_OP_addr 0x200
    DW_OP_call func

The first overlay looks like:

    +---------------------------------+
    |                 <-AX-->         |
    | 100 ... 108 109 --- --- 10C ... |
    +---------------------------------+

Then after func is called the double word is constructed out of the bytes
as follows:

    +------------------------------------------------------------------+
    |               v                                                  |
    |             <-AX-->                            v                 |
    | 100 ... 107 --- ---  10A 10B 10C ... 200 ... 208 209 20A 20B ... |
    |              XX  XX   YY  YY                  ZZ  ZZ  ZZ  ZZ     |
    +------------------------------------------------------------------+
                      |        |                               |
                      v        v                               |
    +-------------------------------------+                    |
    |                 <-AX-->             |                    |
    |                 --- --- 10A 10B     |                    |
    |                  XX  XX  YY  YY     |                    |
    | 208 209 20A 20B --- --- --- --- 210 |                    |
    |  ZZ  ZZ  ZZ  ZZ                 ... |<-------------------+
    +-------------------------------------+

The lack of sensitivity to the type of the locations passed as a
parameter is what makes `DW_OP_overlay` composites composable in a way
that `DW_OP_piece` composites are not.

## Proposal

### Section 3.12
Rename Section 3.12 to "Composite Piece Description Operations"

### Add Section 3.13
Add a new Section 3.13 after "Composite Piece Description Operations",
called "Overlay Composites".

> *Conceptually, these new composite operators create a new composite
> location which is pushed on the stack with an offset the same as the
> base location, and composite storage that is the base location
> storage with a piece of the overlay storage spliced in, extending
> the base storage if necessary with undefined storage.*
>
> 1.  `DW_OP_overlay` `DW_OP_overlay` consumes the top four elements
>     of the stack. The top-of-stack (first) entry is a non-zero
>     unsigned integral type value that represents the width of the
>     region being overlayed on the base location in bytes. The second
>     from the top element is another unsigned integral type value
>     that represents the offset in bytes from the base location where
>     the overlay is positioned. The third element from the top is the
>     location that is being overlayed on top of the base
>     location. The fourth element from the top of the stack is the
>     base location onto which the overlay is being placed. These four
>     elements are popped and a new composite location is pushed. The
>     composite overlay location left on the stack presents the
>     underlying base location with the overlayed location of the
>     specified size positioned over the base location at the
>     specified offset. The resulting composite may be larger than the
>     original base storage and any gaps between the extent of the
>     base storage and the overlay are undefined.
>
> *For example two 32b registers can be combined into an 8-byte by
> overlaying the second one with an offset of 4. Undefined storage
> can also be implicitly inserted into a composite by simply placing
> an overlay at an offset that leaves a gap between the extent of the
> base storage and the beginning of the overlay.
>
> The action is the same for `DW_OP_bit_overlay`, except that the
> overlay size and overlay offset are specified in bits rather than
> bytes.
>
> 2.  `DW_OP_bit_overlay` `DW_OP_bit_overlay` consumes the top four
>     elements of the stack. The top-of-stack (first) entry is a
>     non-zero unsigned integral type value that represents the width
>     of the region being overlayed on the base location in bits. The
>     second from the top element is an unsigned integral type value
>     that represents the offset in bits from the base location where
>     the overlay is positioned. The third element from the top is the
>     location that is being overlayed on top of the base
>     location. The fourth element from the top of the stack is the
>     base location onto which the overlay is being placed. These four
>     elements are popped and a new composite location is pushed. The
>     composite location left on the stack presents the underlying
>     base location with the overlayed location of the specified size
>     positioned over the base location at the specified offset.The
>     resulting composite may be larger than the original base storage
>     and any gaps between the extent of the base storage and the
>     overlay are undefined.

### Section 8.7.1 "DWARF Expressions"

Add the following rows to Table 8.9 "DWARF Operation Encodings":

> Table 8.9: DWARF Operation Encodings
>
> | Operation                          | Code  | Number of Operands | Notes |
> | ---------------------------------- | ----- | ------------------ | ----- |
> | `DW_OP_overlay`                    | TBA   | 0                  |       |
> | `DW_OP_bit_overlay`                | TBA   | 0                  |       |

### Appendix D.1.3 DWARF Location Description Examples

After the piece example:

    DW_OP_reg3
    DW_OP_piece (4)
    DW_OP_reg10
    DW_OP_piece (2)

>   A variable whose first four bytes reside in register 3 and whose next two
>   bytes reside in register 10.

Add:

> a similar effect can be done with an overlay:

    DW_OP_reg3
    DW_OP_reg10
    DW_OP_lit4
    DW_OP_lit2
    DW_OP_overlay

Then add:

> There is a difference in these two examples though: The first one
> creates a six byte location. The second one creates a location that
> is either 6 bytes long if reg3 is less than or equal to 6 bytes long
> or a location as long as reg3 if it is longer than 6 bytes. In this
> case, the difference probably doesn't matter because the consumer is
> only expecting to read 6 bytes. However, if a producer wants to
> ensure that any bits in reg3 which extend beyond the 6 bytes of the
> overlay are masked off they can use an expression like this:

    DW_OP_composite
    DW_OP_reg3
    DW_OP_lit0
    DW_OP_lit4
    DW_OP_overlay
    DW_OP_reg10
    DW_OP_lit4
    DW_OP_lit2
    DW_OP_overlay

Then after the example:

    DW_OP_reg0
    DW_OP_piece (4)
    DW_OP_piece (4)
    DW_OP_fbreg (-12)
    DW_OP_piece (4)

>   A twelve byte value whose first four bytes reside in register zero, whose
>   middle four bytes are unavailable (perhaps due to optimization), and
>   whose last four bytes are in memory, 12 bytes before the frame base.

Add:

> A similar effect can be done with overlays a couple of different
> ways. The most general way places all the pieces over an empty
> composite.

    DW_OP_composite
    DW_OP_reg0
    DW_OP_lit0
    DW_OP_lit4
    DW_OP_overlay
    DW_OP_fbreg (-12)
    DW_OP_lit8
    DW_OP_lit4
    DW_OP_overlay

> However, the preferred way is more compact but it will only work if
> reg0 is at least 32b but not more than 96b or if the consumer is
> unlikely to read more than 12 bytes, is to create an overlay on top
> of reg0.

    DW_OP_reg0
    DW_OP_fbreg (-12)
    DW_OP_lit4
    DW_OP_lit8
    DW_OP_overlay

> As long reg0 is less than 32b this will leave a hole of undefined
> storage between the last bits of reg0 and the beginning of the value
> in fbreg(-12).
>
> If reg0 is more than 32b and less than 96b or if more than 12 bytes
> are likely to be read by the consumer then the undefined bits can be
> explicitly overlayed onto the reg0's upper bytes. This undefined
> storage will extend beyond the value in fbreg(-12).

    DW_OP_reg0
    DW_OP_undefined
    DW_OP_lit4
    DW_OP_lit4
    DW_OP_overlay
    DW_OP_fbreg (-12)
    DW_OP_lit8
    DW_OP_lit4
    DW_OP_overlay

After the example below:

    DW_OP_lit1
    DW_OP_stack_value
    DW_OP_piece (4)
    DW_OP_breg3 (0)
    DW_OP_breg4 (0)
    DW_OP_plus
    DW_OP_stack_value
    DW_OP_piece (4)

> The object value is found in an anonymous (virtual) location whose value
> consists of two parts, given in memory address order: the 4 byte value 1
> followed by the four byte value computed from the sum of the contents of
> r3 and r4

Add: 

> The equivilent expression using overlays would be:

    DW_OP_lit1
    DW_OP_stack_value # while 1 can fit into a single byte stack_value
                      # promotes it to a generic type
    DW_OP_breg3 (0)
    DW_OP_breg4 (0)
    DW_OP_plus
    DW_OP_stack_value
    DW_OP_lit4
    DW_OP_lit4
    DW_OP_overlay

After:

    DW_OP_reg0
    DW_OP_bit_piece (1, 31)
    DW_OP_bit_piece (7, 0)
    DW_OP_reg1
    DW_OP_piece (1)

>   A variable whose first bit resides in the 31st bit of register 0,
>   whose next seven bits are undefined and whose second byte resides
>   in register 1.

Add:

> A similar expression done with bit overlays:

    DW_OP_composite
    DW_OP_reg0
    DW_OP_lit31
    DW_OP_bit_offset
    DW_OP_lit0
    DW_OP_lit1
    DW_OP_bit_overlay
    DW_OP_reg1
    DW_OP_lit8
    DW_OP_lit8
    DW_OP_bit_overlay

> While this looks considerably longer, the piece operators take
> inline operands while the overlay operators have stack operands. The
> extra stack operands take more operations, but the same number of
> bytes in most cases.

*FIXME: I think we should add an example using predicate
registers that are spilled to memory while a pair of nested
conditional loops run on a vector register. Spilling predicate
registers is something realistic that happens. Because it is a spilled
register, the endianness matters and the problems with bit piece
become obvious.

### Section D.13 Figure D.66 C implicit pointer example #1

Replace:

    21$:   DW_TAG_variable
           DW_AT_name("s")
           DW_AT_type(reference to S at 1$)
           DW_AT_location(expression=
                DW_OP_breg5(1) DW_OP_stack_value DW_OP_piece(2)
                DW_OP_breg5(2) DW_OP_stack_value DW_OP_piece(1)
                DW_OP_breg5(3) DW_OP_stack_value DW_OP_piece(1))

With:

    21$:   DW_TAG_variable
           DW_AT_name("s")
           DW_AT_type(reference to S at 1$)
           DW_AT_location(expression=
                DW_OP_breg5 (1) DW_OP_stack_value
		DW_OP_breg5 (2) DW_OP_stack_value
                DW_OP_lit2 DW_OP_lit1 DW_OP_overlay
                DW_OP_breg5 (3) DW_OP_lit3 DW_OP_lit1 DW_OP_overlay)

*FIXME: Add an example where a structure stays in memory while one of
its members is promoted to a register -- where we can use the memory
location as the base.

### Section D.13 Figure D.68 C implicit pointer example #2

Replace:

    98$: DW_LLE_start_end[<label0 in main> .. <label1 in main>)
            DW_OP_lit1 DW_OP_stack_value DW_OP_piece(4)
            DW_OP_lit2 DW_OP_stack_value DW_OP_piece(4)
        DW_LLE_start_end[<label1 in main> .. <label2 in main>)
            DW_OP_lit2 DW_OP_stack_value DW_OP_piece(4)
            DW_OP_lit2 DW_OP_stack_value DW_OP_piece(4)
        DW_LLE_start_end[<label2 in main> .. <label3 in main>)
            DW_OP_lit2 DW_OP_stack_value DW_OP_piece(4)
            DW_OP_lit3 DW_OP_stack_value DW_OP_piece(4)
        DW_LLE_end_of_list

With:

    98$: DW_LLE_start_end[<label0 in main> .. <label1 in main>)
            DW_OP_lit1 DW_OP_stack_value DW_OP_lit2 DW_OP_stack_value
            DW_OP_lit4 DW_OP_lit4 DW_OP_overlay
        DW_LLE_start_end[<label1 in main> .. <label2 in main>)
            DW_OP_lit2 DW_OP_stack_value DW_OP_lit2 DW_OP_stack_value
            DW_OP_lit4 DW_OP_lit4 DW_OP_overlay
        DW_LLE_start_end[<label2 in main> .. <label3 in main>)
            DW_OP_lit2 DW_OP_stack_value DW_OP_lit3 DW_OP_stack_value
            DW_OP_lit4 DW_OP_lit4 DW_OP_overlay
        DW_LLE_end_of_list

### Section D.18 SIMD Lane Example

Replace figure "D.8? SIMD Lane Example: DWARF Encoding" with:

    DW_TAG_subprogram
    DW_AT_name ("vec_add")
    DW_AT_num_lanes .vallist.0
    DW_TAG_formal_parameter
        DW_AT_name ("dst")
        DW_AT_type (reference to type: pointer to int)
        DW_AT_location .loclist.1
    DW_TAG_formal_parameter
        DW_AT_name ("src")
        DW_AT_type (reference to type: pointer to int)
        DW_AT_location .loclist.2
    DW_TAG_formal_parameter
        DW_AT_name ("len")
        DW_AT_type (reference to type int)
        DW_AT_location DW_OP_reg2
    ...
    DW_TAG_variable
        DW_AT_name ("i")
        DW_AT_type (reference to type int)
        DW_AT_location .loclist.3
    ...
    $DP1: DW_TAG_dwarf_procedure # Used by src in loclist.1
              DW_AT_location [
                DW_OP_breg1 (0)   # base location of the array
                DW_OP_regx v1     # location of the overlay
                DW_OP_breg3 (0)
                DW_OP_lit4
                DW_OP_mul         # the offset of the overlay in bytes
                DW_OP_constu (32) # sizeof(int)*num_lanes
                DW_OP_overlay
                ]
    $DP2:  DW_TAG_dwarf_procedure # Used by dst in loclist.2
              DW_AT_location [
                DW_OP_breg0 (0)
                DW_OP_regx v0
                DW_OP_breg3 (0)
                DW_OP_lit4
                DW_OP_mul
                DW_OP_constu (32)
                DW_OP_overlay
                ]
    .vallist.0:
        range [.l1, .l2)
            DW_OP_lit8
        end-of-list

    .loclist.1:          # src
        range [.l0, .l1)
	    DW_OP_reg1
        range [.l1, .l2)
            DW_OP_implicit_pointer ($DP1, 0)
        range [.l2, .l4)
            DW_OP_reg1
    .loclist.2:          # dst
        range [.l0, .l1)
            DW_OP_reg0
        range [.l1, .l2)
            DW_OP_implicit_pointer ($DP2, 0)
        range [.l2, .l4)
            DW_OP_reg0
    .loclist.3:          # i
        range [.l0, .l1)
            DW_OP_reg3
        range [.l1, .l2)
            DW_OP_breg3 (0)
            DW_OP_push_lane
            DW_OP_plus
            DW_OP_stack_value
        range [.l2, .l4)
            DW_OP_reg3
        end-of-list
