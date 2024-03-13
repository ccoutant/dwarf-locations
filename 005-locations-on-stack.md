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
anything but a memory location on the stack.

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

The `DW_OP_deref*` and `DW_OP_xderef*` operators are extended to operate on
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
element of the stack. A new operator, `DW_OP_piece_end`, is defined for
use when a composite location description is complete, and there is a
need to continue the expression.

The `DW_OP_call*` operators are now allowed to leave a location
on the stack.

The contents of Section 2.5 and 2.6 are reorganized as follows:

- 2.5 DWARF Expressions
    - 2.5.1 Value Operations
        - 2.5.1.1 Literal Encodings
        - 2.5.1.2 Register Values
        - 2.5.1.3 Stack Operations
        - 2.5.1.4 Arithmetic and Logical Operations
    - 2.5.2 Location Operations
        - 2.5.2.1 Memory Locations
        - 2.5.2.2 Register Locations
        - 2.5.2.3 Implicit Locations
        - 2.5.2.4 Undefined Locations
        - 2.5.2.5 Composite Locations
        - 2.5.2.6 Offset Operations
    - 2.5.3 Control Flow Operations
    - 2.5.4 Type Conversions
    - 2.5.5 Special Operations
- 2.6 Expression Lists



Proposed Changes
----------------

### Section 2.2 Attribute Types

[Page 24]
In Table 2.3, Classes of attribute value, in the row for "locdesc,"
replace the reference to Section 2.6 with a reference to Section 2.5.


### Section 2.5 DWARF Expressions

[Page 26]
Replace the contents of the preamble to this section with:

> DWARF expressions describe how to compute a value or specify a
> location. They are expressed in terms of DWARF operations that
> operate on a stack. Each element on the stack may be
> either a value or a location.
> 
> Values on the stack are typed, and can represent a value of any
> supported base type of the target machine, or of the generic type,
> which is an integral type that has the size of an address on the
> target machine and unspecified signedness.
> 
> _The generic type is the same as the unspecified type used for stack
> operations defined in DWARF Version 4 and before._
> 
> Locations represent a place where a value is stored, which may
> be any of the following:
> 
> - Memory locations, for values that are stored in memory, starting
>   at a given address within a given address space.
> 
> - Register locations, for values that are stored in registers.
> 
> - Undefined locations, for values that are unavailable
>   (perhaps due to optimization).
> 
> - Implicit locations, for values that are not stored anywhere (like
>   undefined locations), but whose values can be reconstructed.
> 
> - Composite locations, for values that are stored in a combination
>   of storage (e.g., aggregate values where certain components have
>   been promoted to registers).
> 
> A DWARF expression is encoded as a stream of operations, each consisting of an
> opcode followed by zero or more literal operands. The number of operands is
> implied by the opcode.
> 
> The result of a DWARF expression is the value or location on the top
> of the stack after evaluating the operations.
> 
> A DWARF expression is evaluated in a context that determines whether
> its result is expected to be a value or a location. Expressions that are
> expected to produce a location are called "location descriptions."
> 
> Implicit conversions between memory addresses and values may happen
> during the execution of any operation or when evaluation of the
> expression is completed. If a location is expected, but the result is
> a value, the value is implicitly treated as a memory address in the
> default address space, and converted to a memory location. If a value
> is expected, but the result is a memory location in the default
> address space, the address is implicitly converted to a value.


### Section 2.5.1 General Operations

[Page 26]
In the first paragraph, delete all but the first sentence of the first
two paragraphs, leaving only this:

> Each general operation represents a postfix operation on a simple
> stack machine.

(These deleted sentences have been moved up to Section 2.5.)


### Section 2.5.1.1 Literal Encodings

Move `DW_OP_addr` and `DW_OP_addrx` to Section 2.5.2.2.


### Section 2.5.1.3 Stack Operations

[Pages 30-31]
Under `DW_OP_deref`, `DW_OP_deref_size`, and `DW_OP_deref_type`,
replace the descriptions with:

> 7\. `DW_OP_deref`  
> The `DW_OP_deref` operation pops the top stack entry and treats it as
> a location. The first `S` bytes, where `S` is the size of an address on
> the target machine, are retrieved from the location and pushed onto
> the stack as a value of the generic type.
> 
> 8\. `DW_OP_deref_size`  
> The `DW_OP_deref_size` takes a single 1-byte unsigned integral operand
> that specifies the size `S`, in bytes, of the value to be retrieved. The
> operation behaves like the `DW_OP_deref` operation: it
> pops the top stack entry and treats it as a location. The first `S` bytes
> are retrieved from the location, zero extended to the size of an
> address on the target machine, and pushed onto the stack as a value of
> the generic type.
> 
> 9\. `DW_OP_deref_type`  
> The `DW_OP_deref_type` operation takes two operands. The first operand
> is a 1-byte unsigned integer that specifies the size `S` of the type
> given by the second operand. The second operand is an unsigned LEB128
> integer that represents the offset of a debugging information entry in
> the current compilation unit, which must be a `DW_TAG_base_type` entry
> that provides the type `T` of the value to be retrieved. This operation
> pops the top stack entry and treats it as a location. The first `S`
> bytes are retrieved from the location and pushed onto the stack as a
> value of type `T`.
> 
> _While the size of the pushed value could be inferred from the base type
> definition, it is encoded explicitly into the operation so that the
> operation can be parsed easily without reference to the `.debug\_info`
> section._

Move items 13 (`DW_OP_push_object_address`), 14 (`DW_OP_form_tls_address`),
and 15 (`DW_OP_call_frame_cfa`) to (new) Section 2.5.2 Location Operations.


### Section 2.5.2 Location Operations [NEW]

Insert the following (adapted from parts of Section 2.6) into this new section:

> Information about the location of program objects is provided by
> location descriptions. Location descriptions can be either of two forms:
> 
> 1. Single location descriptions, which are a language-independent
> representation of addressing rules of arbitrary complexity built from
> DWARF expressions. They are sufficient for describing the location of
> any object as long as its lifetime is either static or the same as the
> lexical block that owns it, and it does not move during its lifetime.
> 
> 2. Location lists, which are used to describe objects that have a
> limited lifetime or change their location during their lifetime.
> Location lists are described in Section 2.6.
> 
> Location descriptions are distinguished in a context-sensitive manner.
> As the value of an attribute, a single location description is encoded
> using class `locdesc`, and a location list is encoded using class `loclist`
> (which serves as an index into a separate section containing location
> lists).
> 
> The following operation can be used to push a location onto the stack:
> 
> 1. `DW_OP_push_object_address` [moved from Section 2.5.1.3]  
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


### Section 2.5.2.1 Memory Locations

Insert the following (adapted from Section 2.6.1.1.2):

> A memory location represents the location of a piece or all of an
> object or other entity in memory. On architectures that support
> multiple address spaces, a memory location contains a component that
> identifies the address space (which may be provided by the
> `DW_OP_xderef` operation). A memory location is considered
> _unbounded_, as the size of the location storage is only implied by
> the type of object stored at that location.
> 
> In contexts that expect a location, a value of the generic type
> may be implicitly converted to a memory location in the default
> address space.
> 
> The following operations push memory locations onto the DWARF stack:
> 
> 1. `DW_OP_addr` [moved from Section 2.5.1.1]  
> The `DW_OP_addr` operation has a single operand that encodes a
> machine address and whose size is the size of an address on the
> target machine. The value of this operand is treated as an address
> in the default address space and pushed onto the stack.
> 
> 2. `DW_OP_addrx` [moved from Section 2.5.1.1]  
> The `DW_OP_addrx` operation has a single operand that encodes an
> unsigned LEB128 value, which is a zero-based index into the `.debug\_addr`
> section, where a machine address is stored. This index is relative to the
> value of the `DW_AT_addr_base` attribute of the associated compilation
> unit. The address obtained is treated as an address in the default address
> space and pushed onto the stack.
>
> 3. `DW_OP_form_tls_address`... [moved from Section 2.5.1.3]
> 
> 4. `DW_OP_call_frame_cfa`... [moved from Section 2.5.1.3]


