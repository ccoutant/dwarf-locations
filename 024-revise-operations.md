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

DWARF operations potentially have three sources of input: inline
parameters which are encoded in their byte stream, operands on the
stack, and the context in which they are executed. They can also
modify the stack and leave items on the stack. For every operator each
of these sources of input, output, and side effects are specified in a
standardized format similar to how other stack machine operations have
been documented in the past.

Also as the DWARF standard has evolved, the use of of the word
"operand" and "paramter" has been used inconsistently within the
standard. In the past "operand" was often used to describe the "inline
parameters" encoded in the DWARF expression byte stream while "stack
paramters" or "stack entries" were used for the "stack operands" which
were acted upon by the operator.  This proposal seeks to use the
terminology consistently within Chapter 3 of the standard.

**NOTE FOR DISCUSSION** While it makes sense within the context of
  DWARF expressions to call things in the DWARF byte stream inline
  parameters and the things on the stack "operands", there are many
  other cases within the DWARF spec where the values within the DWARF
  byte stream are refered to as "operands". It is going to be a very
  big lift to change all those other uses of the word operand and
  might be a bit much for the DWARF community.


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
>      - inline parameters:
>        - first_operand [*encoding type*]
>        - second_operand [*encoding type*]
>      - stack operands:
>        - T [*expected types*]: <*optional*> operand1_name # 4th entry on the stack
>        - Z [*expected types*]: <*optional*> operand2_name # 3rd entry on the stack
>        - Y [*expected types*]: <*optional*> operand3_name # 2nd entry on stack
>        - X [*expected types*]: <*optional*> operand4_name # top of stack
>      - stack output:
>        - X [*resulting type*]
>
>
> *The top four elements of the stack are given the names X, Y, Z, T as
> a matter of descriptive typographical convenience. In these
> descriptions, the stack is presented as growing down with the deepest
> entry presented first. Unless otherwise stated, it is assumed that
> all stack operands are consumed by the operation and all stack
> output is pushed onto the stack.*

In Section 3.1 replace `operand` with `parameter` in the sentance:

>   * For example, the evaluation of the expression E associated with
>   a `DW_AT_location` attribute of the debug information entry
>   operand of the `DW_OP_call<n>` operations is evaluated with the
>   compilation unit that contains E and not the one that contains the
>   `DW_OP_call<n>` operation expression.*


In Section 3.2 replace the operation descriptions as follows:

> 1. `DW_OP_dup`
>       - stack operands:
>         - X [*any*]
>       - stack output:
>         - Y: X
>         - X: X
>
>   The DW_OP_dup operation pops the operand at the top of the stack, then
>   pushes it twice.
>
> 2. `DW_OP_drop`
>       - stack operands:
>         - Y [*any*]
>         - X [*any*]
>       - stack output:
>         - X: Y
>
>   The DW_OP_drop operation pops the operand at the top of the stack.
>
> 3. `DW_OP_pick`
>       - inline parameters:
>         - stack index [unsigned 1-byte integral in the range 0-255 inclusive]
>       - stack output:
>         - X: stack entry @ stack index
>
>    The single parameter of the DW_OP_pick operation provides a 1-byte
>    index. A copy of the stack entry with the specified index (0
>    through 255, inclusive) is pushed onto the stack.
>

**NOTE FOR DISCUSSION**: It should be noted that there is no pick
operation which can pick based upon an index which is passed on the
stack instead of as a literal operand. The fact that it hasn't been
requested may suggest that it is not needed.

> 4. `DW_OP_over`
>       - stack operands:
>         - Y [*any*]
>         - X [*any*]
>       - stack output:
>         - Z: Y
>         - Y: X
>         - X: Y
>
>    The `DW_OP_over` operation duplicates the entry currently second in
>    the stack and pushes it to the top of the stack.
>
>    *This is equivalent to a `DW_OP_pick 1` operation.*
>
> 5. `DW_OP_swap`
>       - stack operands:
>         - Y [*any*]
>         - X [*any*]
>       - stack output:
>         - Y: X
>         - X: Y
>
>   The `DW_OP_swap` operation pops the top two stack entries then
>   pushes them back onto the stack in the opposite order.
>
>
> 6. `DW_OP_rot`
>       - stack operands:
>         - Z [*any*]
>         - Y [*any*]
>         - X [*any*]
>       - stack output:
>         - Z: X
>         - Y: Z
>         - X: Y
>
>   The DW_OP_rot operation rotates the first three stack entries. The
>   entry at the top of the stack (including its type identifier)
>   becomes the third stack entry, the second entry (including its
>   type identifier) becomes the top of the stack, and the third entry
>   (including its type identifier) becomes the second entry.

In section 3.3 replace the operation descriptions as follows:

