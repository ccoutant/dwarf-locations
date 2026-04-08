# An Improved Model for Locations and Storage

This concept proposal is meant to establish a solid basis
for registers, register pairs, and vector registers as
DWARF locations, and to introduce a series of related
proposals to improve the DWARF location model’s ability
to describe more complex scenarios.


## I. Background

When a location description places a variable or other data object
in a register, it’s usually fairly obvious what that means,
but there are exceptional cases.
The following sections consider various scenarios where values
are promoted to registers.


### Scalar located in a register

When the variable is the same size (as determined by its
type) as the register, we can safely assume that the
location description means that the value of the variable is
located in the entire register, with the most-significant
bit (msb) of the value aligned with the most-significant bit of
the register, and the least-significant bit (lsb) of the value
aligned with the least-significant bit of the register.

    Value:      ........ ........ ........ ........
                |||||||| |||||||| |||||||| ||||||||
              +-------------------------------------+
    Register: | ........ ........ ........ ........ |
              +-------------------------------------+
               msb                               lsb

    Example 1: Variable placed in a register

(In all diagrams, whether big- or little-endian, the msb is at
the left and the lsb is at the right.)

If the value is shorter than the register, the DWARF spec
doesn’t specify, but it’s usually correct to assume that the
value is right-aligned in the register; i.e., the
least-significant bits are aligned.

    Value:                        ........ ........
                                  |||||||| ||||||||
              +-------------------------------------+
    Register: | xxxxxxxx xxxxxxxx ........ ........ |
              +-------------------------------------+

    Example 2: Variable placed in a wider register

There’s still a question of what those upper bits should be.
Again, the spec doesn’t say, but it’s usually safe to assume
that a signed value (as determined by the variables’s type)
is sign-extended to the full width of the register, and an
unsigned value is zero-extended.

It may also be the case that the upper bits are undisturbed
by the load instruction, and the debugger should assume
they are to be ignored. This is a safe assumption
when reading the value from the register, but when writing,
the debugger must know whether to write the upper bits.

On the x86-64 architecture, for example, the `RAX` register is
64-bits wide, but can also be accessed as the 32-bit wide `EAX` register,
the 16-bit wide `AX` register, or a pair of 8-bit wide `AH` and `AL` registers.
These all share the same DWARF register number 0.

    +-------------------------------------------------------------------------+
    | ........ ........ ........ ........ ........ ........ ........ ........ |
    +-------------------------------------------------------------------------+
                                                            <--AH--> <--AL-->
                                                            <-------AX------>
                                           <--------------EAX--------------->
      <----------------------------------RAX-------------------------------->

When a variable is mapped to DWARF `reg0`, the debugger must
make an assumption based on the size of the variable (as given
by its type) and on its knowledge of how the compiler would
generate code to load the variable into the register.

Suppose the compiler has two `uint8_t` variables `x` and `y`,
and loads them onto AH and AL, respectively. To describe
where `y` is, it could simply say `DW_OP_reg0`, but the
debugger will need to assume from out-of-band knowledge
that the compiler is using just the 8 bits of AL.

Describing `x` is more complicated, as the only way we have
of describing its placement in AH is with the `DW_OP_bit_piece`
operator, event though `x` is not a composite:

    DW_OP_reg0
    DW_OP_bit_piece(8, 8)

This location description says that the first 8-bit “piece” of
`x` (i.e., all of it) is placed in `reg0` (the `RAX/EAX/AX` register) at
a bit offset of 8, matching where `AH` maps onto the `AX`
register. Note here that the bit offset of 8 in the `DW_OP_bit_piece`
operator always refers to an offset relative to the lsb of the
register, even on big-endian machines.

On the other hand, suppose we describe the location of `y` as:

    DW_OP_reg0
    DW_OP_piece(1)

This location description says that the first 1-byte “piece” of `y`
(i.e., all of it) is placed in `reg0`. That extra piece operator makes
a big difference: now it’s up to the ABI to determine where the
bits of `y` are placed in the register. I don’t know of any ABIs
that would place `y` anywhere but at the right end of the register
(even big-endian ABIs), but the DWARF spec allows for an exception.

