# Improvements to CFI for GPUs


## BACKGROUND

**Fixme: a much stronger justification is needed**

**Ben: I don't understand why we need DW_OP_call_frame_entry_reg**

**Ben: a considerable portion of the changes are the formal definition
kind of changes which we didn't universally make in the proposals that
were ultimately passed. The size of this proposal can be substantially
reduced if we limit those. It seems like they are only important in a
few cases**

With the addition of address spaces in General Support for Address
Spaces https://dwarfstd.org/issues/260211.1.html the CFA Definition
Instructions need to be properly defined to specify which address
space they refer to. There alaso needs to be CFA Definition
Instructions which can reference the address space.

GPUs can spill registers to local GPU memory and so CFI needs to support this.

Almost all uses of addresses in DWARF 5 are limited to defining
location descriptions, or to be dereferenced to read memory. The
exception is `DW_CFA_val_offset` which uses the address to set the
value of a register. In order to support address spaces, the CFA DWARF
expression is defined to be a memory location description. This allows
it to specify an address space which is used to convert the offset
address back to an address in that address space.

As described in [DWARF Operations to Create Vector Composite Location
Descriptions], a DWARF expression involving the set of SIMT lanes
active on entry to a subprogram is required. The SIMT active lane mask
may be held in a register that is modified as the subprogram
executes. However, when a function returns the active lane mask
register must be restored to the value it had on entry.

The Call Frame Information (CFI) already encodes such register saving,
so it is more efficient to provide an operation to return the location
of a saved register than have to generate a loclist to describe the
same information. This is now possible since [Allow location
description on the DWARF evaluation stack] allows location
descriptions on the stack.

## PROPOSED CHANGES

In Section 3.6 Conxtext Query Perations add the following operation
after `DW_OP_push_lane`:

> 5.  `DW_OP_call_frame_entry_reg`([ULEB] R)
>
>     → <[location] register's location>
>
>     `DW_OP_call_frame_entry_reg` has a single ULEB128 integer
>     operand that represents a target architecture register number R.
>
>     It pushes a location description that holds the value of
>     register R on entry to the current subprogram as defined by the
>     call frame information (see Section 7.4 "Call Frame
>     Information").
>

**Ben: Is this supposed to partially replace DW_OP_entry_value?**

**Ben: The following was copied over as non-normative but it sounds to
me like text that should be in normative**

>     *If there is no call frame information defined, then the default
>     rules for the target architecture are used. If the register rule
>     is <i>undefined</i>, then the undefined location description is
>     pushed.  If the register rule is <i>same value</i>, then a
>     register location description for R is pushed.*

### FIXME: [[ Reword definition of CFA in 7.4 Call Frame Information ]]

In Section 7.4.1 Structure of Call Frame Information:

**Ben for discussion: Could we simplify these rules down to:
undefined, same value, expression(E), val_expression(E),
architectural. Register, offset and val_offset are all locations.**

In the "undefined" register rule, add the following after the first
sentence:

> The previous value of this register is the undefined location
> description (see 3.9 Undefined LocationS).

Remove the parenthetical comment and add the following non-normative
text:

> *(By convention, the register is not preserved by a callee.)*

For the "same value" register rule change "frame" to "caller frame",
and add the following paragraphs after the first sentence:

**Ben: I've read the following multiple times and I'm not sure what it
means or does.**

> If the current frame is the top frame, then the previous value of
> this register is the location description L that specifies one
> register location description SL. SL specifies the register location
> storage that corresponds to the register with a bit offset of 0 for
> the current thread.
>
> If the current frame is not the top frame, then the previous value
> of this register is the location description obtained using the call
> frame information for the callee frame and callee program location
> invoked by the current caller frame for the same register.

Remove the parenthetical comment and add the following non-normative
text:

> *(By convention, it is preserved by the callee, but the callee has
> not modified the register.)*

For the "offset(N)" register rule, replace the description with the
following:

> N is a signed byte offset. The previous value of this register is
> saved at the location description L, where L is the location
> description of the current CFA (see Chapter 3 DWARF Operation
> Expressions) updated with the bit offset N scaled by 8 (the byte
> size).

For the "val_offset(N)" register rule, replace the description with
the following:

> N is a signed byte offset. The previous value of this register is
> the memory byte address of the location description L, where L is
> the location description of the current CFA (Chapter 3 DWARF
> Expressions) updated with the bit offset N scaled by 8 (the byte
> size).
>
> The DWARF is ill-formed if the CFA location description is not a
> memory byte address location description, or if the register size
> does not match the size of an address in the target architecture
> default address space.
>
> *Since the CFA location description is required to be a memory byte
> address location description, the value of val_offset(N) will also
> be a memory byte address location description since it is offsetting
> the CFA location description by N bytes. Furthermore, the value of
> val_offset(N) will be a memory byte address in the target
> architecture default address space.*

**For further discussion...] Should DWARF allow the address size to be
a different size to the size of the register? Requiring them to be the
same bit size avoids any issue of conversion as the bit contents of
the register is simply interpreted as a value of the address.**

