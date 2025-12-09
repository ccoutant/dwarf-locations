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

## Current approaches

### Why `DW_OP_regval_type` is not enough

`DW_OP_regval_type` is not sufficient to solve this problem because

### Swizzled pointers

## Proposed fix

## PROPOSAL
