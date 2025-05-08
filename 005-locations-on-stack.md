Location Descriptions on the DWARF Stack
========================================

Background
----------

The DWARF 5 concept of location descriptions (Section 2.6) limits their
use to cases where the location described is final, and not subject to
some further modification, with two exceptions. First, if the location
description is a memory location description, it is a simple DWARF
expression (Section 2.5) that can be modified by further DWARF
expression operators. Second, for any form of location description, it
can be offset by a fixed number of bits by using a `DW_OP_bit_piece`
composition operator.

Where do these limitations matter?

Consider the case of a FORTRAN array (as shown in Appendix D in D.2.1)
that has been partially promoted to a register or registers. The
evaluation of its lower and upper bounds depends on the location of the
array as provided by `DW_OP_push_object_address`. If the array is not
entirely in memory, this operation is not able to provide the address of
the object, as it can only provide a memory address. If
`DW_OP_push_object_address` were allowed to push a composite location
description on the stack, we could apply further operations to locate
the bounds of the array.

Similarly, consider the case of a pointer-to-member type in C++, where
the object (or part of the object) has been promoted to a register. In
this case, `DW_AT_use_location` is not able to provide the address of
the object. In optimized code, it sometimes would need to provide a
register location description or a composite location description, but
these cannot be pushed onto the DWARF stack. If `DW_AT_use_location`
were allowed to push a composite location description on the stack, we
could apply further operations to determine the register location of the
member being referenced.

Also consider the case where a `DW_OP_call*` operator is used to get the
location of a variable. If the variable happens to be in a register at
the current PC, the call operator cannot succeed, as it cannot push
anything but a memory address on the stack.

All of these cases have a common limiting factor: that location
descriptions cannot be pushed onto the stack, and subsequently operated
on to produce derived location descriptions.


Overview
--------

This proposal removes that limitation. A DWARF expression may evaluate
to either a value or a location. Location descriptions are now simply
DWARF expressions that evaluate to a location, and are now called
location expressions.

The DWARF stack is extended so that it can hold elements that are either
(typed) values or (single) locations. The operators in Section 2.6 that
previously defined register and implicit locations are now considered
part of a DWARF expression, and are no longer "terminal" in the sense
that they cannot be part of a larger expression.

The literal encoding operations, defined in Section 2.5.1.1, push values
onto the stack, except for `DW_OP_addr` and `DW_OP_addrx`, which push
memory locations. These latter two operations are moved to a new
section.

Stack operations, defined in Section 2.5.1.3, can operate on values or
locations, or any combination of the two.

Most existing arithmetic and logical operators, defined in Section 2.5.1.4,
continue to be limited to operating on values only.

The `DW_OP_deref*` operator is extended to operate on
any location, and provide the value contained at that
location, whether in memory, in a register, in implicit storage, or a
composite value.

The `DW_OP_push_object_address`, renamed `DW_OP_push_object_location` operator pushes a location,
which may be a memory address (as before), or a register, implicit
storage, or a composite.

The `DW_AT_use_location` attribute provides an expression used to compute
the address of a member for a pointer-to-member type, and expects the
evaluation mechanism to provide the value of the pointer and the
location of the object as implicitly-pushed elements on the stack. The
latter element is now allowed to be any location.

Two new operators, `DW_OP_offset` and `DW_OP_bit_offset`, are introduced
that allow a location on the stack to be modified by a byte
or a bit offset.

The composite location operators, `DW_OP_piece` and `DW_OP_bit_piece`,
are redefined to build up a composite location, which is held in the top
element of the stack. A new operator, `DW_OP_composite`, is added
to begin a new (empty) composite location.

The `DW_OP_call*` operators are now allowed to leave a location
on the stack.

A new Section 2.5 "Values and Locations" is added, and the old Sections
2.5, "DWARF Expressions," and 2.6, "Location Descriptions," are moved into a
new Chapter 3, "DWARF Expressions," and reorganized as follows:

- (Remove) 2.5 DWARF Expressions
- (Remove) 2.6 Location Descriptions

- (New) 2.5 Values and Locations

- (New) Chapter 3: DWARF Expressions
    - 3.1 DWARF Expression Evaluation Context (was 2.5.1)
    - 3.2 Stack Operations (was 2.5.2.3)
    - 3.3 Literal and Constant Operations (was 2.5.2.1)
    - 3.4 Register Value Operations (was 2.5.2.2)
    - 3.5 Arithmetic and Logical Operations (was 2.5.2.4)
    - 3.6 Context Query Operations (new)
    - 3.7 Memory Locations (was 2.6.1.1.2)
    - 3.8 Register Locations (was 2.6.1.1.3)
    - 3.9 Undefined Locations (was 2.6.1.1.1)
    - 3.10 Implicit Locations (was 2.6.1.1.4)
    - 3.11 Implicit Pointer Locations (was 2.6.1.1.4)
    - 3.12 Composite Locations (was 2.6.1.2)
    - 3.13 Dereferencing Operations (new)
    - 3.14 Offset Operations (new)
    - 3.15 Control Flow Operations (was 2.5.2.5)
    - 3.16 Type Conversions (was 2.5.2.6)
    - 3.17 Special Operations (was 2.5.2.7)
    - 3.18 Value Lists (was 2.5.2)
    - 3.19 Location Lists (was 2.6.2)


Proposed Changes
----------------

### Section 2.2 Attribute Types

In Table 2.3, Classes of attribute value, in the rows for "exprval" and
"locdesc," replace the reference to Sections 2.5 and 2.6 with
references to Chapter 3. In the row for "locdesc," change "locdesc" to
"exprloc" and change "location description" to "location expression"
(2 places).