> 1. `DW_OP_lit0`, ..., `DW_OP_lit31`
>       - stack output:
>         - X [unsigned integer in the range 0-31 inclusive]
>
>   The `DW_OP_lit<n>` operations encode the unsigned literal values from 0
>   through 31, inclusive.
>
> 2. `DW_OP_const1u`, `DW_OP_const2u`, `DW_OP_const4u`, `DW_OP_const8u`
>       - inline parameter:
>         - constant [unsigned integer of the specified width]
>       - stack output:
>         - X [unsigned integer]
>
>    The single inline parameter of a `DW_OP_const<n>u` operation
>    provides a 1, 2, 4, or 8-byte unsigned integer constant,
>    respectively.
>
> 3. `DW_OP_const1s`, `DW_OP_const2s`, `DW_OP_const4s`, `DW_OP_const8s`
>       - inline parameter:
>         - constant [signed integer of the specified width]
>       - stack output:
>         - X [signed integer]
>
>   The single inline parameter of a `DW_OP_const<n>s` operation
>   provides a 1, 2, 4, or 8-byte signed integer constant,
>   respectively.
>
> 4. `DW_OP_constu`
>       - inline parameter:
>         - constant [ULEB128]
>       - stack output:
>         - X [unsigned integer]
>
>   The single inline parameter of the `DW_OP_constu` operation provides an unsigned
>   LEB128 integer constant.
>
> 5. `DW_OP_consts`
>       - inline parameter:
>         - constant [LEB128]
>       - stack output:
>         - X [signed integer]
>
>   The single inline parameter of the `DW_OP_consts` operation provides a signed
>   LEB128 integer constant.
>
> 6. `DW_OP_constx`
>       - inline parameter:
>         - .debug_addr offset [LEB128]
>       - stack output:
>         - X [unsigned integer]
>
>   The `DW_OP_constx` operation has a single inline parameter that encodes an
>   unsigned LEB128 value, which is a zero-based index into the
>   .debug_addr section, where a constant, the size of a generic type,
>    is stored. This index is relative to the value of the DW_AT_addr_base
>   attribute of the associated compilation unit.
>
>   *The `DW_OP_constx` operation is provided for constants that require
>   link-time relocation but should not be interpreted by the consumer as
>   a relocatable address (for example, offsets to thread-local storage).*
>
> 7. `DW_OP_const_type`
>       - inline parameters:
>         - type DIE offset [LEB128]
>         - size [1-byte unsigned integer]
>         - constant
>       - stack output:
>         - X [*specified type*]
>
>   The `DW_OP_const_type` operation takes three inline parameters. The first
>   is an unsigned LEB128 integer that represents the offset of
>   a debugging information entry in the current compilation unit, which
>   must be a `DW_TAG_base_type` entry that provides the type of the
>   constant provided.  The second inline parameter is a 1-byte unsigned integer
>   that specifies the size of the constant value, which is the same as
>   the size of the base type referenced by the first operand. The third
>   inline parameter is a sequence of bytes of the given size that is interpreted
>   as a value of the referenced type.
>
>   *While the size of the constant can be inferred from the base type
>   definition, it is encoded explicitly into the operation so that the
>   operation can be parsed easily without reference to the .debug_info
>   section.*

In section 3.4 replace the operation descriptions as follows:

> 1. `DW_OP_regval_type`
>       - inline parameters:
>         - register number [LEB128]
>         - offset of type DIE [LEB128]
>       - stack output:
>         - X [*specified type*]
>
>   The `DW_OP_regval_type` operation pushes the contents of a given
>   register interpreted as a value of a given type. The first inline
>   parameter is an unsigned LEB128 number, which identifies a
>   register whose contents is to be pushed onto the stack. The second
>   inline parameter is an unsigned LEB128 number that represents the
>   offset of a debugging information entry in the current compilation
>   unit, which must be a DW_TAG_base_type entry that provides the
>   type of the value contained in the specified register.

In section 3.5 replace the operation descriptions as follows:

> 1. `DW_OP_abs`
>       - stack operands
>         - X [numeric type]
>        - stack output:
>          - X [numeric type]
>
>   The `DW_OP_abs` operation pops the operand from the stack,
>   interprets it as a signed value and pushes its absolute value. If
>   the absolute value cannot be represented, the result is undefined.
>
> 2. `DW_OP_and`
>       - stack operands
>         - Y [integral base type or generic type]
>         - X [integral base type or generic type]
>       - stack output:
>         - X [integral base type or generic type]
>
>   The `DW_OP_and` operation pops the two operands, performs a
>   bitwise and operation, and pushes the result.
>
> 3. `DW_OP_div`
>       - stack operands
>         - Y [numeric type] numerator
>         - X [numeric type] denomenator
>       - stack output:
>         - X [numeric type]
>
>   The `DW_OP_div` operation pops the top operands, divides the
>   former second entry by the former top of the stack using signed
>   division, and pushes the result.
>
> 4. `DW_OP_minus`
>       - stack operands
>         - Y [numeric type]
>         - X [numeric type]
>       - stack output:
>         - X [numeric type]
>
>   The `DW_OP_minus` operation pops the two operands, subtracts the
>   former top of the stack from the former second entry, and pushes
>   the result.
>
> 5. `DW_OP_mod`
>       - stack operands
>         - Y [integral base type or generic type]
>         - X [integral base type or generic type]
>       - stack output:
>         - X [integral base type or generic type]
>
>   The `DW_OP_mod` operation pops the two operands and pushes the
>   result of the calculation: former second stack entry modulo the
>   former top of the stack.
>
> 6. `DW_OP_mul`
>       - stack operands
>         - Y [numeric type]
>         - X [numeric type]
>       - stack output:
>         - X [numeric type]
>
>   The `DW_OP_mul` operation pops the two operands, multiplies them
>   together, and pushes the result.
>
> 7. `DW_OP_neg`
>       - stack operands
>         - X [integral base type or generic type]
>       - stack output:
>         - X [integral base type or generic type]
>
>   The `DW_OP_neg` operation pops the operand, interprets it as a
>   signed value and pushes its negation. If the negation cannot be
>   represented, the result is undefined.
>
> 8. `DW_OP_not`
>       - stack operands
>         - X [integral base type or generic type]
>       - stack output:
>         - X [integral base type or generic type]
>
>   The `DW_OP_not` operation pops the operand, and pushes its bitwise
>   complement.
>
> 9. `DW_OP_or`
>       - stack operands
>         - Y [integral base type or generic type]
>         - X [integral base type or generic type]
>       - stack output:
>         - X [integral base type or generic type]
>
>   The `DW_OP_or` operation pops the operands, performs a bitwise or
>   operation on the two, and pushes the result.
>
> 10. `DW_OP_plus`
>        - stack operands
>          - Y [numeric type]
>          - X [numeric type]
>        - stack output:
>          - X [numeric type]
>
>   The `DW_OP_plus` operation pops the operands, adds them together,
>   and pushes the result.
>
> 11. DW_OP_plus_uconst
>       - inline parameters:
>         - X [ULEB128]
>       - stack operands
>         - Y [numeric type]
>       - stack output:
>         - X [numeric type]
>
>   The `DW_OP_plus_uconst` operation pops the operand, adds
>   it to the unsigned LEB128 constant operand interpreted as the same
>   type as the operand popped from the top of the stack and pushes
>   the result.
>
>   *This operation is supplied specifically to be able to
>   encode more field offsets in two bytes than can be done with
>   `DW_OP_lit<n> DW_OP_plus.`*