Similarly, the 64-bit Arm architecture has 64-bit `X` registers that can
also be addressed as 32-bit `W` registers. The debugger must determine
whether a 32-bit or smaller variable is using the `W` register or
the full-width `X` register. It can usually make the right assumption
based on out-of-band knowledge of the ABI and typical compiler
code generation.


### Data member located in a register

Now consider a structure type:

    struct s {
        uint16_t a;
        uint8_t b;
        uint8_t c;
    } v;

If a member of a struct, say `v.a`, is promoted to a register over
a certain pc range, the variable `v` might be mapped as shown in
Example 3.

                                  <-------a-------> <---b--> <---c-->
    Value:                        ........ ........ ........ ........
                                  |||||||| |||||||| |||||||| ||||||||
              +-------------------------------------+ |||||| ||||||||
              | <----------------a----------------> | |||||| ||||||||
    reg0:     | 00000000 00000000 ........ ........ | |||||| ||||||||
              +-------------------------------------+ |||||| ||||||||
                                                    |||||||| ||||||||
                                +-------------------------------------+
                                | <----(hidden)---> <---b--> <---c--> |
    Memory:                     | ........ ........ xxxxxxxx xxxxxxxx |
                                +-------------------------------------+
                                  low addresses   ->   high addresses

    Example 3(a): Data member placed in a wider register (big-endian)

                                  <---c--> <---b--> <-------a------->
    Value:                        ........ ........ ........ ........
                                  |||||||| |||||||| |||||||| ||||||||
                                +-------------------------------------+
                                | <----------------a----------------> |
    reg0:                       | 00000000 00000000 ........ ........ |
                                +-------------------------------------+
                                  |||||||| ||||||||
                                +-------------------------------------+
                                | <---c--> <---b--> <----(hidden)---> |
    Memory:                     | ........ ........ xxxxxxxx xxxxxxxx |
                                +-------------------------------------+
                                  high addresses   <-   low addresses

    Example 3(b): Data member placed in a wider register (little-endian)

In this example, the 16 bits corresponding to `v.a` are placed in
`reg0`, right-aligned and possibly zero-filled, while the bits corresponding
to the other two fields are still found in memory.
(The `x’s` mark “hidden” locations in memory that held the value of `v.a` prior
to promotion to the register.)

A DWARF location list for `v` might have a default entry like the following:

    DW_OP_addr(&v)

For the pc range where `v.a` has been promoted, there would be a location list
entry like the following:

    DW_OP_reg0
    DW_OP_piece(2)     # field a
    DW_OP_addr(&v + 2)
    DW_OP_piece(2)     # fields b and c

If we imagine the overlay operator from the GPU proposals, or the map operator
from 250408.1, we could describe this more naturally as:

    DW_OP_addr(&v) # base location for the whole struct
    DW_OP reg0     # location where v.a is mapped
    DW_OP_lit0     # offset within base location
    DW_OP_lit2     # size of mapping
    DW_OP_overlay  # overlay reg0 on top of memory base location

The point of this example is to show that when a data member is placed
in a register, we have the same questions about which bits of the register
are used, zeroed, or sign-extended as for scalars.


### Multiple data members or whole structure in a register

With a composite, however, we also must be careful when working with offsets.
To access `v.b`, for example, we find from the type DIE that the data member
offset for field `b` is 2 (bytes). When looking at the mapping for the
structure, it’s important to count the register mapping as only 2 bytes,
even though the register itself is 32 bits long.
This distinction may seem obvious, but less so if, for some reason,
we had two contiguous fields, say `v.b` and `v.c`, mapped together in one register,
as in Example 4.

                <-------a-------> <---b--> <---c-->
    Value:      ........ ........ ........ ........
                |||||||| |||||||| |||||||| ||||||||
              +-------------------------------------+
              |                   <---b--> <---c--> |
    reg0:     | 00000000 00000000 ........ ........ |
              +-------------------------------------+
                |||||||| ||||||||
              +-------------------------------------+
              | <-------a-------> <----(hidden)---> |
    Memory:   | ........ ........ xxxxxxxx xxxxxxxx |
              +-------------------------------------+
                low addresses   ->   high addresses

    Example 4: Two data members placed together in a register (big-endian)