**GDB has a per register hook that allows a target specific conversion
on a register by register basis. It defaults to truncation of bigger
registers, and to actually reading bytes from the next register (or
reads out of bounds for the last register) for smaller
registers. There are no GDB tests that read a register out of bounds
(except an illegal hand written assembly test).**

**Ben: can't we just use DW_OP_regval_type for that?**

For the "register(R)" register rule, replace the description with the
following:

> This register has been stored in another register numbered R.
>
> The previous value of this register is the location description
> obtained using the call frame information for the current frame and
> current program location for register R.
>
> The DWARF is ill-formed if the size of this register does not match
> the size of register R or if there is a cyclic dependency in the
> call frame information.

**[For further discussion...] Should this also allow R to be larger than
this register? If so is the value stored in the low order bits and it
is undefined what is stored in the extra upper bits?**

For the "expression(E)" register rule, replace the description with the
following:

> The previous value of this register is located at the location
> description produced by evaluating the DWARF operation expression E
> (see Chapter 3 DWARF Expressions).
>
> E is evaluated with the current context, except the result kind is a
> location description, the compilation unit is unspecified, the
> object is unspecified, and an initial stack comprising the location
> description of the current CFA (see Chapter 3 DWARF Expressions).

For the "val_expression(E)" register rule, replace the description with the
following:

> The previous value of this register is located at the implicit location
> description created from the value produced by evaluating the DWARF
> operation expression E (see Chapter 3 DWARF Expressions).
>
> E is evaluated with the current context, except the result kind is a
> value, the compilation unit is unspecified, the object is
> unspecified, and an initial stack comprising the location
> description of the current CFA (see Chapter 3 DWARF Expressions).
>
> The DWARF is ill-formed if the resulting value type size does not
> match the register size.

**[For further discussion...] This has limited usefulness as the DWARF
expression E can only produce values up to the size of the generic
type. This is due to not allowing any operations that specify a type
in a CFI operation> expression. This makes it unusable for registers
that are larger than the generic type. However, expression(E) can be
used to create an implicit location description of any size.**

Change the paragraph that begins:

> A Common Information Entry holds information that is shared among many Frame
> Description Entries.

to:

> A Common Information Entry (CIE) holds information that is shared among many
> Frame Description Entries (FDE).

In the subsection on CIE, in item 1 (length), change "...of the address
size" to "...of the address size specified in the address_size field."

In item 2 (CIE_id), add the following:

> In the 32-bit DWARF format, the value of the CIE id in the CIE
> header is 0xffffffff; in the 64-bit DWARF format, the value is
> 0xffffffffffffffff.

In item 3 (version), add the following:

> The value of the CIE version number is 4.

**[For further discussion...] Should this be increased to 5?**

In item 4 (augmentation), change " target-specific information" to
"vendor and target architecture specific information".

In item 9 (return_address_register), change "function" to
"subprogram", and add the following:

> The value of the return address register is used to determine the
> program location of the caller frame. The program location of the
> top frame is the target architecture program counter value of the
> current thread.

