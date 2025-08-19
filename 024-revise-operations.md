Standardize Operation Formatting
================================

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

DWARF operations potentially have three sources of input, operands
which are encoded in their byte stream, values on the stack, and the
context in which they are executed. They also can also modify the
stack and leave items on the stack. For every operand each of these
sources of input, output, and side effects are specified in a
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
> operation consisting of an opcode followed by zero or more literal
> operands. The number of operands is implied by the opcode. An may
> also receive parameters from the stack and make use of information
> from its evaluation context.
>
> *The description of the inputs and each of each operation will be as
> follows:*
>
> 1. DW_OP_name
>   - operands:
>     - first_operand [encoding type]
>     - second_operand [encoding type]
>   - stack parameters:
>     - T [expected types]: <optional> parameter1_name   # 4th entry on the stack
>     - Z [expected types]: <optional> parameter2_name   # 3rd entry on the stack
>     - Y [expected types]: <optional> parameter3_name  # 2nd entry on stack
>     - X [expected types]: <optional> parameter4_name   # top of stack
>   - stack output:
>     - X [resulting type]
>
> *The top four elements of the stack are given the names X, Y, Z, T as
> a matter of descriptive typographical convenience. In these
> descriptions the stack is presented as growing down with the deepest
> entry presented first. Unless otherwise stated, it is assumed that
> all stack parameters are consumed by the operation and all stack
> output is pushed onto the stack.*

In Section 3.2 replace the operation descriptions as follows:

> 1. `DW_OP_dup
>   stack parameters:
>     X [*any*]
>   stack output:
>     Y: X
>     X: X
> 
>    The DW_OP_dup operation duplicates the entry at the top of the stack.
>
> 2. `DW_OP_drop`
>   stack parameters:
>     Y [*any*]
>     X [*any*]
>   stack output:
>     X: Y
>
>   The DW_OP_drop operation pops the entry at the top of the stack.
>
> 3. `DW_OP_pick`
>   operand:
>     stack index [unsigned 1-byte integral in the range 0-255 inclusive]
>   stack_output:
>     X: stack entry @ stack index
>
>    The single operand of the DW_OP_pick operation provides a 1-byte
>    index. A copy of the stack entry with the specified index (0
>    through 255, inclusive) is pushed onto the stack.
>

NOTE FOR DISCUSSION. It should be noted that there is no pick
operation which can pick based upon an index which is passed on the
stack instead of as a literal operand. The fact that it hasn't been
requested may suggest that it is not needed.

> 4. `DW_OP_over`
>   stack_input:
>     Y [*any*]
>     X [*any*]
>   stack_output:
>     Z: Y
>     Y: X
>     X: Y
>
>    The DW_OP_over operation duplicates the entry currently second in
>    the stack at the top of the stack.
>
> *This is equivalent to a DW_OP_pick operation, with index 1.*
>
> 5. `DW_OP_swap`
>   stack_input:
>     Y [*any*]
>     X [*any*]
>   stack_output:
>     Y: X
>     X: Y
>
>    The DW_OP_swap operation swaps the top two stack entries. The entry
>    at the top of the stack becomes the
>    second stack entry, and the second entry (including its type
>    identifier) becomes the top of the stack.
>
> 6. `DW_OP_rot`
>   stack_input:
>     Z [*any*]
>     Y [*any*]
>     X [*any*]
>   stack_output:
>     Z: X
>     Y: Z
>     X: Y

In section 3.3 replace the operation descriptions as follows:

> 1. DW_OP_lit0`, ..., `DW_OP_lit31`
>   stack output:
>     X [unsigned integer in the range 0-31 inclusive]
>
>   The DW_OP_lit<n> operations encode the unsigned literal values from 0
>   through 31, inclusive.
>
> 2. `DW_OP_const1u`, DW_OP_const2u, DW_OP_const4u, DW_OP_const8u
>    operand:
>      constant [unsigned integer of the specified width]
>    stack output:
>      X [unsigned integral]: constant
>
>    The single operand of a DW_OP_const<n>u operation provides a 1, 2, 4, or
>    8-byte unsigned integer constant, respectively.
>
> 3. `DW_OP_const1s`, DW_OP_const2s, DW_OP_const4s, DW_OP_const8s
>    operand:
>      constant [signed integer of the specified width]
>    stack output:
>      X [signed integral]: constant
>
>   The single operand of a DW_OP_const<n>s operation provides a 1, 2, 4, or
>   8-byte signed integer constant, respectively.
>
> 4. `DW_OP_constu`
>    operand:
>      constant [LEB128 encoded unsigned integer]
>    stack output:
>      X [unsigned integer]: constant
>
>   The single operand of the DW_OP_constu operation provides an unsigned
>   LEB128 integer constant.
>
> 5. `DW_OP_consts`
>    operand:
>      constant [LEB128 encoded signed integer]
>    stack output:
>      X [signed integer]: constant
>
>   The single operand of the DW_OP_consts operation provides a signed
>   LEB128 integer constant.
>
> 6. `DW_OP_constx`
>    operand:
>      debug_addr offset [LEB128 encoded unsigned integer]
>    stack output:
>      X [unsigned integer]: debug_addr offset
>
>   The DW_OP_constx operation has a single operand that encodes an
>   unsigned LEB128 value, which is a zero-based index into the
>   .debug_addr section, where a constant, the size of a generic type,
>    is stored. This index is relative to the value of the DW_AT_addr_base
>   attribute of the associated compilation unit.
>
>   *The DW_OP_constx operation is provided for constants that require
>   link-time relocation but should not be interpreted by the consumer as
>   a relocatable address (for example, offsets to thread-local storage).*
>
> 7. `DW_OP_const_type`
>    operands:
>      type die offset [LEB128 encoded offset]
>      size [1-byte unsigned integer]
>      constant
>    stack output:
>      X [*specified type*]: constant
>
>   The DW_OP_const_type operation takes three operands. The first
>   operand is an unsigned LEB128 integer that represents the offset of
>   a debugging information entry in the current compilation unit, which
>   must be a DW_TAG_base_type entry that provides the type of the
>   constant provided.  The second operand is a 1-byte unsigned integer
>   that specifies the size of the constant value, which is the same as
>   the size of the base type referenced by the first operand. The third
>   operand is a sequence of bytes of the given size that is interpreted
>   as a value of the referenced type.
>
>   *While the size of the constant can be inferred from the base type
>   definition, it is encoded explicitly into the operation so that the
>   operation can be parsed easily without reference to the .debug_info
>   section.*

