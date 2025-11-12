# DWARF Operation to Create Runtime Overlay Composite Location Description

## Introduction

Composite storage and pieces are powerful DWARF abstractions that
describe objects whose parts may reside in different locations.

While the terms "composite" and "piece" are often associated with the
piece operators, it is important to distinguish between the concepts
themselves and the operators used to construct them.

Currently, you can create composite storage with the
`DW_OP_composite`, `DW_OP_piece`, and `DW_OP_bit_piece` operators.
The `DW_OP_bit_piece` operator in particular has the problem that its
offset operand depends on the type of location. This behavior
conflicts with the unified location storage abstraction provided by
the [locations on the
stack](https://dwarfstd.org/issues/230524.1.html) proposal. It should
be noted that this is a problem with the operator itself, not with
composite storage or the concept of pieces.

Another limitation of the piece operators is that their operands are
inline operands, and thus cannot be computed at runtime. This was
previously discussed in issue
[211206.2](https://dwarfstd.org/issues/211206.2.html) (stack piece
operators), and is also discussed further down this document. The
solution proposed in 211206.2 predates the acceptance of the locations
on the stack proposal, back when piece operators were still syntactic
separators that operated on their own expression stack, forcing
211206.2 to rely on DWARF expressions as inline bytecode operands.
After locations on the stack, it is now possible to design replacement
operators that simply pop arguments from the current, shared stack.

Yet another limitation is one of natural composition. With the piece
operators, it is not possible to start from a pre-computed, shared
composite and replace some part of it. For example when a field of
a structure or an array element is promoted to a register for a
specific PC range. The producer must instead rebuild a new composite
piece by piece, which results in DWARF expressions that are not as
compact as they could be.

It is important to emphasize that these are not limitations of the
model of composite storage itself but of the existing operators used
to construct composite locations.

To address these limitations, this proposal defines a new operator,
overlay (with variants `DW_OP_overlay` and `DW_OP_bit_overlay`). It
produces composite locations and is designed to replace usages of the
aforementioned piece operators without the limitations while allowing
objects to be described in a more natural and composable way. The new
operator semantics align naturally with the post-"locations on the
stack" model. However, the resulting composite locations are just
normal composite locations that are completely indistinguishable from
ones created by the preexisting operators.

In its basic form, the overlay operator produces a composite location
by conceptually taking a base location and overlaying a new location
on top of a range of storage from that base location, overriding the
original location with the new one for that range of storage.

As a simple example, consider the location of a variable of a struct
type with three 32-bit fields, whose default location is on the stack
at offset 0x40 relative to the frame base FB:

              +-----------------------------------------+
    memory:   |        |   a   |   b   |   c   |        |
              +-----------------------------------------+
               0       |FB + 0x40                       |end-of-memory

    DW_AT_location: [ DW_OP_fbreg(0x40) ]

If the compiler chooses to promote the field b to reg1 for some
section of code, we would describe this as an overlay on top of the
memory location, of 4 bytes starting at offset 4 and a new location of
reg1:

                               +-------+
    reg1:                      |   b   |
              +-----------------------------------------+
    memory:   |        |   a   |///////|   c   |        |
              +-----------------------------------------+
               +0      |+FB + 0x40                      |end-of-memory

Resulting in the following composite with three pieces:

              +-----------------------------------------+
              | memory ...     | reg1  | memory ...     |
              +-----------------------------------------+
               +0      |O                               |end-of-composite

Where O is the resulting composite's offset, which is the same offset
as in the base memory location, i.e., +FB + 0x40, and points at the
beginning of the object.

In addition to simple overlaying as above, the overlay operators can
also be used for concatenation, by allowing a piece to be overlaid to
the right of the base location, beyond base's end. This gives it the
power to replace `DW_OP_piece` and `DW_OP_bit_piece`.

For example, consider the location of a variable of a 64-bit integer
type, that has been split into a 32-bit register pair. We would
describe this as the second 32-bit register overlaying on the right of
the first register:

             +--------+
             |   h2   |     (reg2)
    +-----------------+
    |   h1   |              (reg1)
    +--------+
     +0       +4       +8

This results in the following composite:

    +--------+--------+
    |  reg1  |  reg2  |
    +--------+--------+
     +0       +4       +8

As a further extension to concatenation, the operator allows
overlaying beyond the extent of the base storage. When an overlay is
placed beyond the extent of the base storage, the gap is "filled" with
undefined storage, much like `DW_OP_piece` does when the piece is
empty. For example, consider the location of a variable of the same
struct type with three 32-bit fields, mentioned earlier. If the
compiler chooses to promote fields a and c to registers, and optimize
out field b, we may describe this with concatenation with a gap, like
so:

                      +--------+
                      |   c    |     (reg2)
    +--------+        +--------+
    |   a    |                       (reg1)
    +--------+
     +0       +4       +8

This results in the following composite:

    +--------+--------+--------+
    |  reg1  | undef  |  reg2  |
    +--------+--------+--------+
     +0       +4       +8

The following section describes in detail the use cases that motivated
the overlay operator and illustrates additional ways it can be
applied.

## Problem Description

It is common in SIMD vectorization for the compiler to generate code
that promotes portions of an array into vector registers. For example,
if the hardware has vector registers with 8 elements, and 8-wide SIMD
instructions, the compiler may vectorize a loop so that it executes 8
iterations concurrently for each vectorized loop iteration.

On the first iteration of the generated vectorized loop, iterations 0
to 7 of the source language loop will be executed using SIMD
instructions. Then on the next iteration of the generated vectorized
loop, iterations 8 to 15 will be executed, and so on.

If the source language loop accesses an array element based on the
loop iteration index, the compiler may read the element into a
register for the duration of that iteration. Then on the next
iteration it will read the next element into the register, and so
on. However with SIMD, this generalizes to the compiler reading array
elements 0 to 7 into a vector register on the first vectorized loop
iteration, then array elements 8 to 15 on the next iteration, and so
on.

The DWARF location description for the array needs to express that all
elements are in memory, except the slice that has been promoted to the
vector register. The starting position of the slice is a runtime
value based on the iteration index modulo the vectorization size. This
cannot be expressed by `DW_OP_piece` and `DW_OP_bit_piece` which only
allow constant offsets to be expressed.

Therefore, a new operator is defined that takes two location
descriptions plus an offset and a size, and creates a composite that uses
the second location description as an overlay of the first, positioned
according to the offset and size.

Consider an array that has been partially registerized such that the
currently processed elements are held in registers, whereas the
remainder of the array remains in memory. Consider the loop in this C
function, for example:

    extern void foo(uint32_t dst[], uint32_t src[], int len) {
        for (int i = 0; i < len; ++i)
            dst[i] += src[i];
    }

Inside the vectorized loop body, the machine code loads
src[i]..src[i+7] and dst[i]..dst[i+7] into registers, adds them, and
stores the result back into dst[i]..dst[i+7].

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
    `DW_OP_const1u 32`

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
when the consumer accesses dst[2], it would reference the byte located at
offset 8 of register R2.

Then on the second iteration of the vectorized loop after i had been
incremented by 8:

                                 +------------------------+
     R2                          | 8  9  A  B  C  D  E  F |
         +-------------------------------------------------------+
    dst  | 0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F 10 ... |
         +-------------------------------------------------------+

The situation would be reversed. The consumer accessing dst[8] would now
reference the first byte of R2, but when accessing dst[2], it
would reference the unsigned int located at R0+2.

An overlay can also be used to create a composite location without
using `DW_OP_piece`. For example GPUs often store doubles in two
32b parts. An overlay can be used to combine the locations.
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%29&input=DW_OP_addr+0x100%0ADW_OP_addr+0x200%0ADW_OP_lit4+%3B+offset%0ADW_OP_lit4+%3B+width%0ADW_OP_overlay%0A))

    DW_OP_addr 0x100
    DW_OP_addr 0x200
    DW_OP_lit4  # offset 4
    DW_OP_lit4  # size 4
    DW_OP_overlay

            +-----------------------------------------+
    overlay |                 200 201 202 203         |
    base    | 100 101 102 103                 108 ... |
            +-----------------------------------------+

