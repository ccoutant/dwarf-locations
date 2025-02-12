# Clarifications for Expression Evaluation

This is part of a series of proposals related to support for debugging
on heterogeneous architectures. Although this proposal is not directly
related to heterogeneous architectures, it lays the groundwork for
subsequent proposals in this series.


## Background

This proposal is editorial in nature: it does not intend to change the
meaning of any DWARF constructs, but merely to clarify aspects of
DWARF expression evaluation that were unclear to the teams implementing
DWARF consumers.

The changes proposed below provide additional rigor in the description
of DWARF expressions. This proposal defines a context within which
expression evaluation takes place, specifies the conditions under
which an expression is ill formed, and adopts a more formal notation
in the description of the stack operations.


## Proposed Changes

In Section 2.5 DWARF Expressions, add the following text after the first
paragraph:

> [non-normative] The evaluation of a DWARF expression can provide the
> location of an object, the value of an array bound, the length of a
> dynamic string, the desired value itself, and so on.
>
> If the evaluation of a DWARF expression does not encounter an error,
> then it can either result in a value (see 2.5.2 DWARF Expression
> Value) or a location description (see 2.5.3 DWARF Location
> Description). When a DWARF expression is evaluated, it may be
> specified whether a value or location description is required as the
> result kind.
>
> If the evaluation of a DWARF expression encounters an evaluation
> error, then the result is an evaluation error.
>
> If a DWARF expression is ill-formed, then the result is undefined.
>
> The following sections detail the rules for when a DWARF expression
> is ill-formed or results in an evaluation error.

#### FIXME: Add the following somewhere:

> The `address_size` fields must match in the headers of all the entries
> in the `.debug_info`, `.debug_addr`, `.debug_line`, `.debug_rnglists`,
> `.debug_rnglists.dwo`, `.debug_loclists`, and `.debug_loclists.dwo`
> sections corresponding to any given program counter.


In Section 2.5.x (originally 2.5.1) General Operations, replace the
first paragraph:

> Each general operation represents a postfix operation on a simple
> stack machine. Each element of the stack has a type and a value, and
> can represent a value of any supported base type of the target
> machine. Instead of a base type, elements can have a generic type,
> which is an integral type that has the size of an address on the
> target machine and unspecified signedness. The value on the top of
> the stack after “executing” the DWARF expression is taken to be the
> result (the address of the object, the value of the array bound, the
> length of a dynamic string, the desired value itself, and so on).

... with the following:

> Operations represent a postfix operation on a simple stack machine.
>
> A value on the stack has a type and a literal value. It can
> represent a literal value of any supported base type of the target
> architecture. The base type specifies the size, encoding, and
> endianity of the literal value.
>
> There is a distinguished base type termed the generic type, which is
> an integral type that has the size of an address in the target
> architecture default address space, a target architecture defined
> endianity, and unspecified signedness.

Insert the following after the end of the same section:

> An integral type is a base type that has an encoding of
> `DW_ATE_signed`, `DW_ATE_signed_char`, `DW_ATE_unsigned`,
> `DW_ATE_unsigned_char`, `DW_ATE_boolean`, or any target architecture
> defined integral encoding in the inclusive range `DW_ATE_lo_user` to
> `DW_ATE_hi_user`.
>
> Operations can act on entries on the stack, including adding entries
> and removing entries.
>
> Evaluation of an expression starts with an empty stack on which the
> entries from the initial stack provided by the context are pushed in
> the order provided. Then the operations are evaluated, starting with
> the first operation of the stream. Evaluation continues until either
> an operation has an evaluation error, or until one past the last
> operation of the stream is reached.
>
> The result of the evaluation is:
>
>   * If an operation has an evaluation error, or an operation
>     evaluates an expression that has an evaluation error, then the
>     result is an evaluation error.
>
>   * If the current result kind specifies a location description,
>     then:
>
>       * If the stack is empty, the result is an empty location
>         description.
>
>       * If the stack is not empty, then the result is a memory
>         location description (See Section 2.6.1.1.2 Memory Location
>         Descriptions). Any other entries on the stack are discarded.
>
>       * Otherwise the DWARF expression is ill-formed.
>
>         > [For further discussion...]
>         > Could define this case as returning an implicit location
>         > description as if the `DW_OP_implicit` operation is
>         > performed.
>
>   * If the current result kind specifies a value, then:
>
>       * If the stack is not empty, then the result is the top
>         value on the stack. Any other entries on the stack are
>         discarded.
>
>       * Otherwise the DWARF expression is ill-formed.
>
>   * If the current result kind is not specified, then:
>
>       * If the stack is empty, the result is an empty location
>         description.
>
>       * Otherwise, the top stack entry is returned. Any other
>         entries on the stack are discarded.
>
> An operation expression is encoded as a byte block with some form of
> prefix that specifies the byte count. It can be used:
>
>   * as the value of a debugging information entry attribute that is
>     encoded using class exprloc (see 7.5.5 Classes and Forms).
>
>   * as the operand to certain operation expression operations.
>
>   * as the operand to certain call frame information operations (see
>     6.4 Call Frame Information).
>
>   * in location list entries (see 2.5.x DWARF Location List
>     Expressions).

