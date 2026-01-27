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

>    A location list consists of a sequence of zero or more bounded location
>    expression or base address entries, optionally followed by a default location
>    entry, and terminated by an end-of-list entry.

add the following:

>    Bounded or default location entries may optionally have preceeding refined
>    type entries.

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