A similar construct using the piece operators would be as follows:
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%29&input=DW_OP_addr+0x100%0ADW_OP_piece+4%0ADW_OP_addr+0x200%0ADW_OP_piece+4%0A))

    DW_OP_addr 0x100
    DW_OP_piece (4)
    DW_OP_addr 0x200
    DW_OP_piece (4)

However, there is a difference. The piece operator creates a location
referencing composite storage which is 8 bytes long. It is assumed
that the value to be read are those eight bytes. With locations on the
stack, a composite piece location can be offset, but offsetting into
that location is only meaningful for those 8 bytes. Offsetting beyond
those 8 bytes is an error.

On the other hand, an overlay creates a location that extends out to
the full extent of the underlying base storage, both to the left and
to the right. Thus in this example, if the base address space is 64b
long, any offset that does not overlow the generic type would be
valid. In this way, composite overlay locations are more similar to an
address where the consumer determines how many bytes to read from the
location.

If the producer wants to make a location a bit more like what is created
when using piece operators and ensure that the composite storage that
a location references only contains 8 usable bytes, then two overlays
can be placed over an empty composite location.
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%29&input=DW_OP_composite%0ADW_OP_addr+0x100%0ADW_OP_lit0++%3B+offset+0%0ADW_OP_lit4++%3B+width+4%0ADW_OP_overlay%0ADW_OP_addr+0x200%0ADW_OP_lit4++%3B+offset+4%0ADW_OP_lit4++%3B+width+4%0ADW_OP_overlay%0A))

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

