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
language type which may apply to objects allocated in different kinds
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

While some hardware's memory controllers may implement different
address spaces by swizzling pointers, it cannot be assumed that this
is a universal hardware design. Different GPU designs could have a
completely separate connection to a particular pool of memory and have
a completely different numbering system to reference it. On the other
hand there is nothing in this design which precludes hardware from
using swizzled pointers to implement address spaces.

If address spaces were implemented by a consumer by swizzling
pointers, then there would likely need to be reserved ranges of
addresses defined in the ABI which could interfere with other
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

    DW_OP_lit0 ; DW_AS_default is by definition 0
    DW_OP_const<N>u X
    DW_OP_mem

The address space being a stack parameter may be less obvious but at
least one language, OpenCL, allows the address space to be computed
and so it was determined that it was best to make this a stack
parameter as well.

Implicit conversion back to a value is limited only to the default
address space to maintain compatibility with DWARF 5. This approach
of extending memory location to support address spaces, allows all
existing DWARF 5 expressions to have the identical semantics.

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
other operators which refer to address spaces and making the old
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

In Section 2.11 "Address Classes and Address Spaces", remove the last
paragraph and replace it with the following paragraphs:

> Any debugging information entry representing a pointer or
> reference type may also have a `DW_AT_address_space` attribute,
> whose value is a non-negative integer constant which identifies
> the address space to which the pointer refers. The value
> `DW_ASPACE_default` identifies the default address space; other
> values and their uses are assigned by the ABI committee for the
> target.
> 
> Address space identifiers are also used by the DWARF
> operations `DW_OP_mem` (see Section 3.7), `DW_OP_aspace_bregx` (see
> Section 3.7), and `DW_OP_aspace_deref*` (see Section 3.13).

In Section 3.1 DWARF Expression Evaluation Context in point 5 Current
thread change the second paragraph to:

>    The current thread identifies a current thread of execution. By
>    extension, the current thread is also used by consumers to
>    identify which processor within a multi-processor target, a
>    thread is executing on. The processor that a thread is executing
>    on determines which instance of a register to refer to, and when
>    a target has address spaces which are local to a particular
>    processor, it defines which instance of that address space it
>    should refer to.
>
>    When debugging a multi-threaded program, the current thread may
>    be selected by a user command that focuses on a specific thread,
>    or it may be selected automatically when the running thread stops
>    at a breakpoint.

Then after the last paragraph add:

>    On a multi-processor target a current thread is required to
>    identify which instance of a register any register operation is
>    referring to.
>
>    On multi-processor targets that support address spaces which are
>    local to a processor or a thread, a current thread is required to
>    identify the instance of the address space that a memory
>    operation refers to.

In Section 3.7 "Memory Locations", add the following at the end of the
first paragraph:

>    If not specified, the storage associated with a memory location
>    defaults to `DW_ASPACE_default`, the name for the default
>    address space.

After the definition of `DW_OP_addrx` add:

>    3. `DW_OP_mem`
>
>       ![DW_OP_mem](../images/issue-260127-1/op-mem2.png)
>
>        `DW_OP_mem` pops top two stack entries, an address A and an
>    address space identifier AS. The address A must be an integral
>    value which represents a valid offset into the address space
>    AS. The address space AS must be an integral type value that
>    represents a target architecture specific address space
>    identifier.
>
>        *`DW_OP_addr(X)` is a more compact form of `DW_OP_lit0;
>    DW_OP_constNu(X); DW_OP_mem`.*
>
>        The address space identifier value AS must be one of the
>    values defined by the architecture's ABI.

After the definition of `DW_OP_bregx` add:

>    7. `DW_OP_aspace_bregx` (udata R, sdata B)
>
>       ![DW_OP_aspace_bregx](../images/issue-260127-1/op-aspace-bregx.png)
>
>        `DW_OP_aspace_bregx` has two immediate operands. The first is a ULEB
>    integer that represents a register number R. The second is a
>    SLEB integer that represents a byte displacement B. It
>    pops one stack entry that is required to be an integral type
>    value that represents a target architecture specific address
>    space identifier AS.
>
>        The action is the same as for `DW_OP_bregx`, except that AS is
>    used as the address space identifier.
>
>        The address space identifier value AS must be one of the
>    values defined by the architecture's ABI.
>
>        *Target architectures are encouraged to define `DW_ASPACE_*`
>    constants for their address spaces.*