**NOTE FOR DISCUSSION:** The justification for `DW_OP_plus_uconst`
  seems questionable to me. I do not see why you could not just use:
  `DW_OP_uconst <some number> DW_OP_plus`. Is this a historic
  limitation? Should this text be reconsidered?

> 12. `DW_OP_shl`
>        - stack operands
>          - Y: value to be shifted [integral base type or generic type]
>          - X: bits to shift [integral base type or generic type]
>        - stack output:
>          - X [integral base type or generic type]
>
>   The `DW_OP_shl` operation pops the two operands, shifts Y, the
>   former operand, left (filling with zero bits) by the number of
>   bits specified in X, the former top of the stack. It then pushes
>   the result.
>
> 13. `DW_OP_shr`
>        - stack operands
>          - Y: value to be shifted [integral base type or generic type]
>          - X: bits to shift [integral base type or generic type]
>        - stack output:
>          - X [integral base type or generic type]
>
>   The `DW_OP_shr` operation pops the two operands, shifts Y,
>   the former second entry, right logically (filling with zero bits)
>   by the number of bits specified in X, the former top of the stack.
>   It then pushes the result.
>
> 14. `DW_OP_shra`
>        - stack operands
>          - Y: value to be shifted [integral base type or generic type]
>          - X: bits to shift [integral base type or generic type]
>        - stack output:
>          - X [integral base type or generic type]
>
>   The `DW_OP_shra` operation pops the two operands, shifts Y, the
>   former second entry, right arithmetically (divide the magnitude by
>   2, keeping the same sign for the result) by the number of bits
>   specified in X, the former top of the stack. It then pushes the
>   result.
>
> 15. `DW_OP_xor`
>        - stack operands
>          - Y [integral base type or generic type]
>          - X [integral base type or generic type]
>        - stack output:
>          - X [integral base type or generic type]
>
>   The `DW_OP_xor` operation pops the two operands, performs a
>   bitwise exclusive-or operation, and pushes the result.

In section 3.6 replace the operation descriptions as follows:

> 1. `DW_OP_push_object_location`
>       - stack output:
>         - X [location]
>
>     The `DW_OP_push_object_location` operation pushes the
>     location of the current object (see section 3.1) onto the
>     stack, as part of evaluation of a user-presented
>     expression.
>
>     *This object may correspond to an independent variable described
>     by its own debugging information entry; or it may be a component
>     of an array, structure, or class whose address has been
>     dynamically determined by an earlier step during user expression
>     evaluation.*
>
>     *This operator provides explicit functionality (especially for
>     arrays involving descriptors) that is analogous to the implicit
>     push of the base address of a structure prior to evaluation of a
>     `DW_AT_data_member_location` to access a data member of a
>     structure. For an example, see Appendix D.2 on page 304.*
>
>     *In previous versions of DWARF, this operator was named
>     `DW_OP_push_object_address`.  An alias to the old name is still
>     provided in DWARF 6 for compatibility.*
>
> 2. `DW_OP_form_tls_location`
>       - stack operands:
>         - X: thread ID [integral type]
>       - stack output:
>         - X [location] location for TLS for thread
>
>     The `DW_OP_form_tls_location` operation pops an integral type
>     operand for the thread ID. The meaning of the value for the
>     thread ID prior to this operation is defined by the run-time
>     environment. If the run-time environment supports multiple
>     thread-local storage blocks for a single thread, then the block
>     corresponding to the executable or shared library containing
>     this DWARF expression is used. This operation then translates
>     this value into a location in the thread-local storage for the
>     current thread (see Section 3.1), and pushes the location onto
>     the stack.

