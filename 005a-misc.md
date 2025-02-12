# Deferred Issues from Tony’s Review of Locations on Stack

## Section 2.5.1 DWARF Expression Evaluation Context

In item 6, "Current call frame," CFA is referred to as "Call Frame Address."
Should be "Canonical Frame Address."
Need to check document for consistency.

[ttye:] Change "must be an active call frame in the current call stack"
to "must be an active call frame in the current thread's call stack."
[ccoutant: I think this was already decided in the discussion that added
this section to the document.]

## Section 2.5.2 Values and Locations

Re: multiple locations.

> ...but can also be returned by a DWARF expression if it includes a
> `DW_OP_call*` to a debugging information entry that has a location
> list attribute.

## Section 2.5.2.3 Stack Operations

For `DW_OP_deref_size`:

> ... `L` from the stack. If `S` times 8 is larger than the bit size `TS` of the
> generic type, then the first `TS` bits are retrieved from the location `L`
> and pushed onto the stack as a value of the generic type. Otherwise, the
> first `S` bytes are retrieved from the location `L`, zero extended to the
> bit size of the generic type, and pushed onto the stack as a value of the
> generic type. </span>

For `DW_OP_deref_type`:

> The `DW_OP_deref_type` operation takes two operands. The first operand
> is a 1-byte unsigned integer that specifies the size in bytes `S` of the type
> given by the second operand. The second operand is an unsigned LEB128
> integer that represents the offset of a debugging information entry in
> the current compilation unit, which must be a `DW_TAG_base_type` entry
> that provides the type `T` of the value to be retrieved.
> The bit size `TS` of `T` rounded up to a byte size, must equal `S`.
> This operation pops a location `L` from the stack. The
> first `TS` bits are retrieved from the location `L` and pushed onto the
> stack as a value of type `T`.

For `DW_OP_xderef_size`:

> The top two stack
> elements are popped, and a data item is retrieved through an
> implementation-defined address calculation and pushed as the
> new stack top. In the `DW_OP_xderef_size` operation, however,
> the size in bytes of the data retrieved from the
> dereferenced
> address is specified by the single operand. This operand is a
> 1-byte unsigned integral constant <span class="del">whose value may not be larger
> than the size of an address on the target machine</span>. The data
> retrieved is <span class="add">truncated or</span> zero extended to the <span class="add">bit</span> size of <span class="del">an address on the
> target machine</span><span class="add">generic type</span> before being pushed onto the expression stack together
> with the generic type<span class="del"> identifier</span>.

For `DW_OP_xderef_type`:

> The `DW_OP_xderef_type` operation behaves like the `DW_OP_xderef_size`
> operation: it pops the top two stack entries, treats them as an address
> and an address space identifier, and pushes the value retrieved. In the
> `DW_OP_xderef_type` operation, the size in <span
> class="del">bytes</span><span class="add">bits</span> of the data
> retrieved from the dereferenced address is <span class="del">the
> first</span> <span class="add">the bit size of the type specified by the
> second</span> operand. <span class="del">This</span><span
> class="add">The first</span> operand is a 1-byte unsigned integral
> constant whose value value which is the same as the <span
> class="add">bit</span> size of the base type <span class="add">rounded
> up to a byte size</span> referenced by the second operand. The second
> operand is an unsigned LEB128 integer that represents the offset of a
> debugging information entry in the current compilation unit, which must
> be a `DW_TAG_base_type` entry that provides the type of the data pushed.

## 2.5.4 Literal and Constant Operations

> The following operations all push a value onto the DWARF stack.
> Operations other than `DW_OP_const_type` push a value with the
> generic type, and if the value of a constant in one of these
> operations is larger than can be <span class="del">stored in a single stack element</span><span class="add">represented by the stack element's type</span>,
> the value is truncated to the element<span class="add">'s type</span> size and the low-order bits
> are pushed on the stack.

## 2.5.5 Register Value Operations

[ttye: Note that this definition covers what happens if T is smaller
(truncation) or larger (illegal) than R.]

For `DW_OP_regval_type`:

> The `DW_OP_regval_type` operation
> pushes
> the contents of
> a given register interpreted as a value of a given type. The first
> operand is an unsigned LEB128 number,
> which identifies a register whose contents is to
> which identifies a register <span class="add">R</span> whose contents is to
> be pushed onto the stack. The second operand is an unsigned LEB128 number
> that represents the offset of a debugging information entry in the current
> compilation unit, which must be a `DW_TAG_base_type` entry that provides the
> type of the value contained in the specified register.
> type <span class="add">T</span> of the value contained in the specified register.
> <span class="add">It is equivalent to doing `DW_OP_regx R; DW_OP_deref_type T`.</span>

For `DW_OP_regval_bits`:

[ttye: There are a number of problems with this definition. It is not needed
with locations on the stack. It gets the register number from the stack so
it can be runtime computed: all other register operations use a literal. Is
it really needed to runtime compute this? Not sure when a compiler would
ever need to use this ability. The offset is defined in terms of the least
significant bit. This has the same problems as DW_OP_bit_piece and is not
helpful for architectures that support multiple endianess. Using the storage
bank ordering defined by the architecture provides full flexibility. Since
this was added as part of DWARF 6, would advocate to remove it from the
final version.]

## 2.5.14 Control Flow Operations

> <span class="add">If the `DW_AT_location` attribute is encoded using class
> `locdesc`, then these</span>
> operations transfer control of DWARF expression evaluation to
> the `DW_AT_location` attribute of the referenced debugging information entry. If
> there is no such attribute, then there is no effect....
>
> ...
> 
> If the `DW_AT_location` attribute is encoded using class
> `loclist` (or `vallist`), then the location (or value) list is evaluated
> using a separate empty stack, and the resulting location (or value) is
> pushed on the stack.