In the subsection on FDE, in item 1 (length), change "function" to
"subprogram".

In Section 7.4.2 Call Frame Instructions, in the second paragraph,
change "...encoded as DWARF expressions" to "...encoded as DWARF
operation expressions E." Change the following sentence to "The DWARF
operations that can be used in E have the following restrictions:".

**Ben: Do we really want to do the first change.**

In the first bullet item, add the following opcodes to the list:
`DW_OP_fbreg`, `DW_OP_implicit_pointer`, `DW_OP_aspace_deref_type`. Change
"operators are not allowed in an operand of these instructions" to
"operations are not allowed".

**Ben: assuming the operator name changes from the address space proposal.**

Change the second bullet to:

> DW_OP_push_object_location is not allowed because there is no object
> context to provide a value to push.

In the third bullet, add `DW_OP_entry_value`. Change "is not
meaningful...because its use would be circular" to "are not allowed
because their use would be circular."

Add the following bullet to the end of the list:

> * `DW_OP_call_frame_entry_reg` is not allowed if evaluating E causes a
>   circular dependency between `DW_OP_call_frame_entry_reg` operations.
>
>   *For example, if a register R1 has a `DW_CFA_def_cfa_expression`
>   instruction that evaluates a `DW_OP_call_frame_entry_reg`
>   operation that specifies register R2, and register R2 has a
>   `DW_CFA_def_cfa_expression` instruction that that evaluates a
>   `DW_OP_call_frame_entry_reg` operation that specifies register R1.*

**Ben: Huh? The following look the same**

In the last paragraph, change "`DW_CFA_expression` and
`DW_CFA_val_expression`" to "`DW_CFA_expression`, and
`DW_CFA_val_expression`".

**Ben: add operands to 7.4.2.[1-5]**

In Section 7.4.2.2 "CFA Definition Instructions", replace the
description of the six CFI instructions with the following:

> 1.  `DW_CFA_def_cfa (ULEB register, ULEB byte displacement)`
>
>     The `DW_CFA_def_cfa` instruction takes two ULEB128 operands
>     representing a register number R and a (non-factored) byte
>     displacement B. AS is set to the target architecture default
>     address space identifier. The required action is to define the
>     current CFA rule to be equivalent to the result of evaluating
>     the DWARF expression `DW_OP_constu AS; DW_OP_aspace_bregx R, B`
>     as a location description.
>
> 2.  `DW_CFA_def_cfa_sf (ULEB register, LEB factored byte
>     displacement)`
>
>     The `DW_CFA_def_cfa_sf` instruction takes two operands: a
>     ULEB128 value representing a register number R and a LEB128
>     factored byte displacement B. AS is set to the target
>     architecture default address space identifier. The required
>     action is to define the current CFA rule to be equivalent to the
>     result of evaluating the DWARF expression `DW_OP_constu AS;
>     DW_OP_aspace_bregx R, B * data_alignment_factor` as a location
>     description.
>
>     [non-normative] The action is the same as `DW_CFA_def_cfa`,
>     except that the second operand is signed and factored.
>
> 3.  `DW_CFA_def_cfa_register (ULEB register)`
>
>     The `DW_CFA_def_cfa_register` instruction takes a single ULEB128
>     operand representing a register number R. The required action is
>     to define the current CFA rule to be equivalent to the result of
>     evaluating the DWARF expression `DW_OP_constu AS;
>     DW_OP_aspace_bregx R, B` as a location description. B and AS are
>     the old CFA byte displacement and address space respectively.
>
>     If the subprogram has no current CFA rule, or the rule was
>     defined by a `DW_CFA_def_cfa_expression` instruction, then the
>     DWARF is ill-formed.
>
> 4.  `DW_CFA_def_cfa_offset (ULEB byte displacement)`
>
>     The `DW_CFA_def_cfa_offset` instruction takes a single ULEB128
>     operand representing a (non-factored) byte displacement B. The
>     required action is to define the current CFA rule to be
>     equivalent to the result of evaluating the DWARF expression
>     `DW_OP_constu AS; DW_OP_aspace_bregx R, B` as a location
>     description. R and AS are the old CFA register number and
>     address space respectively.
>
>     If the subprogram has no current CFA rule, or the rule was
>     defined by a `DW_CFA_def_cfa_expression` instruction, then the
>     DWARF is ill-formed.
>
> 5.  `DW_CFA_def_cfa_offset_sf (LEB factored byte displacement)`
>
>     The `DW_CFA_def_cfa_offset_sf` instruction takes a LEB128
>     operand representing a factored byte displacement B. The
>     required action is to define the current CFA rule to be
>     equivalent to the result of evaluating the DWARF expression
>     `DW_OP_constu AS; DW_OP_aspace_bregx R, B *
>     data_alignment_factor` as a location description. R and AS are
>     the old CFA register number and address space respectively.
>
>     If the subprogram has no current CFA rule, or the rule was
>     defined by a `DW_CFA_def_cfa_expression` instruction, then the
>     DWARF is ill-formed.
>
>     [non-normative] The action is the same as
>     `DW_CFA_def_cfa_offset`, except that the operand is signed and
>     factored.
>

