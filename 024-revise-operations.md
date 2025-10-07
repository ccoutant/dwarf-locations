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
> `DW_OP_name` ([*type*] first inline operand>, [*type*] second inline operand, ...)
>
>     <[*type*] first stack argument> <[*type*>] second stack argument> ...
>         → <[*type*] first stack result> <[*type*] second stack result> ...
>

> *The first line mentions the inline operands and their expected
> types or binary encodings. These encodings can be found in Section X.Y*

> *The second line explains the ways that the operator affects the
> DWARF stack. The arguments consumed from the stack are listed
> first. Then after the "→" the results that are left on the
> stack. Each entry on the DWARF stack has a type. In most cases, it
> is broadly a value or location. However, there are often limits
> placed on the each operator's input stack arguments limiting the
> sub-types over which the operator is defined. Since one operator's
> output becomes the input for subsequent operators in a DWARF
> expression, the type of the operation's output is also often listed
> when it cannot be immediately inferred.*

> *Symbolic names are often given to the stack arguments and the stack
> result when it is considered to be helpful or meaningful. In the
> cases where no sybolic name would be meaningful, the generic names A
> B C and D are used. When, an operator modifies a stack argument in
> meaningful way before returning it to the stack, the name is given
> an apostrophe suffix, for example turning A into A' ("A prime").*

In Section 3.2

> `DW_OP_dup`
>
>       <[*any*] A> <[*any*] B> → < A> < A>
>

> `DW_OP_drop`
>
>       <[*any*] A> <[*any*] B>  → < B>
>

> `DW_OP_pick` ([1-byte integral] N)
>
>       <[*any*] Nth> ... < first> → < Nth> ... < first> < Nth>
>

**NOTE FOR DISCUSSION**: It should be noted that there is no pick
operation which can pick based upon an index which is passed on the
stack instead of as a literal operand. The fact that it hasn't been
requested may suggest that it is not needed.

> `DW_OP_over`
>
>     <[*any*] B> <[*any*] A> → < B> < A> < B>
>

> `DW_OP_swap`
>
>    <[*any*] B> <[*any*] A> 2→ < A> < B>
>

> `DW_OP_rot`
>
>    <[*any*] C> <[*any*] B> <[*any*] A> → < A> < C> < B>
>

In section 3.3 add the following heading between operator and its
description as follows:

> `DW_OP_lit<n>`
>
>    → <[generic] n>
>

> `DW_OP_const<n>u` ([n-byte unsigned] A)
>
>    → <[unsigned] A>
>

> `DW_OP_const<n>s` ([n-byte integer] A)
>
>    → <[integer] A>
>

> `DW_OP_constu` (ULEB A)
>
>    → <[unsigned] A>
>

> `DW_OP_consts`(SLEB A)
>
>    → <[integer] A>
>

> `DW_OP_constx` ([ULEB] .debug_addr offset>)
>
>    → <[unsigned] A>
>

> `DW_OP_const_type`([SLEB] type DIE offset, [1-byte unsigned] size, constant)
>
>    → <[*specified type*] A>
>

*NOTE FOR DISCUSSION*: Why do some operators take SLEBs or ULEBs while
others take 2- or 4- byte unsigned integrals or 4- or 8- byte unsigned
integrals. Historic?

In section 3.4 add the following heading between operator and its
description as follows:

> `DW_OP_regval_type` ([SLEB] register number, [SLEB] offset of type DIE )
>
>       → <[*specified type*] A>
>

In section 3.5 add the following heading between operator and its
description as follows:

> `DW_OP_abs`
>
>    <[numeric] A> → <[numeric] A'>
>

> `DW_OP_and`
>
>    <[integral base type or generic type] B> <[integral base type or generic type] A> → <[integral base type or generic type] A'>
>

**NOTE FOR DISCUSSION**: Do we want to expand logical operators like
and to include vector types? The concern that I have is predicate
registers may be bigger than a generic type. I'm also not entirely
sure that they are base types in all GPU architectures. In other
words, can we expand the domain of these operators.

> `DW_OP_div`
>
>    <[numeric] B>  <[numeric] A> → <[numeric] B/A>
>

> `DW_OP_minus`
>
>    <[numeric] B> <[numeric] A> → <[numeric] B-A>
>

> `DW_OP_mod`
>
>    <[integral base type or generic type] B> <[integral base type or generic type] A> → <[integral base type or generic type] B mod A>

