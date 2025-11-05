# General Support for Address Spaces

## PROBLEM DESCRIPTION

AMDGPU needs to be able to describe addresses that are in different kinds of
memory. Optimized code may need to describe a variable that resides in pieces
that are in different kinds of storage which may include parts of registers,
memory that is in a mixture of memory kinds, implicit values, or be undefined.

DWARF 5 has the concept of segment addresses. However, the segment cannot be
specified within a DWARF expression, which is only able to specify the offset
portion of a segment address. The segment index is only provided by the entity
that specifies the DWARF expression. Therefore, the segment index is a property
that can only be put on complete objects, such as a variable. That makes it only
suitable for describing an entity (such as variable or subprogram code) that is
in a single kind of memory.

AMDGPU uses multiple address spaces. For example, a variable may be allocated in
a register that is partially spilled to the call stack which is in the private
address space, and partially spilled to the local address space. DWARF mentions
address spaces, for example as an argument to the `DW_OP_xderef*` operations. A
new section that defines address classes is added.

A new attribute `DW_AT_address_space` is added to pointer and reference types.
This allows the compiler to specify which address space is being used to
represent the pointer or reference type.

DWARF 5 uses the concept of an address in many expressions but does
not define how it relates to address spaces. For example,
`DW_OP_push_object_address` pushes the address of an object. Other contexts
implicitly push an address on the stack before evaluating an expression. For
example, the `DW_AT_use_location` attribute of the `DW_TAG_ptr_to_member_type`. The
expression belongs to a source language type which may apply to objects
allocated in different kinds of storage. Therefore, it is desirable that the
expression that uses the address can do so without regard to what kind of
storage it specifies, including the address space of a memory location
description. For example, a pointer to member value may want to be applied to an
object that may reside in any address space.

The DWARF 5 `DW_OP_xderef*` operations allow a value to be converted into an
address of a specified address space which is then read. But it provides no way
to create a memory location description for an address in the non-default
address space. For example, AMDGPU variables can be allocated in the local
address space at a fixed address.

A `DW_OP_form_aspace_address` operation is added and defined to create a memory
location description from an address and address space. It can be used to
specify the location of a variable that is allocated in a specific address
space. This allows the size of addresses in an address space to be larger than
the generic type. It also allows a consumer great implementation freedom. It
allows the implicit conversion back to a value to be limited only to the default
address space to maintain compatibility with DWARF 5. For other address spaces
the producer can use the new operations that explicitly specify the address
space.

In contrast, if the `DW_OP_form_aspace_address` operation had been defined to
produce a value, and an implicit conversion to a memory location description was
defined, then it would be limited to the size of the generic type (which matches
the size of the default address space). An implementation would likely have to
use _reserved ranges_ of value to represent different address spaces. Such
a value would likely not match any address value in the actual hardware. That
would require the consumer to have special treatment for such values.

`DW_OP_breg*` treats the register as containing an address in the default address
space. A `DW_OP_aspace_bregx` operation is added to allow the address space of the
address held in a register to be specified.

Similarly, `DW_OP_implicit_pointer` treats its implicit pointer value as being in
the default address space. A `DW_OP_aspace_implicit_pointer` operation is added to
allow the address space to be specified.

Almost all uses of addresses in DWARF 5 are limited to defining location
descriptions, or to be dereferenced to read memory. The exception is
`DW_CFA_val_offset` which uses the address to set the value of a register. In
order to support address spaces, the CFA DWARF expression is defined to be a
memory location description. This allows it to specify an address space which is
used to convert the offset address back to an address in that address space.

This approach of extending memory location descriptions to support address
spaces, allows all existing DWARF 5 expressions to have the identical semantics.
It allows the compiler to explicitly specify the address space it is using. For
example, a compiler could choose to access private memory in a swizzled manner
when mapping a source language thread to the lane of a wavefront in a SIMT
manner. Or a compiler could choose to access it in an unswizzled manner if
mapping the same language with the wavefront being the thread.