When a portion of the location is undefined, `DW_OP_piece` can
concatenate an empty piece of a specific size to a previous piece to
leave a section undefined.
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%29&input=DW_OP_reg0%0ADW_OP_piece+4%0ADW_OP_piece+4%0A))

     DW_OP_reg0
     DW_OP_piece (4)
     DW_OP_piece (4)

To do the same thing with overlays, the undefined portion must be
explicit. Either by using it as the underlying base storage:
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%28TargetReg+0+%2212345678%22%29%0A%29&input=DW_OP_undefined%0ADW_OP_reg0%0ADW_OP_lit0%0ADW_OP_lit4%0ADW_OP_overlay%0A))

    DW_OP_undefined
    DW_OP_reg0
    DW_OP_lit0
    DW_OP_lit4
    DW_OP_overlay

or by making the actual bytes undefined:
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%28TargetReg+0+%2212345678%22%29%0A%29&input=DW_OP_reg0%0ADW_OP_undefined%0ADW_OP_lit4%0ADW_OP_lit4%0ADW_OP_overlay%0A))

    DW_OP_reg0
    DW_OP_undefined
    DW_OP_lit4
    DW_OP_lit4
    DW_OP_overlay

In the former case, any offset into that location beyond the first
four bytes up to the size of undefined storage (i.e., the largest
address space or register) is meaningful but will be undefined. In the
latter case, the range of meaningful offsets is limited to the maximum
of the size of the underlying base storage and overlay itself. Thus if
reg0 were 4 bytes, then the meaningful offsets of the composite location
would be 8 bytes, where bytes 4 up to and including 7 are undefined.
However, if the size of reg0 were 64 bytes, then the range of meaningful
offsets would be 64 bytes, where bytes 4 up to and including 7 are still
undefined.