**FIXME? -- Ben: fix what?**

> 6.  `DW_CFA_def_cfa_expression (exploc expression)`
>
>     The `DW_CFA_def_cfa_expression` instruction takes a single
>     operand encoded as a DW_FORM_exprloc value representing a DWARF
>     expression E.  The required action is to define the current CFA
>     rule to be equivalent to the result of evaluating E with the
>     current context, except the result kind is a location
>     description, the compilation unit is unspecified, the object is
>     unspecified, and an empty initial stack.
>
>     See Section 7.4.2 Call Frame Instructions regarding restrictions on
>     the DWARF expressions that can be used in E.

Add the following CFI instructions:

> 7.  `DW_CFA_def_aspace_cfa (ULEB register, ULEB byte displacement,
>     ULEB address space)`
>
>     The `DW_CFA_def_aspace_cfa` instruction
>     takes three unsigned LEB128 operands representing a register
>     number R, a (non-factored) byte displacement B, and a target
>     architecture specific address space identifier AS. The required
>     action is to define the current CFA rule to be equivalent to the
>     result of evaluating the DWARF expression `DW_OP_constu AS;
>     DW_OP_aspace_bregx R, B` as a location description.
>
>     If AS is not one of the values defined by the target
>     architecture specific `DW_ASPACE_*` values then the DWARF
>     expression is ill-formed.
>
> 8.  `DW_CFA_def_aspace_cfa_sf (ULEB register, LEB factored byte
>     displacement, ULEB address space)`
>
>     The `DW_CFA_def_aspace_cfa_sf`
>     instruction takes three operands: an unsigned LEB128 value
>     representing a register number R, a signed LEB128 factored byte
>     displacement B, and an unsigned LEB128 value representing a
>     target architecture specific address space identifier AS. The
>     required action is to define the current CFA rule to be
>     equivalent to the result of evaluating the DWARF expression
>     `DW_OP_constu AS; DW_OP_aspace_bregx R, B *
>     data_alignment_factor` as a location description.
>
>     If AS is not one of the values defined by the target
>     architecture specific `DW_ASPACE_*` values, then the DWARF
>     expression is ill-formed.
>
>     [non-normative] The action is the same as
>     `DW_CFA_def_aspace_cfa`, except that the second operand is
>     signed and factored.

In Section 7.4.2.3 Register Rule Instructions, in item 1,
`DW_CFA_undefined`, change "a register number" to "a register number
R."  Change "the specified register" to "the register specified by R."

Make the same change in item 2, `DW_CFA_same_value`.

Replace the text of item 3, `DW_CFA_offset`, with the following:

> The `DW_CFA_offset` instruction takes two operands: a register number
> R (encoded with the opcode) and an unsigned LEB128 constant
> representing a factored displacement B. The required action is to
> change the rule for the register specified by R to be an offset(B *
> data_alignment_factor) rule.

