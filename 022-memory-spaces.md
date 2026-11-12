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
We propose to add new type modifier entries, similar to the existing
entries such as `DW_TAG_volatile_type` and `DW_TAG_restrict_type`,
that decorate a type with a memory space qualifier.  The example above
would be described as follows:

    $1 DW_TAG_variable
         DW_AT_name       ("ptr")
         DW_AT_type       ($2)
         DW_AT_location   (...)

    $2 DW_TAG_local_type
         DW_AT_type       ($3)

    $3 DW_TAG_pointer_type
         DW_AT_type       ($4)

    $4 DW_TAG_private_type
         DW_AT_type       ($5)

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
    can be used in the source program for annotating a type.  DWARF can
    express memory spaces by using `DW_TAG_global_type`,
    `DW_TAG_constant_type`, `DW_TAG_local_type`, and `DW_TAG_private_type`
    modifiers (see 6.3 Type Modifier Entries), one for each OpenCL memory
    space qualifier.  For an example, see D.21 Memory Space Example.

    Note that memory spaces are specified at the source language level,
    whereas address spaces that are expressed with the `DW_AT_address_space`
    attribute and DWARF operators like `DW_OP_mem` are a target-level
    concept (see 2.11 Address Classes and Address Spaces).  If and how a
    memory space would map to an address space depends on compiler choices
    and target properties.

In Table 2.1 "Tag names", add `DW_TAG_global_type`, `DW_TAG_constant_type`,
`DW_TAG_local_type`, and `DW_TAG_private_type`.

In Table 6.3 "Type modifier tags", add the following rows:

    DW_TAG_global_type   | global qualified type (for example, in OpenCL)
    DW_TAG_constant_type | constant qualified type (for example, in OpenCL)
    DW_TAG_local_type    | local qualified type (for example, in OpenCL)
    DW_TAG_private_type  | private qualified type (for example, in OpenCL)

Extend Figure 6.1 "Type modifier examples" with the following:

    Consider the OpenCL declaration below:

        private int *local ptr;

    This can be encoded in DWARF as

        DW_TAG_variable(ptr) -->
          DW_TAG_local_type -->
            DW_TAG_pointer_type -->
              DW_TAG_private_type -->
                DW_TAG_base_type(int)

In Table 8.3 "Tag encodings", add the following rows:

    DW_TAG_global_type   | 0x...
    DW_TAG_constant_type | 0x...
    DW_TAG_local_type    | 0x...
    DW_TAG_private_type  | 0x...

In Table A.1 "Attributes by tag value", add the following rows:

    DW_TAG_global_type   | DECL
                         | DW_AT_name
                         | DW_AT_type

    DW_TAG_constant_type | DECL
                         | DW_AT_name
                         | DW_AT_type

    DW_TAG_local_type    | DECL
                         | DW_AT_name
                         | DW_AT_type

    DW_TAG_private_type  | DECL
                         | DW_AT_name
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

    $2 DW_TAG_local_type
         DW_AT_type       ($3)

    $3 DW_TAG_pointer_type
         DW_AT_type       ($4)

    $4 DW_TAG_private_type
         DW_AT_type       ($5)

    $5 DW_TAG_base_type
         DW_AT_name      ("int")
         DW_AT_byte_size (4)
         DW_AT_encoding  (DW_ATE_signed)
