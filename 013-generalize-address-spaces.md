# General Support for Address Spaces

## PROBLEM DESCRIPTION

GPUs need to be able to describe addresses that are in different
pools of memory that are not part of the system's normal memory
pool. These memory pools have their own address spaces and often have
addresses which are a different size than the system's default size.

DWARF 5 has the concept of an address in many expressions but does
not define how it relates to address spaces. For example,
`DW_OP_push_object_location` pushes the address of an object. Other
contexts implicitly push an address on the stack before evaluating an
expression. For example, the `DW_AT_use_location` attribute of the
`DW_TAG_ptr_to_member_type`. The expression belongs to a source
language type which may apply to objects allocated in different kinds
of storage. Therefore, it is desirable that an expression that uses
the address can do so without regard to what kind of storage it
specifies, including the address space of a memory location
description. For example, a pointer to member value may want to be
applied to an object that may reside in any address space.

### Why DW_OP_xderef is not sufficient

The DWARF 5 `DW_OP_xderef*` operations allow a value to be converted
into an address within a specified address space which is then
read. But it provides no way to create a memory location description
for an address in the non-default address space. For example, GPU
variables can be allocated in the local address space at a fixed
address.

**Insert basic example of OMP mapping to GPU memory.

## Proposed solution
A new attribute `DW_AT_address_space` is added to pointer and
reference types.  This allows the compiler to specify which address
space is being used to represent the pointer or reference type.

A `DW_OP_mem` operation is added and defined to create a memory
location description from an address and address space. It can be used
to specify the location of a variable that is allocated in a specific
address space. Since a memory location within a target's address space
is known to be a certain length, this allows to be an allows the size
of addresses within an address space to be potentially larger than the
generic type. While no GPUs currently use 128b addressing, GPUs
frequently use a kind of pointer swizzling to simplify the design of
their memory management units and it is reasonably foreseeable that in
the not too distant future some addresses may be larger than the host
target's address and thus larger than their generic type.

Having the address within the location also allows a consumer great
implementation freedom. Implicit conversion back to a
value is  limited only to the default address space to maintain
compatibility with DWARF 5. 

In contrast, if the `DW_OP_mem` operation had been defined to produce
a value, and an implicit conversion to a memory location description
was defined, then it would be limited to the size of the generic type
(which matches the size of the default address space). An
implementation would likely have to use _reserved ranges_ of value to
represent different address spaces. Such a value would likely not
match any address value in the actual hardware. That would require the
consumer to have special treatment for such values.

This approach of extending memory location descriptions to support
address spaces, allows all existing DWARF 5 expressions to have the
identical semantics.  It allows the compiler to explicitly specify the
address space it is using. For example, a compiler could choose to
access private memory in a swizzled manner when mapping a source
language thread to the lane of a wavefront in a SIMT manner. Or a
compiler could choose to access it in an unswizzled manner if mapping
the same language with the wavefront being the thread.

It also allows the compiler to mix the address space it uses to access
private memory. For example, for SIMT it can still spill entire vector
registers in an unswizzled manner, while using a swizzled private
memory for SIMT variable access.

This approach also allows memory location descriptions for different
address spaces to be components of composite storage.

Since location descriptions are an abstraction of storage. The same
set of operations can operate on locations independent of their
underlying storage. The `DW_OP_deref*` therefore can be used on any
storage kind, including memory location descriptions referring to
different address spaces. Therefore, the `DW_OP_xderef*` operations
are unnecessary, except as a more compact way to encode a non-default
address space address followed by dereferencing it. Therefore we
propose renaming those operators to `DW_OP_aspace_deref*` and making
the old `DW_OP_xderef` names alias.

## PROPOSAL

In Section 2.2 "Attribute Types", add the following row to Table 2.2
"Attribute names":

Table 2.2: Attribute names

> Attribute             | Usage
> --------------------- | --------------------------------------------------------------
> `DW_AT_address_space` | Architecture specific address space (see 2.x "Address Spaces")

Add the following after Section 2.11 "Address Classes":

> 2.12 Address Spaces
>
> DWARF address spaces correspond to target architecture specific
> linear addressable memory areas. They are used in DWARF expression
> location descriptions to describe in which target architecture
> specific memory area data resides.
>
> * Target architecture specific DWARF address spaces
> may correspond to hardware supported facilities such as memory
> utilizing base address registers, scratchpad memory, and memory with
> special interleaving.  The size of addresses in these address spaces
> may vary. Their access and allocation may be hardware managed with
> each thread or group of threads having access to independent
> storage. For these reasons they may have properties that do not
> allow them to be viewed as part of the unified global virtual
> address space accessible by all threads._*
>
> * It is target architecture specific whether multiple
> DWARF address spaces are supported and how source language memory
> spaces map to target architecture specific DWARF address spaces. A
> target architecture may map multiple source language memory spaces
> to the same target architecture specific DWARF address
> class. Optimization may determine that variable lifetime and access
> pattern allows them to be allocated in faster scratchpad memory
> represented by a different DWARF address space than the default for
> the source language memory space._*
>