Rename class `locdesc` to `exprloc` throughout the document.


### Section (OLD) 2.5 DWARF Expressions [REMOVED]

This section is moved from Chapter 2 into a new Chapter 3.


### Section (OLD) 2.6 Location Descriptions [REMOVED]

This section is moved from Chapter 2 into a new Chapter 3.


### Section (NEW) 2.5 Values and Locations [NEW]

Add the following as a new subsection:

> As described in Section 2.2, DWARF attributes may have
> different classes of values. In many cases, attribute values
> directly provide the numeric value of a property such as
> bounds of an array, the size of a type, the value of a
> named constant in the program, or a string value such as
> the name of a variable or type.
> In other cases, they provide the *location* of a data
> object, such as the address of a variable or common block
> (see Section 4.1), instead of directly providing the value
> of the object.
> 
> The attribute value classes *constant* and *string* are used
> to provide static values for properties, but often the
> values need to be computed dynamically. The classes
> *exprval* and *vallist* indicate that the attribute value is
> to be computed by evaluating a DWARF expression (described
> in Chapter 3).
> 
> A DWARF expression of class *exprval* can be used to compute
> a value that cannot be given statically, such as the upper
> bound of a dynamic array.
> 
> Sometimes, an attribute value may depend on the program
> counter, such as the loop vectorization factor. A value list
> (see Section 3.17), encoded using class *vallist*, can be
> used in this case. A value list is a list of DWARF
> expressions, each associated with a range of program
> counters.
> 
> The class *address* is used to provide a static
> memory address for an object located in memory, and the
> classes *exprloc* and *loclist* indicate that the location
> of an object is to be computed by evaluating a DWARF
> expression. In these cases, where the DWARF expression is
> expected to produce a location, the expression is called a
> “location expression.”
> 
> *(In DWARF 5, the term “location description” was used to
> refer both to a sequence of DWARF operations that evaluates
> to a location, and to the result of that evaluation. For
> DWARF 6, we use the terms “location expression” for the
> expression and “location” for the result of
> evaluating the expression.)*
> 
> The class *exprloc* is used to provide a location
> expression that yields a single location. This is
> sufficient to describe the location of an object whose
> lifetime is either static or the same as the lexical block
> that owns it (excluding any prologue or epilogue ranges),
> and that does not move during its lifetime.
> 
> A location list (see Section 3.18), encoded using class
> *loclist*, describes objects that have a limited lifetime
> or that change their location during their lifetime. A
> location list is a list of location expressions, each
> associated with a range of program counters.
> 
> A location list may have overlapping PC ranges, and thus may
> yield multiple locations at a given PC.
> In this case, the consumer may assume that the object value
> stored is the same in all locations, excluding bits of the
> object that serve as padding. Non-padding bits that are
> undefined (for example, as a result of optimization) must be
> undefined in all the locations.
> 
> *A location list that yields multiple locations can be
> used to describe objects that reside in more than one
> piece of storage at the same time. An object may have more
> than one location as a result of optimization. For
> example, a value that is only read may be promoted from
> memory to a register for some region of code, but later
> code may revert to reading the value from memory as the
> register may be used for other purposes.*
> 
> DWARF can describe the location of program objects in
> several kinds of storage, such as a memory address space or
> a register. Each kind of storage is treated as a linear
> stream of bits of finite size, organized into bytes and/or
> words. The ordering of bits uses the bit numbering and
> direction conventions that are appropriate to the target
> architecture and the ABI.
> *For example, on a little-endian architecture, bits are
> numbered within bytes and words from least-significant
> to most-significant (right-to-left), while on a big-endian
> architecture, bits are numbered left-to-right.*
> 
> A location names a block of storage and a
> (non-negative, zero-based) bit offset relative to the start
> of that storage. It gives the location of the program
> object, which occupies a sequence of contiguous bits
> starting at that location.
> The bit offset of a location must be less than the size of
> the named block of storage.
> 
> DWARF can describe locations in six kinds of storage:
> 
>   - Memory.
>     Corresponds to the target architecture memory address
>     spaces. There is always a default address space, and the
>     target architecture may define additional address
>     spaces.
>     *(The offset can be thought of as a byte or word address
>     combined with a bit offset within the byte or word.)*
> 
>   - Registers.
>     Corresponds to the target architecture registers.
> 
>   - Undefined storage.
>     Indicates no value is available, as when a variable has
>     been optimized out. The location names a block of
>     imaginary storage, whose size is that of the largest
>     memory address space, that cannot be read from or
>     written to.
> 
>   - Implicit storage.
>     Corresponds to fixed values that can only be read, as when
>     a variable has been optimized out but can nevertheless be
>     rematerialized from program context. The location
>     names a block of imaginary storage, whose size is
>     the same as the fixed value that it holds.
>     
>   - Implicit pointer storage.
>     A special form of implicit storage, created by the
>     `DW_OP_implicit_pointer` operator (see Section 3.10).
>     The location names a block of imaginary storage that
>     holds a secondary location. Its size is the size of a
>     pointer in the default memory address space, but no
>     physical pointer is available.
> 
>   - Composite storage.
>     A hybrid form of storage where different pieces of a
>     program object map to different locations, as when a
>     field of a structure or a slice of an array is promoted
>     to a register while the rest remains in memory.


### Chapter 3 DWARF Expressions [NEW]

Replace the contents of the preamble to the old Section 2.5 with:

> DWARF expressions describe how to compute a value or specify a
> location. They are expressed in terms of DWARF operations that
> operate on a stack of elements, each of which may be
> either a value or a location.
> 
> A DWARF expression is encoded as a stream of operations, each
> consisting of an opcode followed by zero or more literal operands. The
> number of operands is implied by the opcode.
> 
> The result of a DWARF expression is the value or location on the top
> of the stack after evaluating the operations.
>
> Values on the stack are typed, and can represent a value
> of any supported base type of the target machine, or of
> the generic type, which is an integral type that has the
> size of an address in the default address space on the
> target machine, and unspecified signedness.
>
> An implicit conversion between a memory location and a value
> may happen during the execution of any operation or when
> evaluation of the expression is completed. If a location is
> expected, but the result is a value, the value is implicitly
> treated as a memory address in the default address space,
> and converted to a memory location. If a value is expected,
> and the result is an addressable memory location (i.e., if
> the offset is a multiple of the byte or word size) in the
> default address space, the address is implicitly converted
> to a value of the generic type.


### Section 3.1 DWARF Expression Evaluation Context [was 2.5.1]

Move the contents of the old Section 2.5.1 DWARF Expression Evaluation
Context here.

In the first paragraph, remove "and location descriptions (see Section ...)".

In item 1, "Required result kind," 2nd paragraph, change "location
description" to "location".

In item 8, "Current program counter (PC)," 5th paragraph, change "only
default location descriptions may be used" to "only default value or location
list entries may be used."

In item 9, "Current object," 2nd paragraph, change "and by some attributes"
to "and is implicitly defined by some attributes."

In the final (non-normative) paragraph of the section, change "A DWARF expression
for a location description..." to "A DWARF expression...."


### Section 3.2 Stack Operations [was 2.5.2.3 Stack Operations]

Move the contents of 2.5.2.3 Stack Operations here.
Change the first sentence to:

> The following operations manipulate the DWARF stack, and may
> operate on both values and locations.... [remainder of paragraph unchanged]

Change the second paragraph to:

> Each entry on the stack is either a value (with an associated type) or
> a location.

Include the descriptions for the following operations:

- `DW_OP_dup`
- `DW_OP_drop`
- `DW_OP_pick`
- `DW_OP_over`
- `DW_OP_swap`
- `DW_OP_rot`

For the above operations, remove all occurrences of "including its type identifier".

For `DW_OP_dup`, change the description to:

> The `DW_OP_dup` operation duplicates the entry at the top of the stack.

For `DW_OP_drop`, change the description to:

> The `DW_OP_drop` operation pops the entry at the top of the stack.


The following operations that were in section 2.5.2.3 are moved to other sections:

- `DW_OP_deref` (to 3.13 Dereferencing Operations)
- `DW_OP_deref_size` (to 3.13 Dereferencing Operations)
- `DW_OP_deref_type` (to 3.13 Dereferencing Operations)
- `DW_OP_xderef` (to 3.13 Dereferencing Operations)
- `DW_OP_xderef_size` (to 3.13 Dereferencing Operations)
- `DW_OP_xderef_type` (to 3.13 Dereferencing Operations)
- `DW_OP_form_tls_address` (to 3.6 Context Query Operations)
- `DW_OP_call_frame_cfa` (to 3.6 Context Query Operations)
- `DW_OP_push_object_address` (to 3.6 Context Query Operations)
- `DW_OP_push_lane` (to 3.6 Context Query Operations)


### Section 3.3 Literal and Constant Operations [was 2.5.2.1 Literal Encodings]

Rename and place the contents of old section 2.5.2.1 here.

Include the descriptions of the following operations:

- `DW_OP_lit0`, ..., `DW_OP_lit31`
- `DW_OP_const1u`, etc.
- `DW_OP_const1s`, etc.
- `DW_OP_constu`
- `DW_OP_consts`
- `DW_OP_constx`
- `DW_OP_const_type`

For `DW_OP_constx`, change "size of a machine address"
to "size of the generic type."

For `DW_OP_const_type`, insert "a" after "The second operand is".

The following operations that were in 2.5.2.1 are moved to Section 3.7 Memory Locations:

- `DW_OP_addr`
- `DW_OP_addrx`


### Section 3.4 Register Value Operations [was 2.5.2.2]

Place the contents of old section 2.5.2.2 here.

Replace the first paragraph with the following:

> The following operation pushes the contents of a
> register onto the stack.

Include the descriptions of the following operations:

- `DW_OP_regval_type`

Remove the `DW_OP_regval_bits` operator, which was added in DWARF 6
by Issue [201007.1][201007.1]. With the ability to dereference
register locations, this operator is not needed.

[201007.1]: https://dwarfstd.org/issues/201007.1.html

The following operations that were in 2.5.2.2 are moved to other sections:

- `DW_OP_fbreg` (to 3.6 General Location Operations)
- `DW_OP_breg0`, ..., `DW_OP_breg31` (to 3.7 Memory Locations)
- `DW_OP_bregx` (to 3.7 Memory Locations)


### Section 3.5 Arithmetic and Logical Operations [was 2.5.2.4]

Place the contents of old section 2.5.2.4 here.

_Remove_ the second paragraph:

> <span class="del">If the type of the operands is the generic type,
> except as otherwise specified, the arithmetic operations
> perform addressing arithmetic, that is, unsigned arithmetic that is performed
> modulo one plus the largest representable address.</span>

Include the descriptions of the following operations:

- `DW_OP_abs`
- `DW_OP_and`
- `DW_OP_div`
- `DW_OP_minus`
- `DW_OP_mod`
- `DW_OP_mul`
- `DW_OP_neg`
- `DW_OP_not`
- `DW_OP_or`
- `DW_OP_plus`
- `DW_OP_plus_uconst`
- `DW_OP_shl`
- `DW_OP_shr`
- `DW_OP_shra`
- `DW_OP_xor`


