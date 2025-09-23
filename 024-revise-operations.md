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

DWARF operations potentially have three sources of input: operands
which are encoded in their byte stream, values on the stack, and the
context in which they are executed. They can also modify the stack and
leave items on the stack. For every operand each of these sources of
input, output, and side effects are specified in a standardized format
similar to how other stack machine operations have been documented in
the past.

Proposed Changes
----------------

In Chapter 3 replace the following paragraph:

> A DWARF expression is encoded as a stream of operations, each
> consisting of an opcode followed by zero or more literal operands. The
> number of operands is implied by the opcode.

With:

> A DWARF expression is encoded as a stream of operations, each
> operation consisting of an opcode followed by zero or more literal
> operands. The number of operands is implied by the opcode. It may
> also receive parameters from the stack and make use of information
> from its evaluation context.
>
> *The structure description of the inputs and each of each operation
> will be as follows:*
> 1. `DW_OP_name`
>      - operands:
>        - first_operand [*encoding type*]
>        - second_operand [*encoding type*]
>      - stack parameters:
>        - T [*expected types*]: <*optional*> parameter1_name   # 4th entry on the stack
>        - Z [*expected types*]: <*optional*> parameter2_name   # 3rd entry on the stack
>        - Y [*expected types*]: <*optional*> parameter3_name  # 2nd entry on stack
>        - X [*expected types*]: <*optional*> parameter4_name   # top of stack
>      - stack output:
>        - X [*resulting type*]
>
>
> *The top four elements of the stack are given the names X, Y, Z, T as
> a matter of descriptive typographical convenience. In these
> descriptions the stack is presented as growing down with the deepest
> entry presented first. Unless otherwise stated, it is assumed that
> all stack parameters are consumed by the operation and all stack
> output is pushed onto the stack.*

In Section 3.2 replace the operation descriptions as follows:

> 1. `DW_OP_dup`
>       - stack parameters:
>         - X [*any*]
>       - stack output:
>         - Y: X
>         - X: X
>
>    The DW_OP_dup operation duplicates the entry at the top of the stack.
>
> 2. `DW_OP_drop`
>       - stack parameters:
>         - Y [*any*]
>         - X [*any*]
>       - stack output:
>         - X: Y
>
>   The DW_OP_drop operation pops the entry at the top of the stack.
>
> 3. `DW_OP_pick`
>       - operand:
>         - stack index [unsigned 1-byte integral in the range 0-255 inclusive]
>       - stack output:
>         - X: stack entry @ stack index
>
>    The single operand of the DW_OP_pick operation provides a 1-byte
>    index. A copy of the stack entry with the specified index (0
>    through 255, inclusive) is pushed onto the stack.
>

**NOTE FOR DISCUSSION**: It should be noted that there is no pick
operation which can pick based upon an index which is passed on the
stack instead of as a literal operand. The fact that it hasn't been
requested may suggest that it is not needed.

> 4. `DW_OP_over`
>       - stack parameters:
>         - Y [*any*]
>         - X [*any*]
>       - stack output:
>         - Z: Y
>         - Y: X
>         - X: Y
>
>    The `DW_OP_over` operation duplicates the entry currently second in
>    the stack at the top of the stack.
>
> *This is equivalent to a `DW_OP_pick` operation, with index 1.*
>
> 5. `DW_OP_swap`
>       - stack parameters:
>         - Y [*any*]
>         - X [*any*]
>       - stack output:
>         - Y: X
>         - X: Y
>
>    The `DW_OP_swap` operation swaps the top two stack entries. The entry
>    at the top of the stack becomes the
>    second stack entry, and the second entry (including its type
>    identifier) becomes the top of the stack.
>
> 6. `DW_OP_rot`
>       - stack parameters:
>         - Z [*any*]
>         - Y [*any*]
>         - X [*any*]
>       - stack output:
>         - Z: X
>         - Y: Z
>         - X: Y

In section 3.3 replace the operation descriptions as follows:

> 1. `DW_OP_lit0`, ..., `DW_OP_lit31`
>       - stack output:
>         - X [unsigned integer in the range 0-31 inclusive]
>
>   The `DW_OP_lit<n>` operations encode the unsigned literal values from 0
>   through 31, inclusive.
>
> 2. `DW_OP_const1u`, `DW_OP_const2u`, `DW_OP_const4u`, `DW_OP_const8u`
>       - operand:
>         - constant [unsigned integer of the specified width]
>       - stack output:
>         - X [unsigned integer]: constant
>
>    The single operand of a `DW_OP_const<n>u` operation provides a 1, 2, 4, or
>    8-byte unsigned integer constant, respectively.
>
> 3. `DW_OP_const1s`, `DW_OP_const2s`, `DW_OP_const4s`, `DW_OP_const8s`
>       - operand:
>         - constant [signed integer of the specified width]
>       - stack output:
>         - X [signed integer]: constant
>
>   The single operand of a `DW_OP_const<n>s` operation provides a 1, 2, 4, or
>   8-byte signed integer constant, respectively.
>
> 4. `DW_OP_constu`
>       - operand:
>         - constant [LEB128 encoded unsigned integer]
>       - stack output:
>         - X [unsigned integer]: constant
>
>   The single operand of the `DW_OP_constu` operation provides an unsigned
>   LEB128 integer constant.
>
> 5. `DW_OP_consts`
>       - operand:
>         - constant [LEB128 encoded signed integer]
>       - stack output:
>         - X [signed integer]: constant
>
>   The single operand of the `DW_OP_consts` operation provides a signed
>   LEB128 integer constant.
>
> 6. `DW_OP_constx`
>       - operand:
>         - debug_addr offset [LEB128 encoded unsigned integer]
>       - stack output:
>         - X [unsigned integer]: debug_addr offset
>
>   The `DW_OP_constx` operation has a single operand that encodes an
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
>       - operands:
>         - type DIE offset [LEB128 encoded offset]
>         - size [1-byte unsigned integer]
>         - constant
>       - stack output:
>         - X [*specified type*]: constant
>
>   The `DW_OP_const_type` operation takes three operands. The first
>   operand is an unsigned LEB128 integer that represents the offset of
>   a debugging information entry in the current compilation unit, which
>   must be a `DW_TAG_base_type` entry that provides the type of the
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

In section 3.4 replace the operation descriptions as follows:

> 1. `DW_OP_regval_type`
>      - operands:
>        - register number [LEB128 encoded register number]
>        - offset of a DIE in the current CU [LEB128 offset]
>      - stack output:
>        - contents of a given register interpreted as a value of a given type
>
>   The `DW_OP_regval_type` operation pushes the contents of a given
>   register interpreted as a value of a given type. The first operand
>   is an unsigned LEB128 number, which identifies a register whose
>   contents is to be pushed onto the stack. The second operand is an
>   unsigned LEB128 number that represents the offset of a debugging
>   information entry in the current compilation unit, which must be a
>   DW_TAG_base_type entry that provides the type of the value
>   contained in the specified register.

In section 3.5 replace the operation descriptions as follows:

> 1. `DW_OP_abs`
>    - stack operands
>      - X [numeric type]
>    - stack output:
>      - X [numeric type]
>
>   The `DW_OP_abs` operation pops the top stack entry, interprets it
>   as a signed value and pushes its absolute value. If the absolute
>   value cannot be represented, the result is undefined.
>
> 2. `DW_OP_and`
>    - stack operands
>      - X [integral base type or generic type]
>      - Y [integral base type or generic type]
>    - stack output:
>      - X [integral base type or generic type]
>
>   The `DW_OP_and` operation pops the top two stack values, performs
>   a bitwise and operation on the two, and pushes the result.
>
> 3. `DW_OP_div`
>    - stack operands
>      - X [numeric type]
>      - Y [numeric type]
>    - stack output:
>      - X [numeric type]
>
>   The `DW_OP_div` operation pops the top two stack values, divides
>   the former second entry by the former top of the stack using
>   signed division, and pushes the result.
>
> 4. `DW_OP_minus`
>    - stack operands
>      - X [numeric type]
>      - Y [numeric type]
>    - stack output:
>      - X [numeric type]
>
>   The `DW_OP_minus` operation pops the top two stack values,
>   subtracts the former top of the stack from the former second
>   entry, and pushes the result.
>
> 5. `DW_OP_mod`
>    - stack operands
>      - X [integral base type or generic type]
>      - Y [integral base type or generic type]
>    - stack output:
>      - X [integral base type or generic type]
>
>   The `DW_OP_mod` operation pops the top two stack values and pushes
>   the result of the calculation: former second stack entry modulo
>   the former top of the stack.

**NOTE FOR DISCUSSION** Why doesn't this take any numeric type the way
that DW_OP_div does? Ref: 2.5.2.4 Line 24-27 pg. 37 in DWARF6 draft. I
believe that mod should be added to that list.