(I’ve omitted the little-endian version of this example,
which doesn’t exhibit the problem I’m illustrating here.)

For this pc range, where `v.b` and `v.c` have been promoted,
there would be a location list entry like the following:

    DW_OP_addr(&v)
    DW_OP_piece(2)     # field a
    DW_OP_reg0
    DW_OP_piece(2)     # fields b and c

The data member offset for `v.c` is 3 (bytes), which would translate
to an offset of 1 byte relative to the piece that is mapped to `reg0`.
The 1-byte offset needs to be interpreted relative to the bits that
have been mapped directly, not counting the zero-extension bits.

One might instead use `DW_OP_bit_piece(16,0)` for the second
piece, to make explicit the placement of the piece at the
right end of the register (recall that the offset field of
`DW_OP_bit_piece` is always relative to the lsb).

Sometimes, a whole structure might be loaded into a register;
for example, when being passed by value or when being copied
as a whole. In this case, the location list entry is simple:

    DW_OP_reg0

Now suppose that the register is 64 bits wide. The structure
would then likely be placed at the right end of the register and
zero-filled to the left. When the debugger looks up the
data member offset for any of the fields, that offset is relative
to the 32-bit representation of the structure, not the widened one.
On a big-endian machine, this means that a data member offset of 0
would translate to an offset of 32 bits from the msb of the register.
On a little-endian machine, the offset is relative to the lsb, so
no adjustment would be necessary.

On the other hand, if a big-endian machine were to place the
32-bit structure in the high-order bits (i.e., left aligned),
no adjustment would be necessary.


### Scalar or entire struct located in a register pair

When a variable is larger than a general register,
it is often promoted to a pair of registers, as in Example 5.

    Value:        ........ ........ ........ ........ ........ ........ ........ ........
                  //////// //////// //////// //////// \\\\\\\\ \\\\\\\\ \\\\\\\\ \\\\\\\\
               +-------------------------------------+-------------------------------------+
    Reg0/Reg1: | ........ ........ ........ ........ | ........ ........ ........ ........ |
               +-------------------------------------+-------------------------------------+

    Example 5: Variable placed in a pair of 32-bit registers

(On a little-endian machine with hardware instructions supporting
register pair operations, it may be that `reg0` and `reg1` would be reversed,
with the low-order half in `reg0` and the high-order half in `reg1`.)

This must be described using a composite location, even though the variable
is itself a scalar:

    DW_OP_reg0
    DW_OP_piece(4)
    DW_OP_reg1
    DW_OP_piece(4)

A more natural description of this situation might be something like:

    DW_OP_reg0
    DW_OP_reg1
    DW_OP_register_pair  # Form a virtual 64-bit register from reg0 and reg1

Suppose the variable is not a multiple of the register size
(e.g., a 6-byte structure with 32-bit registers). The
variable would occupy all of the first register or registers
and part of the last, as in Example 6.

    Value:        ........ ........ ........ ........ ........ ........
                  //////// //////// //////// //////// \\\\\\\\ \\\\\\\\
               +-------------------------------------+-------------------------------------+
    Reg0/Reg1: | ........ ........ ........ ........ | ........ ........ xxxxxxxx xxxxxxxx |
               +-------------------------------------+-------------------------------------+

    Example 6(a): 48-bit variable placed in a pair of 32-bit registers (big-endian)

    Value:                          ........ ........ ........ ........ ........ ........
                                    //////// //////// \\\\\\\\ \\\\\\\\ \\\\\\\\ \\\\\\\\
               +-------------------------------------+-------------------------------------+
    Reg1/Reg0: | xxxxxxxx xxxxxxxx ........ ........ | ........ ........ ........ ........ |
               +-------------------------------------+-------------------------------------+

    Example 6(b): 48-bit variable placed in a pair of 32-bit registers (little-endian)