### Section 3.6 Context Query Operations [NEW]

Insert the following into this new section:

> The following operations can be used to push a value
> or location obtained from the expression evaluation context
> (see Section 3.1) onto the stack:
> 
> 1. `DW_OP_push_object_location` [moved from section 2.5.2.3 and renamed]
>
>     The `DW_OP_push_object_location` operation pushes the
>     location of the current object (see section 3.1) onto the
>     stack, as part of evaluation of a user-presented
>     expression.
>  
>     *This object may correspond to an independent
>     variable described by its own debugging information entry; or it may be a
>     component of an array, structure, or class whose address has been
>     dynamically determined by an earlier step during user expression
>     evaluation.*
>  
>     *This operator provides explicit functionality (especially
>     for arrays involving descriptors) that is analogous to the implicit push
>     of the base address of a structure prior to evaluation of a
>     `DW_AT_data_member_location` to access a data member of a structure. For
>     an example, see Appendix D.2 on page 304.*
>  
>     *In previous versions of DWARF, this operator was named `DW_OP_push_object_address`.
>     The old name is still supported in DWARF 6 for compatibility.*
>
> 2. `DW_OP_form_tls_location` [moved from section 2.5.2.3]
>
>     The `DW_OP_form_tls_location` operation pops a value from the stack,
>     which must have an integral type, translates this value into a location
>     in the thread-local storage for the current thread (see Section
>     3.1), and pushes the location onto the stack.
>     The meaning of the value.... [remainder of paragraph unchanged]
>  
>     *Some implementations of C, C++, Fortran, and other languages,
>     support a thread-local storage class. Variables with this storage
>     class have distinct values and addresses in distinct threads, much
>     as automatic variables have distinct values and addresses in each
>     function invocation. Typically, there is a single block of storage
>     containing all thread-local variables declared in the main
>     executable, and a separate block for the variables declared in each
>     shared library. Each thread-local variable can then be accessed in
>     its block using an identifier. This identifier is typically an
>     offset into the block and pushed onto the DWARF stack by one of the
>     `DW_OP_const<n><x>` operations prior to the
>     `DW_OP_form_tls_location`
>     operation. Computing the address of the appropriate block can be
>     complex (in some cases, the compiler emits a function call to do
>     it), and difficult to describe using ordinary DWARF location
>     expressions. Instead of forcing complex thread-local storage
>     calculations into the DWARF expressions, the
>     `DW_OP_form_tls_location` operation
>     allows the consumer to perform the computation based on the run-time
>     environment.*
>  
>     *In previous versions of DWARF, this operator was named `DW_OP_form_tls_address`.
>     The old name is still supported in DWARF 6 for compatibility.*
> 
> 3. `DW_OP_call_frame_cfa`... [moved unchanged from section 2.5.2.3]
>
> 4. `DW_OP_push_lane`... [moved unchanged from section 2.5.2.3]

Elsewhere in the document, change `DW_OP_push_object_address`
to `DW_OP_push_object_location`, and `DW_OP_form_tls_address`
to `DW_OP_form_tls_location`

### Section 3.7 Memory Locations [adapted from 2.6.1.1.2]

Insert the following:

> A memory location represents the location of a piece or all of an
> object or other entity in memory. On architectures that support
> multiple address spaces, a memory location contains a component that
> identifies the address space.
> 
> In contexts that expect a location, a value of the generic type
> will be implicitly converted to a memory location in the default
> address space.
> 
> The following operations push memory locations onto the stack:
> 
> 1. `DW_OP_addr` [moved from section 2.5.2.1]
>
>     The `DW_OP_addr` operation has a single operand that encodes a
>     machine address and whose size is the size of an address on the
>     target machine. The value of this operand is treated as an address
>     in the default address space and the corresponding memory location is
>     pushed onto the stack.
> 
> 2. `DW_OP_addrx` [moved from section 2.5.2.1]
>
>     The `DW_OP_addrx` operation has a single operand that encodes an
>     unsigned LEB128 value, which is a zero-based index into the `.debug_addr`
>     section, where a machine address is stored. This index is relative to the
>     value of the `DW_AT_addr_base` attribute of the associated compilation
>     unit. The address obtained is treated as an address in the default address
>     space and the corresponding memory location is pushed onto the stack.
> 
> 3. `DW_OP_fbreg`... [moved from section 2.5.2.2]
>
>     The `DW_OP_fbreg` operation provides a signed LEB128 byte offset `B` from
>     the location specified by the location expression in the
>     `DW_AT_frame_base` attribute of the current function (see Section 3.1).
>     The frame base location, offset by `B` bytes, is pushed onto
>     the stack.
>
>     *This is typically a stack pointer register plus or minus some offset.*
> 
> 4. `DW_OP_breg0`, ..., `DW_OP_breg31` [moved from section 2.5.2.2]
>
>     The single operand of the `DW_OP_breg<n>` operations provides a signed
>     LEB128 byte offset. The contents of the specified register (0–31) are
>     treated as a memory address in the default address space. The offset is
>     added to the address obtained from the register and the resulting memory
>     location is pushed onto the stack.
> 
> 5. `DW_OP_bregx` [moved from section 2.5.2.2]
>
>     The `DW_OP_bregx` operation has two operands. The first
>     operand is a register number which is specified by an unsigned LEB128
>     number. The second operand is a signed LEB128 byte offset. It is the same as
>     `DW_OP_breg<n>` except it uses the register and offset provided by the
>     operands.


### Section 3.8 Register Locations [adapted from 2.6.1.1.3]