> `DW_OP_mul`
>
>    <[numeric] B> <[numeric] A> → <[numeric] B*A>
>

> `DW_OP_neg`
>
>    <[numeric] A> → <[numeric] A'>
>

> `DW_OP_not`
>
>    <[integral base type or generic type] A> → <[integral base type or generic type] A'>
>

> `DW_OP_or`
>
>    <[integral base type or generic type] B> <[integral base type or generic type] A> → <[integral base type or generic type] A'>

> `DW_OP_plus`
>
>    <[numeric] B> <[numeric] A> → <[numeric] B+A>
>

> `DW_OP_plus_uconst` (ULEB B)
>
>       <[numeric] A> → <[numeric] B+A>
>

**NOTE FOR DISCUSSION:** The justification for `DW_OP_plus_uconst`
  seems questionable to me. I do not see why you could not just use:
  `DW_OP_uconst <some number> DW_OP_plus`. Is this a historic
  limitation? Should this text be reconsidered?

> `DW_OP_shl`
>
>    <[integral base type or generic type] B> <[integral base type or generic type] A> → <[integral base type or generic type] B<<A >
>

**NOTE FOR DISCUSSION:** What happens if you shift left a negative
number of bytes? Should the number of bits to be shifted be limited to
to a positive value.

>
> `DW_OP_shr`
>
>    <[integral base type or generic type] B> <[integral base type or generic type] A> → <[integral base type or generic type] B>>A >

> `DW_OP_shra`
>
>    <[integral base type or generic type] B> <[integral base type or generic type] A> → <[integral base type or generic type] B shra A >
>

> `DW_OP_xor`
>
>    <[integral base type or generic type] B> <[integral base type or generic type] A> → <[integral base type or generic type] B xor A >
>

In section 3.6 add the following heading between operator and its
description as follows:

> `DW_OP_push_object_location`
>
>    → <[location] A >
>

> `DW_OP_form_tls_location`
>
>    <[integral] offset into TLS> → <[location] in thread's TLS >
>

**Fixme** IMHO the description needs to be rewritten.

> `DW_OP_call_frame_cfa`
>
>    → <[location] call frame location >
>

> `DW_OP_push_lane`
>
>    → <[unsigned integer] lane number >
>

In section 3.7 add the following heading between operator and its
description as follows:

> `DW_OP_addr` ([unsigned integer] address)
>
>    → <[location] memory storage >
>

> `DW_OP_addrx` ([ULEB] offset into .debug_addr )
>
>    → <[location] memory storage >
>

> `DW_OP_fbreg` ([ULEB] offset from frame base )
>
>    → <[location] memory_storage >
>

> `DW_OP_breg<n>` ([SLEB] offset)
>
>    → <[location] memory storage >
>