Replace the first paragraph in Section 2.5.1.1 Literal Encodings:

> The following operations all push a value onto the DWARF stack. Operations
> other than `DW_OP_const_type` push a value with the generic type, and if the
> value of a constant in one of these operations is larger than can be stored
> in a single stack element, the value is truncated to the element size and
> the low-order bits are pushed on the stack.

with:

> The following operations all push a literal value onto the DWARF stack.
>
> Operations other than `DW_OP_const_type` push a value V with the generic type.
> If V is larger than the generic type, then V is truncated to the generic
> type size and the low-order bits used.

In Section 2.5.1.1 Literal Encodings, replace the description of the following
operations:

> 1.  `DW_OP_lit0`, `DW_OP_lit1`, ..., `DW_OP_lit31`
>     `DW_OP_lit<N>` operations encode an unsigned literal value N from 0
>     through 31, inclusive. They push the value N with the generic type.
>
> 3.  `DW_OP_const1u`, `DW_OP_const2u`, `DW_OP_const4u`, `DW_OP_const8u`
>     `DW_OP_const<N>u` operations have a single operand that is a 1, 2, 4, or
>     8-byte unsigned integer constant U, respectively. They push the value U
>     with the generic type.
>
> 4.  `DW_OP_const1s`, `DW_OP_const2s`, `DW_OP_const4s`, `DW_OP_const8s`
>     `DW_OP_const<N>s` operations have a single operand that is a 1, 2, 4, or
>     8-byte signed integer constant S, respectively. They push the value S
>     with the generic type.
>
> 5.  `DW_OP_constu`
>     `DW_OP_constu` has a single unsigned LEB128 integer operand N. It pushes
>     the value N with the generic type.
>
> 6.  `DW_OP_consts`
>     `DW_OP_consts` has a single signed LEB128 integer operand N. It pushes the
>     value N with the generic type.
>
> 8.  `DW_OP_constx`
>     `DW_OP_constx` has a single unsigned LEB128 integer operand that
>     represents a zero-based index into the .debug_addr section relative to
>     the value of the `DW_AT_addr_base` attribute of the associated compilation
>     unit. The value N in the .debug_addr section has the size of the generic
>     type. It pushes the value N with the generic type.
>
>     [non-normative] The `DW_OP_constx`... [same]

In Section 2.5.1.1 Literal Encodings, replace the description of
`DW_OP_const_type` with the following:

> `DW_OP_const_type` has three operands. The first is an unsigned LEB128
> integer DR that represents the byte offset of a debugging
> information entry D relative to the beginning of the current
> compilation unit, that provides the type T of the constant value.
> The second is a 1-byte unsigned integral constant S. The third is a
> block of bytes B, with a length equal to S.
>
> TS is the bit size of the type T. The least significant TS bits of B
> are interpreted as a value V of the type D. It pushes the value V
> with the type D.
>
> The DWARF is ill-formed if D is not a `DW_TAG_base_type` debugging
> information entry in the current compilation unit, or if TS divided
> by 8 (the byte size) and rounded up to a whole number is not equal
> to S.
>
> [non-normative] While the size of the byte block B can be inferred
> from the type D definition, it is encoded explicitly into the
> operation so that the operation can be parsed easily without
> reference to the .debug_info section.

#### FIXME: Add new wording for breg0..31

In Section 2.5.1.2 Register Values, replace the description of
`DW_OP_bregx` with the following:

> `DW_OP_bregx` has two operands. The first is an unsigned LEB128
> integer that represents a register number R. The second is a signed
> LEB128 integer that represents a byte displacement B.
>
> The action is the same as for `DW_OP_breg<N>`, except that R is used
> as the register number and B is used as the byte displacement.

In Section 2.5.1.2 Register Values, replace the description of
`DW_OP_regval_type` with the following:

> `DW_OP_regval_type` has two operands. The first is an unsigned LEB128
> integer that represents a register number R. The second is an
> unsigned LEB128 integer DR that represents the byte offset of a
> debugging information entry D relative to the beginning of the
> current compilation unit, that provides the type T of the register
> value.
>
> > [For further discussion...]
> > Should DWARF allow the type T to be a larger size than the size of
> > the register R? Restricting a larger bit size avoids any issue of
> > conversion as the, possibly truncated, bit contents of the
> > register is simply interpreted as a value of T. If a conversion is
> > wanted it can be done explicitly using a `DW_OP_convert` operation.
> >
> > GDB has a per register hook that allows a target specific
> > conversion on a register by register basis. It defaults to
> > truncation of bigger registers. Removing use of the target hook
> > does not cause any test failures in common architectures. If the
> > compiler for a target architecture did want some form of
> > conversion, including a larger result type, it could always
> > explicitly use the `DW_OP_convert` operation.
> >
> > If T is a larger type than the register size, then the default GDB
> > register hook reads bytes from the next register (or reads out of
> > bounds for the last register!). Removing use of the target hook
> > does not cause any test failures in common architectures (except
> > an illegal hand written assembly test). If a target architecture
> > requires this behavior, these extensions allow a composite
> > location description to be used to combine multiple registers.

In Section 2.5.1.3 Stack Operations, replace the description of
`DW_OP_pick` with the following:

> `DW_OP_pick` has a single unsigned 1-byte operand that represents an
> index I. A copy of the stack entry with index I is pushed onto the
> stack.

Replace the description of `DW_OP_over` with the following:

> `DW_OP_over` pushes a copy of the entry with index 1.
>
> [non-normative] This is equivalent to a `DW_OP_pick` 1 operation.

Replace the description of `DW_OP_deref` with the following:

> S is the bit size of the generic type divided by 8 (the byte size)
> and rounded up to a whole number. DR is the offset of a hypothetical
> debug information entry D in the current compilation unit for a base
> type of the generic type.
>
> The operation is equivalent to performing `DW_OP_deref_type` S, DR.

Replace the description of `DW_OP_deref_size` with the following:

> `DW_OP_deref_size` has a single 1-byte unsigned integral constant that
> represents a byte result size S.
>
> TS is the smaller of the generic type bit size and S scaled by 8
> (the byte size). If TS is smaller than the generic type bit size
> then T is an unsigned integral type of bit size TS, otherwise T is
> the generic type. DR is the offset of a hypothetical debug
> information entry D in the current compilation unit for a base type
> T.
>
> > [For further discussion...]
> > Truncating the value when S is larger than the generic type
> > matches what GDB does. This allows the generic type size to not be
> > an integral byte size. It does allow S to be arbitrarily large.
> > Should S be restricted to the size of the generic type rounded up
> > to a multiple of 8?
>
> The operation is equivalent to performing `DW_OP_deref_type` S, DR,
> except if T is not the generic type, the value V pushed is
> zero-extended to the generic type bit size and its type changed to
> the generic type.

Replace the description of `DW_OP_deref_type` with the following:

> `DW_OP_deref_type` has two operands. The first is a 1-byte unsigned
> integral constant S. The second is an unsigned LEB128 integer DR
> that represents the byte offset of a debugging information entry D
> relative to the beginning of the current compilation unit, that
> provides the type T of the result value.
>
> TS is the bit size of the type T.
>
> [non-normative] While the size of the pushed value V can be inferred
> from the type T, it is encoded explicitly as the operand S so that
> the operation can be parsed easily without reference to the
> .debug_info section.
>
> > [For further discussion...]
> > It is unclear why the operand S is needed. Unlike
> > `DW_OP_const_type`, the size is not needed for parsing. Any
> > evaluation needs to get the base type T to push with the value to
> > know its encoding and bit size.
>
> It pops one stack entry and treats it as an address. The popped
> value must have an integral type. A value V of TS bits is retrieved
> from that address and pushed on the stack with the type T.
>
> The DWARF is ill-formed if D is not in the current compilation unit,
> D is not a `DW_TAG_base_type` debugging information entry, or if TS
> divided by 8 (the byte size) and rounded up to a whole number is not
> equal to S.

