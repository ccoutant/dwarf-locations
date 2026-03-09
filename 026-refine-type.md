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

A better option to address this situation is to have the producer use
DW_OP_regval_type so long as the variable is in a register. However,
if one of these extended floating point registers representing a
smaller type is spilled to memory without conversion back to a 64b
double then the problem still exists.

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

## Other uses

There are other cases where the producer can assist the consumer by
providing more precise type information.

### Derived types

One example could be when a compiler knows the type more precisely
than the variable's type natively indicates. For eample: `BaseV *bv2 =
new DerivedV;` adding more precise type information could allow the
consumer to print the full contents of the variable rather than just
the parts included in the base class. Some consumers can already do
this by referring to the vtable to identify the derived type but
having the actual known type which is known by the producer makes this
simpler.

It also allows a consumer handle situations which are currently not
handled such as `(gdb) p func(*bv2)` when `int func(Derived arg);`
without the user having to cast the variable to its known type.

### void pointers with known types

Another potential use is to give the objects pointed to by void
pointers their type. For example given `int x; void *generic=&x` the
compiler knows that object pointed to by generic is an int.

## Proposed fix

Add a new operator `DW_OP_refine_type` which tells the consumer that
object at the location specified should be interpreted as the
following type.

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
a DWARF expression can change the type. This includes portions which
are called from other DWARF expressions.

Some members of the GPU team were concerned that this would force all
DWARF expressions to have a type and that this type was not limited to
base types. Many DWARF operations are currently only defined for base
types to allow the implementaiton of consumers to be more practically
feasable. Since the `DW_OP_refine_type` operator is only defined for
locations, there is no problem with backward compatibility with
DWARF5. Furthermore, operators which are currently limited to values
and base types do not need to be changed.

## Alternatives considered

### Redefine DW_OP_reinterpret

Remove the restrictions on `DW_OP_reinterpret` that requires the type
to be a base type and the one that specifies that "The type of the
operand and result type must have the same size in bits". Then specify
that when the top of the stack is a location, then the object pointed
to by that location is understood to have changed type.

It was thought that this change would be backward compatible with
DWARF5 and earlier because all previous uses of DW_OP_reinterpret
would be subject to the bit size limitations. However, we discovered
an incompatibilty.

    DW_OP_addr 0xf00
	DW_OP_reinterpret <type>

In DWARF 5 `DW_OP_reinterpret` would pop a generic-type 0xf00 value
and reinterpret it as <type>. However, in DWARF6 with locations on the
stack `DW_OP_addr` pushes a location. Then the object at that location
is interpreted as a new type. In one case 0xf00 is treated as an
object which has the specified type. In the other case, whatever is
found at 0xf00 is understood to have the specified type. It was this
behavior that hilighted the problem with confusion between the type of
a value on the stack and the type of an object pointed to by a
location on the stack.

### Type lists

An other alternative that we considered was to create a type list
similar to a location list except instead of providing locations it
provided reference to type DIEs.

In most of the examples that we considered, we realized that the
change in type was highly correlated with changes in locations. For
example moving a double to an extended precision floating point
register changes both its location and type. We dropped this approach
when we considered the overhad of having to encode the bounds of the
range of PCs for both the type and location's bounds. We felt that
there were very few cases where the location would stay the same but
the type of the object would change.

### Annotate location list entries

A refinement of the type list approach that we also seriously
considered is to to annotate location list entries with type
information for that location. We would do this by prefixing entries
with `DW_LLE_refined_type` followed by a reference to the type DIE
then the normal location list entry.

    DW_LLE_refined_type( DIE reference to new type)
    DW_LLE_start_end[PC1 .. PC2) DW_OP_reg0

For variables which do not have a location list we would also add
`DW_AT_refined_type`.

One disadvatage of this approach is that the type of an expression can
only be refined at the end of evaluating a location expression. This
would prevent the type change from being factored into portions of
expressions which are called. It was limitations similar to this which
led to locations on the stack, therefore some members of the GPU team
were concerned by that limitation. However, this approach was favored
by a minority of the GPU team and so this approach is included in its
entirety as a alternative for the committee to consider.

## PROPOSAL

In section 3.4 after the description of DW_OP_regval_type add the
following non-nomaltive paragraph:

>    *When the compiler changes the type of a variable and its
>    location is stored in a register, this operation can be used to
>    inform the consumer that its type has changed. One such example
>    would be when a double float is promoted to an extended precision
>    floating point number by copying it to an extended precision
>    floating point register.*

In section 3.16 Type Conversions

After DW_OP_reinterpret add a new operator:

>    DW_OP_refine_type([ULEB] DIE offset for type)
>    <[location] A > → <[location] A with specified type>
>
>    The `DW_OP_refine_type` operation pops the location from the top
>    stack entry. It then changes the apparent type of the object
>    found at that location, in effect informing the consumer that the
>    type has been changed by the compiler. It takes one operand,
>    which is an ULEB integer that represents the offset of a
>    debugging information entry for a type in the current compilation
>    unit.

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
>                DW_AT_location( DW_OP_reg0;
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
>
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
>        1$: DW_TaAG_base_type
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
>                DW_OP_overlay DW_OP_refine_type <class __foo_refined_1>
>

## Alternative proposal

To the end of Section 2.6 "Types of Program Entities", add the
following paragraph:

>    A debugging information entry with a `DW_AT_type` attribute may
>    optionally have a `DW_AT_refined_type` attribute to specify a
>    runtime type that is different from the declared type.
>
>    *Refined type information may be provided, for example, when the
>    producer was able to find out that an object has a particular
>    type that is a subtype of the declared type throughout the
>    lifetime of the object, such as a pointer variable declared as a
>    pointer to a base class always pointing to a derived class
>    instance at runtime.  Refined type information may also be
>    provided to describe type changes due to optimizations, such as a
>    64-bit variable being narrowed to 32-bit as a result of value
>    range analysis.*
>
>    If an object has a refined type during a certain range of PCs
>    rather than its whole lifetime, a `DW_LLE_refined_type` entry can
>    be used in the location list of the object, instead of a
>    `DW_AT_refined_type` attribute.  See Section 3.19 "Location
>    Lists".

In Section 3.19 "Location Lists", before the bullet item "End-of-list", add
the following new bullet item:

>    * Refined type.  If the object whose location is being described
>      has a type that is different from its declared type for a
>      particular address range, the object is said to have a refined
>      type.  This kind of entry describes the refined type of the
>      object within the address range specified by the subsequent
>      location expression.  (Also see `DW_AT_refined_type`, Section
>      2.6.)

To the end of the paragraph

>    A location list consists of a sequence of zero or more bounded
>    location expression or base address entries, optionally followed
>    by a default location entry, and terminated by an end-of-list
>    entry.

add the following:

>    Bounded or default location entries may optionally have
>    preceeding refined type entries.

After the "target address operand" bullet item, add the following new
bullet item:

>    * A *refined type* operand that is a `reference` to a debugging
>      information entry describing a type.

After the `DW_LLE_include_loclistx` bullet item, add the following new
bullet item:

>    8. *DW_LLE_refined_type*
>
>       This is a form of *refined type* entry that has one operand,
>       which is a `reference` to a debugging information entry that
>       describes a type.  The referenced type is then the refined
>       (i.e.  runtime) type of the object within the address range
>       specified by the subsequent location expression entry, which
>       may be bounded or default.

In Table 8.5 "Attribute encodings", add a new row:

>    DW_AT_refined_type | 0x... | `reference`

In Table 8.10 "Location list entry encoding values", add a new row:

>    DW_LLE_refined_type | 0x0b

In Table A.1 "Attributes by tag value", add `DW_AT_refined_type` as
an applicable attribute to the following TAG values:

>    DW_TAG_formal_parameter
>    DW_TAG_subprogram
>    DW_TAG_variable

Add new example section to the appendix:

>    D.N Refined Type Examples
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
>                DW_AT_refined_type(reference to 3$ "Derived *")
>                DW_AT_location(...)
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
>
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
>        $9: DW_LLE_refined_type(reference to $3 "Derived *")
>            DW_LLE_start_end[PC1 .. PC2) ! Then-block range.
>                ...
>            DW_LLE_refined_type(reference to $4 "DerivedTwo *")
>            DW_LLE_start_end[PC3 .. PC4) ! Else-block range.
>                ...
>            ...
>            DW_LLE_end_of_list