**Fixme** IMHO needs to be rewritten.

>
>     *Some implementations of C, C++, Fortran, and other languages,
>     support a thread-local storage class. Variables with this
>     storage class have distinct values and addresses in distinct
>     threads, much as automatic variables have distinct values and
>     addresses in each function invocation. Typically, there is a
>     single block of storage containing all thread-local variables
>     declared in the main executable, and a separate block for the
>     variables declared in each shared library. Each thread-local
>     variable can then be accessed in its block using an
>     identifier. This identifier is typically an offset into the
>     block and pushed onto the DWARF stack by one of the
>     `DW_OP_const<n><x>` operations prior to the
>     `DW_OP_form_tls_location` operation. Computing the address of
>     the appropriate block can be complex (in some cases, the
>     compiler emits a function call to do it), and difficult to
>     describe using ordinary DWARF location expressions. Instead of
>     forcing complex thread-local storage calculations into the DWARF
>     expressions, the `DW_OP_form_tls_location` operation allows the
>     consumer to perform the computation based on the run-time
>     environment.*
>
>     *In previous versions of DWARF, this operator was named
>     `DW_OP_form_tls_address`. An alias to the old name is still
>     provided in DWARF 6 for compatibility.*
>
> 3. `DW_OP_call_frame_cfa`
>       - stack output:
>         - X: [location] call frame location
>
>   The `DW_OP_call_frame_cfa` operation pushes the value of the
>   current call frame address (CFA), obtained from the Call Frame
>   Information (see Section X on page Y and Section Z on page
>   T).
>
>   *Although the value of `DW_AT_frame_base` can be computed using
>   other DWARF expression operators, in some cases this would require
>   an extensive location list because the values of the registers
>   used in computing the CFA change during a subroutine. If the Call
>   Frame Information is present, then it already encodes such
>   changes, and it is space efficient to reference that.*
>
> 4. `DW_OP_push_lane`
>       - stack output:
>         - X [unsigned integer] lane number
>
>   The DW_OP_push_lane operation pushes a lane index value of the
>   generic type, which provides the context of the lane in which the
>   expression is being evaluated (see Section X on page Y and
>   Section Z on page T).

In section 3.7 replace the operation descriptions as follows:

> 1. `DW_OP_addr`
>       - inline parameters:
>         - address [unsigned integer]
>       - stack output:
>         - X [location]
>
>   The `DW_OP_addr` operation has a single parameter that encodes a
>   machine address and whose size is the size of an address on the
>   target machine. The value of this operand is treated as an address
>   in the default address space and the corresponding memory location
>   is pushed onto the stack.
>
> 2. `DW_OP_addrx`
>       - inline parameters:
>         - offset into .debug_addr [ULEB128]
>       - stack output:
>         - X [location]
>
>   The `DW_OP_addrx` operation has a single parameter that encodes an
>   unsigned LEB128 value, which is a zero-based index into the
>   `.debug_addr` section, where a machine address is stored. This
>   index is relative to the value of the `DW_AT_addr_base` attribute
>   of the associated compilation unit. The address obtained is
>   treated as an address in the default address space and the
>   corresponding memory location is pushed onto the stack.
>
> 3. `DW_OP_fbreg`
>       - inline parameters:
>         - offset from frame base [LEB128]
>       - stack output:
>         - X [location]
>
>   The `DW_OP_fbreg` operation provides a signed LEB128 byte offset
>   `B` from the location specified by the location expression in the
>   `DW_AT_frame_base` attribute of the current function (see Section
>   3.1).  The frame base location, offset by `B` bytes, is pushed
>   onto the stack.
>
>   *This is typically a stack pointer register plus or minus some
>   offset.*

**NOTE FOR DISCUSSION**: The use of a variable B here doesn't match
  any previous operator descriptions. Also why is there an elipse
  following the operator in 005? (removed)

>
> 4. `DW_OP_breg0`, ..., `DW_OP_breg31`
>       - inline parameters:
>         - offset from register [LEB128]
>       - stack output:
>         - X [location] register storage
>
>   The single operand of the `DW_OP_breg<n>` operations provides a
>   signed LEB128 byte offset. The contents of the specified register
>   (0–31) are treated as a memory address in the default address
>   space. The offset is added to the address obtained from the
>   register and the resulting memory location is pushed onto the
>   stack.
>
> 5. `DW_OP_bregx`
>       - inline parameters:
>         - register number [ULEB128]
>         - offset from register [LEB128]
>       - stack output:
>         - X [location] register storage
>
>   The `DW_OP_bregx` operation has two parameters. The first is a
>   register number which is specified by an unsigned LEB128.  The
>   second parameter is a signed LEB128 byte offset. It is the same as
>   `DW_OP_breg<n>` except it uses the register and offset provided by
>   the parameters.

In section 3.8 replace the operation descriptions as follows:

> 1. `DW_OP_reg0`, `DW_OP_reg1`, ..., `DW_OP_reg31`
>       - stack output:
>         - X [location] register storage
>
>   The `DW_OP_reg<n>` operations encode the names of up to 32
>   registers, numbered from 0 through 31, inclusive. A location that
>   names the designated register, with an offset of 0, is formed and
>   pushed on the stack.
>