In the case of mapping such a structure onto registers, a big-endian ABI
may choose to map the extra bytes to the high-order bits of the final
register. This is a likely consequence of parameter calling conventions,
where some arguments are passed in registers, and structures passed by value
are not decomposed.

This would be described as:

    DW_OP_reg0
    DW_OP_piece(4)
    DW_OP_reg1
    DW_OP_piece(2)

Unlike the case in Example 4, now the `DW_OP_piece(2)`
operation means to place the bits in the upper half of
`reg1` on a big-endian machine, rather than the lower half,
but we can only distinguish the two cases by guessing what
the conventions in effect are. If the variable is a
structure mapped *as a whole* onto registers, it’s likely to
be left-aligned (i.e., starting at the high-order end).
If the mapping applies to an individual field, it’s likely
to be right-aligned (i.e., starting at the low-order end).

This is the rationale for the ABI dependency in `DW_OP_piece`:

> If the piece is located in a register, but does not occupy
> the entire register, the placement of the piece within that
> register is defined by the ABI.

and the statement:

> Whether or not a `DW_OP_piece` operation is equivalent to any
> `DW_OP_bit_piece` operation with an offset of 0 is ABI
> dependent.

Because the `DW_OP_bit_piece` operation has an offset
parameter and specifies that the offset (as it applies to
register locations) is relative to the lsb, we could write
the big-endian location description for Example 6 as:

    DW_OP_reg0
    DW_OP_piece(4)
    DW_OP_reg1
    DW_OP_bit_piece(16, 16)

But this would be correct only for the big-endian example.


### Array partially mapped to a vector register

In loop optimizations, a loop is often unrolled so that
several iterations of the loop are executed in parallel.
This involves promoting a slice of an array into a vector
register, and possibly a slice of another array into another
vector register, then executing a parallel operation on those
two registers. During that time, the DWARF needs to describe
the location of each such array using a composite location
where the slice that is currently in the vector register is
marked as such, and the rest of the array, both before and
after, remains in its original location (typically memory).

We have an example of this in Section D.18, SIMD Lane Example,
with a loop unroll factor of 4 (i.e., slice of 4 elements at a
time are loaded into a vector register),
but that example so far only describes the location of the
loop control variable `i`.

To describe the location of the array parameters `dst` and `src`,
we first need to describe the locations of the arrays those
parameters point to, and then form implicit pointers as the
locations of the parameters (which, being passed by reference,
decay to pointers in this example). We could use a DWARF procedure
to describe the location of each array, but we run into a
limitation of the piece and bit-piece operators: they cannot
take a computed offset as an operand. Instead, we will need
to rely on one of the proposals to add overlay operators (251120.1),
mapping operators (250408.1), or new dynamic piece operators (211206.2).
Using the overlay operators, we could describe the location of
the array `dst`, while within the vectorized loop body, as follows:

    DW_TAG_dwarf_procedure
      DW_AT_location [
        DW_OP_breg0(0)    # base location of the array, in r0
        DW_OP_regx(v0)    # location of the slice: vector register 0
        DW_OP_breg3(0)    # i, the starting index of the current slice
        DW_OP_lit4
        DW_OP_mul         # the offset of that slice in bytes
        DW_OP_const1u(32) # sizeof(int)*num_lanes
        DW_OP_overlay
        ]

Now, the location of the parameter `dst` (a pointer) can be
described as an implicit pointer to the composite location
given above:

    DW_OP_implicit_pointer (ref to DWARF procedure, 0)

Now that we allow locations on the stack, we could simplify the
location of `dst` into a single location expression, without reference
to a DWARF procedure, by inventing a new implicit pointer operator
that takes its implicit location from the stack:

    DW_OP_breg0(0)    # base location of the array, in r0
    DW_OP_regx(v0)    # location of the slice: vector register 0
    DW_OP_breg3(0)    # i, the starting index of the current slice
    DW_OP_lit4
    DW_OP_mul         # the offset of that slice in bytes
    DW_OP_const1u(32) # sizeof(int)*num_lanes
    DW_OP_overlay
    DW_OP_form_implicit_pointer

