# Support for temporarily changing the type of a variable

## PROBLEM DESCRIPTION

There are situations where the compiler changes the type of variable
temporarily due to optimization or relocation. There is currently no
way for this to be communicated from producers to consumers.

### Register promotion

If a variable's type is shorter than the register than it is currently
being promoted to, the DWARF spec currently doeesn’t specify what to
do. It has been customary and has been usually correct to assume that
the value is right-aligned in the register; i.e., the
least-significant bits are aligned. This models the behavior of a
typical load instruction. However most architectures do not just have
one general load instruction, they have various load instructions and
the producer is using a specific one. These various load instructions
have different semantics. The particular load instruction will specify
the width of data copied into the register and whether sign extension
happens or zero filling of the upper bits will happen. The producer
simply needs an unambigious way to communicate the precise semantics of
particular load instruction that it used and the represntation of the
resulting value when it is stored in the register.

The current situation where the specific semantics of the load
operation and the resulting data representation are not communicated
leads to several ambiguities. Several of which are presented in [Issue
260422.1: An Improved Model for Locations and
Storage](https://dwarfstd.org/issues/260422.1.html)

#### Ambiguity about the size of the register

Many architectures have evolved over the years and the size of the
registers has changed. The ABIs for these platform rarely define a new
register number for a register which has grown in size. One very
common example are the general purpose registers on x86_64 like the
accumulator register:

    +-------------------------------------------------------------------------+
    | ........ ........ ........ ........ ........ ........ ........ ........ |
    +-------------------------------------------------------------------------+
                                                            <--AH--> <--AL-->
                                                            <-------AX------>
                                           <--------------EAX--------------->
      <----------------------------------RAX-------------------------------->

When a variable is mapped to DWARF `reg0`, the accumulator register,
the debugger must make an assumption based on the size of the variable
(as given by its type) and on its knowledge of how the compiler would
generate code to load the variable into the register.

#### Ambiguity about upper bits

When the general assumption is that a value is extended to fill the
entire register, it can cause confusion about what happens to the
upper bits. For example 260422.1 presented an example where when a
member of a structure is promoted to a register.

Consider the structure type:

    struct s {
        uint16_t a;
        uint8_t b;
        uint8_t c;
    } v;

If `v.a`, is promoted to a register over a certain pc range, the
variable `v` might be mapped as shown in Example 3.


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

Since the a member is residing in a larger register than its type,
arithmatic operations can concievably lead to intermediat values whose
value is larger than the declared type. How would a consumer present
that? Neither way of constructing composite storage would allow the
consumer to present the full intermeidate value without masking
potentially important bits. While this would likely be undefined
behavior in a language standard, it is not uncommon for there to be
bugs that unknowingly rely on behavior such as this. If the consumer
then masks the behavior it would make discovering the root of these
bugs exceedingly difficult. Both:

    DW_OP_reg0
    DW_OP_piece(2)     # field a
    DW_OP_addr(&v + 2)
    DW_OP_piece(2)     # fields b and c

and

    DW_OP_addr(&v) # base location for the whole struct
    DW_OP reg0     # location where v.a is mapped
    DW_OP_lit0     # offset within base location
    DW_OP_lit2     # size of mapping
    DW_OP_overlay  # overlay reg0 on top of memory base location

would mask bits outside the range of uint16_t which might be the
source of the bug.

#### Packed registers

There is also ambiguity when a the target ISA allows the use of
different bits within a register as descrete values. A historic but
widespread example is x86_64 where AH and AL are both addressable
within Register 0, the accumulator register. A consumer cannot simply
assume that the full RAX register is being used when a location refers
to Register 0. If the value happens to be signed, the opcode which is
used would determine if the value is sign extended to the full width
of the register or if it was constrained to just the lower 8
bits. While this is a historic example and modern compilers are
unlikely to use AL and AH simultaneously for different values due to
their limited range, the rise of AI has found a lot of utility in low
precision floating point numbers and smaller integers and so current
GPU's ISAs frequently pack discrete values into portions of
registers. This is how nvidia, AMD, and intel all handle FP4 and INT4
types. This sub-register view is highly related to the now withdrawn
proposal [Issue 230712.1: Register Segment Name
Type](https://dwarfstd.org/issues/230712.1.html)

#### Float promotion

A slightly more complicated version of the same problem is if a
variable is declared a double and is given location in memory that
variable will be a normal 64b IEEE double floating point
number. However, if due to optimization that variable is moved into a
80b extended precision register for a block of PCs. If a consumer
stops within that range of PCs and asks to print that variable, the
location will point to 80b register rather than a 64b IEEE double.

This situation is a bit more complicated than normal register
promotion because not only does the width of the type change, the bit
position of fields within the the type changes.

Current versions of GDB handle this situation with special case code
which demotes the 80b register's value back to the 64b base type. This
is less than ideal because if the problem with a calculation is
happening in the last 16b of the extended precision float, then this
will be masked from the programmer. One way to handle this is by
dumping the raw contents of the register and then decoding it
manually. A programmer may not know to do this or know how to do
this. It also is a very cumbersome work around.

A better option to address this situation is to have the producer use
DW_OP_regval_type so long as the variable is in a register. However,
if one of these extended floating point registers representing a
smaller type is spilled to memory without conversion back to a 64b
double then the problem still exists.

It should be noted that in cases such as this both the normal 64 bit
floating point and the extended precision floating point number would
both be base types defined by the target ABI.

### Type width reduction optimization

New optimization passes are starting to be able to prove that the
range of a variable is within known bounds. When it can prove that the
range of a values can fit in a smaller type, the optimizer may choose
to reduce the width of the type. In other words, a 4 byte int can
become a 2 byte short or even a single byte char. This optimization is
particularly useful when applied to vector registers because it can
double or quadruple the number of lanes. Like loop vectorization where
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

## Other uses

There are other cases where the producer can assist the consumer by
providing more precise type information.

### Derived types

One example could be when a compiler knows the type more precisely
than the variable's type natively indicates. For example: `BaseV *bv2
= new DerivedV;` adding more precise type information could allow the
consumer to print the full contents of the variable rather than just
the parts included in the base class. Some times consumers can already
do this by referring to the vtable to identify the derived type but
having the actual known type which is known by the producer makes this
simpler and can handle situations when a class doesn't have a vtable.

It also allows a consumer handle situations which are currently not
handled such as `(gdb) p func(*bv2)` when `int func(Derived arg);`
without the user having to cast the variable to its known type.

### void pointers with known types

Another potential use is to give the objects pointed to by void
pointers their type. For example given `int x; void *generic=&x` the
compiler knows that object pointed to by generic is an int.

## Proposed fix

Add a new context operator `DW_OP_refine_type` to Section 3.6 which
tells the consumer that context object at the location specified
should be interpreted as the following type. This operator does not
affect the type of locations that may be on the stack. It changes the
type of the Current Object defined in Section 3.1 point 9.

It resolves the ambiguity about the size of the register. If the
producer intends for the variable to extended to be the full size of
the register, it can communicate that fact by updating the context's
Current Object's type within the location list's expression that
covers the range of PCs when the variable is in a register.

For example if v is a int16_t and the compiler promotes it to reg0
which has 64 bits and expects it to be sign extended it would use the
expression:

    DW_OP_reg0
	DW_OP_refine_type <die for int64_t>

This technique also removes the ambiguity what happens to the upper
bits.

It can also be applied to aggregates as well as scalar values with
some added complexity. The example given above where a member of the
structure is promoted to a register could be handled like this:

    struct s {
        uint16_t a;
        uint8_t b;
        uint8_t c;
    } v;

would be replaced by a new artificial type:

    struct __s_a_promoted {
        uint32_t a;
        uint8_t b;
        uint8_t c;
    };

then the expression for its location while a was promoted to reg0
would be:

    DW_OP_refine_type <DIE for __s_a_promoted>
    DW_OP reg0     # location where v.a is mapped
    DW_OP_addr(&v) # base location for the whole struct
	DW_OP_lit2
	DW_OP_offset   # the original offset for b within the structure
    DW_OP_lit4     # offset within base location
    DW_OP_lit2     # size of mapping
    DW_OP_overlay  # overlay reg0 on top of memory base location

The consumer would know how to present this without masking
potenitally improtant bits.

Having the producer explicitly communicate when it expects to expand
and zero extension and when it does not also helps resolve another
ambiguity pointed out by 260422.1 in Example 4. The crux of that
example is that the offset within the register for the fields which
are promoted may be diffent depending the particular load instruction
selected by the compiler and how many bytes were copied by that
instruction. Simple assumptions about zero extension
of discrete values to the full width of a register are no longer
sufficient to know where bytes will land within the register.

Float promotion - While the producer could use `DW_OP_regval_type`
pointing to the extended precision float type when the value is in a
register, if it were spilled to memory without conversion back to a
64b double then it could end the location expression with
`DW_OP_refine_type`. Then the location would point to a memory address
and instead of reading 8 bytes it would read 10 bytes.

The opposite would happen when you had type reduction and pointed at a
vector register with an offset or memory location where the
expectation is you would read the size of the refined
type. `DW_OP_refine_type` would be applied to the location on the
stack.

Pointer size changes associated with address space relocations could
be pointed to by having the final op in the location being
`DW_OP_refine_type` referring to a 4 byte pointer to the correct type.

The other uses would just assist consumers in cases where they would
not be able to satisfy the user's request or would require a manual
cast to accomplish the user's request.

An advantage of this approach is that it factors well. Any portion of
a DWARF expression can change the effective type. This includes
portions which are called from other DWARF expressions.

Some members of the GPU team were concerned that this would force all
DWARF expressions to have a type and that this type was not limited to
base types. Many DWARF operations are currently only defined for base
types to allow the implementaiton of consumers to be more practically
feasable. This proposal does not force locations to have
types. Locations continue to be typeless. This changes the context's
Current Object's type.

## PROPOSAL

Rename section 3.6 from "Context Query Operations" to "Context
Operations"

In section 3.6 add an operation after `DW_OP_push_lane`

>    DW_OP_refine_type([ULEB] DIE offset for type)
>
>    `DW_OP_refine_type` changes the effective type of the current
>    object of the context (See section 3.1 pg. 49). This has the
>    effect informing the consumer that the type has been changed by
>    the compiler. It takes one operand, which is an ULEB integer that
>    represents the offset of a debugging information entry for a type
>    in the current compilation unit.
>
>    This operator cannot be used in cases where the DWARF expression
>    doesn't have a current object in its context such as when
>    evaluating expressions restoring a register as part of CFI.

In section 7.4.2 add another bullet point below the one concerning
`DW_OP_push_object_location` which says:

>    * `DW_OP_refine_type` is not meaningful in an operand of these
>      instructions because there is no object context whose type can
>      be changed.

Add a new section to Appendix D

>    D.<n> Refined Type Examples
>
>    Consider a C++-like program as in Figure D.X where a pointer
>    always points to a derived instance.
>
>        Base *bp = new Derived;
>
>    A possible DWARF description is in D.X.
>
>        1$: DW_TAG_variable
>                DW_AT_name("bp")
>                DW_AT_type(reference to 2$ "Base *")
>                DW_AT_location(
>                        DW_OP_reg0;
>                        DW_OP_refined_type( reference to $5 "Derived")
>
>        2$: DW_TAG_pointer_type
>                DW_AT_type(reference to $4 "Base")
>        3$: DW_TAG_pointer_type
>                DW_AT_type(reference to $5 "Derived")
>        4$: DW_TAG_structure_type
>                DW_AT_name("Base")
>                ...
>        5$: DW_TAG_structure_type
>                DW_AT_name("Derived")
>                ...
>
>
>    Consider another case where the pointee type is known in certain
>    PC ranges only:
>
>        Base *bp = ...;
>        ...
>        if (...) {
>            bp = new Derived;
>            ...
>        } else {
>            bp = new DerivedTwo;
>            ...
>        }
>        ...
>
>    Refined type information can be given in a location list, as
>    shown in Figure D.X.
>
>        1$: DW_TAG_variable
>                DW_AT_name("bp")
>                DW_AT_type(reference to 2$ "Base *")
>                DW_AT_location(location list $9)
>        2$: DW_TAG_pointer_type
>                DW_AT_type(reference to $5 "Base")
>        3$: DW_TAG_pointer_type
>                DW_AT_type(reference to $6 "Derived")
>        4$: DW_TAG_pointer_type
>                DW_AT_type(reference to $7 "DerivedTwo")
>        5$: DW_TAG_structure_type
>                DW_AT_name("Base")
>                ...
>        6$: DW_TAG_structure_type
>                DW_AT_name("Derived")
>                ...
>        7$: DW_TAG_structure_type
>                DW_AT_name("DerivedTwo")
>                ...
>
>        ! .debug_loclists section.
>        $9: DW_LLE_start_end[PC1 .. PC2) ! Then-block range.
>                DW_OP_reg0
>                DW_OP_refined_type(reference to $3 "Derived *")
>            DW_LLE_start_end[PC3 .. PC4) ! Else-block range.
>                DW_OP_reg0
>                DW_OP_refined_type(reference to $4 "DerivedTwo *")
>            DW_LLE_end_of_list
>
>    Consider the C++ example where a variable resides in memory but
>    within a certain range of PCs a member of a structure is promoted
>    to a register and the compiler is able to prove that the range of
>    values allow it to reside within 8 bits.
>
>        class foo {
>        public:
>          unsigned int a, b;
>        };
>
>        1$: DW_TAG_base_type
>          DW_AT_byte_size   : 4
>          DW_AT_encoding    : 7	(unsigned)
>          DW_AT_name        : unsigned int
>        2$: DW_TAG_base_type
>          DW_AT_byte_size   : 1
>          DW_AT_encoding    : 7	(unsigned)
>          DW_AT_name        : unsigned char
>        3$: DW_TAG_class_type
>           DW_AT_name        : foo
>           DW_AT_byte_size   : 8
>           DW_TAG_member
>             DW_AT_name        : a
>             DW_AT_type        : <unsigned int>
>             DW_AT_data_member_location: 0
>             DW_AT_accessibility: 1	(public)
>           DW_TAG_member
>             DW_AT_name        : b
>             DW_AT_type        : <unsigned int>
>             DW_AT_data_member_location: 4
>             DW_AT_accessibility: 1	(public)
>        4$: DW_TAG_class_type
>           DW_AT_name        : __foo_refined_1
>           DW_AT_byte_size   : 8
>           DW_AT_artificial  : 1
>           DW_AT_abstract_origin : <class foo>
>           DW_TAG_member
>             DW_AT_name        : a
>             DW_AT_type        : <unsigned char>
>             DW_AT_data_member_location: 0
>             DW_AT_accessibility: 1	(public)
>           DW_TAG_member
>             DW_AT_name        : b
>             DW_AT_type        : <unsigned int>
>             DW_AT_data_member_location: 4
>             DW_AT_accessibility: 1	(public)
>
>        ! .debug_loclists section.
>        5$: DW_LLE_default_location
>                DW_OP_addr 0x100
>        6$: DW_LLE_startx_endx PC1 PC2
>                DW_OP_addr 0x100
>                DW_OP_reg1
>                DW_OP_lit0 DW_OP_lit1
>                DW_OP_overlay
>                DW_OP_refine_type <class __foo_refined_1>
>