> 2. `DW_OP_regx`
>       - inline parameters:
>         - register number [ULEB128]
>       - stack output:
>         - X [location] register storage
>
>   The `DW_OP_regx` operation has a single unsigned LEB128 literal
>   operand that encodes the name of a register. A location that names
>   the designated register, with an offset of 0, is formed and pushed
>   on the stack.

In section 3.9 replace the operation descriptions as follows:

> 1. `DW_OP_undefined`
>       - stack output:
>         - X [location] undefined storage
>
>   The `DW_OP_undefined` operation pushes an undefined location with
>   an offset of 0 onto the stack.

In section 3.10 replace the operation descriptions as follows:

> 1. `DW_OP_implicit_value`
>       - inline parameters:
>         - length [ULEB128]
>         - implicit storage bytes
>       - stack output:
>         - X [location] implicit value storage
>
>   The `DW_OP_implicit_value` operation specifies an immediate value
>   using two inline parameters: an unsigned LEB128 length, followed by a
>   sequence of bytes of the given length that contain the value. A
>   location `L` is formed for a block of implicit storage which
>   contains the given byte sequence. The offset of `L` is set to 0,
>   and `L` is pushed onto the stack.

**NOTE FOR DISCUSSION** Cary says he has removed these single letter
  abbreviations in some draft but it is not in his github or the
  version of the proposal on dwarfstd.org

> 2. `DW_OP_stack_value`
>       - stack operands:
>         - V value [any]
>       - stack output:
>         - X [location] implicit value storage
>
>   The `DW_OP_stack_value` operation specifies that the object does
>   not exist in memory but its value is nonetheless known and is at
>   the top of the DWARF expression stack. The value `V` on top of the
>   stack is popped, and a location for `L` is formed for a block of
>   implicit storage containing the value `V`, represented using the
>   encoding and byte order of the value's type. The offset of `L` is
>   set to 0, and `L` is pushed onto the stack.

**NOTE FOR DISCUSSION** another one that uses single letter
  abbreviations. I don't like the way it works in the new syntax.

In section 3.11 replace the operation descriptions as follows:

> 1. `DW_OP_implicit_pointer`
>       - inline parameters:
>         - value DIE [4 or 8 byte unsigned integral]
>         - ofset [LEB128]
>       - stack output:
>         - X [location] implicit pointer storage
>
>   The `DW_OP_implicit_pointer` operation has two inline parameters: a
>   reference to a debugging information entry that describes the
>   dereferenced object's value, and a signed number that is treated
>   as a byte offset from the start of that value.  The first operand
>   is a 4-byte unsigned value in the 32-bit DWARF format, or an
>   8-byte unsigned value in the 64-bit DWARF format (see Section
>   {32bitand64bitdwarfformats}) that is used as the offset of the
>   debugging information entry in the `.debug_info` section of the
>   current executable or shared object file.  The second operand, the
>   byte offset, is a signed LEB128 number.
>
>   An implicit pointer storage location is created for the location
>   of the pointer object and is pushed onto the stack.
>
>   *The debugging information entry referenced by a
>   `DW_OP_implicit_pointer` operation is typically a
>   `DW_TAG_variable` or `DW_TAG_formal_parameter` entry whose
>   `DW_AT_location` attribute gives a second DWARF expression or a
>   location list that describes the value of the object, but the
>   referenced entry may be any entry that contains a `DW_AT_location`
>   or `DW_AT_const_value` attribute (for example,
>   `DW_TAG_dwarf_procedure`).  By using the second DWARF expression,
>   a consumer can reconstruct the value of the object when asked to
>   dereference the pointer described by the original DWARF expression
>   containing the `DW_OP_implicit_pointer` operation.*

In section 3.12 replace the operation descriptions as follows:

> 1. `DW_OP_composite`
>       - stack output:
>         - X [location] composite storage
>
>   The `DW_OP_composite` pushes a new, empty, composite location onto
>   the stack, with an offset of 0.
>
>   *This operator is provided so that a new series of piece
>   operations can be started to form a composite location when the
>   state of the stack is unknown (e.g., following a `DW_OP_call`
>   operation), or when a new composite is to be started (e.g., rather
>   than add to a previous composite location on the stack).*
>
> 2. `DW_OP_piece`
>       - inline parameters:
>         - size in bytes [ULEB128]
>       - stack output:
>         - X [location] composite storage
>
>   The `DW_OP_piece` operation has a single parameter, which is an
>   unsigned LEB128 number. The number describes the size *N*, in
>   bytes, of the piece of the object to be appended to the composite
>   `LC`.  The *EP* value of the new tuple is set to *SP + N ×
>   byte_size.

**NOTE FOR DISCUSSION** Another single letter abbreviation that needs to be fixed.

