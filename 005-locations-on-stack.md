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
DWARF expressions that evaluate to a location.

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

The `DW_OP_push_object_address` operator pushes a location,
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

The contents of Section 2.5 and 2.6 are reorganized as follows:

- 2.5 DWARF Expressions
    - 2.5.1 Values and Locations
    - 2.5.2 Stack Operations
    - 2.5.3 Value Operations
        - 2.5.3.1 Literal Encodings
        - 2.5.3.2 Register Values
        - 2.5.3.3 Arithmetic and Logical Operations
    - 2.5.4 Location Operations
        - 2.5.4.1 Memory Locations
        - 2.5.4.2 Register Locations
        - 2.5.4.3 Implicit Locations
        - 2.5.4.4 Undefined Locations
        - 2.5.4.5 Composite Locations
        - 2.5.4.6 Offset Operations
    - 2.5.5 Control Flow Operations
    - 2.5.6 Type Conversions
    - 2.5.7 Special Operations
- 2.6 Expression Lists
  - 2.6.1 Value Lists
  - 2.6.2 Location Lists


Proposed Changes
----------------

### Section 2.2 Attribute Types

In Table 2.3, Classes of attribute value, in the row for "locdesc,"
replace the reference to Section 2.6 with a reference to Section 2.5.


### Section 2.5 DWARF Expressions

Replace the contents of the preamble to this section with:

> DWARF expressions describe how to compute a value or specify a
> location. They are expressed in terms of DWARF operations that
> operate on a stack. Each element on the stack may be
> either a value or a location.
> 
> A DWARF expression is encoded as a stream of operations, each
> consisting of an opcode followed by zero or more literal operands. The
> number of operands is implied by the opcode.
> 
> The result of a DWARF expression is the value or location on the top
> of the stack after evaluating the operations.

### Section 2.5.1 Values and Locations [NEW]

Add the following as a new subsection:

> A DWARF expression is evaluated in a context that determines whether
> its result is expected to be a value or a location. Expressions that are
> expected to produce a location are called "location descriptions."
> 
> Values on the stack are typed, and can represent a value of any
> supported base type of the target machine, or of the generic type,
> which is an integral type that has the size of an address on the
> target machine and unspecified signedness.
> 
> [non-normative] The generic type is the same as the unspecified type used for stack
> operations defined in DWARF Version 4 and before.
> 
> [non-normative] Debugging information must provide consumers a way to
> find the location of program variables, determine the bounds of dynamic
> arrays and strings, and possibly to find the base address of a
> subroutine’s stack frame or the return address of a subroutine.
> Furthermore, to meet the needs of recent computer architectures and
> optimization techniques, debugging information must be able to describe
> the location of an object whose location changes over the object’s
> lifetime.
> 
> Information about the location of program objects is provided by
> location descriptions. Location descriptions can be either of two forms:
> 
> - Single location descriptions, which are a language-independent
> representation of addressing rules of arbitrary complexity built from
> DWARF expressions. They are sufficient for describing the location of
> any object as long as its lifetime is either static or the same as the
> lexical block that owns it, excluding any prologue or epilogue ranges,
> and it does not move during its lifetime.
> 
> - Location lists, which are used to describe objects that have a
> limited lifetime or change their location during their lifetime.
> Location lists are described in Section 2.6.
> 
> Location descriptions are distinguished in a context-sensitive manner.
> As the value of an attribute, a single location description is encoded
> using class `locdesc`, and a location list is encoded using class `loclist`
> (which serves as an index into a separate section containing location
> lists).
> 
> A single location description specifies the storage bank that holds
> a program object and a position within that storage bank where the
> program object starts. The position within the storage bank is
> expressed as a bit offset relative to the start of the storage bank.
> 
> A storage bank is a linear stream of bits that can hold values. Each
> storage bank has a size in bits and can be accessed using a
> zero-based bit offset. The ordering of bits within a storage bank
> uses the bit numbering and direction conventions that are appropriate to
> the current language on the target architecture.
> 
> There are five kinds of storage banks:
> 
> - Memory storage.
>   Corresponds to the target architecture memory address spaces.
> 
> - Register storage.
>   Corresponds to the target architecture registers.
> 
> - Implicit storage.
>   Corresponds to fixed values that can only be read.
> 
> - Undefined storage.
>   Indicates no value is available and therefore cannot be read or written.
> 
> - Composite storage.
>   Allows a mixture of these where some bits come from one storage
>   bank and some from another storage bank, or from disjoint parts
>   of the same storage bank.
> 
> Implicit conversions between memory locations and values may happen
> during the execution of any operation or when evaluation of the
> expression is completed. If a location is expected, but the result is
> a value, the value is implicitly treated as a memory address in the
> default address space, and converted to a memory location. If a value
> is expected, but the result is an addressable memory location in the
> default address space, the address is implicitly converted to a value
> of the generic type.
> 
> [begin non-normative]
> 
> Location descriptions are a language independent representation of
> addressing rules.
> 
> They can be the result of evaluating a debugging information entry
> attribute that specifies an operation expression of arbitrary
> complexity. In this usage they can describe the location of an object
> as long as its lifetime is either static or the same as the lexical
> block that owns it, excluding any prologue or epilogue ranges, and it
> does not move during its lifetime.
> 
> They can be the result of evaluating a debugging information entry
> attribute that specifies a location list expression. In this usage they
> can describe the location of an object that has a limited lifetime,
> changes its location during its lifetime, or has multiple locations over
> part or all of its lifetime.
> 
> [end non-normative]  
> 
> If a location description yields more than one single location,
> the DWARF expression is ill-formed if the object value held in each
> single location’s position within the associated location
> storage is not the same value, except for the parts of the value that
> are uninitialized.
> 
> A multiple location can only be created by a location list that has
> overlapping program location ranges.
>
> A multiple location description can be used to describe objects that
> reside in more than one piece of storage at the same time. An object
> may have more than one location as a result of optimization. For
> example, a value that is only read may be promoted from memory to a
> register for some region of code, but later code may revert to reading
> the value from memory as the register may be used for other purposes.
> For the code region where the value is in a register, any change to
> the object value must be made in both the register and the memory so
> both regions of code will read the updated value.
>
> A consumer of a multiple location description can read the object’s
> value from any of the single location descriptions (since they all
> refer to storage that has the same value), but must write any
> changed value to all the single locations.

### Section 2.5.2 Stack Operations [was: 2.5.1.3 Stack Operations]

Move the contents of 2.5.1.3 Stack Operations to this new subsection.
Change the first sentence to:

> The following operations manipulate the DWARF stack, and may
> operate on both values and locations.

Change the second paragraph to:

> Each entry on the stack is either a value (with an associated type) or
> a location. When operating on a value, the associated type is always
> included with the value.

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

The following operations that were in section 2.5.1.3 are moved to other sections:

- `DW_OP_deref`, `DW_OP_deref_size`, `DW_OP_deref_type`,
  `DW_OP_xderef`, `DW_OP_xderef_size`, `DW_OP_xderef_type`, `DW_OP_form_tls_address`,
  `DW_OP_call_frame_cfa` (to 2.5.4.1 Memory Locations)
- `DW_OP_push_object_address`  (to 2.5.4 Location Operations)
- `DW_OP_push_lane` (to 2.5.3.1 Literal Encodings)


### Section 2.5.3 Value Operations [was: 2.5.1 General Operations]

In the first paragraph, delete all but the first sentence of the first
two paragraphs, leaving only this:

> Each value operation represents a postfix operation on a simple
> stack machine.

(The deleted sentences have been moved up to the introduction to Section 2.5.)


### Section 2.5.3.1 Literal Encodings [was: 2.5.1.1]

Place the contents of old section 2.5.1.1 here.

Include the descriptions of the following operations:

- `DW_OP_lit0`, ..., `DW_OP_lit31`
- `DW_OP_const1u`, etc.
- `DW_OP_const1s`, etc.
- `DW_OP_constu`
- `DW_OP_consts`
- `DW_OP_constx`
- `DW_OP_const_type`
- `DW_OP_push_lane` (moved from old section 2.5.1.3)