This is not limited to GPUs: Some modern CPUs have instructions
that can treat 64-bit general registers as a vector of 16- or 8-bit
values, with operations like “parallel add”.


### Vertical slice of an array partially mapped to a vector register

Similarly, if a loop is processing an array vertically
(e.g., working on the columns of an array stored in row-major order),
we could use an alternate form of the overlay operator that
specifies an explicit stride.

Suppose we have an 8x8 array, and can process a slice
of four elements (shaded) in each iteration:

           +----+----+----+----+----+----+----+----+
    array: |    |    |    |    |    |    |    |    |
           +----+----+----+----+----+----+----+----+
       +32 |    |    |    |    |    |    |    |    |
           +----+----+----+----+----+----+----+----+
       +64 |    |    |    |    |    |    |    |    |
           +----+----+----+----+----+----+----+----+
       +96 |    |    |    |    |    |    |    |    |
           +----+----+----+----+----+----+----+----+
      +128 |    |    |////|    |    |    |    |    |
           +----+----+----+----+----+----+----+----+
      +160 |    |    |////|    |    |    |    |    |
           +----+----+----+----+----+----+----+----+
      +192 |    |    |////|    |    |    |    |    |
           +----+----+----+----+----+----+----+----+
      +224 |    |    |////|    |    |    |    |    |
           +----+----+----+----+----+----+----+----+

    Example 7: Vertical slice of an 8x8 array of uint32_t

Here, we have a slice consisting of `array(2, 4:7)` promoted to a
vector register. The starting offset of the overlay would be `4 *
32 + 8`, where 32 is the stride of the array in bytes. This could
be described with a location expression like the following:

    DW_OP_breg0(0)    # (1) base location of the array, in r0
    DW_OP_regx(v0)    # (2) location of the slice: vector register 0
    DW_OP_breg3(0)    # i, the first row of the slice
    DW_OP_const1u(32) # stride
    DW_OP_mul         # offset of the row
    DW_OP_breg4(0)    # j, the current column
    DW_OP_lit4
    DW_OP_mul         # offset within row
    DW_OP_add         # (3) offset of first element of slice
    DW_OP_const1u(32) # (4) stride
    DW_OP_lit4        # (5) number of elements in slice
    DW_OP_lit4        # (6) size of each element
    DW_OP_overlay_stride

In this example, the `DW_OP_overlay_stride` would establish four
separate overlays, each 32 bytes apart, that would map the
corresponding elements of the array to elements of the vector
register.


### Scalar located in a vector register

Vectorized loops sometimes compute a scalar value in each iteration,
and keep that scalar in one element of a vector register, said
element corresponding to the current lane (or iteration).
Suppose the example in Section D.18 is extended to compute a scalar
intermediate value:

    void vec_add (int dst[], int src[], int len) {
        for (int i = 0; i < len; ++i) {
            int temp = dst[i] + src[i];
            ...
            dst[i] = temp;
        }
    }

    Example 8: Scalar in a vectorized loop

While in the vectorized loop body, the compiler might allocate the
`temp` variable to another vector register, say `v4`.
This register would hold four copies of the variable,
one for each of the iterations being processed in parallel
(i.e., one per lane). We could describe the location of `temp`
based on the current lane, as follows:

    DW_OP_regx(v4)      # Vector register 4
    DW_OP_push_lane     # Get current lane
    DW_OP_lit4
    DW_OP_mul           # Compute byte offset within the register
    DW_OP_offset        # Offset into the vector register

But now we have the same ambiguity that we found earlier: How
do we distinguish between a variable that occupies just part of
the register vs. one that has been widened to fill the entire
register? In this case, where we have a small scalar and a
vector register, the consumer is likely able to infer the answer
from context. But consider the case above of a 64-bit general register
with eight 8-bit values, where it may not be as clear that a `uint_8 temp`
in lane 0 should only occupy bits 0–7.