In item 4, `DW_CFA_offset_extended`, change "a register number and a
factored offset" with "a register number R and a factored displacement
B", and add a comma before "except.

In item 5, `DW_CFA_offset_extended_sf`, change "a register number and
a signed LEB128 factored offset" with "a register number R and a
signed LEB128 factored displacement B." Change the final two sentences
to "This instruction is identical to `DW_CFA_offset_extended`, except
that B is signed."

In item 6, `DW_CFA_val_offset`, change "a register number and a
factored offset" to "a register number R and a factored displacement
B."  Change "indicated by the register number to be a val_offset(N)
rule where..." to "indicated by R to be a val_offset(B *
data_alignment_factor) rule."

Replace the text of item 7, `DW_CFA_val_offset_sf`, with the following:

> The `DW_CFA_val_offset_sf` instruction takes two operands: an
> unsigned LEB128 value representing a register number R and a signed
> LEB128 factored displacement B. This instruction is identical to
> `DW_CFA_val_offset`, except that B is signed.

Replace the text of item 8, `DW_CFA_register`, with the following:

> The `DW_CFA_register` instruction takes two unsigned LEB128 operands
> representing register numbers R1 and R2 respectively. The required
> action is to set the rule for the register specified by R1 to be a
> register(R2) rule.

In item 9, `DW_CFA_expression`, replace the description with the
following:

> The `DW_CFA_expression` instruction takes two operands: an unsigned
> LEB128 value representing a register number R, and a `DW_FORM_block`
> value representing a DWARF operation expression E. The required
> action is to change the rule for the register specified by R to be
> an expression(E) rule.
>
> *That is, E computes the location description where the register
> value can be retrieved.*
>
> *See 7.4.2 Call Frame Instructions regarding restrictions on the
> DWARF expression operations that can be used in E.*

In item 10, `DW_CFA_val_expression`, replace the description with the
following:

> The `DW_CFA_val_expression` instruction takes two operands: an
> unsigned LEB128 value representing a register number R, and a
> `DW_FORM_block` value representing a DWARF operation expression
> E. The required action is to change the rule for the register
> specified by R to be a val_expression(E) rule.
>
> *That is, E computes the value of register R.*
>
> *See 7.4.2 Call Frame Instructions regarding restrictions on the
> DWARF expression operations that can be used in E.*
>
> If the result of evaluating E is not a value with a base type size
> that matches the register size, then the DWARF is ill-formed.

In item 11, `DW_CFA_restore`, change "a register number" to "a
register number R." Change "the indicated register" to "the register
specified by R."

In item 12, `DW_CFA_restore_extended`, change "a register number" to
"a register number R", and add a comma before "except".

In Section 8.7.1 "Operation Expressions" of [Allow location
description on the DWARF evaluation stack], add the following row to
Table 8.9 "DWARF Operation Encodings":

    ---------------------------------------------------------------------------

    Table 8.9: DWARF Operation Encodings
    ================================== ===== ======== =========================
    Operation                          Code  Number   Notes
                                             of
                                             Operands
    ================================== ===== ======== =========================
    DW_OP_call_frame_entry_reg         TBA      1     ULEB128 register number
    ================================== ===== ======== =========================

    ---------------------------------------------------------------------------

In section 8.23 Call Frame Information in Table 8.29

In the 4th column add a third operand

    ---------------------------------------------------------------------------

    Table 8.23:Call frame instruction encodings
    ================================== ===== ======== =========================
    Instruction                        High   Low    Operand 1,
                                       2 bits 6 bits Operand 2,
                                                     Operand 3
    ================================== ====== ====== ==========================
    DW_CFA_def_aspace_cfa                0    TBA    ULEB register,
	                                                 ULEB offset,
                                                     ULEB address space
	DW_CFA_def_aspace_cfa_s              0    TBA    ULEB register,
	                                                 SLEB offset,
                                                     ULEB address space
    ================================== ====== ====== ==========================

    ---------------------------------------------------------------------------
