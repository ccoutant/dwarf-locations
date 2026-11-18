# OpenCL Address Space Type Qualifiers

OpenCL defines named address space type qualifers as explained at

  https://registry.khronos.org/OpenCL/specs/unified/refpages/man/html/addressSpaceQualifiers.html

For example, one can write in OpenCL

    private int *local ptr;

This is read from "right to left", similar to how `const` and
`volatile` qualifiers in C/C++ are read: `ptr` is in the `local`
address space and is a pointer to `int` in the `private` address
space.  We propose adding new type modifier entries in DWARF, similar
to the existing entries such as `DW_TAG_volatile_type` and
`DW_TAG_restrict_type`, so that OpenCL address space qualifiers can be
expressed.  The example above would be described as follows:

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

*Note:* OpenMP has memory spaces that "represent storage resources
  where variables can be stored and retrieved" (see Section 8.1 in
  [OpenMP Spec][openmp]).  These are, however, API level constructs
  and do not appear in the source language type system.  OpenMP and
  OpenACC have data-sharing and initialization clauses such as
  `private` and `firstprivate`, but these are directives
  (i.e. pragmas) that also don't appear in the source language type
  system.  Therefore the proposal we make here does not apply to
  OpenMP or OpenACC.  For data-sharing and initialization support in
  DWARF, see [Issue 230530][issue].

[openmp]: https://www.openmp.org/wp-content/uploads/OpenMP-API-Specification-6-0.pdf
[issue]: https://dwarfstd.org/issues/230530.1.html

# PROPOSAL

In Table 2.1 "Tag names", add `DW_TAG_global_type`, `DW_TAG_constant_type`,
`DW_TAG_local_type`, and `DW_TAG_private_type`.

In Table 6.3 "Type modifier tags", add the following rows:

    DW_TAG_global_type   | global qualified type (for example, in OpenCL)
    DW_TAG_constant_type | constant qualified type (for example, in OpenCL)
    DW_TAG_local_type    | local qualified type (for example, in OpenCL)
    DW_TAG_private_type  | private qualified type (for example, in OpenCL)

At the end of section 6.3, before "Examples of modified types...", add
the following paragraph:

    Type modifiers for OpenCL address space qualifiers are a source
    language level concept separate from the target-level address space
    concept (see 2.11 Address Classes and Address Spaces).  If and how
    OpenCL type qualifiers map to target-level address spaces depends on
    compiler choices and target properties.

Extend Figure 6.1 "Type modifier examples" with the following:

    Consider the OpenCL declaration below:

        private int *local p;

    This can be encoded in DWARF as

        DW_TAG_variable(p) -->
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

    D.21 OpenCL Address Space Type Qualifier Example

    The OpenCL declaration

        private int *local ptr;

    can be encoded in DWARF as

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