This could be simplified and made clearer by introducing subregisters
(as suggested in Issue 230712.1).

    DW_OP_regx(v4)      # Vector register 4
    DW_OP_push_lane     # Get current lane as index
    DW_OP_subreg(32)    # Split v4 into N x 32-bit subregisters and select one

Here the new `DW_OP_subreg` operator takes an inline operand that
indicates the size of the vector elements, pops an index `I` and
a vector register `V` from the stack, creates a new register storage block
for the subregister `V[I]`, and returns a location referring to that
new storage.

This new subregister operation would also allow the producer to
indicate that a variable has been placed in, say, `AL`, `AH`,
`AX`, `EAX`, or `RAX` on x86; or to a `W` or `X` register on Arm.


### Scalar located in a pair of vector registers

In the above example, the temporary scalar variable was the same size
as the vector elements. If the scalar is wider, say 64 bits, while the
vector elements are 32 bits, the compiler may choose to store its
various instances in a pair of vector registers: the high halves in
one, and the low halves in another:

          (lane N)                (lane 1)    (lane 0)
        +-----------+-----------+-----------+-----------+
    v3: | low bits  |    ...    | low bits  | low bits  |
        +-----------+-----------+-----------+-----------+
    v4: | high bits |    ...    | high bits | high bits |
        +-----------+-----------+-----------+-----------+

    Example 9: Scalar in a pair of vector registers

With the subregister operation, we could describe this with a composite:

    DW_OP_regx(v4)
    DW_OP_push_lane
    DW_OP_subreg(32)
    DW_OP_piece(4)      # Bits 63-32 in v4[n]
    DW_OP_regx(v3)
    DW_OP_push_lane
    DW_OP_subreg(32)
    DW_OP_piece(4)      # Bits 31-0 in v3[n]

Or with an overlay:

    DW_OP_regx(v4)
    DW_OP_push_lane
    DW_OP_subreg(32)    # v4[n]
    DW_OP_regx(v3)
    DW_OP_push_lane
    DW_OP_subreg(32)    # v3[n]
    DW_OP_lit4          # Offset
    DW_OP_lit4          # Width
    DW_OP_overlay       # Place v3[n] to follow v4[n]

To simplify these expressions, suppose we had an operation that
could form a new kind of composite (or compound) storage by
interleaving the elements of two vectors. This operation would
take two locations from the top of the stack and create a new
block of storage that interleaves the storage underlying the two
locations.

Now we could describe the location of `temp` as:

    DW_OP_regx(v4)
    DW_OP_regx(v3)
    DW_OP_interleave(32)    # Size of each interleaved element = 32 bits
    DW_OP_push_lane
    DW_OP_subreg(64)

The interleave operation would produce a new block of storage:

            +-----------+-----------+-----------+
    v3:     | low bits  |    ...    | low bits  |
            +-----------+-----------+-----------+
                  |                       |
                  +-----------+           +-----------------------------------+
                              |                                               |
            +-----------+-----------+-----------+-----------+-----------+-----------+
    result: | high bits | low bits  |    ...    |    ...    | high bits | low bits  |
            +-----------+-----------+-----------+-----------+-----------+-----------+
                  |                                               |             
                  |                       +-----------------------+
                  |                       |
            +-----------+-----------+-----------+
    v4:     | high bits |    ...    | high bits |
            +-----------+-----------+-----------+

    Example 10: Interleaving two vector registers

The `DW_OP_subreg` operation would then select the 64-bit high/low pair
corresponding to the index (in this example, the current lane).

(The `DW_OP_register_pair` operation suggested above could be
considered a special case of `DW_OP_interleave`.)


### Scalar or entire struct located in a register pair (revisited)

With `DW_OP_subreg`, we can now describe the 48-bit variable in Example 6
unambiguously:

    DW_OP_reg0
    DW_OP_piece(4)
    DW_OP_reg1
    DW_OP_lit0
    DW_OP_subreg(16)
    DW_OP_piece(2)

For the little-endian case, the `DW_OP_subreg` would select register 0,
counting from the lsb end, and select the lower 16 bits of `reg1`.
For the big-endian case, it would select register 0, counting from
the msb end, and select the upper 16 bits of `reg1`. The same DWARF
expression works to describe both cases.