> Although DWARF address space identifiers are target architecture
> specific, `DW_ASPACE_none` is a common address space supported by
> all target architectures, and defined as the target architecture
> default address space.
>

**For Discussion** Do we really want to call it DW_ASPACE_none or
should we call it something like DW_ASPACE_default.

> DWARF address space identifiers are used by:
>
> * The `DW_AT_address_space` attribute.
>
> * The DWARF operations: `DW_OP_mem`, `DW_OP_aspace_bregx` and
>   `DW_OP_aspace_deref*`.

In Section 3.7 "Memory Locations", add the following at the
end of the first paragraph:

> `DW_ASPACE_none` is defined as the target architecture default address space.
> See 2.x Address Spaces.

After the definition of `DW_OP_addrx` add:

> 3.  `DW_OP_mem`
>      <[memory location] L> <[integral] AS> → <[memory location] SL>

**For Discussion** I made this pop a memory location off the stack
rather than expecting an integral type. Implicit conversion takes care
of converting this into a memory location if an integer is
supplied. Otherwise if it is a location, we just have to somehow get a
large enough number into that the location.

>
>     `DW_OP_mem` pops top two stack entries. The first must be an
>     integral type value that represents a target architecture
>     specific address space identifier AS. The second is a location
>     that represents an address A.
>
>     The address size S is defined as the address bit size of the target
>     architecture specific address space that corresponds to AS.
>
>     A is adjusted to S bits by zero extending if necessary, and then
>     treating the least significant S bits as an unsigned value A'.
>
>     It pushes a location description L with one memory location description
>     SL on the stack. SL specifies the memory location storage.
>
>     If AS is an address space that is specific to context elements,
>     then LS corresponds to the location storage associated with the
>     current context.
>
>     *For example, if AS is for per thread storage
>     then LS is the location storage for the current
>     thread. Therefore, if L is accessed by an operation, the
>     location storage selected when the location description was
>     created is accessed, and not the location storage associated
>     with the current context of the access operation.*
>
>     The DWARF expression is ill-formed if AS is not one of the values
>     defined by the target architecture specific `DW_ASPACE_*` values.
>

After the definition of `DW_OP_bregx` add:

> 7.  `DW_OP_aspace_bregx` ([ULEB128] R, [LEB128] B)
>     <[integral] AS> → <[location] A >
>
>     `DW_OP_aspace_bregx` has two operands. The first is an unsigned
>     LEB128 integer that represents a register number R. The second
>     is a signed LEB128 integer that represents a byte displacement
>     B. It pops one stack entry that is required to be an integral
>     type value that represents a target architecture specific
>     address space identifier AS.
>
>
>     The action is the same as for `DW_OP_breg`<N>, except that R is used as
>     the register number, B is used as the byte displacement, and AS is used as
>     the address space identifier.
>
>     The DWARF expression is ill-formed if AS is not one of the
>     values defined by the target architecture specific `DW_ASPACE_*`
>     values.

In section 3.13 Rename `DW_OP_xderef*` to `DW_OP_aspace_deref*`


In Section 6.3 "Type Modifier Entries", after the paragraph starting
"A modified type entry describing a pointer or reference type...", add
the following paragraph:

> A modified type entry describing a pointer or reference type (using
> `DW_TAG_pointer_type`, `DW_TAG_reference_type` or
> `DW_TAG_rvalue_reference_type`) may have a `DW_AT_address_space`
> attribute with a constant value AS representing an architecture
> specific DWARF address space (see 2.x "Address Spaces"). If omitted,
> this defaults to `DW_ASPACE_none`. DR is the offset of a
> hypothetical debug information entry D in the current compilation
> unit for an integral base type matching the address size of AS. An
> object P having the given pointer or reference type are dereferenced
> as if the `DW_OP_push_object_address`; `DW_OP_deref_type` DR;
> `DW_OP_constu` AS; `DW_OP_form_aspace_address` expression was
> evaluated with the current context except: the result kind is
> location description; the initial stack is empty; and the object is
> the location description of P.


In Section 7.1.1.1 "Contents of the Name Index", replace the bullet:

> * `DW_TAG_variable` debugging information entries with a `DW_AT_location`
>   attribute that includes a `DW_OP_addr` or `DW_OP_form_tls_address` operator
>   are included; otherwise, they are excluded.

with:

> * `DW_TAG_variable` debugging information entries with a `DW_AT_location`
>   attribute that includes a `DW_OP_addr`, `DW_OP_form_aspace_address`, or
>   `DW_OP_form_tls_address` operation are included; otherwise, they are
>   excluded.

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
> `DW_OP_mem                `        | TBA   | 0                  |
> `DW_OP_aspace_bregx`               | TBA   | 2                  | ULEB128 register number, ULEB128 byte displacement

After Section 8.13 "Address Class Encodings" add the following section:

> 8.x Address Space Encodings
>
> The value of the common address space encoding `DW_ASPACE_none` is 0.


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