Place the contents of old section 2.6.1.1.3 here.

_Remove_ the first paragraph:

> <span class="del">A register location consists of a register name operation, which
> represents a piece or all of an object located in a given register.</span>

_Remove_ the last sentence of non-normative text that follows:

> <span class="del">_A register location description must stand alone as the entire
> description of an object or a piece of an object._</span>

Include the descriptions of the following operations:

- `DW_OP_reg0`, ..., `DW_OP_reg31`
- `DW_OP_regx`

For `DW_OP_reg<n>`, replace

> The object addressed is in register _n_.

with:

> A location that names the designated register,
> with an offset of 0, is formed and pushed on the stack.

For `DW_OP_regx`, add the sentence:

> A location that names the designated register,
> with an offset of 0, is formed and pushed on the stack.

Replace the non-normative paragraph at the end with the following:

> *These operations name a register, not the contents of the register.
> To fetch the contents of a register, it is necessary to use
> one of the register based addressing operations, such as
> `DW_OP_bregx` (Section 3.7),
> the register value operation
> `DW_OP_regval_type` (Section 3.4),
> or a `DW_OP_reg` operation followed by a derefencing operation
> (Section 3.13).*


### Section 3.9 Undefined Locations [adapted from 2.6.1.1.1]

Insert the following (adapted from Section 2.6.1.1.1):

> An undefined location represents a piece or all of an object that is
> present in the source but not in the object code (perhaps due to
> optimization).
> An undefined location cannot be read from or written to.

> 1. `DW_OP_undefined`
>
>     The `DW_OP_undefined` operation pushes an undefined
>     location with an offset of 0 onto the stack.

> 2. A DWARF expression containing no operations or
> that leaves no elements on the stack also produces an undefined
> location.
> *This is for compatibility with earlier versions of DWARF.*


### Section 3.10 Implicit Locations [adapted from 2.6.1.1.4]

Move the contents of Section 2.6.1.1.4 here, excluding `DW_OP_implicit_pointer`.

Move the last non-normative paragraph to the start of the section
and replace the term "location descriptions" with "location expressions":

> *DWARF location expressions are intended ...*

In the remainder of the section, replace the term
"location description" with "location" throughout.

In the first (normative) paragraph, change "but whose contents are nonetheless
either known or known to be undefined" to "but whose contents are
nonetheless known".

Include the descriptions of the following operations:

- `DW_OP_implicit_value`
- `DW_OP_stack_value`
- `DW_OP_implicit_pointer`

For `DW_OP_implicit_value`, add:

> A location `L` is formed for a block of implicit storage which contains
> the given byte sequence.
> The offset of `L` is set to 0, and `L` is pushed onto the stack.

For `DW_OP_stack_value`, replace:

> In this form
> of location description, the DWARF expression represents the
> actual value of the object, rather than its location.
> The `DW_OP_stack_value` operation terminates the expression.

with:

> The value `V` on top of the stack is popped, and
> a location for `L` is formed for a block of implicit storage
> containing the value `V`, represented using the encoding and byte order of
> the value's type. The offset of `L` is set to 0, and `L` is pushed onto the stack.


### Section 3.11 Implicit Pointer Locations [adapted from 2.6.1.1.4]

Move the description of `DW_OP_implicit_pointer` into this new
section as follows:

> *An optimizing compiler may eliminate a pointer, while
> still retaining the value that the pointer addressed.
> `DW_OP_implicit_pointer` allows a producer to describe
> <del>this</del> <ins>the latter</ins> value.*
>
> An implicit pointer location is used to describe a pointer
> object that cannot be represented as a real pointer, even
> though the value it would point to can be described. It
> refers to a debugging information entry that represents the
> actual value of the object to which the pointer would point.
> Thus, a consumer of the debug information would be able to
> show the value of the dereferenced pointer, even when it
> cannot show the value of the pointer itself.
>
> The following DWARF operation is used to create an implicit
> pointer location:
>
> 1. `DW_OP_implicit_pointer`
>
>     The `DW_OP_implicit_pointer` operation has two operands: a
>     reference to a debugging information entry that describes
>     the dereferenced object's value, and a signed number that
>     is treated as a byte offset from the start of that value.
>     The first operand is a 4-byte unsigned value in the 32-bit
>     DWARF format, or an 8-byte unsigned value in the 64-bit
>     DWARF format (see Section
>     {32bitand64bitdwarfformats})
>     that is used as the offset of the debugging information entry
>     in the `.debug_info` section of the current executable
>     or shared object file.
>     The second operand, the byte offset, is a
>     signed LEB128 number.
>     
>     An implicit pointer storage location is created for the
>     location of the pointer object and is pushed onto the
>     stack.
>
>     *The debugging information entry referenced by a
>     `DW_OP_implicit_pointer` operation is typically a
>     `DW_TAG_variable` or `DW_TAG_formal_parameter` entry whose
>     `DW_AT_location` attribute gives a second DWARF expression or a
>     location list that describes the value of the object, but the
>     referenced entry may be any entry that contains a `DW_AT_location`
>     or `DW_AT_const_value` attribute (for example, `DW_TAG_dwarf_procedure`).
>     By using the second DWARF expression, a consumer can
>     reconstruct the value of the object when asked to dereference
>     the pointer described by the original DWARF expression
>     containing the `DW_OP_implicit_pointer` operation.*


### Section 3.12 Composite Locations [adapted from 2.6.1.2]

Insert the following (adapted from Section 2.6.1.2, and with the new
`DW_OP_composite` operator):