### Section 2.5.2.2 Register Locations

Move the contents of Section 2.6.1.1.3 here, replacing the term
"location description" with "location" throughout.

[page 39] Remove the non-normative text:

> A register location description must stand alone as the entire
> description of an object or a piece of an object.


### Section 2.5.2.3 Implicit Locations

Move the contents of Section 2.6.1.1.4 here, replacing the term
"location description" with "location" throughout.


### Section 2.5.2.4 Undefined Locations

Insert the following (adapted from Section 2.6.1.1.1):

> An undefined location represents a piece or all of an object that is
> present in the source but not in the object code (perhaps due to
> optimization). The `DW_OP_undefined` operation pushes an undefined
> location onto the stack. A DWARF expression containing no operations or
> that leaves no elements on the stack also produces an undefined
> location.


### Section 2.5.2.5 Composite Locations

Insert the following (adapted from Section 2.6.1.2):

> The above kinds of locations are considered "simple" locations.
> 
> A composite location describes an object or value which may be
> contained in part of a register or stored in more than one location.
> Each piece is described by a composition operation. There may be one
> or more composition operations in a single composite location
> description. A composite location may be either _partial_ or
> _complete_. While under construction, it is _partial_, and is
> converted to a _complete_ composite location explicitly by a
> `DW_OP_piece_end` operator, or implicitly at the end of the DWARF
> expression.
>
> A series of piece operations describes the parts of a value in memory
> address order. Each piece operation pops a location `A` from the stack
> and updates the partial composite location `B` in the preceding
> element of the stack by appending the new piece described by `A`. If
> `A` is the only element on the stack, or the second element `B` on the
> stack is not a partial composite location, however, a new partial
> location is pushed onto the stack with `A` as the first piece.
>
> A composite location may be formed from several simple locations by
> the composition operations described in this section. Each simple
> location describes the location of one piece of the object; each
> composition operation describes which part of the object is located
> there.
> 
> 1. `DW_OP_piece`
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
> 2. `DW_OP_bit_piece`
> 
>     The DW_OP_bit_piece operation takes two operands. The first is an
>     unsigned LEB128 number that gives the size `S`, in bits, of the piece.
>     The second is an unsigned LEB128 number that gives the offset in bits
>     from the location defined by the location `A` on the top of the stack.
>
>     Interpretation of the offset depends on the type of location. If the
>     location is an undefined location (see Section 2.5.2.4), the
>     `DW_OP_bit_piece` operation describes a piece consisting of the given
>     number of bits whose values are undefined, and the offset is ignored. If
>     the location is a memory address (see Section 2.5.2.1), the
>     `DW_OP_bit_piece` operation describes a sequence of bits relative to the
>     location whose address is on the top of the DWARF stack using the bit
>     numbering and direction conventions that are appropriate to the current
>     language on the target system. In all other cases, the source of the
>     piece is given by either a register location (see Section 2.5.2.2) or an
>     implicit value location (see Section 2.5.2.3); the offset is from the
>     least significant bit of the source value.
> 
> 3. `DW_OP_piece_end`
> 
>     The `DW_OP_piece_end` operation terminates a composition operation by
>     converting the partial composite location description on top of the
>     stack to a complete composite location description. This operation
>     is necessary only if the location description is not at the end of
>     the DWARF expression; otherwise, the conversion is implicit.
>
> 
> _`DW_OP_bit_piece` is used instead of `DW_OP_piece` when the piece to be
> assembled into a value or assigned to is not byte-sized or is not at the
> start of a register or addressable unit of memory._
> 
> _Whether or not a `DW_OP_piece` operation is equivalent to any
> `DW_OP_bit_piece` operation with an offset of 0 is ABI dependent._