If an overlay leaves a gap between the maximum size of the underlying
base storage and the overlay, then those bits are inferred to be
undefined and the size of the overlay grows to include the overlaid
section. For example:

    DW_OP_reg0  # assume a 64b general purpose register
    DW_OP_reg1  # another 64b register
    DW_OP_lit16 # well beyond the 8 bytes of reg0
    DW_OP_lit8
    DW_OP_overlay

would leave the offsets between 8 and 15 undefined.
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%28TargetReg+0+%2212345678%22%29%0A%28TargetReg+1+%22abcdefgh%22%29%0A%29&input=DW_OP_reg0++%3B+assume+a+64b+general+purpose+register%0ADW_OP_reg1++%3B+another+64b+register%0ADW_OP_lit16+%3B+well+beyond+the+8+bytes+of+reg0%0ADW_OP_lit8%0ADW_OP_overlay%0A))

Using an overlay this way as opposed to using the piece operators has
two big advantages.

The first being that `DW_OP_piece` has an ABI dependency:

     2.5.4.5 Composite Location Descriptions, point 2.:

        "If the piece is located in a register, but does not occupy the
        entire register, the placement of the piece within that register
        is defined by the ABI. "

While this is not often a problem with typical, relatively small CPU
registers, GPUs make much more heavy use of vector registers, which
may be thousands of bits long. Furthermore, to work around this ABI
dependency on CPU registers, the ABI often defines different names for
sub-registers. For example on the x86, the lowest 16b of the 32b
register known as EAX can be referred to as AX and within those 16b
the lowest 8b can be referred to as AL and the next 8b are AH as if
they were different registers. This workaround is not practical when
there are already 256 vector registers that have 2048 bits.

These large vector registers can also be spilled to fast GPU memory to
free up registers for a portion of a computation. Thus it is helpful
to have DWARF expressions which refer to these source variables in the
same way whether they are in a vector register or in memory. Other
approaches to solving this ABI problem have been suggested including
[Stack piece operators](https://dwarfstd.org/issues/211206.2.html)

A very common thing done in GPUs is combining two 32b slices of
different vector registers to make a 64b double. This can be done
trivially with overlay:

     DW_OP_regx vreg1
     DW_OP_push_lane
	 DW_OP_lit4
	 DW_OP_mul
     DW_OP_offset
     DW_OP_regx vreg0
     DW_OP_push_lane
	 DW_OP_lit4
	 DW_OP_mul
     DW_OP_offset

This could very easily be factored with boiler plate DWARF functions
which are called from many expressions:

     lane_slice:       ; expects a location on the stack
	   DW_OP_push_lane
	   DW_OP_lit4
	   DW_OP_mul
       DW_OP_offset
     merge_split_double: ; expects two locations on the stack
	   DW_OP_call lane_slice
	   DW_OP_swap
	   DW_OP_call lane_slice
	   DW_OP_swap


Then the expression could be reduced to a a very compact:

    DW_OP_regx vreg1
    DW_OP_regx vreg0
    DW_OP_call merge_split_double

yields:
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%28TargetReg+0++%220000%22%29%0A%28TargetReg+90+%2212345678123456781234567812345678%22%29%0A%28TargetReg+91+%22abcdefghabcdefghabcdefghabcdefgh%22%29%0A%29%0A&input=%28%0A+%28DW_OP_regx+91%29%0A+%28DW_OP_regx+90%29%0A+%28DW_OP_call+%28%0A+++%3B+func%3A%0A++++DW_OP_lit8%0A++++DW_OP_offset%0A++++DW_OP_swap%0A++++DW_OP_lit8%0A++++DW_OP_offset%0A++++DW_OP_lit4%0A++++DW_OP_lit4%0A++++DW_OP_overlay%0A++%29%0A+%29%0A%29%0A))

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

These functions also work if one or both of those vector registers has
been spilled to memory:

    DW_OP_addr 0x200
    DW_OP_addr 0x100
    DW_OP_call merge_split_double

