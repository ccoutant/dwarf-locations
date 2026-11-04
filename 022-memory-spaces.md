# PROBLEM DESCRIPTION

OpenCL defines source language memory spaces as explained at

  https://registry.khronos.org/OpenCL/specs/unified/refpages/man/html/addressSpaceQualifiers.html

For example, one can write in OpenCL

    private int *local ptr;

This is read from "right to left", similar to how `const` and
`volatile` qualifiers in C/C++ are read: `ptr` is in the `local`
memory space and is a pointer to `int` in the `private` memory space.

In [Issue 260617.1](https://dwarfstd.org/issues/260617.1.html) and
[Issue 260211.1](https://dwarfstd.org/issues/260211.1.html), address
space support in DWARF was clarified and generalized.  It is important
to note that address spaces discussed in those issues is a target
level concept, whereas the memory spaces in OpenCL are a source
language level concept.  To not cause terminology confusion, we
deliberately say "memory space" to refer to the source language
concept and "address space" to refer to the target level concept.  The
two are separate concepts: how a memory space maps to an address space
depends on compiler choices and target properties.

This proposal adds ability to DWARF to describe memory spaces, which
are type qualifiers, very similar to `const` and `volatile` in C/C++.
We propose to add a new type modifier entry,
`DW_TAG_memory_space_type`, that decorates a type with a memory space.
The example above would be described as follows:

    $1 DW_TAG_variable
         DW_AT_name       ("ptr")
         DW_AT_type       ($2)
         DW_AT_location   (...)

    $2 DW_TAG_memory_space_type
         DW_AT_type         ($3)
         DW_AT_memory_space (DW_MSPACE_local)

    $3 DW_TAG_pointer_type
         DW_AT_type         ($4)

    $4 DW_TAG_memory_space_type
         DW_AT_type         ($5)
         DW_AT_memory_space (DW_MSPACE_private)

    $5 DW_TAG_base_type
         DW_AT_name      ("int")
         DW_AT_byte_size (4)
         DW_AT_encoding  (DW_ATE_signed)

# PROPOSAL

Add the following after Section 2.11 "Address Classes and Address Spaces":

    ---------------------------------------------------------------------
    2.12 Memory Spaces

    In some languages, such as OpenCL, memory spaces can be specified at
    the source level as type qualifiers.  For example, OpenCL defines four
    qualifiers: `global`, `constant`, `local`, `private`.  These qualifiers
    can be used in the source program for annotating a type.  DWARF can express
    memory spaces by using the `DW_TAG_memory_space_type` modifier (see 6.3
    Type Modifier Entries) together with the `DW_AT_memory_space` attribute
    (see Section 8.15 Memory Space Encodings).  For an example, see D.21
    Memory Space Example.

In Table 2.1 "Tag names", add `DW_TAG_memory_space_type`.

In Section 2.2 "Attribute Types", add the following row to Table 2.2 "Attribute
names":

    DW_AT_memory_space       | Memory space qualified type entries (see 6.3
                             | "Type Modifier Entries")

In Table 6.3 "Type modifier tags", add the following row:

    DW_TAG_memory_space_type | type specified at a memory space (for example,
                             | in OpenCL)

Extend Figure 6.1 "Type modifier examples" with the following:

    Consider the OpenCL declaration below:

        private int *local ptr;

    This can be encoded in DWARF as

        DW_TAG_variable(ptr) -->
          DW_TAG_memory_space_type(local) -->
            DW_TAG_pointer_type -->
              DW_TAG_memory_space_type(private) -->
                DW_TAG_base_type(int)

In Table 8.3 "Tag encodings", add the following row:

    DW_TAG_memory_space_type | 0x...

**Note to DWARF committee: consider using the reserved value 0x14.**

In Table 8.5 "Attribute encodings", add the following row:

    DW_AT_memory_space(*) | 0x... | constant
    (*) New in DWARF Version 6

After Section 8.14 "Address Space Encodings", add

    Section 8.15 Memory Space Encodings

    The set of currently defined DWARF memory spaces is given in Table
    8.x "Source language memory spaces".  Vendor defined source
    language memory spaces may be defined using codes in the range
    `DW_MSPACE_lo_user` to `DW_MSPACE_hi_user`.

    Table 8.x: Source language memory spaces
    ================== ============
    Memory Space Name  Value
    ================== ============
    DW_MSPACE_default  0x00
    DW_MSPACE_global   0x01
    DW_MSPACE_constant 0x02
    DW_MSPACE_local    0x03
    DW_MSPACE_private  0x04
    DW_MSPACE_lo_user  0x80
    DW_MSPACE_hi_user  0xff
    ================== ============

In Table 8.33 "Attributes used in type signature computation", add
`DW_AT_memory_space`.

In Table A.1 "Attributes by tag value", add the following rows:

    DW_TAG_memory_space_type | DECL
                             | DW_AT_memory_space
                             | DW_AT_type

Add the following new subsection to Appendix D.

    D.21 Memory Space Example

    The OpenCL declaration

        private int *local ptr;

    which means "`ptr` is in the `local` memory space and is a
    pointer to `int` in the `private` memory space.", can be encoded in
    DWARF as

    $1 DW_TAG_variable
         DW_AT_name       ("ptr")
         DW_AT_type       ($2)
         DW_AT_location   (...)

    $2 DW_TAG_memory_space_type
         DW_AT_type         ($3)
         DW_AT_memory_space (DW_MSPACE_local)

    $3 DW_TAG_pointer_type
         DW_AT_type         ($4)

    $4 DW_TAG_memory_space_type
         DW_AT_type         ($5)
         DW_AT_memory_space (DW_MSPACE_private)

    $5 DW_TAG_base_type
         DW_AT_name      ("int")
         DW_AT_byte_size (4)
         DW_AT_encoding  (DW_ATE_signed)