The following operations that were in 2.5.1.1 are moved to Section 2.5.4.1 Memory Locations:

- `DW_OP_addr`
- `DW_OP_addrx`


### Section 2.5.3.2 Register Values [was: 2.5.1.2]

Place the contents of old section 2.5.1.2 here.

Include the descriptions of the following operations:

- `DW_OP_regval_type`
- `DW_OP_regval_bits`

The following operations that were in 2.5.1.2 are moved to other sections:

- `DW_OP_fbreg` (to 2.5.4 Location Operations)
- `DW_OP_breg0`, ..., `DW_OP_breg31` (to 2.5.4.1 Memory Locations)
- `DW_OP_bregx` (to 2.5.4.1 Memory Locations)


### Section 2.5.3.3 Arithmetic and Logical Operations [was: 2.5.1.4]

Place the contents of old section 2.5.1.4 here.

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


### Section 2.5.4 Location Operations [NEW]

Insert the following into this new section:

> The following operations can be used to push a location onto the stack:
> 
> 1. `DW_OP_fbreg`... [moved unchanged from section 2.5.1.2]
> 
> 2. `DW_OP_push_object_address` [moved from section 2.5.1.3]  
> The DW_OP_push_object_address operation pushes the location of the
> object currently being evaluated as part of evaluation of a user
> presented expression. This object may correspond to an independent
> variable described by its own debugging information entry or it may be a
> component of an array, structure, or class whose address has been
> dynamically determined by an earlier step during user expression
> evaluation. This operator provides explicit functionality (especially
> for arrays involving descriptors) that is analogous to the implicit push
> of the base address of a structure prior to evaluation of a
> DW_AT_data_member_location to access a data member of a structure. For
> an example, see Appendix D.2 on page 304.


### Section 2.5.4.1 Memory Locations [adapted from 2.6.1.1.2]

Insert the following:

> A memory location represents the location of a piece or all of an
> object or other entity in memory. On architectures that support
> multiple address spaces, a memory location contains a component that
> identifies the address space.
> 
> In contexts that expect a location, a value of the generic type
> may be implicitly converted to a memory location in the default
> address space.
> 
> The following operations push memory locations onto the DWARF stack:
> 
> 1. `DW_OP_addr` [moved from section 2.5.1.1]  
> The `DW_OP_addr` operation has a single operand that encodes a
> machine address and whose size is the size of an address on the
> target machine. The value of this operand is treated as an address
> in the default address space and pushed onto the stack.
> 
> 2. `DW_OP_addrx` [moved from section 2.5.1.1]  
> The `DW_OP_addrx` operation has a single operand that encodes an
> unsigned LEB128 value, which is a zero-based index into the `.debug_addr`
> section, where a machine address is stored. This index is relative to the
> value of the `DW_AT_addr_base` attribute of the associated compilation
> unit. The address obtained is treated as an address in the default address
> space and pushed onto the stack.
> 
> 3. `DW_OP_breg0`, ..., `DW_OP_breg31` [moved from section 2.5.1.2]  
> The single operand of the `DW_OP_breg<n>` operations provides a signed
> LEB128 byte offset. The contents of the specified register (0–31) are
> treated as a memory address in the default address space. The offset is
> added to the address obtained from the register and the resulting memory
> location is pushed onto the stack.
> 
> 4. `DW_OP_bregx` [moved from section 2.5.1.2]  
> The `DW_OP_bregx` operation provides a memory location formed from its
> two operands. The first operand is a register number which is specified
> by an unsigned LEB128 number. The contents of the specified register are
> treated as a memory address in the default address space. The second
> operand is a signed LEB128 byte offset. The offset is added to the
> address obtained from the register and the resulting location is pushed
> onto the stack.
>
> 5. `DW_OP_form_tls_address`... [moved unchanged from section 2.5.1.3]
> 
> 6. `DW_OP_call_frame_cfa`... [moved unchanged from section 2.5.1.3]
> 
> 7. `DW_OP_deref` [moved from section 2.5.1.3]  
> The `DW_OP_deref` operation pops a location `L` from the top of the
> stack. The first `S` bytes, where `S` is the size of an address on the
> target machine, are retrieved from the location `L` and pushed onto the
> stack as a value of the generic type.
> 
> 8. `DW_OP_deref_size` [moved from section 2.5.1.3]  
> The `DW_OP_deref_size` takes a single 1-byte unsigned integral operand
> that specifies the size `S`, in bytes, of the value to be retrieved.
> The size `S` must be no larger than the size of the generic type. The
> operation behaves like the `DW_OP_deref` operation: it pops a location
> `L` from the stack. The first `S` bytes are retrieved from the
> location `L`, zero extended to the size of an address on the target
> machine, and pushed onto the stack as a value of the generic type.
> 
> 9. `DW_OP_deref_type` [moved from section 2.5.1.3]  
> The `DW_OP_deref_type` operation takes two operands. The first operand
> is a 1-byte unsigned integer that specifies the size `S` of the type
> given by the second operand. The second operand is an unsigned LEB128
> integer that represents the offset of a debugging information entry in
> the current compilation unit, which must be a `DW_TAG_base_type` entry
> that provides the type `T` of the value to be retrieved. The size `S`
> must be the same as the size of the base type represented by the
> second operand. This operation pops a location `L` from the stack. The
> first `S` bytes are retrieved from the location `L` and pushed onto the
> stack as a value of type `T`.
> 
> [non-normative] While the size of the pushed value could be inferred from the base type
> definition, it is encoded explicitly into the operation so that the
> operation can be parsed easily without reference to the `.debug_info`
> section.
> 
> 10. `DW_OP_xderef`... [moved unchanged from section 2.5.1.3]
> 
> 11. `DW_OP_xderef_size`... [moved unchanged from section 2.5.1.3]
> 
> 12. `DW_OP_xderef_type`... [moved unchanged from section 2.5.1.3]