It also allows the compiler to mix the address space it uses to access private
memory. For example, for SIMT it can still spill entire vector registers in an
unswizzled manner, while using a swizzled private memory for SIMT variable
access.

This approach also allows memory location descriptions for different address
spaces to be combined using the regular `DW_OP_*piece` operations.

Location descriptions are an abstraction of storage. They give freedom to the
consumer on how to implement them. They allow the address space to encode lane
information so they can be used to read memory with only the memory location
description and no extra information. The same set of operations can operate on
locations independent of their kind of storage. The `DW_OP_deref*` therefore can
be used on any storage kind, including memory location descriptions of different
address spaces. Therefore, the `DW_OP_xderef*` operations are unnecessary, except
to become a more compact way to encode a non-default address space address
followed by dereferencing it.

## PROPOSAL

In Section 2.2 "Attribute Types", add the following row to Table 2.2 "Attribute
names":

Table 2.2: Attribute names

> Attribute             | Usage
> --------------------- | --------------------------------------------------------------
> `DW_AT_address_space` | Architecture specific address space (see 2.x "Address Spaces")

Add the following after Section 2.11 "Address Classes":

> 2.x Address Spaces
> 
> DWARF address spaces correspond to target architecture specific linear
> addressable memory areas. They are used in DWARF expression location
> descriptions to describe in which target architecture specific memory area
> data resides.
> 
> [non-normative] _Target architecture specific DWARF address spaces may
> correspond to hardware supported facilities such as memory utilizing base
> address registers, scratchpad memory, and memory with special interleaving.
> The size of addresses in these address spaces may vary. Their access and
> allocation may be hardware managed with each thread or group of threads
> having access to independent storage. For these reasons they may have
> properties that do not allow them to be viewed as part of the unified global
> virtual address space accessible by all threads._
> 
> [non-normative] _It is target architecture specific whether multiple DWARF
> address spaces are supported and how source language memory spaces map to
> target architecture specific DWARF address spaces. A target architecture may
> map multiple source language memory spaces to the same target architecture
> specific DWARF address class. Optimization may determine that variable
> lifetime and access pattern allows them to be allocated in faster scratchpad
> memory represented by a different DWARF address space than the default for
> the source language memory space._
> 
> Although DWARF address space identifiers are target architecture specific,
> `DW_ASPACE_none` is a common address space supported by all target
> architectures, and defined as the target architecture default address space.
> 
> DWARF address space identifiers are used by:
> 
> * The `DW_AT_address_space` attribute.
> 
> * The DWARF expressions: `DW_OP_aspace_bregx`,
>   `DW_OP_form_aspace_address`, `DW_OP_aspace_implicit_pointer`, and
>   `DW_OP_xderef*`.
> 
> * The CFI instructions: `DW_CFA_def_aspace_cfa` and `DW_CFA_def_aspace_cfa_sf`.

[For further discussion]
May want to clarify what `DW_AT_address_class` is for, now that the x86 examples
have been removed, in a separate issue.

In Section 3.7 "Memory Locations", add the following at the
end of the first paragraph:

> `DW_ASPACE_none` is defined as the target architecture default address space.
> See 2.x Address Spaces.

After the definition of `DW_OP_addrx` add:

> 3.  `DW_OP_form_aspace_address`  
>     `DW_OP_form_aspace_address` pops top two stack entries. The first must be
>     an integral type value that represents a target architecture specific
>     address space identifier AS. The second must be an integral type value
>     that represents an address A.
> 
>     The address size S is defined as the address bit size of the target
>     architecture specific address space that corresponds to AS.
> 
>     A is adjusted to S bits by zero extending if necessary, and then
>     treating the least significant S bits as an unsigned value A'.
> 
>     It pushes a location description L with one memory location description
>     SL on the stack. SL specifies the memory location storage LS that
>     corresponds to AS with a bit offset equal to A' scaled by 8 (the byte
>     size).
> 
>     If AS is an address space that is specific to context elements, then LS
>     corresponds to the location storage associated with the current context.
> 
>     [non-normative] For example, if AS is for per thread storage then LS is
>     the location storage for the current thread. Therefore, if L is accessed
>     by an operation, the location storage selected when the location
>     description was created is accessed, and not the location storage
>     associated with the current context of the access operation.
> 
>     The DWARF expression is ill-formed if AS is not one of the values
>     defined by the target architecture specific `DW_ASPACE_*` values.
> 
>     See Section 2.5.2.3 "Implicit Locations" for
>     special rules concerning implicit pointer values produced by
>     dereferencing implicit location descriptions created by the
>     `DW_OP_implicit_pointer` and `DW_OP_aspace_implicit_pointer` operations.

