# General Support for Address Spaces

## PROBLEM DESCRIPTION

GPUs need to be able to describe addresses that are in different pools
of memory that are not part of the system's normal memory pool. These
memory pools can often be quite different than the system's normal
memory address space and this requires that they be treated
differently.

## How GPU memory addresses are different

While the system's normal address space is universally accessible and
therefore context independent memory, GPU memory is often context
dependent. The definition of this context has already been accepted
into the DWARF6 standard with issue [241011.1 Expression Evaluation
Context][242011.1]. Unlike system
memory where an address like 0x1000 refers to the same location
independent of the evaluation context, GPU memory pools can be local
to a GPU's processing unit and every processing unit may have its own
memory pool. A couple of examples of this are: [Intel's Shared Local
Memory][shared-local-mem] and [AMD's LDS][lds].
Thus, within a specific address space, the address 0x1000 may
reference different bits depending on which context it is evaluated
from. This is very similar to registers where every processor has its
own set of registers and the context disambiguates which one the
consumer should refer to. More formally when a pool of memory is
context dependent each address space defines a locality scope and
locations within that address space are bound to a particular instance
of storage with that scope.

[242011.1]: https://dwarfstd.org/issues/241011.1.html
[shared-local-mem]: https://www.intel.com/content/www/us/en/docs/oneapi/optimization-guide-gpu/2025-2/shared-local-memory.html
[lds]: https://rocm.docs.amd.com/projects/HIP/en/develop/understand/hardware_implementation.html#local-data-share-lds

There is no requirement that a particular set of bits is only
accessible through only one address space. In fact, it is common for a
GPU to provide access to the same bits through different address
spaces. This allows uses of memory locations within alternative
address spaces that are quite different than on CPUs. In some GPUs
some address spaces have embedded access patterns. Compilers often use
one access pattern for serial code while another is used for
vectorized code. If this access pattern were not conceptually
abstracted into an address space, then the compiler would have to
generate DWARF expressions that would undo the work done by the
addressing mode in order to point the consumer to the correct
location. This would expand the size of the DWARF generated and it
would make the job of generating debuginfo harder for the
producer. Another common example of an embedded access pattern is
strided memory.

These GPU memories can also have address widths that are different
than the system memory addresses. It is currently common for an
address in GPU memory to be 32b long while a system's memory address
is 64b.

## An address as a location rather than as a value

DWARF 5 has the concept of an address in many expressions but does not
define how it relates to address spaces. For example,
`DW_OP_push_object_address` pushes the address of an object. Other
contexts implicitly push an address on the stack before evaluating an
expression. For example, the `DW_AT_use_location` attribute of the
`DW_TAG_ptr_to_member_type`. The expression belongs to a source
language type that may apply to objects allocated in different kinds
of storage. Therefore, it is important that an expression that uses
the location can do so without regard to what kind of storage it
specifies. As it applies to memory location this means that a simple
value is insufficient. The address space of a memory location
description is also required. For example, a pointer to member value
may want to be applied to an object that may reside in any address
space.

### Why `DW_OP_xderef` is not sufficient

The DWARF 5 `DW_OP_xderef*` operations allow a value to be converted
into an address within a specified address space which is then
read. But it provides no way to create a memory location for an
address in the non-default address space. For example, GPU variables
can be allocated in local scratch pad memory at a fixed address.

### Address spaces are not address classes

Address class is an attribute of a pointer type that distinguishes
between different representations of a pointer; e.g., NEAR and FAR
pointers from old 16-bit x86 code. The attribute affects the size and
interpretation of the bits of the pointer, but the pointer still
refers to the "default" address space of the target.

The attribute has also been used for subprograms and subroutine types,
to specify what addressing mode should be used to call the subprogram.

While it is similar to address classes in that a pointer type for one
address space may be a different size from that for another address
space, the address space is not part of the pointer value, but is
implicit in the type of the pointer. The bits of the pointer refer to
an address in a completely separate address space.

### Address spaces are not just swizzled pointers

While some target's memory controllers may implement different address
spaces by what is sometimes called "swizzling pointers" (e.g., using
otherwise unused bits of the address to encode the address spaces), it
cannot be assumed that this is a universal hardware design. Different
GPU designs could have a completely separate connection to a
particular pool of memory and have a completely different numbering
system to reference it. On the other hand, there is nothing in this
design that precludes hardware from using swizzled pointers to
implement address spaces.

If address spaces were implemented by a consumer by swizzling
pointers, then there would likely need to be reserved ranges of
addresses defined in the ABI that could interfere with other
addresses in the system memory. That would require the consumer to
have special treatment for such values. Furthermore, purely swizzled
pointers do not capture the semantic differences of address spaces
like context dependence, pointer size, or embedded access patterns.

## Proposed solution

Add a new attribute `DW_AT_address_space` to pointer and reference
types.  This allows the compiler to specify which address space is
being used to encode the value of the pointer or reference type.

Add a new `DW_OP_mem` operation defined to create a memory location
description from an address and address space. It can be used to
specify the location of a variable that is allocated in a specific
address space.

Unlike `DW_OP_addr` which takes the address as an inline operand,
`DW_OP_mem` takes both of its parameters from the stack. This allows
them both to be dynamically computed. The case for the address being
dynamically computed is fairly obvious. One unobvious use is to allow
the address to be relocated when it is supplied using `DW_OP_constx`.

`DW_OP_mem` can be a thought of as a more general form of `DW_OP_addr`
with `DW_OP_addr(X)` being a short hand form of:

    DW_OP_lit0 ; DW_ASPACE_default is by definition 0
    DW_OP_const<N>u X
    DW_OP_mem

The address space being a stack parameter may be less obvious but at
least one language, OpenCL, allows the address space to be computed
and so it was determined that it was best to make this a stack
parameter as well.

Implicit conversion back to a value is limited only to the default
address space to maintain compatibility with DWARF 5. A very common
idiom in previous versions of DWARF would be:

    DW_OP_addr 0xf00
    DW_OP_lit1
    DW_OP_plus

In previous versions DWARF, `DW_OP_addr` pushed a generic-type stack
value. In DWARF 6, `DW_OP_addr` pushes a location. `DW_OP_plus` pops
two VALUEs from the stack.  If we didn't have the backward
compatability rule, this common DWARF expression would be invalid in
DWARF 6.

A similar problem arises with `DW_OP_push_object_address`:

    DW_OP_push_object_address  # pushes generic-type value in DWARF 5
    DW_OP_lit1
    DW_OP_plus

In DWARF 6:

    DW_OP_push_object_location # pushes location in DWARF 6
    DW_OP_lit1
    DW_OP_plus

Still works, provided the object location is global memory, which was
the only case possible in DWARF 5. This approach of extending memory
location to support address spaces, allows all existing DWARF 5
expressions to have the identical semantics. However the correct DWARF
6 expression that does not rely on implicit conversion would be:

    DW_OP_push_object_location
    DW_OP_lit1
    DW_OP_offset

Unlike `DW_OP_plus` which pops two VALUEs, `DW_OP_offset` pops a
LOCATION and a VALUE. Therefore the expression still works even if the
object location is another address space rather than in global memory
or even if the object's location is not in memory at all.

Having the address space included as part of the memory location as
opposed to being separate value, fixes one of the problems with
`DW_OP_xderef` where the address space is a separate value from the
address. This requires producers and consumers to manage two values
rather than one location.

Address spaces need not uniquely reference bits. The same bits may be
accessible through multiple address spaces. This allows great
implementation freedom. For example, a compiler could choose to access
private memory when mapping a source language thread to the lane of a
wavefront in a SIMT manner. Or a compiler could choose to access the
same private memory by mapping the same language construct with a
wavefront being the thread. It also allows the compiler to mix the
address space it uses to access private memory. For example, the
compiler can still spill entire vector registers to an address space
without a strided access pattern, while accessing the same bits for
SIMT variable access using the strided address space.

Having address spaces within the location also makes it available to
consumers and therefore developers. This can also be an aid to
programmers who are seeking to optimize their code.

Since memory locations are an abstraction of storage, the same set of
operations can operate on memory locations independent of their
underlying storage. Therefore, operations like `DW_OP_deref*` can be
used on any memory locations even those referring to different address
spaces. Consequently, the `DW_OP_xderef*` operations are unnecessary,
except as a more compact way to encode a non-default address space
address followed by dereferencing it. Therefore we propose renaming
those operators to `DW_OP_aspace_deref*` to be consistent with the
other operators that refer to address spaces and making the old
`DW_OP_xderef` names aliases.

### Scope of proposal

The scope of this proposal is limited to introducing address spaces to
memory locations. Additional changes are needed to support address
spaces in CFI. However, they are not included in this proposal. Those
changes will be presented in a subsequent proposal.

## PROPOSAL

In Section 2.2 "Attribute Types", add the following row to Table 2.2
"Attribute names":

Table 2.2: Attribute names

| Attribute | Usage |
| :---- | :---- |
| `DW_AT_address_space` | Architecture specific address space (see 2.12 "Address Spaces") |

In Section 2.11 "Address Classes and Address Spaces",
add an additional point to the list of "Examples of alternate address
classes":

> *Some systems support different classes of addresses. The address
> class may affect the representation of a pointer and how it is
> dereferenced. Normally, a pointer is represented as an address in
> the default memory address space, where the memory can be
> accessed by standard load and store instruction. On some systems
> with segmented memory models or global virtual address spaces, an
> alternative pointer representation may consist of a separate
> segment identifier and an offset within that segment.*
>
> *Examples of alternate address classes include:*
>
> - *x86 “far” pointers, which consist of a 16-bit segment identifier
> and either a 16- or 32-bit offset within the segment.
> Dereferencing a far pointer requires moving the segment
> identifier to a free segment register and using a segment
> override for the memory access.*
>
> - *PA-RISC “long” pointers, which consist of a 32-bit segment
> (termed “space” in PA-RISC documentation) identifier and a 32-bit
> offset. Dereferencing a long pointer requires moving the space
> identifier to a free space register and using the explicit space
> register number on the load or store instruction.*
>
> - <ins>*Tagged Pointers, where unused bits are repurposed to store other
> information.*</ins>

Revise the rest of 2.11 as follows:

> *Some systems may also support more than one memory address
> space. There is always a default address space, and the default size
> of a pointer corresponds to the size of the default address
> space. <del>The size of any other address space is not necessarily the
> same as the size of the default address space.</del>*

> <ins>*Address spaces are used when the addressing is independent or
> distinct and when the value of the address is not sufficient to
> unambiguously identify the storage being referenced. They are often
> used when the memory has some contextual locality.
> For example, every compute unit within a GPU may have its own local
> storage. A consumer may need to refer the current thread or lane to
> identify which instance of the address space to refer to.*</ins>
>
> <ins>*The size of any alternative address space is not necessarily the
> same as the size of the default address space. Consequently, the
> size of a pointer into an alternative address space is not
> necessarily the same size as a pointer into the target's default
> address space.*</ins>
>
> <ins>*Even though the addressing is independent or distinct, the storage
> referred to by different address spaces are not guaranteed to be
> independent of one another.
> For example, one address space might provide an alternate
> addressing scheme for the same storage as another address space.*</ins>
>
> *Examples of alternate address spaces include:*
>
> - *”Harvard” architectures with separate code and data spaces.*
>
> - *Embedded systems with a separate address space for flash memory.*
>
> - *Heterogeneous systems with local memory spaces for each GPU,
> or where large register files are mapped to their own address
> space.*
>
> Any debugging information entry representing a pointer or
> reference type may have a DW_AT_address_class attribute, whose
> value is an integer constant. The set of permissible values is
> specific to each target architecture. The value <del>DW_ADDR_none</del>
> <ins>DW_ACLASS_default</ins>, however, is common to all encodings, and means
> that no address class has been specified (that is, the pointer or
> reference type uses the standard encoding of an address on the
> target architecture).
>
> <ins>Any debugging information entry representing a pointer or reference
> type may have a `DW_AT_address_class` attribute, whose value is an
> integer constant. The set of permissible values is specific to each
> target architecture. The value `DW_ACLASS_default`, however, is common to
> all encodings, and means that no address class has been specified.</ins>
>
> <ins>Any debugging information entry representing a pointer or
> reference type may also have a `DW_AT_address_space` attribute,
> whose value is a non-negative integer constant that identifies
> the address space to which the pointer refers. The value
> `DW_ASPACE_default` identifies the default address space; other
> values and their uses are defined by the ABI.</ins>
>
> On multi-processor targets that support address spaces that are
> local to a processor or a thread, a current thread may be required
> to identify the instance of the address space that a memory
> operation refers to. Likewise on vectorized processors with address
> spaces that are local to a lane, a current lane may be required to
> identify the instance of the address space that a memory operation
> refers to.
>
> Address space identifiers are <ins>also</ins> used by the
> <del>DW_OP_xderef* operations</del>
> <ins>DWARF operations `DW_OP_mem` (see Section 3.7), `DW_OP_aspace_bregx` (see
> Section 3.7), and `DW_OP_aspace_deref*`</ins> (see Section 3.13).

In Section 3 DWARF Expressions insert the following paragraph after
the paragraph describing implicit conversion just before Section 3.1.

>    *Implicit conversion between a location and a value is provided
>    for backward compatability with previous versions of DWARF where
>    it was common to treat a VALUE as a memory address. Since this
>    conversion between a LOCATION and a VALUE assumes the location is
>    a memory location and strips the memory location of its
>    qualifying address space, this implicit conversion is limited to
>    memory locations in the default address space.*

In Section 3.1 DWARF Expression Evaluation Context,
change point 5 "Current thread" as follows:

> 5\. Current thread
>
>    *Many programming environments support the concept of independent
>    threads of execution, where the process and its address space are
>    shared among the threads, but each thread has its own stack,
>    program counter, and possibly its own block of memory for
>    thread-local storage (TLS). These threads may be implemented in
>    user-space or with kernel threads, or by a combination of the
>    two.*
>
>    The current thread identifies a current thread of execution. When
>    debugging a multi-threaded program, the current thread may be
>    selected by a user command that focuses on a specific thread, or
>    it may be selected automatically when the running thread stops at
>    a breakpoint.
>
>    <ins>[Make this normative]</ins>
>    If there is no current process (or an image of a process, as
>    from a core file), there is no current thread.
>
>    <ins>On a multi-processor target a current thread is required to
>    identify which instance of a register any register operation is
>    referring to.</ins>
>
>    <ins>*The current thread identifies a current thread of execution. By
>    extension, the current thread is also used by consumers to
>    identify which processor within a multi-processor target, a
>    thread is executing on. The processor that a thread is executing
>    on determines which instance of a register to refer to, and when
>    a target has address spaces that are local to a particular
>    processor, it defines which instance of that address space it
>    should refer to.*</ins>
>
>    <ins>On multi-processor targets that support address spaces that are
>    local to a processor or a thread, a current thread may be
>    required to identify the instance of the address space that a
>    memory operation refers to.</ins>
>
>    <ins>*When debugging a multi-threaded program, the current thread may
>    be selected by a user command that focuses on a specific thread,
>    or it may be selected automatically when the running thread stops
>    at a breakpoint.*</ins>
>
>    A current thread is required for the DW_OP_form_tls_location
>    operation (see Section 3.2 on page 49) which provides access to
>    thread-local storage.

In point 7, "Current lane", replace the third paragraph as follows:

>    The current lane is a SIMD/SIMT lane identifier. This applies to
>    source languages with scalar code that is vectorized by the
>    compiler using a SIMD/SIMT execution model. These implementations
>    map vectorized operations to SIMD/SIMT lanes of execution (see
>    Section 4.3.5.4 on page 102).
>
>    <ins>On SIMD/SIMT targets that support address spaces that are local
>    to a particular lane, a current lane may be required to identify
>    the instance of the address space that a memory operation refers
>    to.</ins>
>
>    <ins>[Change to non-normative]</ins>
>    *When debugging a SIMD/SIMT program, the current lane is
>    typically selected by a user command that focuses on a specific
>    lane.*

And add the following paragraph:

>    <ins>If there is no current process (or an image of a process, as
>    from a core file), there is no current lane.</ins>

In Section 3.7 "Memory Locations", add the following at the end of the
first paragraph:

>    A memory location represents the location of a piece or all of an
>    object or other entity in memory. On architectures that support
>    multiple address spaces, a memory location identifies storage
>    associated with the address space.
>    <ins>If not specified, the storage associated with a memory location
>    defaults to `DW_ASPACE_default`, the name for the default
>    address space.</ins>

After the definition of `DW_OP_addrx` add:

>    3. `DW_OP_mem`
>
>       ![DW_OP_mem](../images/issue-260127-1/op-mem2.png)
>
>        `DW_OP_mem` pops top two stack entries, an address A and an
>    address space identifier ASPACE. The address A must be an
>    integral value that represents a valid offset into the address
>    space ASPACE. The address space ASPACE must be an integral
>    value that represents a target architecture specific address
>    space identifier.
>
>        *`DW_OP_addr(X)` is a more compact form of `DW_OP_lit0;
>    DW_OP_constNu(X); DW_OP_mem`.*
>
>        The address space identifier value ASPACE must be one of the
>    values defined by the architecture's ABI.

After the definition of `DW_OP_bregx` add:

>    7. `DW_OP_aspace_bregx` (udata R, sdata B)
>
>       ![DW_OP_aspace_bregx](../images/issue-260127-1/op-aspace-bregx.png)
>
>        `DW_OP_aspace_bregx` has two immediate operands. The first is
>    a ULEB integer that represents a register number R. The second is
>    a SLEB integer that represents a byte displacement B. It pops one
>    stack entry that is required to be an integral type value that
>    represents a target architecture specific address space
>    identifier ASPACE.
>
>        The action is the same as for `DW_OP_bregx`, except that
>    ASPACE is used as the address space identifier.
>
>        The address space identifier value ASPACE must be one of the
>    values defined by the architecture's ABI.
>
>        *Target architectures are encouraged to define `DW_ASPACE_*`
>    constants for their address spaces.*

In section 3.13, rename `DW_OP_xderef*` to `DW_OP_aspace_deref*` and
note that `DW_OP_xderef*` is still available as an alias.

As a minor editorial clarification change "1-byte unsigned integer" in
`DW_OP_deref_type` to "1-byte unsigned integral operand" to match the
description of the operand used in `DW_OP_deref_size`.

Change the description of `DW_OP_aspace_deref` to

>    `DW_OP_aspace_deref` pops top two stack entries, an address A and
>    an address space identifier ASPACE. The address A must be an
>    integral value that represents the offset into the address space
>    ASPACE. The address space ASPACE must be an integral type value
>    that represents a target architecture specific address space
>    identifier. A data item whose size is the size of the generic
>    type is retrieved from the memory location L whose address space
>    is ASPACE and whose address is A. The retrieved data is pushed
>    onto the stack as a value of generic type.

Change the description of `DW_OP_aspace_deref_size` to:

>    `DW_OP_aspace_deref_size` takes a single 1-byte unsigned integral
>    operand that specifies the size S, in bytes, of the value to be
>    retrieved. It pops the top two stack entries, an address A and an
>    address space identifier ASPACE. The address A must be an
>    integral value that represents the offset into the address space
>    ASPACE. The address space ASPACE must be an integral type value
>    that represents a target architecture specific address space
>    identifier. `DW_OP_aspace_deref_size` behaves like
>    `DW_OP_aspace_deref` except a data item whose size is S rather
>    than the size of a generic type is retrieved from the memory
>    location L whose address space is ASPACE and whose address is
>    A. The data retrieved is zero extended to the size of an generic
>    type, and pushed onto the stack as a value of the generic type.

Change the description of `DW_OP_aspace_deref_type` to:

>    `DW_OP_aspace_deref_type` takes two operands. The first operand
>    is a 1-byte unsigned integral operand that specifies the byte
>    size S of the type given by the second operand. The second
>    operand is an ULEB integer that represents the offset of a
>    debugging information entry in the current compilation unit,
>    which must be a DW_TAG_base_type entry that provides the type T
>    of the value to be retrieved. The size S must be the same as the
>    byte size of the base type represented by the type T. It pops the
>    top two stack entries, an address A and an address space
>    identifier ASPACE. The address A must be an integral value that
>    represents the offset into the address space ASPACE. The address
>    space ASPACE must be an integral type value that represents a
>    target architecture specific address space
>    identifier. `DW_OP_aspace_deref_type` behaves like
>    `DW_OP_aspace_deref` except a data item whose size is S rather
>    than the size of a generic type is retrieved from the memory
>    location L whose address space is ASPACE and whose address is A
>    and pushed onto the stack as a value of type T.

In Section 6.3 "Type Modifier Entries", after the paragraph starting
"A modified type entry describing a pointer or reference type...", add
the following paragraph:

>    A modified type entry describing a pointer or reference type
>    (using `DW_TAG_pointer_type`, `DW_TAG_reference_type` or
>    `DW_TAG_rvalue_reference_type`) may have a `DW_AT_address_space`
>    attribute with a constant value ASPACE representing an
>    architecture specific DWARF address space (see 2.11 "Address
>    Classes and Adress Spaces"). If omitted, this defaults to
>    `DW_ASPACE_default`.

In Section 7.1.1.1 "Contents of the Name Index", replace the bullet:

>    * `DW_TAG_variable` debugging information entries with a
>      `DW_AT_location` attribute that includes a `DW_OP_addr` or
>      `DW_OP_form_tls_address` operator are included; otherwise, they
>      are excluded.

with:

>    * `DW_TAG_variable` debugging information entries with a
>       `DW_AT_location` attribute that includes a `DW_OP_addr`,
>       `DW_OP_mem` or `DW_OP_form_tls_address` operator are
>       included; otherwise, they are excluded.

In Section 8.5.4 "Attribute Encodings", add the following row to Table
8.5 "Attribute encodings":

Table 8.5: Attribute encodings

>    | Attribute Name | Value | Classes |
>    | :---- | :---- | :---- |
>    | `DW_AT_address_space` | TBA | constant |

Add the following rows to table 8.9 in Section "8.7.1 Operator
Encodings":

>    | Operation | Code | Number of Operands | Notes |
>    | :---- | :---- | :---- | :---- |
>    | `DW_OP_mem`                 | TBA | 0 |  |
>    | `DW_OP_aspace_bregx` | TBA | 2 | ULEB register number, SLEB byte displacement |

In Section 8.13 Address Class Encodings, change `DW_ADDR_none` to
`DW_ACLASS_default` with a footnote stating that `DW_ADDR_none` will
continue to be an alias for `DW_ACLASS_default`

After Section 8.13 "Address Class Encodings" add the following
section:

>    8.x Address Space Encodings
>
>    The value that identifies the default address space,
>    `DW_ASPACE_default`, is 0.

In Section 8.31 "Type Signature Computation", Table 8.32 "Attributes
used in type signature computation", add the following attribute in
alphabetical order to the list:

> `DW_AT_address_space`

In Appendix A "Attributes by Tag Value (Informative)", add the
following to Table A.1 Attributes by tag value":

Table A.1: Attributes by tag value

>    | Tag Name | Applicable Attributes |
>    | :---- | :---- |
>    | `DW_TAG_pointer_type` | `DW_AT_address_space` |
>    | `DW_TAG_reference_type` | `DW_AT_address_space` |
>    | `DW_TAG_rvalue_reference_type` | `DW_AT_address_space` |

---

2026-04-13: Revised based on suggestions from Ron B.

2026-05-19: Further [revisions][diff1]: Remove completely old
Section 2.11 on Address Classes.

2026-06-16: [Revised][diff2] based on 6/8/26 meeting.
Restored Section 2.11 Address Classes.
Fixed definition of `DW_OP_aspace_bregx`.

2026-06-18: [Revised][diff3] to address discussion since 6/8/26 meeting.
Now depends on [260617.1][260617.1].
Removed addition of Section 2.12; added extra paragraphs to Section 2.11.
Added text to Section 3.1 DWARF Expression Evaluation Context, "current thread".
Rewrote description of `DW_OP_mem`; removed discussion of context and truncation.
Added new descriptions for `DW_OP_aspace_deref*`.
Shortened new text in Section 6.3 Type Modifier Entries.

2026-07-02: [Revised][diff4] to address discussion in 6/22/26 meeting.

2026-07-15: [Revised][diff5] to address discussion in 7/6/26 meeting
and subsequent email discussions.

[260617.1]: https://dwarfstd.org/issues/260617.1.html