### Section 2.5.4.2 Register Locations [adapted from 2.6.1.1.3]

Place the contents of old section 2.6.1.1.3 here.

_Remove_ the non-normative text:

> _A register location description must stand alone as the entire
> description of an object or a piece of an object._

Include the descriptions of the following operations:

- `DW_OP_reg0`, ..., `DW_OP_reg31`
- `DW_OP_regx`


### Section 2.5.4.3 Implicit Locations [adapted from 2.6.1.1.4]

Move the contents of Section 2.6.1.1.4 here, replacing the term
"location description" with "location" throughout, except for the last
non-normative paragraph, which should remain as is:

> DWARF location descriptions are intended ...

In the first paragraph, change "but whose contents are nonetheless
either known or known to be undefined" to "but whose contents are
nonetheless known".

Under `DW_OP_stack_value`, remove the sentence:

> The `DW_OP_stack_value` operation terminates the expression.


### Section 2.5.4.4 Undefined Locations [adapted from 2.6.1.1.1]

Insert the following (adapted from Section 2.6.1.1.1):

> An undefined location represents a piece or all of an object that is
> present in the source but not in the object code (perhaps due to
> optimization). The `DW_OP_undefined` operation pushes an undefined
> location onto the stack. A DWARF expression containing no operations or
> that leaves no elements on the stack also produces an undefined
> location.


### Section 2.5.4.5 Composite Locations [adapted from 2.6.1.2]

Insert the following (adapted from Section 2.6.1.2, and with the new
`DW_OP_composite` operator):