> A composite location represents the
> location of an object or value that does not exist in a
> single block of contiguous storage
> *(e.g., as the result of compiler optimization where part of the
> object is promoted to a register)*.
> It consists of
> a set of tuples of the form *(S, E, L)*, where each tuple
> represents a mapping from bits in the half-open range *[S,
> E)* of the program object onto a location *L*.
> The offset *D* in a composite location refers
> to a bit position within the program object, selects the
> mapping where *S ≤ D < E*, and translates to the location
> *L* with an additional offset of *D - S* applied. In a
> composite location, each component tuple
> represents a disjoint range of bits (i.e., the ranges may
> not overlap).
> The offset must be less than the largest value of *E*
> among the tuples.
> *In the process of fetching the value from a composite
> location, the consumer may need to reassemble bits from
> more than one of the mappings.*

> The piece operations (`DW_OP_piece` and `DW_OP_bit_piece`) in this
> section are used to build a composite location
> in storage order, one piece at a time.
> Each piece operation expects a location `LP` at the top of the stack,
> and a composite location `LC` in the preceding element
> on the stack. It pops those two locations and
> extends `LC` by adding a new mapping tuple *(SP, EP, LP)*, where *SP* is equal
> to the largest value of *E* from among the existing mapping tuples of `LC`,
> and *EP* is determined by the size operand of the piece operation.
> The extended location `LC` is pushed onto the stack.
> When adding a piece to an empty composite, *SP* is 0.
> 
> 1. `DW_OP_composite`
>
>     The `DW_OP_composite` operator has no operands. It pushes a new, empty,
>     composite location onto the stack, with an offset of 0.
>
>     *This operator is provided so that a new series of
>     piece operations can be started to form a composite location when
>     the state of the stack is unknown (e.g., following a `DW_OP_call`
>     operation), or when a new composite is to be started (e.g., rather
>     than add to a previous composite location on the stack).*
>
> 2. `DW_OP_piece`
>
>     The `DW_OP_piece` operation takes a single operand, which is an unsigned
>     LEB128 number. The number describes the size *N*, in bytes, of the piece
>     of the object to be appended to the composite `LC`.
>     The *EP* value of the new tuple is set to *SP + N × byte_size*.

>     If `LP` is a register location, but the piece does not
>     occupy the entire register, the placement of the piece
>     within that register is defined by the ABI.
>
> 3. `DW_OP_bit_piece`
>
>     The `DW_OP_bit_piece` operation takes two operands. The first is an
>     unsigned LEB128 number that gives the size *N*, in bits, of the piece to be appended.
>     The second is an unsigned LEB128 number that gives the offset in bits
>     to be applied to the location `LP`.
>     The *EP* value of the new tuple is set to *SP + L*.
>
>     Interpretation of the offset depends on the type of location. If the
>     location is an undefined location (see Section 3.9), the
>     `DW_OP_bit_piece` operation describes a piece consisting of the given
>     number of bits whose values are undefined, and the offset is ignored. If
>     the location is a memory location (see Section 3.7), the
>     `DW_OP_bit_piece` operation describes a sequence of bits relative to the
>     location whose address is on the top of the DWARF stack using the bit
>     numbering and direction conventions that are appropriate to the current
>     language on the target system. In all other cases, the source of the
>     piece is given by either a register location (see Section 3.8) or an
>     implicit value location (see Section 3.9); the offset is from the
>     least significant bit of the source value.
> 
>     *The `DW_OP_bit_piece` operator is used instead of `DW_OP_piece`
>     when the piece to be assembled into a value or assigned to is not
>     byte-sized or is not at the start of a register or addressable
>     unit of memory.*
>
>     *Whether or not a `DW_OP_piece` operation is
>     equivalent to any `DW_OP_bit_piece` operation with an offset of 0
>     is ABI dependent.*
>
> For compatibility with DWARF Version 5 and earlier, the following
> additional rules apply to piece operations:
>
> - If `LP` is a non-composite location (or convertible to
> one) and is the only element on the stack, the result
> is a new composite location with the single piece `LP` (as
> if `DW_OP_composite DW_OP_swap` had been processed
> immediately prior to the piece operation).
> *This rule supports the first piece operation in a DWARF 5 expression.*
>
> - If a piece operation is processed while the stack is empty, a new
> empty composite and an undefined location are pushed implicitly (as if
> `DW_OP_composite DW_OP_undefined` had been processed immediately prior
> to the piece operation). The result is a composite with a single
> undefined piece.
> *This rule supports the empty piece operation in DWARF 5 when it is the
> first piece of a composite.*
>
> - If the top of the stack is a composite location, and is
> the only element on the stack, an undefined location is
> pushed implicitly (as if `DW_OP_undefined` had been
> processed immediately prior to the piece operation),
> whereupon the composite on top of the stack is taken as
> `LC` and the undefined location is `LP`. The result is the
> addition of an undefined piece to the existing composite
> location.
> *This rule supports the empty piece operation in DWARF 5 when it is not the
> first piece of a composite.*
>
> - Otherwise, the DWARF expression is not valid.


### Section 3.13 Dereferencing Operations [NEW]

Add the following introductory paragraphs:

> The following operations are used to dereference a location
> on the stack; i.e., to load a value (or another location) stored at that location.
> Each operation defines the number of bytes or bits to be loaded.

> Dereferencing a memory location simply loads a value from memory.
> Dereferencing a register location extracts all or some of the bits from the register,
> depending on the offset of the location and the size of value to be loaded.
> Dereferencing an implicit location loads a value (or, in
> the case of an implicit pointer, another location) from
> the implicit storage. Attempting to dereference an
> undefined location is an error.
> 
> When dereferencing from a composite location, the value to be loaded
> may span more than one of the pieces of the composite, and the
> consumer must reassemble the value from the component pieces.