### Section 2.5.2.6 Offset Operations

Add:

> In addition to the composite operations, locations
> may be modified by the following operations:
> 
> 1. `DW_OP_offset`
> 
>     `DW_OP_offset` pops two stack entries. The first (top of stack)
>     must be an integral type value, which represents a byte
>     displacement. The second must be a location. It forms an updated
>     location by adding the given byte displacement to the original
>     location and pushes the updated location onto the stack.
> 
> 2. `DW_OP_bit_offset`
> 
>     `DW_OP_bit_offset` pops two stack entries. The first
>     (top of stack) must be an integral type value, which represents a bit
>     displacement. The second must be a location. It forms an updated
>     location by adding the given bit displacement to the original
>     location and pushes the updated location onto the stack.
> 
>     _A bit offset of `n*8` is equivalent to a byte offset of `n`._


### Section 2.5.3  Control Flow Operations

Move and renumber Section 2.5.1.5 to here.

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

### Section 2.5.4 Type Conversions

Move and renumber Section 2.5.1.6 to here.


### Section 2.5.5 Special Operations

Move and renumber Section 2.5.1.7 to here.


### Section 2.6 Location Descriptions

The parts of this section dealing with single location descriptions
is moved to Section 2.5.2 Location Operations.

The remainder of this section deals with location lists.
Rename the section to "Location Lists".


### Section 5.7.6 Data Member Entries

[Page 118] In the description for `DW_AT_data_member_location`, change
the second bullet to:

> 2\. Otherwise, the value must be a location description. In this case,
> the beginning of the containing entity must be byte aligned. The
> location of the containing entity is pushed on the DWARF stack before
> the location description is evaluated; the result of the evaluation is
> the location of the member entry.


### Section 5.14 Pointer to Member Type Entries

[Page 131] Replace the paragraph beginning "The `DW_AT_use_location` description..." with:

> The `DW_AT_use_location` description is used in conjunction with the
> location descriptions for a particular object of the given pointer to
> member type and for a particular structure or class instance. The
> `DW_AT_use_location` attribute expects two values to be pushed onto the
> DWARF expression stack before the `DW_AT_use_location` description is
> evaluated. The first value pushed is the value of the pointer to member
> object itself. The second value pushed is the location of the
> entire structure or union instance containing the member whose address
> is being calculated.


### Section 6.4 Call Frame Information

[Page 181]
For the "expression(E)" register rule, change the description to:

> The previous value of this register is located at the location produced
> by executing the location description E (see Section 2.5).


### Section 7.7.1 DWARF Expressions

[Page 226]
Add to Table 7.9:

>                                 No. of
>     Operation             Code  Operands  Notes
>     --------------------  ----  --------  -----
>     DW_OP_offset          TBA      0
>     DW_OP_bit_offset      TBA      0

### Section D.2.1 Fortran Simple Array Example

[Page 296]
In Figure D.4, change `DW_OP_plus` to `DW_OP_offset` in the following
places:

* In the `DW_AT_associated` attribute at `1$`.

* In the `DW_AT_lower_bound` and `DW_AT_upper_bound` attributes at `2$`.

* In the `DW_AT_allocated` and `DW_AT_data_location` attributes at `6$`.

* In the `DW_AT_lower_bound` and `DW_AT_upper_bound` attributes at `7$`.

### Section D.2.3 Fortran 2008 Assumed-rank Array Example

[Page 302]
In Figure D.13, change `DW_OP_plus` to `DW_OP_offset` in the following
places:

* In the `DW_AT_rank` and `DW_AT_data_location` attributes at `10$`.

* In the `DW_AT_lower_bound` and `DW_AT_upper_bound` attributes at `11$`
(immediately following `DW_OP_push_object_address`).