> The above kinds of locations are considered "simple" locations.
> 
> A composite location describes an object or value which may be
> contained in part of a register or stored in more than one location.
> Each piece is described by a composition operation. There may be one
> or more composition operations in a single composite location
> description.
>
> A series of piece operations (`DW_OP_piece` or `DW_OP_bit_piece`)
> describes the parts of a value in storage order. Each piece
> operation pops a location `A` from the stack and updates the composite
> location `B` in the preceding element of the stack by appending the
> new piece described by `A`.
>
> A composite location may be formed from several simple locations by
> the composition operations described in this section. Each simple
> location describes the location of one piece of the object; each
> composition operation describes which part of the object is located
> there.
> 
> 1. `DW_OP_composite`
>
>     The `DW_OP_composite` operator has no operands. It pushes a new, empty,
>     composite location onto the stack.
>
>     [non-normative] This operator is provided so that a new series of
>     piece operations can be started to form a composite location when
>     the state of the stack is unknown (e.g., following a `DW_OP_call*`
>     operation).
> 
> 2. `DW_OP_piece`
>
>     The `DW_OP_piece` operation takes a single operand, which is an unsigned
>     LEB128 number. The number describes the size `S`, in bytes, of the piece
>     of the object referenced by the location `A` on the top of the stack. If
>     the piece is located in a register, but does not occupy the entire
>     register, the placement of the piece within that register is defined by
>     the ABI. Many compilers store a single variable in sets of registers, or
>     store a variable partially in memory and partially in registers.
>     `DW_OP_piece` provides a way of describing how large a part of a
>     variable a particular location refers to.
>
> 3. `DW_OP_bit_piece`
>
>     The DW_OP_bit_piece operation takes two operands. The first is an
>     unsigned LEB128 number that gives the size `S`, in bits, of the piece.
>     The second is an unsigned LEB128 number that gives the offset in bits
>     from the location defined by the location `A` on the top of the stack.
>
>     Interpretation of the offset depends on the type of location. If the
>     location is an undefined location (see Section 2.5.4.4), the
>     `DW_OP_bit_piece` operation describes a piece consisting of the given
>     number of bits whose values are undefined, and the offset is ignored. If
>     the location is a memory address (see Section 2.5.4.1), the
>     `DW_OP_bit_piece` operation describes a sequence of bits relative to the
>     location whose address is on the top of the DWARF stack using the bit
>     numbering and direction conventions that are appropriate to the current
>     language on the target system. In all other cases, the source of the
>     piece is given by either a register location (see Section 2.5.4.2) or an
>     implicit value location (see Section 2.5.4.3); the offset is from the
>     least significant bit of the source value.
> 
>     [non-normative] `DW_OP_bit_piece` is used instead of `DW_OP_piece`
>     when the piece to be assembled into a value or assigned to is not
>     byte-sized or is not at the start of a register or addressable
>     unit of memory.
>
>     [non-normative] Whether or not a `DW_OP_piece` operation is
>     equivalent to any `DW_OP_bit_piece` operation with an offset of 0
>     is ABI dependent.
>
> For compatibility with DWARF Version 5 and earlier, the following
> additional rules apply to piece operations:
>
> - If a piece operation is processed while the stack is empty, a new
> empty composite and an undefined location are pushed implicitly (as if
> `DW_OP_composite DW_OP_undefined` had been processed immediately prior
> to the piece operation). The result is a composite with a single
> undefined piece.
>
> - Otherwise, if the top of the stack `A` is a composite, and is the only
> element on the stack (i.e., `B` does not exist), an undefined location
> is pushed implicitly (as if `DW_OP_undefined` had been processed
> immediately prior to the piece operation), whereupon the composite `A`
> becomes `B` and the undefined location is now `A`. The result is the
> addition of an undefined piece to the existing composite location.
>
> - Otherwise, if the top of the stack `A` is a location, or convertible
> to a location, and the preceding element is not a composite location,
> one or more elements below `A` are popped and discarded until the
> preceding element `B` is a composite location, or until `A` is the
> only element on the stack. If `A` is the only remaining element, a new
> empty composite is inserted before it (as if `DW_OP_composite
> DW_OP_swap` had been processed immediately prior to the piece
> operation), and the result is a new composite location with the single
> piece `A`.


### Section 2.5.4.6 Offset Operations [NEW]

Add:

> In addition to the composite operations, locations
> may be modified by the following operations:
> 
> 1. `DW_OP_offset`
> 
>     `DW_OP_offset` pops two stack entries. The first (top of stack)
>     must be an integral type value, which represents a byte
>     displacement. The second must be a location. It forms an updated
>     location by adding the given byte displacement to the offset
>     component of the original location and pushes the updated location
>     onto the stack.
> 
> 2. `DW_OP_bit_offset`
> 
>     `DW_OP_bit_offset` pops two stack entries. The first
>     (top of stack) must be an integral type value, which represents a bit
>     displacement. The second must be a location. It forms an updated
>     location by adding the given bit displacement to the offset
>     component of the original location and pushes the updated location
>     onto the stack.
> 
>     [non-normative] A bit offset of `n*8` is equivalent to a byte offset of `n`.


### Section 2.5.5  Control Flow Operations [was: 2.5.1.5]

Move the contents of old section 2.5.1.5 here.

Include the descriptions of the following operations:

- `DW_OP_le`, ... `DW_OP_ne`
- `DW_OP_skip`
- `DW_OP_bra`
- `DW_OP_call2`, `DW_OP_call4`, `DW_OP_call_ref`

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


### Section 2.5.6 Type Conversions [was: 2.5.1.6]

Move and renumber Section 2.5.1.6 to here.

Include the descriptions of the following operations:

- `DW_OP_convert`
- `DW_OP_reinterpret`


### Section 2.5.7 Special Operations [was: 2.5.1.7]

Move and renumber Section 2.5.1.7 to here.

Include the descriptions of the following operations:

- `DW_OP_nop`
- `DW_OP_entry_value`
- `DW_OP_extended`
- `DW_OP_user_extended`


### Section 2.6 Expression Lists [was: Location Descriptions]

The parts of this section dealing with single location descriptions (was 2.6.1)
is moved to Section 2.5.4 Location Operations.

The remainder of this section deals with value lists and location lists.
Rename the section to "Expression Lists".

Add the following two subsections:

### Section 2.6.1 Value Lists [was: 2.5.2]

Place the contents of old section 2.5.2 Value Lists here.

### Section 2.6.2 Location Lists [was: 2.6.2]

Place the contents of old Section 2.6.2 Location Lists here.


### Section 5.7.6 Data Member Entries

In the description for `DW_AT_data_member_location`, change
the second bullet to:

> 2\. Otherwise, the value must be a location description. In this case,
> the beginning of the containing entity must be byte aligned. The
> location of the containing entity is pushed on the DWARF stack before
> the location description is evaluated; the result of the evaluation is
> the location of the member entry.

Replace the following non-normative paragraph with:

> > [non-normative] The push on the DWARF expression stack of the location
> > of the containing construct is equivalent to execution of the
> > `DW_OP_push_object_address` operation (see Section 2.5.4);
> > `DW_OP_push_object_address` therefore is not needed at the beginning of
> > a location description for a data member. The result of the evaluation
> > is a location, not an offset to the member.

_Remove_ the second non-normative paragraph:

> > [non-normative] A `DW_AT_data_member_location` attribute that has the
> > form of a location description is not valid for a data member contained
> > in an entity that is not byte aligned because DWARF operations do not
> > allow for manipulating or computing bit offsets.

### Section 5.14 Pointer to Member Type Entries

Replace the paragraph beginning "The `DW_AT_use_location` description..." with:

> The `DW_AT_use_location` description is used in conjunction with the
> location descriptions for a particular object of the given pointer to
> member type and for a particular structure or class instance. The
> `DW_AT_use_location` attribute expects two values to be pushed onto the
> DWARF expression stack before the `DW_AT_use_location` description is
> evaluated. The first value pushed is the value of the pointer to member
> object itself. The second value pushed is the location of the
> entire structure or union instance containing the member whose location
> is being calculated.


### Section 6.4 Call Frame Information

For the "expression(E)" register rule, change the description to:

> The previous value of this register is located at the location produced
> by executing the location description E (see Section 2.5).


### Section 7.7.1 DWARF Expressions

Add to Table 7.9:

>                                 No. of
>     Operation             Code  Operands  Notes
>     --------------------  ----  --------  -----
>     DW_OP_offset          TBA      0
>     DW_OP_bit_offset      TBA      0


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
(immediately following `DW_OP_push_object_address`).