Include the descriptions for the following operations (moved from 2.5.2.3):

- `DW_OP_deref`
- `DW_OP_deref_size`
- `DW_OP_deref_type`
- `DW_OP_xderef`
- `DW_OP_xderef_size`
- `DW_OP_xderef_type`

For `DW_OP_deref`, change the description to:

> The `DW_OP_deref` operation pops a location `L` from the
> top of the stack. The first `S` bits, where `S` is the
> size (in bits) of an address on the target machine, are
> retrieved from the location `L` and pushed onto the stack
> as a value of the generic type.

For `DW_OP_deref_size`, change the description to:

> The `DW_OP_deref_size` takes a single 1-byte unsigned integral operand
> that specifies the size `S`, in bytes, of the value to be retrieved.
> The size `S` must be no larger than the size of the generic type. The
> operation behaves like the `DW_OP_deref` operation: it pops a location
> `L` from the stack. The first `S` bytes are retrieved from the
> location `L`, zero extended to the size of the generic type, and
> pushed onto the stack as a value of the generic type.

For `DW_OP_deref_type`, change the description to:

> The `DW_OP_deref_type` operation takes two operands. The first operand
> is a 1-byte unsigned integer that specifies the byte size `S` of the type
> given by the second operand. The second operand is an unsigned LEB128
> integer that represents the offset of a debugging information entry in
> the current compilation unit, which must be a `DW_TAG_base_type` entry
> that provides the type `T` of the value to be retrieved. The size `S`
> must be the same as the byte size of the base type represented by the
> type `T`. This operation pops a location `L` from the stack. The
> first `S` bytes are retrieved from the location `L` and pushed onto the
> stack as a value of type `T`.

And delete the non-normative paragraph following (this rationale
is appropriate for `DW_OP_const_type`, but not for `DW_OP_deref_type`):

> <span class="del">*While the size of the pushed value could be inferred from the base type
> definition, it is encoded explicitly into the operation so that the
> operation can be parsed easily without reference to the `.debug_info`
> section.*</span>

For `DW_OP_xderef` and `DW_OP_xderef_size`, change "integral type
identifiers" to "integral types," and "generic type identifier" to
"generic type."

For `DW_OP_xderef_type`, change "whose value value which is"
to "whose value is". [This was a typo in the DWARF 5 spec.]


### Section 3.14 Offset Operations [NEW]

Add:

> In addition to the composite operations, locations
> may be modified by the following operations:
> 
> 1. `DW_OP_offset`
> 
>     `DW_OP_offset` pops two stack entries. The first (top of stack)
>     must be an integral type value, which represents a signed byte
>     displacement. The second must be a location. It forms an updated
>     location by adding the given byte displacement to the offset
>     component of the original location and pushes the updated location
>     onto the stack.
> 
> 2. `DW_OP_bit_offset`
> 
>     `DW_OP_bit_offset` pops two stack entries. The first
>     (top of stack) must be an integral type value, which represents a signed bit
>     displacement. The second must be a location. It forms an updated
>     location by adding the given bit displacement to the offset
>     component of the original location and pushes the updated location
>     onto the stack.
> 
>     *A bit offset of `N × byte_size` is equivalent to a byte offset of `N`.*
>
> The resulting offset remain valid for the location.


### Section 3.15  Control Flow Operations [was 2.5.2.5]

Move the contents of old section 2.5.2.5 here.

Include the descriptions of the following operations:

- `DW_OP_le`, ... `DW_OP_ne`
- `DW_OP_skip`
- `DW_OP_bra`
- `DW_OP_call2`, `DW_OP_call4`, `DW_OP_call_ref`

For `DW_OP_bra`, change:

> If the value popped is not the constant 0, the 2-byte constant operand is ...

to

> If the value popped is not 0, the constant operand is ...

Under `DW_OP_call2`, etc., change:

> Execution of the DWARF expression of a `DW_AT_location` attribute may
> add to and/or remove from values on the stack. Execution returns to
> the point following the call when the end of the attribute is reached.
> Values on the stack at the time of the call may be used as parameters
> by the called expression and values left on the stack by the called
> expression may be used as return values by prior agreement between the
> calling and called expressions.

to:

> Execution of the DWARF expression of a `DW_AT_location` attribute may
> pop elements from the stack and/or push values or locations onto the
> stack. Execution returns to the point following the call when the end
> of the attribute is reached. Values and locations on the
> stack at the time of the call may be used as parameters by the called
> expression, and elements (values or locations) left on the stack by
> the called expression may be used as return values by prior agreement
> between the calling and called expressions.


### Section 3.16 Type Conversions [was 2.5.2.6]

Move and renumber Section 2.5.2.6 to here.

Include the descriptions of the following operations:

- `DW_OP_convert`
- `DW_OP_reinterpret`

Under `DW_OP_reinterpret`, in the next-to-last sentence, change
"converted" to "reinterpreted".


### Section 3.17 Special Operations [was 2.5.2.7]

Move and renumber Section 2.5.2.7 to here.

Include the descriptions of the following operations:

- `DW_OP_nop`
- `DW_OP_entry_value`
- `DW_OP_extended`
- `DW_OP_user_extended`

For `DW_OP_nop`, change "location stack" to "stack" and "values" to "elements".

For `DW_OP_user_extended`, in the last (non-normative) paragraph,
change "the standard" to "this specification".


### Section 3.18 Value Lists [was 2.5.2]

Place the contents of old section 2.5.2 Value Lists here.

*Remove* the fifth (non-normative) paragraph:

> <span class="del">*The DWARF expressions in value list entries, being
> expressions and not location descriptions, may not contain
> any of the DWARF operations described in Section
> {locationdescriptions}.*</span>

### Section 3.19 Location Lists [was 2.6.2]

Place the contents of old Section 2.6.2 Location Lists here.

Change the first sentence to:

> Location lists are used in place of location expressions whenever
> the object whose location is being described can change location
> during its lifetime.

In the second paragraph, change "a location or other attribute" to "an
attribute."

Remove the third (non-normative) paragraph.


### Section 5.7.6 Data Member Entries

In the description for `DW_AT_data_member_location`, change
the first paragraph of the second bullet to:

> 2\. Otherwise, the value must be a location expression. The
> location of the containing entity is pushed on the DWARF stack before
> the location expression is evaluated; the result of the evaluation is
> the location of the member entry (see Section 3.1).

Replace the following non-normative paragraph with:

> *The push on the DWARF expression stack of the location
> of the containing construct is equivalent to execution of the
> `DW_OP_push_object_location` operation (see Section 3.6);
> `DW_OP_push_object_location` therefore is not needed at the beginning of
> a location expression for a data member. The result of the evaluation
> is a location, not an offset to the member.*

_Remove_ the second non-normative paragraph:

> <span class="del">*A `DW_AT_data_member_location` attribute that has the
> form of a location description is not valid for a data member contained
> in an entity that is not byte aligned because DWARF operations do not
> allow for manipulating or computing bit offsets.*</span>

### Section 5.7.8 Member Function Entries

Replace the paragraph describing `DW_AT_vtable_elem_location` with:

> An entry for a virtual function also has a
> `DW_AT_vtable_elem_location` attribute whose value contains a
> location expression yielding the location of the slot for
> the function within the virtual function table for the
> enclosing class. The location of an object of the enclosing
> type is pushed onto the expression stack before the location
> expression is evaluated.

### Section 5.14 Pointer to Member Type Entries

In the paragraph beginning “The pointer to member entry has a
`DW_AT_use_location` attribute...”, replace “location description”
with “location expression.”

Replace the paragraph beginning “The `DW_AT_use_location` description...” with:

> The `DW_AT_use_location` expression is used in conjunction with the
> location expressions for a particular object of the given pointer to
> member type and for a particular structure or class instance. The
> `DW_AT_use_location` attribute expects two values to be pushed onto the
> DWARF expression stack before the `DW_AT_use_location` expression is
> evaluated. The first value pushed is the value of the pointer to member
> object itself. The second value pushed is the location of the
> entire structure or union instance containing the member whose location
> is being calculated.


### Section 6.4.1 Structure of Call Frame Information

For the "expression(E)" register rule, change the description to:

> The previous value of this register is located at the location produced
> by executing the DWARF expression E (see Chapter 3).


### Section 7.5.5 Classes and Forms

Replace the `locdesc` bullet with the following:

> - `exprloc`  
>   A DWARF location expression (see Section 2.5).
>   This is represented as an unsigned LEB128 length,
>   followed by a byte sequence of the specified length (`DW_FORM_expression`)
>   containing the location expression.

Rename `DW_FORM_exprloc` to `DW_FORM_expression`.

### Section 7.7 DWARF Expressions <span class="del">and Location Descriptions</span>

Rename Section 7.7 to "DWARF Expressions."

### Section 7.7.1 DWARF Expressions

Add to Table 7.9:

>                                 No. of
>     Operation             Code  Operands  Notes
>     --------------------  ----  --------  -----
>     DW_OP_offset          TBA      0
>     DW_OP_bit_offset      TBA      0
>     DW_OP_composite       TBA      0
>     DW_OP_undefined       TBA      0

### Section 7.7.2 Location Descriptions

Remove Section 7.7.2.

### Section D.1.3 DWARF Location Expression Examples [was DWARF Location Description Examples]

Change the name of the section. Also change "location descriptions" to
"location expressions" in the first paragraph.

Replace the seventh example (`DW_OP_plus_uconst`) with:

> `DW_OP_lit4`  
> `DW_OP_offset`
>
> > A structure member is four bytes from the start of the structure
> > instance. The base location is assumed to be already on the stack.

In the four composite examples that follow, add a `DW_OP_composite` operator
at the beginning of each sequence, and change `DW_OP_plus` to
`DW_OP_offset`.

In the sixth `DW_OP_entry_value` example, replace `DW_OP_plus_uconst (16)`
with `DW_OP_lit16` and `DW_OP_offset`.

### Section D.2.1 Fortran Simple Array Example

In Figure D.4, change `DW_OP_plus` to `DW_OP_offset` in the following
places:

- In the `DW_AT_associated` attribute at `1$`.

- In the `DW_AT_lower_bound` and `DW_AT_upper_bound` attributes at `2$`.

- In the `DW_AT_allocated` and `DW_AT_data_location` attributes at `6$`.

- In the `DW_AT_lower_bound` and `DW_AT_upper_bound` attributes at `7$`.


### Section D.2.3 Fortran 2008 Assumed-rank Array Example

[Page 302]
In Figure D.13, change `DW_OP_plus` to `DW_OP_offset` in the following
places:

- In the `DW_AT_rank` and `DW_AT_data_location` attributes at `10$`.

- In the `DW_AT_lower_bound` and `DW_AT_upper_bound` attributes at `11$`
(immediately following `DW_OP_push_object_location`).


[1]: https://llvm.org/docs/AMDGPUDwarfExtensionsForHeterogeneousDebugging.html
[2]: ../../doc/Issue-230524-1-diffs.html