After the definition of `DW_OP_bregx` add:

> 7.  `DW_OP_aspace_bregx`  
>     `DW_OP_aspace_bregx` has two operands. The first is an unsigned
>     LEB128 integer that represents a register number R. The second is a signed
>     LEB128 integer that represents a byte displacement B. It pops one stack
>     entry that is required to be an integral type value that represents a target
>     architecture specific address space identifier AS.
> 
>     The action is the same as for `DW_OP_breg`<N>, except that R is used as
>     the register number, B is used as the byte displacement, and AS is used as
>     the address space identifier.
> 
>     The DWARF expression is ill-formed if AS is not one of the values defined by
>     the target architecture specific `DW_ASPACE_*` values.

In Section 3.11 "Implicit Pointer Locations", after the
definition of `DW_OP_implicit_pointer`, add:

> 2.  `DW_OP_aspace_implicit_pointer`  
>     `DW_OP_aspace_implicit_pointer` has two operands that are the same as for
>     `DW_OP_implicit_pointer`.
> 
>     It pops one stack entry that must be an integral type value that
>     represents a target architecture specific address space identifier AS.
> 
>     The location description L that is pushed on the stack is the same as
>     for `DW_OP_implicit_pointer`, except that the address space identifier
>     used is AS.
> 
>     The DWARF expression is ill-formed if AS is not one of the values
>     defined by the target architecture specific `DW_ASPACE_*` values.

In Section 6.3 "Type Modifier Entries", after the paragraph starting "A modified
type entry describing a pointer or reference type...", add the following
paragraph:

> A modified type entry describing a pointer or reference type (using
> `DW_TAG_pointer_type`, `DW_TAG_reference_type` or `DW_TAG_rvalue_reference_type`)
> may have a `DW_AT_address_space` attribute with a constant value AS
> representing an architecture specific DWARF address space (see 2.x "Address
> Spaces"). If omitted, defaults to `DW_ASPACE_none`. DR is the offset of a
> hypothetical debug information entry D in the current compilation unit for
> an integral base type matching the address size of AS. An object P having
> the given pointer or reference type are dereferenced as if the
> `DW_OP_push_object_address`; `DW_OP_deref_type` DR; `DW_OP_constu` AS;
> `DW_OP_form_aspace_address` expression was evaluated with the
> current context except: the result kind is location description; the initial
> stack is empty; and the object is the location description of P.

[For further discussion -- remove?]
With the expanded support for DWARF address spaces, it may be worth examining
if they can be used for what was formerly supported by DWARF 5 segments that
are being removed in DWARF 6. That would include specifying the address space
of all code addresses (compilation units, subprograms, subprogram entries,
labels, subprogram types, etc.). Either the code address attributes could be
extended to allow a exprloc form (so that `DW_OP_form_aspace_address` can be
used) or the `DW_AT_address_space` attribute be allowed on all DIEs that
formerly allowed `DW_AT_segment`.

In Section 7.1.1.1 "Contents of the Name Index", replace the bullet: 

> * `DW_TAG_variable` debugging information entries with a `DW_AT_location`
>   attribute that includes a `DW_OP_addr` or `DW_OP_form_tls_address` operator
>   are included; otherwise, they are excluded.

with:

> * `DW_TAG_variable` debugging information entries with a `DW_AT_location`
>   attribute that includes a `DW_OP_addr`, `DW_OP_form_aspace_address`, or
>   `DW_OP_form_tls_address` operation are included; otherwise, they are
>   excluded.

### FIXME: [[ Reword definition of CFA in 7.4 Call Frame Information ]]

In Section 7.4.2.2 "CFA Definition Instructions", replace the description of the
six CFI instructions with the following:

> 1.  `DW_CFA_def_cfa`  
>     The `DW_CFA_def_cfa` instruction takes two unsigned LEB128 operands
>     representing a register number R and a (non-factored) byte displacement
>     B. AS is set to the target architecture default address space
>     identifier. The required action is to define the current CFA rule to be
>     equivalent to the result of evaluating the DWARF expression
>     `DW_OP_constu AS; DW_OP_aspace_bregx R, B` as a location description.
> 
> 2.  `DW_CFA_def_cfa_sf`  
>     The `DW_CFA_def_cfa_sf` instruction takes two operands: an unsigned LEB128
>     value representing a register number R and a signed LEB128 factored byte
>     displacement B. AS is set to the target architecture default address
>     space identifier. The required action is to define the current CFA rule
>     to be equivalent to the result of evaluating the DWARF
>     expression `DW_OP_constu AS; DW_OP_aspace_bregx R, B *
>     data_alignment_factor` as a location description.
> 
>     [non-normative] The action is the same as `DW_CFA_def_cfa`, except that
>     the second operand is signed and factored.
> 
> 3.  `DW_CFA_def_cfa_register`  
>     The `DW_CFA_def_cfa_register` instruction takes a single unsigned LEB128
>     operand representing a register number R. The required action is to
>     define the current CFA rule to be equivalent to the result of evaluating
>     the DWARF expression `DW_OP_constu AS; DW_OP_aspace_bregx R, B`
>     as a location description. B and AS are the old CFA byte displacement
>     and address space respectively.
> 
>     If the subprogram has no current CFA rule, or the rule was defined by a
>     `DW_CFA_def_cfa_expression` instruction, then the DWARF is ill-formed.
> 
> 4.  `DW_CFA_def_cfa_offset`  
>     The `DW_CFA_def_cfa_offset` instruction takes a single unsigned LEB128
>     operand representing a (non-factored) byte displacement B. The required
>     action is to define the current CFA rule to be equivalent to the result
>     of evaluating the DWARF expression `DW_OP_constu AS;
>     DW_OP_aspace_bregx R, B` as a location description. R and AS are the old
>     CFA register number and address space respectively.
> 
>     If the subprogram has no current CFA rule, or the rule was defined by a
>     `DW_CFA_def_cfa_expression` instruction, then the DWARF is ill-formed.
> 
> 5.  `DW_CFA_def_cfa_offset_sf`  
>     The `DW_CFA_def_cfa_offset_sf` instruction takes a signed LEB128 operand
>     representing a factored byte displacement B. The required action is to
>     define the current CFA rule to be equivalent to the result of evaluating
>     the DWARF expression `DW_OP_constu AS; DW_OP_aspace_bregx R, B
>     * data_alignment_factor` as a location description. R and AS are the old
>     CFA register number and address space respectively.
> 
>     If the subprogram has no current CFA rule, or the rule was defined by a
>     `DW_CFA_def_cfa_expression` instruction, then the DWARF is ill-formed.
> 
>     [non-normative] The action is the same as `DW_CFA_def_cfa_offset`, except
>     that the operand is signed and factored.
>
> 6.  `DW_CFA_def_cfa_expression`  *FIXME?*  
>     The `DW_CFA_def_cfa_expression` instruction takes a single operand encoded
>     as a DW_FORM_exprloc value representing a DWARF expression E.
>     The required action is to define the current CFA rule to be equivalent
>     to the result of evaluating E with the current context, except the
>     result kind is a location description, the compilation unit is
>     unspecified, the object is unspecified, and an empty initial stack.
>     
>     See A.6.4.2 Call Frame Instructions regarding restrictions on the DWARF
>     expressions that can be used in E.

Add the following CFI instructions:

> 7.  `DW_CFA_def_aspace_cfa`  
>     The `DW_CFA_def_aspace_cfa` instruction takes three unsigned LEB128
>     operands representing a register number R, a (non-factored) byte
>     displacement B, and a target architecture specific address space
>     identifier AS. The required action is to define the current CFA rule to
>     be equivalent to the result of evaluating the DWARF expression
>     `DW_OP_constu AS; DW_OP_aspace_bregx R, B` as a location description.
> 
>     If AS is not one of the values defined by the target architecture
>     specific `DW_ASPACE_*` values then the DWARF expression is ill-formed.
> 
> 8.  `DW_CFA_def_aspace_cfa_sf`  
>     The `DW_CFA_def_aspace_cfa_sf` instruction takes three operands: an
>     unsigned LEB128 value representing a register number R, a signed LEB128
>     factored byte displacement B, and an unsigned LEB128 value representing
>     a target architecture specific address space identifier AS. The required
>     action is to define the current CFA rule to be equivalent to the result
>     of evaluating the DWARF expression `DW_OP_constu AS;
>     DW_OP_aspace_bregx R, B * data_alignment_factor` as a location
>     description.
> 
>     If AS is not one of the values defined by the target architecture
>     specific `DW_ASPACE_*` values, then the DWARF expression is ill-formed.
> 
>     [non-normative] The action is the same as `DW_CFA_def_aspace_cfa`, except
>     that the second operand is signed and factored.

In Section 8.4 "32-Bit and 64-Bit DWARF Formats" list item 3's table, add the
following row:

> Form                             | Role
> -------------------------------- | -----------------------
> `DW_OP_aspace_implicit_pointer`  |   offset in .debug_info

In Section 8.5.4 "Attribute Encodings", add the following row to Table 7.5
"Attribute encodings":

> Table 8.5: Attribute encodings
> 
> Attribute Name                     | Value  | Classes
> ---------------------------------- | ------ | ----------------------------------
> `DW_AT_address_space`              | TBA    | constant

Add the following rows to table 8.9 in Section "8.7.1 Operator Encodings":

> Operation                          | Code  | Number of Operands | Notes
> ---------------------------------- | ----- | ------------------ | --------
> `DW_OP_form_aspace_address`        | TBA   | 0                  |
> `DW_OP_aspace_bregx`               | TBA   | 2                  | ULEB128 register number, ULEB128 byte displacement
> `DW_OP_aspace_implicit_pointer`    | TBA   | 2                  | 4-byte or 8-byte offset of DIE, SLEB128 byte displacement

After Section 8.13 "Address Class Encodings" add the following section:

> 8.x Address Space Encodings
> 
> The value of the common address space encoding `DW_ASPACE_none` is 0.

In Section 8.23 "Call Frame Information", add the following rows to Table 8.29
"Call frame instruction encodings":

> Instruction                | High 2 Bits | Low 6 Bits | Operand 1        | Operand 2        | Operand 3
> ------------------------   | ----------- | ---------- | ---------------- | ---------------- | ---------------------
> `DW_CFA_def_aspace_cfa`    | 0           | TBA        | ULEB128 register | ULEB128 offset   | ULEB128 address space
> `DW_CFA_def_aspace_cfa_sf` | 0           | TBA        | ULEB128 register | SLEB128 offset   | ULEB128 address space

In Section 8.31 "Type Signature Computation", Table 8.32 "Attributes used in
type signature computation", add the following attribute in alphabetical order
to the list:

> `DW_AT_address_space`

In Appendix A "Attributes by Tag Value (Informative)", add the following to
Table A.1 Attributes by tag value":

> Table A.1: Attributes by tag value
> 
> Tag Name                       | Applicable Attributes
> ------------------------------ | ---------------------
> `DW_TAG_pointer_type`          | `DW_AT_address_space`
> `DW_TAG_reference_type`        | `DW_AT_address_space`
> `DW_TAG_rvalue_reference_type` | `DW_AT_address_space`