>
>   If `LP` is a register location, but the piece does not occupy the
>   entire register, the placement of the piece within that register
>   is defined by the ABI.
>
> 3. `DW_OP_bit_piece`
>       - inline parameters:
>         - size in bits [ULEB128]
>         - offset in bits [ULEB128]
>      - stack output:
>         - X [location] composite storage
>
>   The `DW_OP_bit_piece` operation takes two parameters. The first is
>   an unsigned LEB128 number that gives the size *N*, in bits, of the
>   piece to be appended.  The second is an unsigned LEB128 number
>   that gives the offset in bits to be applied to the location `LP`.
>   The *EP* value of the new tuple is set to *SP + L*.
>
>   Interpretation of the offset depends on the type of location. If
>   the location is an undefined location (see Section 3.9), the
>   `DW_OP_bit_piece` operation describes a piece consisting of the
>   given number of bits whose values are undefined, and the offset is
>   ignored. If the location is a memory location (see Section 3.7),
>   the `DW_OP_bit_piece` operation describes a sequence of bits
>   relative to the location whose address is on the top of the DWARF
>   stack using the bit numbering and direction conventions that are
>   appropriate to the current language on the target system. In all
>   other cases, the source of the piece is given by either a register
>   location (see Section 3.8) or an implicit value location (see
>   Section 3.9); the offset is from the least significant bit of the
>   source value.
>
>   *The `DW_OP_bit_piece` operator is used instead of `DW_OP_piece`
>   when the piece to be assembled into a value or assigned to is not
>   byte-sized or is not at the start of a register or addressable
>   unit of memory.*
>
>   *Whether or not a `DW_OP_piece` operation is equivalent to any
>   `DW_OP_bit_piece` operation with an offset of 0 is ABI dependent.*

In section 3.13 replace the operation descriptions as follows:

> 1. `DW_OP_deref`
>       - stack operands:
>         - X [location]
>       - stack output:
>         - X [generic type]
>
> The `DW_OP_deref` operation pops a location `L` from the
> top of the stack. The first `S` bits, where `S` is the
> size (in bits) of an address on the target machine, are
> retrieved from the location `L` and pushed onto the stack
> as a value of the generic type.
>
> 2. `DW_OP_deref_size`
>       - inline parameters:
>         - size [1-byte integral]
>       - stack operands:
>         - X [location]
>       - stack output:
>         - X [generic type]
>
> The `DW_OP_deref_size` takes a single 1-byte unsigned integral
> parameter that specifies the size `S`, in bytes, of the value to be
> retrieved.  The size `S` must be no larger than the size of the
> generic type. The operation behaves like the `DW_OP_deref`
> operation: it pops a location `L` from the stack. The first `S`
> bytes are retrieved from the location `L`, zero extended to the size
> of the generic type, and pushed onto the stack as a value of the
> generic type.
>
> 3. `DW_OP_deref_type`
>       - inline parameters:
>         - size [1-byte integral]
>         - DIE offset for type [ULEB128]
>       - stack operands:
>         - X [location]
>       - stack output:
>         - X [*specified type*]
>
> The `DW_OP_deref_type` operation takes two parameters. The first is
> a 1-byte unsigned integer that specifies the byte size `S` of the
> type given by the second parameter. The second parameter is an
> unsigned LEB128 integer that represents the offset of a debugging
> information entry in the current compilation unit, which must be a
> `DW_TAG_base_type` entry that provides the type `T` of the value to
> be retrieved. The size `S` must be the same as the byte size of the
> base type represented by the type `T`. This operation pops a
> location `L` from the stack. The first `S` bytes are retrieved from
> the location `L` and pushed onto the stack as a value of type `T`.
>
> 4. `DW_OP_xderef`
>       - stack operands:
>         - Y [integral] address space identifier
>         - X [location]
>       - stack output:
>         - X [generic type]
>
>   The `DW_OP_xderef` operation provides an extended dereference
>   mechanism.  The entry at the top of the stack is treated as an
>   address. The second stack entry is treated as an “address space
>   identifier” for those architectures that support multiple address
>   spaces. Both of these entries must have integral types. The top
>   two stack elements are popped, and a data item is retrieved
>   through an implementation-defined address calculation and pushed
>   as the new stack top together with the generic type. The size of
>   the data retrieved from the dereferenced address is the size of
>   the generic type.
>
> 5. `DW_OP_xderef_size`
>       - inline parameters:
>         - size [1-byte integral]
>       - stack operands:
>         - Y [integral] address space identifier
>         - X [location]
>       - stack output:
>         - X [generic type]
>
>   The `DW_OP_xderef_size` operation behaves like the `DW_OP_xderef`
>   operation. The entry at the top of the stack is treated as an
>   address. The second stack entry is treated as an “address space
>   identifier” for those architectures that support multiple address
>   spaces. Both of these entries must have integral types. The top
>   two stack elements are popped, and a data item is retrieved
>   through an implementation-defined address calculation and pushed
>   as the new stack top. In the DW_OP_xderef_size operation, however,
>   the size in bytes of the data retrieved from the dereferenced
>   address is specified by the single operand. This operand is a
>   1-byte unsigned integral constant whose value may not be larger
>   than the size of an address on the target machine. The data
>   retrieved is zero extended to the size of an address on the target
>   machine before being pushed onto the expression stack as a generic
>   type.
>
> 6. `DW_OP_xderef_type`
>       - inline parameters:
>         - size [1-byte integral]
>         - DIE offset for type [ULEB128]
>       - stack operands:
>         - Y [integral] address space identifier
>         - X [location]
>       - stack output:
>         - X [*specified type*]
>
>   The `DW_OP_xderef_type` operation behaves like the
>   `DW_OP_xderef_size` operation: it pops the top two stack entries,
>   treats them as an address and an address space identifier, and
>   pushes the value retrieved. In the `DW_OP_xderef_type` operation,
>   the size in bytes of the data retrieved from the dereferenced
>   address is specified by the first parameter. This parameter is a
>   1-byte unsigned integral constant whose is the same as the size of
>   the base type referenced by the second parameter. The second
>   paramters is an unsigned LEB128 integer that represents the offset
>   of a debugging information entry in the current compilation unit,
>   which must be a DW_TAG_base_type entry that provides the type of
>   the data pushed.