In section 3.13, rename `DW_OP_xderef*` to `DW_OP_aspace_deref*` and
note that `DW_OP_xderef*` is still available as an alias.

Change the description of `DW_OP_aspace_deref` to

>    `DW_OP_aspace_deref` pops top two stack entries, an address A and
>    an address space identifier AS. The address A must be an integral
>    value which represents the offset into the address space AS. The
>    address space AS must be an integral type value that represents a
>    target architecture specific address space identifier. A data
>    item whose size is the size of the generic type is retrieved from
>    the memory location L whose address space is AS and whose address
>    is A.

Change the description of `DW_OP_aspace_deref_size` to:

>    `DW_OP_aspace_deref_size` takes a single 1-byte unsigned integral
>    operand that specifies the size S, in bytes, of the value to be
>    retrieved. It pops the top two stack entries, an address A and an
>    address space identifier AS. The address A must be an integral
>    value which represents the offset into the address space AS. The
>    address space AS must be an integral type value that represents a
>    target architecture specific address space
>    identifier. `DW_OP_aspace_deref_size` behaves like
>    `DW_OP_aspace_deref` except a data item whose size is S rather
>    than the size of a generic type is retrieved from the memory
>    location L whose address space is AS and whose address is A.

Change the description of `DW_OP_aspace_deref_size` to:

>    `DW_OP_aspace_deref_type` takes two operands. The first operand
>    is a 1-byte unsigned integer that specifies the byte size S of
>    the type given by the second operand. The second operand is an
>    ULEB integer that represents the offset of a debugging
>    information entry in the current compilation unit, which must be
>    a DW_TAG_base_type entry that provides the type T of the value to
>    be retrieved. The size S must be the same as the byte size of the
>    base type represented by the type T. It pops the top two stack
>    entries, an address A and an address space identifier AS. The
>    address A must be an integral value which represents the offset
>    into the address space AS. The address space AS must be an
>    integral type value that represents a target architecture
>    specific address space identifier. `DW_OP_aspace_deref_type`
>    behaves like `DW_OP_aspace_deref` except a data item whose size
>    is S rather than the size of a generic type is retrieved from the
>    memory location L whose address space is AS and whose address is
>    A and pushed onto the stack as a value of type T.

In Section 6.3 "Type Modifier Entries", after the paragraph starting
"A modified type entry describing a pointer or reference type...", add
the following paragraph:

>    A modified type entry describing a pointer or reference type
>    (using `DW_TAG_pointer_type`, `DW_TAG_reference_type` or
>    `DW_TAG_rvalue_reference_type`) may have a `DW_AT_address_space`
>    attribute with a constant value AS representing an architecture
>    specific DWARF address space (see 2.12 "Address Spaces"). If
>    omitted, this defaults to `DW_ASPACE_default`. The address space
>    of a memory location which is not in the default address space is
>    set as if the expression `DW_OP_constu` AS; `DW_OP_mem` were
>    evaluated for that instance of the variable.

In Section 7.1.1.1 "Contents of the Name Index", replace the bullet:

>    * `DW_TAG_variable` debugging information entries with a
>      `DW_AT_location` attribute that includes a `DW_OP_addr` or
>      `DW_OP_form_tls_address` operator are included; otherwise, they
>      are excluded.

with:

>    * `DW_TAG_variable` debugging information entries with a
>       `DW_AT_location` attribute that includes a `DW_OP_addr`<ins>,
>       `DW_OP_mem`,</ins> or `DW_OP_form_tls_address` operator are
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

After Section 8.13 "Address Class Encodings" add the following
section:

>    8.x Address Space Encodings
>
>    The value `DW_ASPACE_default` is 0, which identifies the default
>    address space.

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

[260617.1]: https://dwarfstd.org/issues/260617.1.html