yields:
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%28TargetReg+0++%220000%22%29%0A%28TargetReg+90+%2212345678123456781234567812345678%22%29%0A%28TargetReg+91+%22abcdefghabcdefghabcdefghabcdefgh%22%29%0A%29%0A&input=%28%0A+%28DW_OP_addr+0x200%29%0A+%28DW_OP_addr+0x100%29%0A+%28DW_OP_call+%28%0A+++%3B+func%3A%0A++++DW_OP_lit8%0A++++DW_OP_offset%0A++++DW_OP_swap%0A++++DW_OP_lit8%0A++++DW_OP_offset%0A++++DW_OP_lit4%0A++++DW_OP_lit4%0A++++DW_OP_overlay%0A++%29%0A+%29%0A%29%0A))

    +---------------------------------------------------------------------+
    |                   v                               v                 |
    | ... 100 101 ... 108 109 10A 10B ... 200 201 ... 208 209 20A 20B ... |
    |                  XX  XX  XX  XX                  YY  YY  YY  YY     |
    +---------------------------------------------------------------------+
                              |                               |
                              v                               |
    +-----------------------------------------------------+   |
    |                 108 109 10A 10B                     |   |
    |            ...   XX  XX  XX  XX                     |   |
    |                                 208 209 20A 20B ... |   |
    |                                  YY  YY  YY  YY     |<--+
    +-----------------------------------------------------+

It even works if one or both of the locations is already composite location.

    DW_OP_addr 0x200
    DW_OP_addr 0x100
    DW_OP_regx AX
    DW_OP_lit10
    DW_OP_lit2
    DW_OP_overlay # This creates the first overlay
    DW_OP_call merge_split_double

the first overlay looks like:
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%28TargetReg+0++%220000%22%29%0A%28TargetReg+90+%2212345678123456781234567812345678%22%29%0A%28TargetReg+91+%22abcdefghabcdefghabcdefghabcdefgh%22%29%0A%29%0A&input=%28%0A+%28DW_OP_addr+0x200%29%0A+%28DW_OP_addr+0x100%29%0A+DW_OP_reg0+%3B+reg+AX%0A+DW_OP_lit10%0A+DW_OP_lit2%0A+DW_OP_overlay+%3B+This+creates+the+first+overlay%0A%29%0A))

    +---------------------------------+
    |  v              <-AX-->         |
    | 100 ... 108 109 --- --- 10C ... |
    +---------------------------------+

Then after merge_split_double is called the double word is constructed
out of the bytes as follows:
([Try it!](https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%28TargetReg+0++%220000%22%29%0A%28TargetReg+90+%2212345678123456781234567812345678%22%29%0A%28TargetReg+91+%22abcdefghabcdefghabcdefghabcdefgh%22%29%0A%29%0A&input=%28%0A+%28DW_OP_addr+0x200%29%0A+%28DW_OP_addr+0x100%29%0A+DW_OP_reg0+%3B+reg+AX%0A+DW_OP_lit10%0A+DW_OP_lit2%0A+DW_OP_overlay+%3B+This+creates+the+first+overlay%0A%0A+%28DW_OP_call+%28%0A+++%3B+func%3A%0A++++DW_OP_lit8%0A++++DW_OP_offset%0A++++DW_OP_swap%0A++++DW_OP_lit8%0A++++DW_OP_offset%0A++++DW_OP_lit4%0A++++DW_OP_lit4%0A++++DW_OP_overlay%0A++%29%0A+%29%0A%29%0A))

    +------------------------------------------------------------------+
    |              v                                                   |
    |                                              v                   |
    | 100 ... 107 108 109 <-AX--> 10C ... 200 ... 208 209 20A 20B ...  |
    |              XX  XX  YY  YY                  ZZ  ZZ  ZZ  ZZ      |
    +------------------------------------------------------------------+
                      |        |                           |
                      v        v                           |
    +--------------------------------------------------+   |
    |              v                                   |   |
    |             108 109 <-AX-->                      |   |
    |        ...   XX  XX  YY  YY                      |   |
    |                             208 209 20A 20B ...  |   |
    |                              ZZ  ZZ  ZZ  ZZ      |<--+
    +--------------------------------------------------+