In section 3.14 replace the operation descriptions as follows:

> 1. `DW_OP_offset`
>       - stack operands:
>         - Y [location]
>         - X [signed integral] displacement
>       - stack output:
>         - X [location]
>
>   `DW_OP_offset` pops two operands. The first (top of stack)
>   must be an integral type value, which represents a signed byte
>   displacement. The second must be a location. It forms an updated
>   location by adding the given byte displacement to the offset
>   component of the original location and pushes the updated location
>   onto the stack.
>
> 2. `DW_OP_bit_offset`
>       - stack operands:
>         - Y [location]
>         - X [signed integral] displacement in bits
>       - stack output:
>         - X [location]
>
>   `DW_OP_bit_offset` pops two operands. The first (top of
>   stack) must be an integral type value, which represents a signed
>   bit displacement. The second must be a location. It forms an
>   updated location by adding the given bit displacement to the
>   offset component of the original location and pushes the updated
>   location onto the stack.
>
>   *A bit offset of `N × byte_size` is equivalent to a byte offset of
>   `N`.*

In section 3.15 replace the operation descriptions as follows:

> 1. `DW_OP_le`, `DW_OP_ge`, `DW_OP_eq`, `DW_OP_lt`, `DW_OP_gt`, `DW_OP_ne`
>       - stack operands:
>         - Y [base type or generic type]
>         - X {same base type or generic type]
>       - stack output:
>         - X [generic type with a value of 0 or 1]
>
>   The six relational operators each:
>   - pop the two operands, which have the same type, either the same
>     base type or the generic type,
>   - compare the operands:
>     Y < relational operator > X
>   - push the constant value 1 onto the stack if the result of the
>     operation is true or the constant value 0 if the result of the
>     operation is false. The pushed value has the generic type.  If
>     the operands have the generic type, the comparisons are
>     performed as signed operations.

**NOTE FOR DISCUSSION** Does this work for only numeric types or does
  it work for all base types?

> 2. `DW_OP_skip`
>    - inline parameters:
>      - bytes to skip [2-byte signed integer]
>
>   `DW_OP_skip` is an unconditional branch. Its single operand is a
>   2-byte signed integer constant. The 2-byte constant is the number
>   of bytes of the DWARF expression to skip forward or backward from
>   the current operation, beginning after the 2-byte constant.
>
> 3. `DW_OP_bra`
>       - inline parameters:
>         - bytes to skip [2-byte signed integer]
>       - stack operands:
>         - X
>
>   `DW_OP_bra` is a conditional branch. This operation pops the
>   operand; if the value popped is not 0, the number of bytes
>   specified in the inline parameter is skipped either forward or
>   backward. The counting of the bytes begins after the 2-byte
>   constant.

> 4. `DW_OP_call2`, `DW_OP_call4`
>       - inline parameters:
>         - DIE offset [2 or 4 byte unsigned integral]
>
>   `DW_OP_call2`and `DW_OP_call4` perform DWARF procedure calls
>   during evaluation of a DWARF expression or location
>   description. For `DW_OP_call2` and `DW_OP_call4` the parameter is
>   the 2- or 4-byte unsigned offset, respectively, of a debugging
>   information entry in the current compilation unit.

>
>   *Parameter interpretation of `DW_OP_call2` and `DW_OP_call4` is
>   exactly like that for `DW_FORM_ref2` and `DW_FORM_ref4`
>   respectively (see Section 7.5.4 on page 222).*
>
>   These operations transfer control of DWARF expression evaluation
>   to the `DW_AT_location` attribute of the referenced debugging
>   information entry. If there is no such attribute, then there is no
>   effect. Execution of the DWARF expression of a `DW_AT_location`
>   attribute may pop elements from the stack and/or push values or
>   locations onto the stack. Execution returns to the point following
>   the call when the end of the attribute is reached. Values and
>   locations on the stack at the time of the call may be used as
>   parameters by the called expression, and elements (values or
>   locations) left on the stack by the called expression may be used
>   as return values by prior agreement between the calling and called
>   expressions.

**NOTE FOR DISCUSSION** While it makes sense within the context to
  call things in the DWARF byte stream inline parameters and the
  things on the stack "operands", there are many other cases within
  the DWARF spec where the values within the DWARF byte stream are
  refered to as "operands". One example is the DW_FORM_ref*