> > [For further discussion...]
> > This definition allows the base type to be a bit size since there seems no
> > reason to restrict it.

Replace the description of `DW_OP_xderef` with the following:

> `DW_OP_xderef` pops two stack entries. The first must be an integral
> type value that represents an address A. The second must be an
> integral type value that represents a target architecture specific
> address space identifier AS.
>
> The address size S is defined as the address bit size of the target
> architecture specific address space that corresponds to AS.
>
> A is adjusted to S bits by zero extending if necessary, and then
> treating the least significant S bits as an unsigned value A’.
>
> It creates a location description L with one memory location
> description SL. SL specifies the memory location storage LS that
> corresponds to AS with a bit offset equal to A’ scaled by 8 (the
> byte size).
>
> If AS is an address space that is specific to context elements, then
> LS corresponds to the location storage associated with the current
> context.
>
> [non-normative] For example, if AS is for per thread storage then LS
> is the location storage for the current thread. Therefore, if L is
> accessed by an operation, the location storage selected when the
> location description was created is accessed, and not the location
> storage associated with the current context of the access operation.
>
> The DWARF expression is ill-formed if AS is not one of the values
> defined by the target architecture specific `DW_ASPACE_*` values.
>
> The value V is retrieved from the location described by L, and
> pushed on the stack with the generic type.

Replace the description of `DW_OP_xderef_size` with the following:

> `DW_OP_xderef_size` has a single 1-byte unsigned integral constant
> that represents a byte result size S.
>
> It pops two stack entries. The first must be an integral type value
> that represents an address A. The second must be an integral type
> value that represents a target architecture specific address space
> identifier AS.
>
> It creates a location description L as described for `DW_OP_xderef`.
>
> The value V of size S is retrieved from the location described by L,
> zero-extended, and pushed on the stack with the generic type.

Replace the description of `DW_OP_xderef_type` with the following:

> `DW_OP_xderef_type` has two operands. The first is a 1-byte unsigned
> integral constant S. The second operand is an unsigned LEB128
> integer DR that represents the byte offset of a debugging
> information entry D relative to the beginning of the current
> compilation unit, that provides the type T of the result value.
>
> It pops two stack entries. The first must be an integral type value
> that represents an address A. The second must be an integral type
> value that represents a target architecture specific address space
> identifier AS.
>
> It creates a location description L as described for `DW_OP_xderef`.
>
> The value V is retrieved from the location described by L, and
> pushed on the stack with the type T.

In Section 2.5.1.5 Control Flow Operations, add the following to the
description of `DW_OP_skip`:

> If the updated position is at one past the end of the last
> operation, then the operation expression evaluation is complete.
>
> Otherwise, the DWARF expression is ill-formed if the updated
> operation position is not in the range of the first to last
> operation inclusive, or not at the start of an operation.

Add the following to the description of `DW_OP_bra`:

> If the updated position is at one past the end of the last
> operation, then the operation expression evaluation is complete.
>
> Otherwise, the DWARF expression is ill-formed if the updated
> operation position is not in the range of the first to last
> operation inclusive, or not at the start of an operation.

Replace the first paragraph of the description of `DW_OP_call2`/4/ref with
the following:

> `DW_OP_call2`, `DW_OP_call4`, and `DW_OP_call_ref` perform DWARF procedure
> calls during evaluation of a DWARF expression.
>
> `DW_OP_call2` and `DW_OP_call4`, have one operand that is, respectively,
> a 2-byte or 4-byte unsigned offset DR that represents the byte
> offset of a debugging information entry D relative to the beginning
> of the current compilation unit.
>
> `DW_OP_call_ref` has one operand that is a 4-byte unsigned value in
> the 32-bit DWARF format, or an 8-byte unsigned value in the 64-bit
> DWARF format, that represents the byte offset DR of a debugging
> information entry D relative to the beginning of the .debug_info
> section that contains the current compilation unit. D might not be
> in the current compilation unit.

