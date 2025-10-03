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
> parameters. The number of inline parameters is implied by the
> opcode. It may also receive operands from the stack and make use of
> information from its evaluation context.

>
> *The structure description of the inputs and each of each operation
> will be as follows:*
> 1. `DW_OP_name`
>
>       <[*type*] first stack argument> <[*type*>] second stack argument> ...
>        `DW_OP_name ([*type*] first inline operand>, [*type*] second inline operand, ...)
>        --> <[*type*] first stack result> <[*type*] second stack result> ...
>

In Section 3.2

> 1. `DW_OP_dup`
>
>       <[*any*] X> <[*any*] Y> `DW_OP_dup` → < X> < X>
>

> 2. `DW_OP_drop`
>
>       <[*any*] X> <[*any*] Y> `DW_OP_drop` → < Y>
>

> 3. `DW_OP_pick`
>
>       <[*any*] Nth> ... < first> `DW_OP_pick` ([1-byte integral] N)
>        → < Nth> ... < first> < Nth>
>

**NOTE FOR DISCUSSION**: It should be noted that there is no pick
operation which can pick based upon an index which is passed on the
stack instead of as a literal operand. The fact that it hasn't been
requested may suggest that it is not needed.

> 4. `DW_OP_over`
>
>       <[*any*] Y> <[*any*] X> `DW_OP_over` → < Y> < X> < Y>
>

> 5. `DW_OP_swap`
>
>       <[*any*] Y> <[*any*] X> `DW_OP_swap` → < X> < Y>
>

> 6. `DW_OP_rot`
>
>       <[*any*] Z> <[*any*] Y> <[*any*] X> `DW_OP_rot` → < X> < Z> < Y>
>

In section 3.3 add the following heading between operator and its
description as follows:

> 1. `DW_OP_lit0`, ..., `DW_OP_lit31`
>
>       `DW_OP_lit<n>` → <[generic] n>
>

> 2. `DW_OP_const1u`, `DW_OP_const2u`, `DW_OP_const4u`, `DW_OP_const8u`
>
>       `DW_OP_const<n>u` ([n-byte unsigned] X) → <[unsigned] X'>
>

> 3. `DW_OP_const1s`, `DW_OP_const2s`, `DW_OP_const4s`, `DW_OP_const8s`
>
>       `DW_OP_const<n>s` ([n-byte integer] X) → <[integer] X'>
>

> 4. `DW_OP_constu`
>
>       `DW_OP_constu` (ULEB128 X) → <[unsigned] X'>
>

> 5. `DW_OP_consts`
>
>       `DW_OP_constu` (LEB128 X) → <[integer] X'>
>

> 6. `DW_OP_constx`
>
>       `DW_OP_constx` ([ULEB128] .debug_addr offset>) → <[unsigned] X>
>

> 7. `DW_OP_const_type`
>
>       `DW_OP_const_type` ([LEB128] type DIE offset, [1-byte unsigned] size, constant) → <[*specified type*] X>
>

In section 3.4 add the following heading between operator and its
description as follows:

> 1. `DW_OP_regval_type`
>
>       `DW_OP_regval_type` ([LEB128] register number, [LEB128] offset of type DIE ) → <[*specified type*] X>
>

In section 3.5 add the following heading between operator and its
description as follows:

> 1. `DW_OP_abs`
>
>       <[numeric] X> `DW_OP_abs` → <[numeric] X'>
>

> 2. `DW_OP_and`
>
>       <[integral base type or generic type] Y> <[integral base type or generic type] X> `DW_OP_and` → <[integral base type or generic type] X'>
>

> 3. `DW_OP_div`
>
>       <[numeric] Y>  <[numeric] X> `DW_OP_div` → <[numeric] Y/X>
>

> 4. `DW_OP_minus`
>
>       <[numeric] Y> <[numeric] X> `DW_OP_minus` → <[numeric] Y-X>
>

> 5. `DW_OP_mod`
>
>       <[integral base type or generic type] Y> <[integral base type or generic type] X> `DW_OP_mod` → <[integral base type or generic type] Y mod X>

> 6. `DW_OP_mul`
>
>       <[numeric] Y> <[numeric] X> `DW_OP_mul` → <[numeric] Y*X>
>

> 7. `DW_OP_neg`
>
>       <[numeric] X> `DW_OP_neg` → <[numeric] X'>
>

> 8. `DW_OP_not`
>
>       <[integral base type or generic type] X> `DW_OP_not` → <[integral base type or generic type] X'>
>

> 9. `DW_OP_or`
>
>       <[integral base type or generic type] Y> <[integral base type or generic type] X> `DW_OP_or` → <[integral base type or generic type] X'>

> 10. `DW_OP_plus`
>
>       <[numeric] Y> <[numeric] X> `DW_OP_plus` → <[numeric] Y+X>
>

> 11. DW_OP_plus_uconst
>
>       <[numeric] X> `DW_OP_plus_ucost` (ULEB128 Y) → <[numeric] Y+X>
>

**NOTE FOR DISCUSSION:** The justification for `DW_OP_plus_uconst`
  seems questionable to me. I do not see why you could not just use:
  `DW_OP_uconst <some number> DW_OP_plus`. Is this a historic
  limitation? Should this text be reconsidered?

> 12. `DW_OP_shl`
>
>       <[integral base type or generic type] Y> <[integral base type or generic type] X> `DW_OP_shl` → <[integral base type or generic type] Y<<X >
>

**NOTE FOR DISCUSSION:** What happens if you shift left a negative
number of bytes? Should the number of bits to be shifted be limited to
to a positive value.

>
> 13. `DW_OP_shr`
>
>       <[integral base type or generic type] Y> <[integral base type or generic type] X> `DW_OP_shr` → <[integral base type or generic type] Y>>X >

> 14. `DW_OP_shra`
>
>       <[integral base type or generic type] Y> <[integral base type or generic type] X> `DW_OP_shra` → <[integral base type or generic type] Y shra X >
>

> 15. `DW_OP_xor`
>
>       <[integral base type or generic type] Y> <[integral base type or generic type] X> `DW_OP_xor` → <[integral base type or generic type] Y xor X >
>

In section 3.6 add the following heading between operator and its
description as follows:

> 1. `DW_OP_push_object_location`
>
>       `DW_OP_push_object_location` → <[location] X >
>

> 2. `DW_OP_form_tls_location`
>
>       <[integral] offset into TLS> `DW_OP_form_tls_location` → <[location] in thread's TLS >
>

**Fixme** IMHO the description needs to be rewritten.

> 3. `DW_OP_call_frame_cfa`
>
>       `DW_OP_call_frame_cfa` → <[location] call frame location >
>

> 4. `DW_OP_push_lane`
>
>       `DW_OP_push_lane` → <[unsigned integer] lane number >
>

In section 3.7 add the following heading between operator and its
description as follows:

> 1. `DW_OP_addr`
>
>       `DW_OP_addr` ([unsigned integer] address) → <[location] memory storage >
>

> 2. `DW_OP_addrx`
>
>       `DW_OP_addrx` ([ULEB128] offset into .debug_addr ) → <[location] memory storage >
>

> 3. `DW_OP_fbreg`
>
>       `DW_OP_fbreg` ([ULEB128] offset from frame base) → <[location] memory_storage >
>

> 4. `DW_OP_breg0`, ..., `DW_OP_breg31`
>
>       `DW_OP_breg<n>` ([LEB128] offset) → <[location] memory storage >
>

> 5. `DW_OP_bregx`
>
>       `DW_OP_breg<n>` ([ULEB128 register number, [LEB128] offset) → <[location] X >
>

In section 3.8 add the following heading between operator and its
description as follows:

> 1. `DW_OP_reg0`, `DW_OP_reg1`, ..., `DW_OP_reg31`
>
>       `DW_OP_reg<n>` → <[location] register storage >
>

> 2. `DW_OP_regx`
>
>       `DW_OP_regx` ([ULEB128] register number) → <[location] register storage >
>

In section 3.9 add the following heading between operator and its
description as follows:

> 1. `DW_OP_undefined`
>
>       `DW_OP_undefined` → <[location] undefined storage >
>

In section 3.10 add the following heading between operator and its
description as follows:

> 1. `DW_OP_implicit_value`
>
>       `DW_OP_implicit_value` ([ULEB128] length, implicit storage bytes) → <[location] implicit value storage >
>

> 2. `DW_OP_stack_value`
>
>       <[*any*] value> `DW_OP_stack_value` <[location] implicit value storage >
>

In section 3.11 add the following heading between operator and its
description as follows:

> 1. `DW_OP_implicit_pointer`
>
>       `DW_OP_implicit_pointer` ([unsigned 4 or 8 byte integral] .debug_info offset, [LEB128] byte offset) → <[location] implicit pointer storage>
>

In section 3.12 add the following heading between operator and its
description as follows:

> 1. `DW_OP_composite`
>       `DW_OP_composite` → <[location] composite storage >
>

> 2. `DW_OP_piece`
>
>       `DW_OP_piece` ([ULEB128] size in bytes) → <[location] composite storage>
>

**NOTE FOR DISCUSSION** The original conception of piece had the
composite being created on the side not on the stack. Now that it is
on the stack, a way of understanding piece may be that it peeks at the
top of the stack and if the type is a composite location then it pops
off of the stack, modifies it and then pushes it again. Maybe we
should rewrite the description of piece this way?

<[location] composite storage > `DW_OP_piece` ([ULEB128] size in
bytes) → <[location] composite storage>

If the top of the stack is not a composite, then it creates an empty
composite storage and then pushes that. Note that there is currently
no peek or typeof operator.

The problem with this is in stack based machines maintaining stack
alignment is very important. Not knowing and not being able to know if
a stack element is going to be popped can lead to stack misalignment.

> 3. `DW_OP_bit_piece`
>
>       `DW_OP_bit_piece` ([ULEB128] size in bits) → <[location] composite storage>
>

In section 3.13 add the following heading between operator and its
description as follows:

> 1. `DW_OP_deref`
>
>       <[location] L> `DW_OP_deref` → <[generic] X>
>

> 2. `DW_OP_deref_size`
>
>       <[location] L> `DW_OP_deref_size` ([1-byte integral] size) → <[generic] X>
>

> 3. `DW_OP_deref_type`
>
>       <[location] L> `DW_OP_deref_type` ([1-byte integral] size, [ULEB128] DIE offset for type ) → <[*specified type*] X>
>

> 4. `DW_OP_xderef`
>
>       <[integral] address space identifier> <[location] L> `DW_OP_xderef` → <[generic] X>

> 5. `DW_OP_xderef_size`
>
>       <[integral] address space identifier> <[location] L> `DW_OP_xderef_size` ([1-byte integral] size) → <[generic] X>
>

> 6. `DW_OP_xderef_type`
>
>       <[integral] address space identifier> <[location] L> `DW_OP_xderef_type` ([1-byte integral] size, [ULEB128] DIE offset for type) → <[*specified type*] X>
>

In section 3.14 add the following heading between operator and its
description as follows:

> 1. `DW_OP_offset`
>
>       <[location] L> <[signed integral] displacement> `DW_OP_offset` → <[location] L'>
>

> 2. `DW_OP_bit_offset`
>
>       <[location] L> <[signed integral] displacement> `DW_OP_bit_offset` → <[location] L'>
>

In section 3.15 add the following heading between operator and its
description as follows:

> 1. `DW_OP_le`, `DW_OP_ge`, `DW_OP_eq`, `DW_OP_lt`, `DW_OP_gt`, `DW_OP_ne`
>
>       <[base type or generic] Y> <[base type or generic] X> DW_OP_<relation> → <[[generic] 0 or 1>
>

**NOTE FOR DISCUSSION** Does this work for only integral base types or
  does it work for all base types? Some of the base types like vector
  types are kind of weird like the vector types? I think that this
  should be limited to integral base types.

> 2. `DW_OP_skip`
>
>       `DW_OP_skip` ([2-byte signed integer] bytes to skip)
>

> 3. `DW_OP_bra`
>
>       <[numeric] condition> `DW_OP_bra` ([2-byte signed integer] bytes to skip)
>

> 4. `DW_OP_call2`, `DW_OP_call4`
>
>       `DW_OP_call[24]` ([2 or 4 byte unsigned integral] DIE offset)
>

> 5. `DW_OP_call_ref`
>
>       `DW_OP_call_ref` ([4 or 8 byte unsigned integral] debug_info offset)
>

In section 3.16 add the following heading between operator and its
description as follows:

> 1. `DW_OP_convert`
>
>       <[*any*] X> `DW_OP_convert` ([ULEB128] or 0 for generic type) → <[*specified type*] X'>
>

**NOTE FOR DISCUSSION** Should the types that can be converted be
specified? Some of the conversions can be kind of weird.

> 2. `DW_OP_reinterpret`
>
>       <[*any*] X> `DW_OP_reinterpret` ([ULEB128] or 0 for generic type) → <[*specified type*] X'>
>

In section 3.17 add the following heading between operator and its
description as follows:

> 1. `DW_OP_nop`
>
>       `DW_OP_nop`
>

> 2. `DW_OP_entry_value`
>
>       `DW_OP_entry_value` ([ULEB128] length, [DWARF expression] expression) → <[generic type] value>
>

**NOTE FOR DISCUSSION** What type does it push on the stack. It isn't
  clear to me from the operation description. Presumably it is the
  same as the formal parameter to which it is applied. This operator
  doesn't make sense to me for different reasons than the problems
  that Tony has with it.

> 3. `DW_OP_extended`
>
>       `DW_OP_extended` ([ULEB128] extended opcode)
>

> 4. `DW_OP_user_extended`
>
>       `DW_OP_user_extended` ([ULEB128] extended opcode)
>

**NOTE FOR DISCUSSION** We should consider expanding table 8.9 to
  include all the information in the header. We could easily expand it
  to include the number of stack operands and their expected types as
  well.