> 6. `DW_OP_mul`
>    - stack operands
>      - X [numeric type]
>      - Y [numeric type]
>    - stack output:
>      - X [numeric type]
>
>   The `DW_OP_mul` operation pops the top two stack entries,
>   multiplies them together, and pushes the result.
>
> 7. `DW_OP_neg`
>
>   The `DW_OP_neg` operation pops the top stack entry, interprets it
>   as a signed value and pushes its negation. If the negation cannot
>   be represented, the result is undefined.
>
> 8. `DW_OP_not`
>    - stack operands
>      - X [integral base type or generic type]
>    - stack output:
>      - X [integral base type or generic type]
>
>   The `DW_OP_not` operation pops the top stack entry, and pushes its
>   bitwise complement.
>
> 9. `DW_OP_or`
>    - stack operands
>      - X [integral base type or generic type]
>      - Y [integral base type or generic type]
>    - stack output:
>      - X [integral base type or generic type]
>
>   The `DW_OP_or` operation pops the top two stack entries, performs
>   a bitwise or operation on the two, and pushes the result.
>
> 10. `DW_OP_plus`
>    - stack operands
>      - X [numeric type]
>      - Y [numeric type]
>    - stack output:
>      - X [numeric type]
>
>   The `DW_OP_plus` operation pops the top two stack entries, adds
>   them together, and pushes the result.
>
> 11. DW_OP_plus_uconst
>    - stack operands
>      - X [ULEB128]
>      - Y [numeric type]
>    - stack output:
>      - X [numeric type]
>
>   The `DW_OP_plus_uconst` operation pops the top stack entry, adds
>   it to the unsigned LEB128 constant operand interpreted as the same
>   type as the operand popped from the top of the stack and pushes
>   the result.
>
>   *This operation is supplied specifically to be able to
>   encode more field offsets in two bytes than can be done with
>   `DW_OP_lit<n> DW_OP_plus.`*

**NOTE FOR DISCUSSION:** It looks to me like the comment should be
  non-normative text. I made it so. Also the justification for
  DW_OP_plus_uconst seems questionable to me. I do not see why you
  could not just use: `DW_OP_uconst <some number> DW_OP_plus`. Is this
  a historic limitation? Should this text be reconsidered?

> 12. `DW_OP_shl`
>    - stack operands
>      - X: bits to shift [integral base type or generic type]
>      - Y: value to be shifted [integral base type or generic type]
>    - stack output:
>      - X [integral base type or generic type]
>
>   The `DW_OP_shl` operation pops the top two stack entries, shifts
>   the former second entry left (filling with zero bits) by the
>   number of bits specified by the former top of the stack, and
>   pushes the result.
>
> 13. `DW_OP_shr`
>    - stack operands
>      - X: bits to shift [integral base type or generic type]
>      - Y: value to be shifted [integral base type or generic type]
>    - stack output:
>      - X [integral base type or generic type]
>
>   The `DW_OP_shr` operation pops the top two stack entries, shifts
>   the former second entry right logically (filling with zero bits)
>   by the number of bits specified by the former top of the stack,
>   and pushes the result.

> 14. `DW_OP_shra`
>    - stack operands
>      - X: bits to shift [integral base type or generic type]
>      - Y: value to be shifted [integral base type or generic type]
>    - stack output:
>      - X [integral base type or generic type]
>
>   The `DW_OP_shra` operation pops the top two stack entries, shifts
>   the former second entry right arithmetically (divide the magnitude
>   by 2, keep the same sign for the result) by the number of bits
>   specified by the former top of the stack, and pushes the result.

> 15. `DW_OP_xor`
>    - stack operands
>      - X [integral base type or generic type]
>      - Y [integral base type or generic type]
>    - stack output:
>      - X [integral base type or generic type]
>
>   The `DW_OP_xor` operation pops the top two stack entries, performs
>   a bitwise exclusive-or operation on the two, and pushes the
>   result.

In section 3.6 replace the operation descriptions as follows:

> 1. `DW_OP_push_object_location`
>    - stack output:
>      - object [location]
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
>     `DW_OP_push_object_address`.  The old name is still supported in
>     DWARF 6 for compatibility.*

**NOTE FOR DISCUSSION**: is "supported' the right word in this context.

> 2. `DW_OP_form_tls_location`
>    - stack output:
>      - thread [integral type]
>    - stack output:
>      - location for TLS for thread [location]
>
>     The `DW_OP_form_tls_location` operation pops a value from the
>     stack, which must have an integral type, translates this value
>     into a location in the thread-local storage for the current
>     thread (see Section 3.1), and pushes the location onto the
>     stack.  The meaning of the value on the top of the stack prior
>     to this operation is defined by the run-time environment. If the
>     run-time environment supports multiple thread-local storage
>     blocks for a single thread, then the block corresponding to the
>     executable or shared library containing this DWARF expression is
>     used.
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
>     `DW_OP_form_tls_address`.  The old name is still supported in
>     DWARF 6 for compatibility.*
