# Clarification: Add an example for DWARF6 vtable

## Introduction

How vtables are handled in DWARF was significantly changed without an
example in [250506.1: Improve Support for Finding
vtables](https://dwarfstd.org/issues/250506.1.html) [250506.2 Improve
Support for Downcasting
Objects](https://dwarfstd.org/issues/250506.2.html) [250506.3 Replace
DW_AT_vtable_elem_location Attribute with
DW_AT_vtable_elem_index](https://dwarfstd.org/issues/250506.3.html). One
of the reasons for this change was that current compilers had not
iomplemented DWARF5 correctly. To prevent future confusion during the
DWARF6 implementation phase, we propose adding an example.

## Proposal

Add a new example to Appendix D.

D.n Virtual Table Example

Consider the C++ class with a virtual table in Figure D.x

>    class MyClass {
>    public:
>        virtual void doSomething();
>        int myData; // 4 bytes
>    };

A DWARF representation for this example, with many details elided, is
shown in Figure D.y

>    DW_TAG_class_type
>        DW_AT_name        : "MyClass"
>        DW_AT_byte_size   : 16
>        DW_AT_vtable_location: (DW_OP_deref)
>
>        DW_TAG_member
>            DW_AT_name        : "myData"
>            DW_AT_type        : <Reference to 'int' type>
>            DW_AT_data_member_location: 8
>
>        DW_TAG_subprogram)
>            DW_AT_name        : "doSomething"
>            DW_AT_virtuality  : DW_VIRTUALITY_virtual
>            DW_AT_vtable_elem_index: 0
>            DW_AT_declaration : 1
>
>    DW_TAG_vtable)
>        DW_AT_artificial     : 1
>        DW_AT_location       : <Address of the vtable>
>        DW_AT_vtable_for_type: <reference to MyClass class>