NOTE: I'm not really buying the non-normative explanation. I think it
was a mistake and you should flat out say that and possibly suggest
DE_OP_const_type2 which drops the size operand.

> X. DW_OP_overlay
>   stack parameters:
>     T [location]: base location
>     Z [location]: overlay location
>     Y [unsigned int]: base offset
>     X [unsigned int]: overlay width
>   stack output:
>     X [location]: composite_location

>   DW_OP_overlay pushes a new composite location whose storage is the
>   result of replacing a slice in the storage of `base location` with
>   a slice from the storage of `overlay location`.
>
>   The slice of bytes obtained from the storage of `overlay location`
>   is referred to as the `overlay`.  The `overlay` begins at `overlay
>   location` and has a size of `overlay width`.  The `overlay`
>   must not extend outside of the bounds of the storage of `overlay
>   location`.
>
>   The slice of bytes replaced in the storage of `base location` is
>   referred to as the `overlay base`.  It begins at `base location`
>   offset by `base offset` and has a size of `overlay width`.
>
>   *If the `overlay width` is zero. Then the consumer may leave the
>   `base location` on the top of the stack rather than creating
>   composite storage.*
>
>   If the `overlay base` extends beyond the bounds of the storage of
>   `base location`, the storage of the resulting location is first
>   extended by adding undefined storage sufficient to cover the
>   `overlay base`.
>
>   The offset of the resulting location is the offset of `base
>   location`.
>



LocalWords:  DW lit0 lit31 const1u const2u const4u const8u const addr
LocalWords:  const1s const2s const4s const8s constu consts constx
LocalWords:  type2