### Widening and other representation changes

As we saw in Example 2 above, when a variable or field is loaded into
a register, it may be widened by zero- or sign-extension. For floating-point
registers, a value loaded from memory might be converted from a memory
format to a different register format (and vice versa, when stored).

How can we indicate these representation changes? Do we need to?
Consumers have been dealing with these ambiguities up to now, but
the inability to communicate exceptional conditions constrains the
producers in many situations, especially when dealing with vector
registers. The `DW_OP_subreg` operation helps in many of these
situations, but still leaves it up to the consumer to infer whether
a value is sign- or zero-extended. Occasionally, a floating-point
register needs to be spilled to memory in its register format,
but there’s no good way for a consumer to guess that a value in
memory is actually in register format.

There’s also a subtler problem. Consider the following location expression
for a 16-bit variable `x` assigned to a 32-bit register:

    DW_OP_reg0
    DW_OP_lit1
    DW_OP_offset

In this expression, the location of `x` is bit 8
of register 0. If the location were at bit 0, the consumer would
likely infer that `x` occupies the entire register. In this case,
what should the consumer infer? Does it occupy bits 8-23, or was
it extended to occupy bits 8-31? On a little-endian machine, the
answer might not make much difference, but the consumer would need
to know if asked to write to that location. On a big-endian machine,
the answer makes a big difference: In one case, the value is in bits 8-23,
but in the other case, the actual value is in bits 16-31, with zeros
or sign extension in bits 8-15.

Now consider the following:

    DW_OP_call(ref to x)    # returns location of register 0
    DW_OP_lit1
    DW_OP_offset

In the first case, the offset is used to modify the register location
where `x` is stored, but in the second, the offset is (likely) meant
to offset *into* the value of `x`; i.e., we want to set the offset to point to
bit 8 of `x`.

There’s a problem with this: `DW_OP_call*` isn’t currently
specified to work this way — it’s simply a DWARF procedure call,
where the `DW_AT_location` expression of the referenced DIE is
executed as if it were simply part of the calling expression, so this
sequence would be indistinguishable from the first case. We really
have two kinds of calls in practice: one where we want to simply
call a DWARF procedure using the current stack, and a second
(currently unspecified) where we want to obtain the location of
some object and push it on the stack.

Suppose we had the second form, as a new `DW_OP_push_location_of` operator.
This operation would evaluate the `DW_AT_location` attribute of the
referenced DIE, on its own evaluation stack, and then push the resulting
location onto the caller’s stack. Now we could write the following:

    DW_OP_push_location_of(ref to x)
    DW_OP_lit1
    DW_OP_offset

The difference is still subtle, but in one case, we have the
simple location pushed by `DW_OP_reg0`, and in the other case,
we have a *mapping* of the object `x` onto the register.
When we have a *mapping*, we’re working in the address space
of the data object’s memory layout, rather than in the address
space of the location’s underlying storage.

Look back at Example 2 (copied here):

    Value:                        ........ ........
                                  |||||||| ||||||||
              +-------------------------------------+
    Register: | xxxxxxxx xxxxxxxx ........ ........ |
              +-------------------------------------+

    Example 2 (restated): Variable placed in a wider register

Here, the register storage has its address space consisting of bits 0-31,
while the value of the object has its address space consisting
of bits 0-15, and these bits are mapped onto the register.


## II. Storage, Locations, and Mappings

As currently defined, a *location* is just a point in some
*storage*, referencing a specific bit position within that
storage. A location has no size or type.

We currently define several kinds of *storage*: memory,
registers, undefined, implicit, implicit pointer, and composite.
Each kind of storage has a size, but no type.

A data object can have a location, and also has a type.
Its `DW_AT_location` attribute maps the bits of that object onto
a location. This *mapping* has a size determined by the type
(and/or other attributes or children of the object).

To address some of the issues raised in part I,
I propose to formalize the concept of a *mapping*,
and to replace the concept of composite storage with
a *compound mapping*. Thus, there would only be five kinds
of storage.