Replace the last paragraph (beginning “These operations transfer
control...”) with the following:

> The call operation is evaluated by:
>
>   * If D has a `DW_AT_location` attribute that is encoded as a exprloc
>     that specifies an operation expression E, then execution of the
>     current operation expression continues from the first operation
>     of E. Execution continues until one past the last operation of E
>     is reached, at which point execution continues with the
>     operation following the call operation. The operations of E are
>     evaluated with the same current context, except current
>     compilation unit is the one that contains D and the stack is the
>     same as that being used by the call operation. After the call
>     operation has been evaluated, the stack is therefore as it is
>     left by the evaluation of the operations of E. Since E is
>     evaluated on the same stack as the call operation, E can use,
>     and/or remove entries already on the stack, and can add new
>     entries to the stack.
>
>     [non-normative] Values on the stack at the time of the call may
>     be used as parameters by the called expression and values left
>     on the stack by the called expression may be used as return
>     values by prior agreement between the calling and called
>     expressions.

### FIXME: encoded as loclist or loclistptr?

>   * Otherwise, there is no effect and no changes are made to the
>     stack.
>
> > [For further discussion...]
> > In DWARF Version 5, if D does not have a `DW_AT_location` then
> > `DW_OP_call*` is defined to have no effect. It is unclear that this
> > is the right definition as a producer should be able to rely on
> > using `DW_OP_call*` to get a location description for any
> > non-`DW_TAG_dwarf_procedure` debugging information entries. Also,
> > the producer should not be creating DWARF with `DW_OP_call*` to a
> > `DW_TAG_dwarf_procedure` that does not have a `DW_AT_location`
> > attribute. So, should this case be defined as an ill-formed DWARF
> > expression?
>
> [non-normative] The `DW_TAG_dwarf_procedure` debugging information
> entry can be used to define DWARF procedures that can be called.

In Section 2.5.1.7 Special Operations, replace the description of
`DW_OP_entry_value` with the following:

> `DW_OP_entry_value` pushes the value of an expression that is
> evaluated in the context of the calling frame.
>
> It has two operands. The first is an unsigned LEB128 integer S. The
> second is a block of bytes, with a length equal S, interpreted as a
> DWARF operation expression E.
>
> E is evaluated with the current context, with the following
> modifications: the result kind is unspecified, the call frame is the
> one that called the current frame, the program location is the call
> site in the calling frame, the object is unspecified, and the
> initial stack is empty. The calling frame information is obtained by
> virtually unwinding the current call frame using the call frame
> information (see 6.4 Call Frame Information).
>
> If the result of E is a location description L (see 2.5.x.x Register
> Location Description Operations), and the last operation executed by
> E is a `DW_OP_reg*` for register R with a target architecture specific
> base type of T, then the contents of the register are retrieved as
> if a `DW_OP_deref_type` DR operation was performed where DR is the
> offset of a hypothetical debug information entry in the current
> compilation unit for T. The resulting value V s pushed on the stack.
>
> [non-normative] Using `DW_OP_reg*` provides a more compact form for
> the case where the value was in a register on entry to the
> subprogram.
>
> > [For further discussion...]
> > It is unclear how this provides a more
> > compact expression, as `DW_OP_regval_type` could be used which is
> > marginally larger.
>
> If the result of E is a value V, then V is pushed on the stack.
>
> Otherwise, the DWARF expression is ill-formed.

In Section 2.6.1.1.3 Register Location Descriptions, in the description of
`DW_OP_reg0`...`DW_OP_reg31`, change "The `DW_OP_reg<n>` operations encode the
names..." with "The `DW_OP_reg<n>` operations encode the numbers...", and change
"The object addressed is in register n" with "The target architecture register
number R corresponds to the N in the operation name."

Add the following paragraph at the end of the description of
`DW_OP_reg0`...`DW_OP_reg31`:

> The operation is equivalent to performing `DW_OP_regx` R.

2.6.1.1.3 Register Location Descriptions

In Section 7.4, 32-Bit and 64-Bit DWARF Formats, in item 3, add the following
row to the table:

> | Form                      | Role                                   |
> | ------------------------  | -------------------------------------- |
> | `DW_OP_implicit_pointer`  | offset in .debug_info                  |