>
> 5. `DW_OP_call_ref`
>       - inline parameters:
>         - .debug_info offset [4 or 8-byte unsigned integral]
>
>   The `DW_OP_call_ref` operator has a single parameter. In the
>   32-bit DWARF format, the parameter is a 4-byte unsigned value; in
>   the 64-bit DWARF format, it is an 8-byte unsigned value (see
>   Section 7.4 on page 210). The parameter is used as the offset of a
>   debugging information entry in the .debug_info section of the
>   current executable or shared object file.
>
>   *Operand interpretation of `DW_OP_call_ref` is exactly like that
>   for `DW_FORM_ref_addr` (see Section 7.5.4 on page 222).*
>
>   This operation transfers control of DWARF expression evaluation
>   to the `DW_AT_location` attribute of the referenced debugging
>   information entry. If there is no such attribute, then there is no
>   effect. Execution of the DWARF expression of a `DW_AT_location`
>   attribute may pop elements from the stack and/or push values or
>   locations onto the stack. Execution returns to the point following
>   the call when the end of the attribute is reached. Values and
>   locations on the stack at the time of the call may be used as
>   parameters by the called expression, and elements (values or
>   locations) left on the stack by the called expression may be used
>   as return values by prior agreement between the calling and called
>   expressions.

In section 3.16 replace the operation descriptions as follows:

> 1. `DW_OP_convert`
>       - inline parameters:
>         - type DIE [ULEB128] or 0 for generic type
>       - stack operands:
>         - X
>       - stack output:
>         - X [*specified type*]
>
>   The `DW_OP_convert` operation pops the operand, converts it to a
>   different type, then pushes the result. It takes one parameter,
>   which is an unsigned LEB128 integer that represents the offset of
>   a debugging information entry in the current compilation unit, or
>   value 0 which represents the generic type. If the parameter is
>   non-zero, the referenced entry must be a `DW_TAG_base_type` entry
>   that provides the type to which the value is converted.
>
> 2. `DW_OP_reinterpret`
>       - inline parameters:
>         - type DIE [ULEB128] or 0 for generic type
>       - stack operands:
>         - X
>       - stack output:
>         - X [*specified type*]
>
>   The `DW_OP_reinterpret` operation pops the operand,
>   reinterprets the bits in its value as a value of a different type,
>   then pushes the result. It takes one parameter, which is an unsigned
>   LEB128 integer that represents the offset of a debugging
>   information entry in the current compilation unit, or value 0
>   which represents the generic type. If the operand is non-zero, the
>   referenced entry must be a `DW_TAG_base_type` entry that provides
>   the type to which the value is reinterpreted. The type of the
>   parameter and result type must have the same size in bits.

In section 3.17 replace the operation descriptions as follows:

> 1. `DW_OP_nop`
>
>   The `DW_OP_nop` operation is a place holder. It has no effect on
>   the location stack or any of its values.
>
> 2. `DW_OP_entry_value`
>       - inline parameters:
>         - length [ULEB128]
>         - [DWARF expression]
>       - stack output:
>         - [generic type]
>
>   The `DW_OP_entry_value` operation pushes the value that an
>   expression would have had, or a register location would have held,
>   upon entering the current subprogram. It has two parameters: an
>   unsigned LEB128 length, followed by a block containing a DWARF
>   expression or a register location description (see Section
>   2.6.1.1.3 on page 44). The length parameter specifies the length in
>   bytes of the block. If the block contains a DWARF expression, the
>   DWARF expression is evaluated as if it had been evaluated upon
>   entering the current subprogram. The DWARF expression assumes no
>   values are present on the DWARF stack initially and results in
>   exactly one value being pushed on the DWARF stack when
>   completed. If the block contains a register location description,
>   DW_OP_entry_value pushes the value that register held upon
>   entering the current subprogram.
>
>   `DW_OP_push_object_address` is not meaningful inside of this DWARF
>   operation.
>
>   *The register location description provides a more compact form
>   for the case where the value was in a register on entry to the
>   subprogram. The values needed to evaluate `DW_OP_entry_value`
>   could be obtained in several ways. The consumer could suspend
>   execution on entry to the subprogram, record values needed by
>   `DW_OP_entry_value` expressions within the subprogram, and then
>   continue; when evaluating `DW_OP_entry_value`, the consumer would
>   use these recorded values rather than the current values. Or, when
>   evaluating `DW_OP_entry_value`, the consumer could virtually
>   unwind using the Call Frame Information (see Section 6.4 on page
>   185) to recover register values that might have been clobbered
>   since the subprogram entry point.*
>

**NOTE FOR DISCUSSION** What type does it push on the stack. It isn't
  clear to me from the operation description.

> 3. `DW_OP_extended`
>       - inline parameters:
>         - extended opcode [ULEB128]
>
>   The `DW_OP_extended` opcode encodes an extension operation. It has
>   at least one parameter: a ULEB128 constant identifying the extension
>   operation. The remaining parameters are defined by the extension
>   opcode, which are named using a prefix of . The extension opcode 0
>   is reserved.

> 4. `DW_OP_user_extended`
>       - inline parameters:
>         - extended opcode [ULEB128]
>
>   The `DW_OP_user_extended` opcode encodes a producer extension
>   operation.  It has at least one parameter: a ULEB128 constant
>   identifying a producer extension operation. The remaining
>   parameter are defined by the producer extension. The producer
>   extension opcode 0 is reserved and cannot be used by any producer
>   extension.
>
>   *The DW_OP_user_extended encoding space can be understood to
>   supplement the space defined by DW_OP_lo_user and DW_OP_hi_user
>   that is allocated by the standard for the same purpose.*