A *mapping* establishes the relationship between the bits of
the data object, using the memory layout defined by the compiler,
and the bits in the underlying storage, as given by the *location*.

There are several kinds of mapping:

- Bit-for-bit. This is the default for memory storage,
and for register storage where the size of the mapping matches
the size of the storage. The bits of the object are mapped onto
consecutive bits of the underlying storage, starting with the bit
position specified by the location.

- Sign-extended. This is the default for register
storage, where the object has a signed integral type and the size
of the mapping is smaller than the size of the storage. The bits of
the object are mapped onto the low-order bits of the register,
and the unmapped bits of the register contain sign-extension
bits.

- Zero-extended. This is the default for register
storage, where the object has an unsigned integral type and the size
of the mapping is smaller than the size of the storage. The bits of
the object are mapped onto the low-order bits of the register,
and the unmapped bits of the register contain zeroes.

- Sign-truncated. This is the default for register storage, where
the object has a signed integral type and the size of the mapping
is larger than the size of the storage. The low-order bits of the
object are mapped onto the register, and the high-order bits of
the object are defined to match the sign bit of the register.

- Zero-truncated. This is the default for register storage, where
the object has an unsigned integral type and the size of the mapping
is larger than the size of the storage. The low-order bits of the
object are mapped onto the register, and the high-order bits of
the object are defined to be zeroes.

- FP Register. This is the default for floating-point register
storage, on architectures where the usual floating-point load and store
instructions convert the floating-point value to an internal
representation that is different from its memory format.
This may also apply where single-precision values are converted
to double-precision when loaded into registers. An ABI may define
one or more such mappings.

An ABI may define additional mapping kinds as needed.

A *compound mapping* consists of an ordered set of mappings,
where each member of the set maps a range of bits from the data
object onto a location. Ranges in the set may overlap, and later
mappings are defined to overlay earlier mappings for the same
range (or overlapping portions of a range).

In a compound mapping, a mapping also has either the
*transparent* or *opaque* property. Opaque mappings hide earlier
overlapping mappings, such that the underlying storage for the
mapped location is the only live copy of that range of bits from
the object. Transparent mappings indicate that the corresponding
bits have been copied from the location(s) specified by earlier
overlapping mappings, and the values are live in both locations.
By default, mappings are opaque.

Mappings are established both implicitly and explicitly. The
composite piece operators and overlay operators establish
explicit mappings. For data objects with a `DW_AT_location`
attribute, a mapping for the entire object is established
implicitly after the location expression has been evaluated, if
the result is not already a mapping.

A new `DW_OP_mapping` operation is defined to override the
default mapping kind. It may precede an operation that
establishes an explicit mapping, or be placed at the end of a
location expression to define the implicit mapping. It takes a
single ULEB inline parameter specifying the kind of mapping.

A new `DW_OP_transparent` operation is defined to set the
transparent property on the following explicit mapping operation.
(This would replace the `DW_OP_copy`, `DW_OP_overlay_copy`,
and `DW_OP_bit_overlay_copy` operations proposed in 251017.1.)


## III. Proposals

The following new concrete proposals would follow from this:

1. Storage, Locations, and Mappings. Update Section 2.5
with the new model given in Part II. Add `DW_OP_mapping`
and `DW_OP_transparent` operators.

2. Add `DW_OP_overlay_stride` operator.

3. Add `DW_OP_register_pair`, `DW_OP_interleave`, and
`DW_OP_subreg` operators. These new operators would form new
storage and return a location in that storage.

4. Add `DW_OP_form_implicit_pointer` operator.

5. Add `DW_OP_push_location_of` operator. For DIEs with `DW_AT_location`,
it would evaluate the location on a separate stack, then push the resulting location on the caller’s stack.
For DIEs with `DW_AT_const_value`, it would push an implicit storage location.
Define the `DW_OP_call*` operators to work only on `DW_TAG_DWARF_procedure` DIEs
with (new) `DW_AT_DWARF_procedure` attributes.


