Standardize Operation Heading
=============================

Background
----------

This proposal is editorial in nature: it does not intend to change the
meaning of any DWARF constructs, but merely to standardize the
presentation of the existing DWARF operations to facilitate
understanding.

It is believed that tool authors referring to the the standard would
find it much easier to understand if each operand followed a standard
template, rather than explaining these in the textual description of
the operand. This issue proposes such a template and converts each
operand to that template.

DWARF operations potentially have three sources of input: inline
operands which are encoded in their byte stream, arguments on the
stack, and the context in which they are executed. They can also
modify the stack and leave items on the stack. For every operator each
of these sources of input, output, and side effects are specified in a
standardized format similar to how other stack machine operations have
been documented in the past.

Proposed Changes
----------------

In Chapter 3 replace the following paragraph:

> A DWARF expression is encoded as a stream of operations, each
> consisting of an opcode followed by zero or more literal operands. The
> number of operands is implied by the opcode.

With:

> A DWARF expression is encoded as a stream of operations, each
> operation consisting of an opcode followed by zero or more inline
> operands. The number of inline operands is implied by the opcode. It
> may also receive parameters from the stack and make use of
> information from its evaluation context.

> The description of each operation begins with a heading that shows
> the name of the operation and its inline operands, if any, as a
> C-like function prototype. Each inline operand has one of the
> following types:
>
> - ubyte (unsigned 1-byte value)
> - uhalf (unsigned 2-byte value)
> - uword (unsigned 4-byte value)
> - ulong (unsigned 8-byte value)
> - sbyte (signed 1-byte value)
> - shalf (signed 2-byte value)
> - sword (signed 4-byte value)
> - slong (signed 8-byte value)
> - uleb (unsigned LEB128 value)
> - sleb (signed LEB128 value)
> - block (a block of bytes, encoded as DW_FORM_block)
> - address (an address in the default memory address space)
> - offset2 (2-byte offset to a DIE)
> - offset4 (4-byte offset to a DIE)
> - offset (4- or 8-byte offset to a DIE)
>
> Following the operation heading is a stack diagram, showing the
> state of the stack before and after the operation, with the top of
> the stack at the top of the diagram, and the base of the stack shown
> as an open rectangle at the bottom of the diagram. In some cases,
> where the stack diagrams are the same for a group of operations, the
> diagram is given at the top of the section.
>
> example stack diagrams are illustrated in a separate [PDF
> attachment](https://dwarfstd.org/doc/Issue-260127-1-op-diagrams.pdf)
>
> *Symbolic names are often given to the stack arguments and the stack
> result when it is considered to be helpful or meaningful. In the
> cases where no sybolic name would be meaningful, the generic names A
> B C and D are used.*
>

In Section 3.2 add the following heading between operator and its
description as follows:

> `DW_OP_dup`
>
> ![DW_OP_dup](../images/issue-260127-1/op-dup.png)

> `DW_OP_drop`
>
> ![DW_OP_drop](../images/issue-260127-1/op-drop.png)

> `DW_OP_pick` ([1-byte unsigned] N)
>
> ![DW_OP_pick](../images/issue-260127-1/op-pick.png)

> `DW_OP_over`
>
> ![DW_OP_over](../images/issue-260127-1/op-over.png)

> `DW_OP_swap`
>
> ![DW_OP_swap](../images/issue-260127-1/op-swap.png)

> `DW_OP_rot`
>
> ![DW_OP_rot](../images/issue-260127-1/op-rot.png)

In section 3.3 add the following heading between operator and its
description as follows:

> `DW_OP_lit<n>`
>
> ![DW_OP_lit](../images/issue-260127-1/op-lit.png)

> `DW_OP_const<n>u` ([n-byte unsigned] A)
>
> ![DW_OP_lit](../images/issue-260127-1/op-lit.png)

> `DW_OP_const<n>s` ([n-byte integer] A)
>
> ![DW_OP_lit](../images/issue-260127-1/op-lit.png)

> `DW_OP_constu` (ULEB A)
>
> ![DW_OP_lit](../images/issue-260127-1/op-lit.png)

> `DW_OP_consts`(SLEB A)
>
> ![DW_OP_lit](../images/issue-260127-1/op-lit.png)

> `DW_OP_constx` ([ULEB] .debug_addr offset>)
>
> ![DW_OP_lit](../images/issue-260127-1/op-lit.png)

> `DW_OP_const_type`([SLEB] type DIE offset, [1-byte unsigned] size, constant)
>
> ![DW_OP_lit](../images/issue-260127-1/op-lit.png)

In section 3.4 add the following heading between operator and its
description as follows:

> `DW_OP_regval_type` ([SLEB] register number, [SLEB] offset of type DIE )
>
> ![DW_OP_regval-type](../images/issue-260127-1/op-regval-type.png)

In section 3.5 add the following heading between operator and its
description as follows:

> `DW_OP_abs`
>
> `DW_OP_neg`
>
> `DW_OP_not`
>
> `DW_OP_plus_uconst` (ULEB b)
>
> ![DW_OP_unary](../images/issue-260127-1/op-unary.png)

> `DW_OP_and`
>
> `DW_OP_div`
>
> `DW_OP_minus`
>
> `DW_OP_mod`
>
> `DW_OP_mul`
>
> `DW_OP_or`
>
> `DW_OP_plus`
>
> `DW_OP_shl`
>
> `DW_OP_shr`
>
> `DW_OP_shra`
>
> `DW_OP_xor`
>
> ![DW_OP_binary](../images/issue-260127-1/op-binary.png)
>

In section 3.6 add the following heading between operator and its
description as follows:

> `DW_OP_push_object_location`
>
> ![DW_OP_push_object_location](../images/issue-260127-1/op-push-loc.png)

> `DW_OP_form_tls_location`
>
> ![DW_OP_form_tls_location](../images/issue-260127-1/op-tls.png)

> `DW_OP_call_frame_cfa`
>
> ![DW_OP_call_frame_cfa](../images/issue-260127-1/op-push-loc.png)

> `DW_OP_push_lane`
>
> ![DW_OP_push_lane](../images/issue-260127-1/op-push-lane.png)

In section 3.7 add the following heading between operator and its
description as follows:

> `DW_OP_addr` ([unsigned integer] address)
>
> `DW_OP_addrx` ([ULEB] offset into .debug_addr )
>
> `DW_OP_fbreg` ([ULEB] offset from frame base )
>
> `DW_OP_breg<n>` ([SLEB] offset)
>
> `DW_OP_bregx` ([ULEB register number, [SLEB] offset)
>
> ![DW_OP_addr](../images/issue-260127-1/op-mem.png)

In section 3.8 add the following heading between operator and its
description as follows:

> `DW_OP_reg<n>`
>
> `DW_OP_regx` ([ULEB] register number)
>
> ![DW_OP_reg](../images/issue-260127-1/op-reg.png)

In section 3.9 add the following heading between operator and its
description as follows:

> `DW_OP_undefined`
>
> ![DW_OP_undefined](../images/issue-260127-1/op-undef.png)

In section 3.10 add the following heading between operator and its
description as follows:

> `DW_OP_implicit_value` ([ULEB] length, implicit storage bytes)
>
> ![DW_OP_implicit_value](../images/issue-260127-1/op-implicit.png)

> `DW_OP_stack_value`
>
> ![DW_OP_stack_value](../images/issue-260127-1/op-stack-value.png)

In section 3.11 add the following heading between operator and its
description as follows:

> `DW_OP_implicit_pointer` ([DIE reference] .debug_info offset, [SLEB] byte offset)
>
> ![DW_OP_implicit_pointer](../images/issue-260127-1/op-implicit-ptr.png)

In section 3.12 add the following heading between operator and its
description as follows:

> `DW_OP_composite`
>
> ![DW_OP_composite](../images/issue-260127-1/op-composite.png)

> `DW_OP_piece` ([ULEB] size in bytes)
>
> ![DW_OP_piece](../images/issue-260127-1/op-piece.png)

> `DW_OP_bit_piece` ([ULEB] size in bits)
>
> ![DW_OP_piece](../images/issue-260127-1/op-piece.png)

In section 3.13 add the following heading between operator and its
description as follows:

> `DW_OP_deref`
>
> `DW_OP_deref_size` ([1-byte integral] size)
>
> `DW_OP_deref_type` ([1-byte integral] size, [ULEB] DIE offset for type )
>
> ![DW_OP_deref](../images/issue-260127-1/op-deref.png)

> `DW_OP_xderef`
>
> `DW_OP_xderef_size` ([1-byte integral] size)
>
> `DW_OP_xderef_type` ([1-byte integral] size, [ULEB] DIE offset for type)
>
> ![DW_OP_xderef](../images/issue-260127-1/op-xderef.png)

In section 3.14 add the following heading between operator and its
description as follows:

> `DW_OP_offset`
>
> `DW_OP_bit_offset`
>
> ![DW_OP_offset](../images/issue-260127-1/op-offset.png)

In section 3.15 add the following heading between operator and its
description as follows:

 `DW_OP_le`, `DW_OP_ge`, `DW_OP_eq`, `DW_OP_lt`, `DW_OP_gt`, `DW_OP_ne`
>
> ![DW_OP_relational](../images/issue-260127-1/op-relational.png)

> `DW_OP_skip` ([2-byte signed integer] bytes to skip)
>
>     *no stack effects*
>

> `DW_OP_bra` ([2-byte signed integer] bytes to skip)
>
>     <[numeric] condition> → *no stack result*
>

> `DW_OP_call[24]` ([2- or 4- byte unsigned integral] DIE offset)
>
>     *stack effects by agreement*
>

> `DW_OP_call_ref` ([4- or 8- byte unsigned integral] .debug_info offset)
>
>     *stack effects by agreement*
>

In section 3.16 add the following heading between operator and its
description as follows:

> `DW_OP_convert` ([ULEB] or 0 for generic type)
>
> `DW_OP_reinterpret` ([ULEB] DIE offset with 0 for generic type)
>
> ![DW_OP_convert](../images/issue-260127-1/op-convert.png)

In section 3.17 add the following heading between operator and its
description as follows:

> `DW_OP_nop`
>
>     *no stack effects*
>

> `DW_OP_entry_value` ([ULEB] length, [DWARF expression] expression)
>
>     →  *stack result by agreement*
>

> `DW_OP_extended` ([ULEB] extended opcode)
>
>     *stack effects defined by extended operation*
>

> `DW_OP_user_extended` ([ULEB] extended opcode)
>
>     *stack effects defined by extended operation*
>

---

2026-03-30: Accepted in principle; will revise to use drawings as shown in
[op-diagrams.pdf](../doc/Issue-260127-1-op-diagrams.pdf).

2026-04-28: [Revised][diff1] to use drawings.

2026-04-29: Updated some of the introductory material.
