# Support for temporarily changing the type of a variable

## PROBLEM DESCRIPTION

There are situations where the compiler changes the type of variable
temporarily due to optimization or relocation. There is currently no
way for this to be communicated from producers to consumers.

### Float promotion

If a variable is declared a double and is given location in memory
that variable will be a normal 64b IEEE double floating point
number. However, if due to optimization that variable is moved into a
80b extended precision register for a block of PCs. If a consumer
stops within that range of PCs and asks to print that variable, the
location will point to 80b register rather than a 64b IEEE double.

Current versions of GDB handle this situation with special case code
which demotes the 80b register's value back to the 64b base type. This
is less than ideal because if the problem with a calculation is
happening in the last 16b of the extended precision float, then this
will be masked from the programmer. One way to handle this is by
dumping the raw contents of the register and then decoding it
manually. A programmer may not know to do this or know how to do
this. It also is a very cumbersome work around.

A better option is to address this situation is to have the producer
use DW_OP_regval_type so long as the variable is in a
register. However, if one of these extended floating point registers
representing a smaller type is spilled to memory without conversion
back to a 64b double then the problem still exists.

### Type width reduction optimization

New optimization passes are starting to be able to prove that the
range of a variable is within known bounds. When it can prove that the
range of a values can fit in a smaller type, the optimizer may choose
to reduce the width of the type. In other words, a 4 byte int can
become a 2 byte short or even a single byte char. This optimization is
particularly useful when applied to vector registers because it can
double or quadriple the number of lanes. Like loop vectorization where
the compiler generates both a vectorized code block as well as a block
to handle the non-vector width remainder, the compiler code generation
can emit blocks of code which are used when the compiler can prove
that the size can be reduced.

This leaves the consumer confused about what to do when stopped within
one of these blocks of code. The type of the variable will be bigger
than the area it takes up in storage. This can be particularly
confusing when the storage pointed to by the location is a vector
register. The variables will appear to overlap and the consumer will
extract too many bits from the location and present an erroneous value
to the programmer.

### Pointer width changes

With the introduction of address spaces, one of the challenges which
must be dealt with is the width of a pointer may be different in
different address spaces. For example a host system may be 64b using 8
byte pointers while an address into GPU memory may only be 32b using 4
byte pointers. If there is an object which has a pointer to it that is
moved from system memory to GPU memory by an optimization, the
consumer will need to know that the size of the pointer has changed
from 8 bytes to 4 bytes when accessing it.

## Proposed fix

Remove the restriction on `DW_OP_reinterpret` that requires "The type
of the operand and result type must have the same size in bits." This
addresses all three of the situations mentioned above.

Float promotion - While the producer could use DW_OP_regval_type
pointing to the extended precision float type when the value is in a
registerm, if it were spilled to memory without conversion back to a
64b double then it could end the location expression with
`DW_OP_reinterpret`. Then the location would point to a memory address
and instead of reading 8 bytes it would read 10 bytes.

The opposite would happen when you had type reduction and pointed at a
vector register with an offset or memory location where the
expectation is you would read the size of the refined
type. `DW_OP_reinterpret` would be applied to the location on the
stack.

Pointer size changes associated with address space relocations could
be pointed to by having the final op in the location being
DW_OP_reinterpret referring to a 4 byte pointer to the correct type.

This change is backward compatible with DWARF5 and earlier because all
previous uses of DW_OP_reinterpret would be subject to the bit size
limitations.

## PROPOSAL

In section 3.4 after the description of DW_OP_regval_type add the
following non-nomraltive paragraph:

>    *When the compiler changes the type of a variable and its
>    location is stored in a register, this operation can be used to
>    inform the consumer that its type has changed. One such example
>    would be when a double float is promoted to an extended precision
>    floating point number by copying it to an extended precision
>    floating point register.*

In section 3.16 Type Conversions 

Remove the sentence at the end of the description of
`DW_OP_reinterpret` which says: "The type of the operand and result
type must have the same size in bits."

Then add the paragraphs.

>    *In previous versions of DWARF `DW_OP_reinterpret` was limited to
>    only reinterpreting types that had the same size in bits. This
>    restriction has been removed in DWARF6`*
>
>    When the compiler changes the type of variable,
>    `DW_OP_reinterpret` signals this to the consumer so that the
>    correct number of bits are fetched and properly interpreted.
>
>    *Some examples of when the compiler may change the type of a
>    variable are when a double is promoted to a extended precision
>    floating point number and that register is then spilled to
>    memory. Another example is when the compiler is able to prove
>    that the range of values is small enough that the width of the
>    type can be reduced within some range of addresses. Another
>    example is if there is a pointer to an object which is moved to
>    an address space which uses addresses which are smaller than
>    those used by the system.*

FIXME: consider adding some best practice examples into appendix D.