The independence to the underlying storage is a very powerful property
of overlays. This composability allows DWARF expressions to be factored
making them much more compact.

## Proposal

### Section 3.12
In Section 3.12 Composite Locations, keep the following introductory paragraphs:

> A composite location represents the location of an object or value
> that does not exist in a single block of contiguous storage (e.g.,
> as the result of compiler optimization where part of the object is
> promoted to a register). It consists of a (possibly empty) sequence
> of pieces, where each piece maps a fixed range of bits from the
> object onto a corresponding range of bits at a new (sub-)location.
>
> The pieces of a composite location are contiguous, so that the size
> of the composite storage is the sum of the sizes of the individual
> pieces, and each piece covers a range of bits immediately following
> the previous piece. The maximum size of a block of composite storage
> is the size of the largest address space or register.

> Typically, the size of a composite storage is the same as that of
> the object it describes. If the composite storage is smaller than the
> object, the remaining bits of the object are treated as undefined.
>
> In the process of fetching a value from a composite location, the consumer
> may need to fetch and assemble bits from more than one piece.

Add the following new paragraph:

> Composite locations can be formed by two methods: piece operations, which
> operate as in previous versions of DWARF, and overlay operations.

Move the rest of the existing section into subsection 3.12.1 Piece Operations:

> 3.12.1 Piece Operations
> The following operation is used to form a new, empty, composite location:
>
>  ...


### Add Section 3.12.2
Add a new Section 3.12.2 after "Composite Piece Description Operations",
called "Overlay Composites".

> *Conceptually, these new composite operators create a new composite
> location which is pushed on the stack with an offset the same as the
> base location, and composite storage that is the base location
> storage with a piece of the overlay storage spliced in, extending
> the base storage if necessary with undefined storage.*
>
> `DW_OP_overlay`
>
>    <[location] base location> <[location] overlay location> <[integral] base offset> <[unsigned] overlay width> → <[location] composite location>
>
>   `DW_OP_overlay` pushes a new composite location whose storage is the
>   result of replacing a slice in the storage of `base location` with
>   a slice from the storage of `overlay location`.
>
>   The slice of bytes obtained from the storage of `overlay location`
>   is referred to as the `overlay`. The `overlay` begins at `overlay
>   location` and has a size of `overlay width`. The `overlay`
>   must not extend outside of the bounds of the storage of `overlay
>   location`.
>

