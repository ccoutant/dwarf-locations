# DWARF Operation to Create Runtime Overlay Composite Location Description

## Problem Description

It is common in SIMD vectorization for the compiler to generate code that
promotes portions of an array into vector registers. For example, if the
hardware has vector registers with 8 elements, and 8 wide SIMD instructions, the
compiler may vectorize a loop so that is executes 8 iterations concurrently for
each vectorized loop iteration.

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

Inside the loop body, the machine code loads src[i] and dst[i] into registers,
adds them, and stores the result back into dst[i].

Considering the location of dst and src in the loop body, the elements dst[i]
and src[i] would be located in registers, all other elements are located in
memory. Let register R0 contain the base address of dst, register R1 contain i,
and register R2 contain the registerized dst[i] element.

     + dst's address stored in register 0
     v
     +---------------------------------------------------+
     | 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 |
     +---------------------------------------------------+

     register 1 contains i
     register 2 has copy of a portion of dst
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

    // 4. The size of the register element:
    `DW_OP_lit4`

    // 5. Make a composite location description for dst that is the memory #1
    //    with the register #2 positioned as an overlay at offset #3 of size #4:
    `DW_OP_overlay`

On the first iteration of the vectorized loop the overlay would look like:

         +---------------------------------------------------+
    dst  | 0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F 10 |
         +---------------------------------------------------+
    reg2 | 0  1  2  3  4  5  6  7 |
         +------------------------+

A consumer accessing dst[8] would reference the unsigned int reg0+32
but when a consumer accesses dst[2] it would reference the unsigned
int located at the 9th byte of reg2.

Then on the second iteration of the loop after i had been incremented by 8:

         +---------------------------------------------------+
    dst  | 0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F 10 |
         +---------------------------------------------------+
    reg2                         | 8  9  A  B  C  D  E  F |
                                 +------------------------+

The situation would be reversed. A consumer accessing dst[8] would
reference the unsigned int at byte 0 of reg0 but when a consumer
accesses dst[2] it would reference the unsigned int located reg0+8

An overlay can also be used to create a composite location without
using `DW_OP_piece`. For example GPUs often store doubles in two
32b parts. An overlay can be used to concatenate the locations.

    DW_OP_undefined
    DW_OP_addr 0x100
    DW_OP_lit0  # offset 0
    DW_OP_lit4  # size 4
    DW_OP_overlay
    DW_OP_addr 0x200
    DW_OP_lit4  # offset 4
    DW_OP_lit4  # size 4
    DW_OP_overlay

                +------------------------------------+
    2nd overlay |                    200 201 202 203 |
                |+-----------------+                 |
    1st overlay || 100 101 102 103 |                 |
    base        || <undefined> ... |                 |
                |+-----------------+                 |
                +------------------------------------+

*Note: my diagram is slightly wrong but I don't think it matters. My
 understanding is that the undefined base layer continues on into
 infinity.

*Note: this was directly from Cary's email about how to do
 concatenation with overlays. I do not understand why we need the
 first overlay with the undefined except that it constrains the size
 of the storage location to be 8 bytes long rather than being as long
 as the first storage in the overlay. I think that there was a slight
 problem in Cary's emailed version. He had the size of the overlay as
 8 but I understand the size of the overlay to be 4. A simpler example
 is:*

    DW_OP_addr 0x100
    DW_OP_addr 0x200
    DW_OP_lit4  # offset 4
    DW_OP_lit4  # size 4
    DW_OP_overlay

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
    DW_OP_offset 8 # this could also be computed using DW_OP_push_lane
                   # I just picked a number
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

Rename Section 3.11 to "Composite Piece Description Operations"

Add a new Section 3.12 after "Composite Piece Description Operations",
called "Overlay Composites" With the the following operations:

> 1.  `DW_OP_overlay`
>     `DW_OP_overlay` pops four stack entries. The first must be an integral
>     type value that represents the overlay byte size value S. The second
>     must be an integral type value that represents the overlay byte offset
>     value O. The third must be a location description that represents the
>     overlay location description OL. The fourth must be a location
>     description that represents the base location description BL.
>
>     The action is the same for `DW_OP_bit_overlay`, except that the overlay
>     bit size BS and overlay bit offset BO used are S and O respectively
>     scaled by 8 (the byte size).
>
> .  `DW_OP_bit_overlay`
>     `DW_OP_bit_overlay` pops four stack entries. The first must be an integral
>     type value that represents the overlay bit size value BS. The second
>     must be an integral type value that represents the overlay bit offset
>     value BO. The third must be a location description that represents the
>     overlay location description OL. The fourth must be a location
>     description that represents the base location description BL.
>

In Section 8.7.1 "DWARF Expressions", add the following
rows to Table 8.9 "DWARF Operation Encodings":

> Table 8.9: DWARF Operation Encodings
>
> | Operation                          | Code  | Number of Operands | Notes |
> | ---------------------------------- | ----- | ------------------ | ----- |
> | `DW_OP_overlay`                    | TBA   | 0                  |       |
> | `DW_OP_bit_overlay`                | TBA   | 0                  |       |