> `DW_OP_bregx` ([ULEB register number, [SLEB] offset)
>
>    → <[location] A >
>

In section 3.8 add the following heading between operator and its
description as follows:

> `DW_OP_reg<n>`
>
>    → <[location] register storage >
>

> `DW_OP_regx` ([ULEB] register number)
>
>    → <[location] register storage >
>

In section 3.9 add the following heading between operator and its
description as follows:

> `DW_OP_undefined`
>
>    → <[location] undefined storage >
>

In section 3.10 add the following heading between operator and its
description as follows:

> `DW_OP_implicit_value` ([ULEB] length, implicit storage bytes)
>
>    → <[location] implicit value storage >
>

> `DW_OP_stack_value`
>
>    <[*any*] value> → <[location] implicit value storage >
>

In section 3.11 add the following heading between operator and its
description as follows:

> `DW_OP_implicit_pointer` ([DIE reference] .debug_info offset, [SLEB] byte offset)
>
>    → <[location] implicit pointer storage>
>

In section 3.12 add the following heading between operator and its
description as follows:

> `DW_OP_composite`
>    → <[location] composite storage >
>

> `DW_OP_piece` ([ULEB] size in bytes)
>
>    → <[location] composite storage>
>

**NOTE FOR DISCUSSION** The original conception of piece had the
composite being created on the side not on the stack. Now that it is
on the stack, a way of understanding piece may be that it peeks at the
top of the stack and if the type is a composite location then it pops
off of the stack, modifies it and then pushes it again. Maybe we
should rewrite the description of piece this way?

<[location] composite storage > `DW_OP_piece` ([ULEB] size in
bytes) → <[location] composite storage>

If the top of the stack is not a composite, then it creates an empty
composite storage and then pushes that. Note that there is currently
no peek or typeof operator.

The problem with this is in stack based machines maintaining stack
alignment is very important. Not knowing and not being able to know if
a stack element is going to be popped can lead to stack misalignment.

> `DW_OP_bit_piece` ([ULEB] size in bits)
>
>    → <[location] composite storage>
>

In section 3.13 add the following heading between operator and its
description as follows:

> `DW_OP_deref`
>
>    <[location] L> → <[generic] A>
>

> `DW_OP_deref_size` ([1-byte integral] size)
>
>    <[location] L> → <[generic] A>
>

> `DW_OP_deref_type` ([1-byte integral] size, [ULEB] DIE offset for type )
>
>    <[location] L> → <[*specified type*] A>
>

> `DW_OP_xderef`
>
>    <[integral] address space identifier> <[location] L> → <[generic] A>

> `DW_OP_xderef_size` ([1-byte integral] size)
>
>    <[integral] address space identifier> <[location] L> → <[generic] A>
>

> `DW_OP_xderef_type` ([1-byte integral] size, [ULEB] DIE offset for type)
>
>    <[integral] address space identifier> <[location] L> → <[*specified type*] A>
>

In section 3.14 add the following heading between operator and its
description as follows:

> `DW_OP_offset`
>
>    <[location] L> <[signed integral] displacement> → <[location] L'>
>

> `DW_OP_bit_offset`
>
>    <[location] L> <[signed integral] displacement> → <[location] L'>
>

In section 3.15 add the following heading between operator and its
description as follows:

> `DW_OP_le`, `DW_OP_ge`, `DW_OP_eq`, `DW_OP_lt`, `DW_OP_gt`, `DW_OP_ne`
>
>    <[base type or generic] B> <[base type or generic] A> → <[[generic] 0 or 1>
>

**NOTE FOR DISCUSSION** Does this work for only integral base types or
  does it work for all base types? Some of the base types like vector
  types are kind of weird like the vector types? I think that this
  should be limited to integral base types.

> `DW_OP_skip` ([2-byte signed integer] bytes to skip)
>
>    *no stack effects*
>

> `DW_OP_bra` ([2-byte signed integer] bytes to skip)
>
>    <[numeric] condition> → *no stack result*
>

> `DW_OP_call[24]` ([2- or 4- byte unsigned integral] DIE offset)
>
>    *stack effects by agreement*
>

**NOTE FOR DISCUSSION**: Should I just call the type "DIE offset" and
"DiE reference" in all places. Should we change the operator's
description?

> `DW_OP_call_ref` ([4- or 8- byte unsigned integral] .debug_info offset)
>
>    *stack effects by agreement*
>

**NOTE FOR DISCUSSION**: These operators are interesting moving the
inline operands up to the operatation's declaration makes the line
below a "stack effects" line. How to express that?

In section 3.16 add the following heading between operator and its
description as follows:

> `DW_OP_convert` ([ULEB] or 0 for generic type)
>
>       <[*any*] A> → <[*specified type*] A'>
>

**NOTE FOR DISCUSSION** Should the types that can be converted be
specified? Some of the conversions can be kind of weird.

> `DW_OP_reinterpret` ([ULEB] or 0 for generic type)
>
>       <[*any*] A> → <[*specified type*] A'>
>

In section 3.17 add the following heading between operator and its
description as follows:

> `DW_OP_nop`
>
>    *no stack effects*
>

> `DW_OP_entry_value` ([ULEB] length, [DWARF expression] expression)
>
>    → <[generic] value>
>

**NOTE FOR DISCUSSION** What type does it push on the stack. It isn't
  clear to me from the operation description. Presumably it is the
  same as the formal parameter to which it is applied. This operator
  doesn't make sense to me for different reasons than the problems
  that Tony has with it.

> `DW_OP_extended` ([ULEB] extended opcode)
>
>    *stack effects defined by extended operation*
>

> `DW_OP_user_extended` ([ULEB] extended opcode)
>
>    *stack effects defined by extended operation*
>

**NOTE FOR DISCUSSION** We should consider expanding table 8.9 to
  include all the information in the header. We could easily expand it
  to include the number of stack operands and their expected types as
  well.