>   The slice of bytes replaced in the storage of `base location` is
>   referred to as the `overlay base`. It begins at `base location`
>   offset by `base offset` and has a size of `overlay width`. The
>   `overlay width` cannot extend beyond the end of the overlay's
>   storage or else the expression is invalid.
>
>   *If the `overlay width` is zero and offset is within the bounds of
>   the base location's storage, then the consumer may leave the `base
>   location` on the top of the stack rather than creating composite
>   storage.*
>
>   If the `overlay base` extends beyond the bounds of the storage of
>   `base location`, the storage of the resulting location is first
>   extended by adding undefined storage sufficient to cover the
>   `overlay base`.
>
>   The offset of the resulting location is the offset of `base
>   location`.
>
> The action is the same for `DW_OP_bit_overlay`, except that the
> overlay size and overlay offset are specified in bits rather than
> bytes.
>
> 2. `DW_OP_bit_overlay`
>
>    <[location] base location> <[location] overlay location> <[integral] base offset in bits> <[unsigned] overlay width in bits> → <[location] composite location>
>
>   `DW_OP_bit_overlay` pushes a new composite location whose storage
>   is the result of replacing a slice in the storage of `base
>   location` with a slice from the storage of `overlay location`.
>
>   The slice of bits obtained from the storage of `overlay location`
>   is referred to as the `overlay`. The `overlay` begins at `overlay
>   location` and has a size in bits of `overlay width`. The
>   `overlay` must not extend outside of the bounds of the storage of
>   `overlay location`.
>
>   The slice of bits replaced in the storage of `base location` is
>   referred to as the `overlay base`. It begins at `base location`
>   offset by `base offset` and has a size of `overlay width` in
>   bits. The `overlay width` cannot extend beyond the end of the
>   overlay's storage or else the expression is invalid.
>
>   *If the `overlay width` is zero and offset is within the bounds of
>   the base location's storage, then the consumer may leave the `base
>   location` on the top of the stack rather than creating composite
>   storage.*
>
>   If the `overlay base` extends beyond the bounds of the storage of
>   `base location`, the storage of the resulting location is first
>   extended by adding undefined storage sufficient to cover the
>   `overlay base`.
>
>   The offset of the resulting location is the offset of `base
>   location`.

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

    DW_OP_composite
    DW_OP_reg3
    DW_OP_piece (4)
    DW_OP_reg10
    DW_OP_piece (2)

Replace:

>   A variable whose first four bytes reside in register 3 and whose next two
>   bytes reside in register 10.

With:

>   A variable whose first four bytes reside in register 3 and whose
>   next two bytes reside in register 10. For example as if a six byte
>   structure were passed by value using two 32 bit registers. An
>   overlay can do the same thing. On a little endian machine the
>   expression would be:

    DW_OP_reg3
    DW_OP_reg10
    DW_OP_lit4
    DW_OP_lit2
    DW_OP_overlay

>   and on a big endian machine the expression would likely be:

    DW_OP_reg3
    DW_OP_reg10
	DW_OP_lit2
	DW_OP_offset
    DW_OP_lit4
    DW_OP_lit2
    DW_OP_overlay

Then after the example:

    DW_OP_composite
    DW_OP_reg0
    DW_OP_piece (4)
    DW_OP_piece (4)
    DW_OP_fbreg (-12)
    DW_OP_piece (4)

Replace:

>   A twelve byte value whose first four bytes reside in register zero, whose
>   middle four bytes are unavailable (perhaps due to optimization), and
>   whose last four bytes are in memory, 12 bytes before the frame base.

With:

>   A twelve byte value whose first four bytes reside in register
>   zero, whose middle four bytes are unavailable, and whose last four
>   bytes are in memory, 12 bytes before the frame base.  This
>   situation most likely would occur in the case where the object
>   lives at FB-12 but the first four bytes got promoted and the next
>   four bytes got optimized away giving us the expression:

    DW_OP_fbreg (-12) ; the default location
    DW_OP_reg0        ; the promoted 32b variable
    DW_OP_lit0
    DW_OP_lit4
    DW_OP_overlay
    DW_OP_undefined   ; the variable that was optimized away
    DW_OP_lit4
    DW_OP_lit4
    DW_OP_overlay

>   This location can be incrementally built up as optimization passes
>   act on the variable promoting some portions and optimizing out
>   others.

>   Assuming that reg0 is a 32b register, and the end result is known
>   a more compact way to specify the location is to create an overlay
>   on top of reg0. Because any gap between the end of the base
>   storage and the overlay is undefined, thhis leaves a hole of
>   undefined storage between the last bit of reg0 and the beginning
>   of the value in fbreg(-12).

    DW_OP_reg0
    DW_OP_fbreg (-12)
    DW_OP_lit8
    DW_OP_lit4
    DW_OP_overlay

After the example below:

    DW_OP_composite
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

> The equivalent expression using overlays would be:

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

    DW_OP_composite
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
                DW_OP_breg5(1) DW_OP_stack_value
                DW_OP_breg5(2) DW_OP_stack_value
                DW_OP_lit2 DW_OP_lit1 DW_OP_overlay
                DW_OP_breg5(3) DW_OP_lit3 DW_OP_lit1 DW_OP_overlay)

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
