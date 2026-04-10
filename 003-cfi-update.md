# Improvements to CFI for GPUs


## BACKGROUND

### Change CFA to CFL

Debugging device code running on GPUs needs to be able to unwind the
stack like other code running on CPUs. However, GPUs have some
requirements that are different than CPUs. In particular, they have
multiple address spaces and and can spill registers to them. Some
architectures of GPUs can even spill registers to other registers
which are farther away in their memory hierarchy. The fact that GPUs
use various locations to store previous frames which cannot be
expressed as a simple address requires the generalization of Call
Frame Addresses or Cononical Frame Addresses CFA (it is used
inconsistently in the current standard) into Call Frame Locations,
CFL, in much the same way that DWARF5's `DW_OP_push_object_address`
needed to change to `DW_OP_push_object_location` in DWARF6.

While this terminology change has significant typographical impact on
the standard itself, the amount of change needed to support these
changes is fairly minimal. Great lengths were made to ensure backward
compatibility. This proposal has been implemented in the ROCm LLVM
compiler.

### Row Creation Instructions

There are a few places where complete generalization was intentionally
not done. For example in Section 7.4.2.1 Row Creation Instructions
still only reference values which are addresses rather than full
locations. To be able to support full locations within the rows, a
couple more row creation instructions would need to be
introduced. Currently on all GPUs studied while preparing this
proposal, program text for device code is executed from device
memory. While this would be a different address space from the
perspective of the system, no GPU studied needs to reference PC
addresses outside of this assumed address space. This property of
which device address space program text is assumed to be in, is a
characteristic of the GPU's target ABI.

This assumed address space for program text is facilitated by the way
that fat binaries are currently implemented. The host code's ELF file
has a section, which contains a complete ELF file for the target
GPU. Therefore, there is no confusion between the host's .debug_frames
section and the .debug_frames section for the target GPU in its
embedded ELF file. If there were an architecture where the host's ELF
file's .debug_frames section mixed addresses between host code and
device code, then additional Row Creation Instructions would be need
to be added to Section 7.4.2.1.

### CFL Definition Instructions

Unlike Row Creation Instructions which deal with program text where
the address space can be assumed, the Call Frame Location, CFL, can be
put anywhere in the GPU's storage and therefore the CFL Defintion
Instructions need a couple new rules to handle locations in other address
spaces. We added: `DW_CFA_def_aspace_cfa`, `DW_CFA_def_aspace_cfa_sf`

### Register rules

In DWARF5 before Locations were allowed on the stack, Call Frame
Information needed a way to reference locations other than the values
that could be put on the stack. It also needed to be determine if the
entry referred to a value which was a memory location or if that was
the value. This led to the diversity of Register Rules found in
section 7.4.2.3. Now that DWARF6 has locations on the stack, many of
these rules are no longer strictly necessary; they could be
implemented as DWARF expressions. Replacing the register rules
exclusively with DWARF Exprressions was not seriously considered; that
would have broken backward compatibility and probably would not be
quite as compact. However, to make the association between register
rules and DWARF expressions more obvious, we included the analogus
DWARF expression with its register rule.

### New Operator:

**FIXME: Justification for new operator** **Ben: This supposed to
partially replace DW_OP_entry_value.  This is going to need a strong
justification why DW_OP_entry_value is not sufficient**

**jakub: the current design comes from the intent to make it useful
for 2 different purposes; one is the most common, gdb unwinds the
stack, looks at the call site info and if it finds usable values
there, propagates into DW_OP_entry_value in the callee the other is
with tools like systemtap in mind; if you know ahead you'll stop in
function xyz and will need to ask for value of some variable, if it
uses DW_OP_entry_value, you can simply add another breakpoint at the
start of the function and remember values that will be needed later,
and then just use them when you hit the final breakpoint**

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

In Section 3.1 DWARF Expression Evaluation Context

In bullet point 1 change:

>    1. Required result kind
>
>    The kind of result required – either a
>    location or a value – is determined by the DWARF construct where
>    the expression is found. For example, DWARF attributes with class
>    valexpr require a value, and attributes with class locexpr
>    require a location (see Section 8.5.5 on page 235).

to

>    1. Required result kind
>
>    The kind of result required – either a
>    location or a value – is determined by the DWARF construct where
>    the expression is found.
>
>    *For example, A variable's DWARF attributes with class valexpr
>    require a value, and attributes with class locexpr require a
>    location (see Section 8.5.5 on page 235). Or when the expression
>    is used in a CFI (see Section 7.4.2 on page 198), then the kind
>    is determined by the register rule that points to the DWARF
>    expression (see Section 7.4.1 on page 195). If the register rule
>    is expression(E) then kind is a location, while if the rule is
>    val_expression(E) then the kind is a value.*

In bullet point 3 Current Compilation Unit add the following paragraph
between the normative text and the non-normative text.

>    DWARF exppressions used in CFI (see Section 7.4.2 on page 198) do
>    not have a Compilation Unit in their context.

In bullet point 6 Current call frame change "call frame address (CFA)"
to "call frame location (CFL)"

In bullet point 9 Current Object add the following paragraph.

>    DWARF exppressions used in CFI (see Section 7.4.2 on page 198) do
>    not have a Current Object in their context.

Add a 10th bullet point:

>    10. Current register
>
>    When a DWARF expression is evaluated in the context of a CFI
>    entry (see Section 7.4.2 on page 198) the current register is the
>    register whose contents are being recovered.
>
>    DWARF exppressions used for variable's location or value do not
>    have a curret register in their context.

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
>     If there is no call frame information defined, then the default
>     rules for the target architecture are used. If the register rule
>     is <i>undefined</i>, then the undefined location description is
>     pushed.  If the register rule is <i>same value</i>, then a
>     register location description for R is pushed.

### FIXME: [[ Reword definition of CFA in 7.4 Call Frame Information ]]

In Section 7.4 Call Frame Information

Replace the 2nd bullet point:

>    An area of memory that is allocated on a stack called a “call
>    frame.” The call frame is identified by an address on the
>    stack. We refer to this address as the Canonical Frame Address or
>    CFA. Typically, the CFA is defined to be the value of the stack
>    pointer at the call site in the previous frame (which may be
>    different from its value on entry to the current frame).

With:

>    An area of memory that is allocated on a stack called a “call
>    frame.” The call frame is identified by an location on the
>    stack. We refer to this location as the Canonical Frame Location or
>    CFL. Typically, the CFL is defined to be the value of the stack
>    pointer at the call site in the previous frame (which may be
>    different from its value on entry to the current frame).
>
>    *Previously, the Call Frame Location, CFL, was known as the Call
>    Frame Address, CFA. However, GPUs do not necessarily store
>    previous frames on the stack, they may pushed to some local
>    storage in the GPU. So it is not quite to appropriate to assume
>    that it is an address anymore.*

In the subsequent non-normative paragraphs and bullet points replace
"CFA" with "CFL".

In Section 7.4.1 Structure of Call Frame Information:

Replace all uses of CFA with CFL including in the abstract very large
table where it is the second column. Also replace "Call Frame Address"
with "Call Frame Location" in the paragraph that describes what the
CFL is.

> The previous value of this register is the undefined location
> description (see 3.9 Undefined LocationS).

Remove the parenthetical comment and add the following non-normative
text:

> *(By convention, the register is not preserved by a callee.)*

For the "same value" register rule change "frame" to "caller frame",
and add the following paragraphs after the first sentence:

**Ben: I've read the following multiple times and I'm not sure what it
means or does. I can't really see what this adds. It seems to try to
resolve some ambiguity but I don't understand where this comes up**

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
> description of the current CFL offset by N bytes.

For the "val_offset(N)" register rule, replace the description with
the following:

> N is a signed byte offset. The previous value of this register is
> saved at the memory address pointed to by the location description
> L, where L is the location description of the current CFL offset by
> N bytes.

**Ben: Is this right? L can be any kind of location. L+offset N must
be a memory location. That address is then dereferenced and theat is
the previous value of the register. It sounds like you are trying to
avoid the situation where the size of the register is bigger than a
generic. I can see this being a problem when trying to restore a
vector register. Since the context in which this is being evaluated
the size register is known this should not be a problem.**

> The DWARF is ill-formed if the CFL is not a memory byte address
> location description,
>
> *The value of this register need not be representable as a generic
> type. For example vector registers are often larger than the generic
> type. A consumer can reference the size of the register from the
> DWARF expression evaluation context (see Section 3.1 page XX) and
> the entire bit contents of the register can be read from the
> specified location.*
>

**Ben: I'm not sure about the following**

> *Since the CFL location description is required to be a memory byte
> address location description, the value of val_offset(N) will also
> be a memory byte address location description since it is offsetting
> the CFL location description by N bytes. Furthermore, the value of
> val_offset(N) will be a memory byte address in the target
> architecture default address space.*

For the "register(R)" register rule, replace the description with the
following:

> This register has been stored in another register numbered R.
>
> The DWARF is ill-formed if the size of this register is larger
> the size of register R or if there is a cyclic dependency in the
> call frame information.

For the "expression(E)" register rule, replace the description with the
following:

> The previous value of this register is located at the location
> description produced by evaluating the DWARF operation expression E
> (see Chapter 3 DWARF Expressions).
>
> E is evaluated with the current context including the register and
> the target architecture but without the compilation unit and the
> current object which are unspecified. The resulting kind is a
> location descriptionm.


> and an initial stack comprising the location
> description of the current CFL (see Chapter 3 DWARF Expressions).

For the "val_expression(E)" register rule, replace the description with the
following:

> The previous value of this register is located either in a generic
> value or in implicit storage produced by evaluating the DWARF
> operation expression E (see Chapter 3 DWARF Expressions).
>
> E is evaluated with the current context including the register and
> the target architecture but without the compilation unit and the
> current object which are unspecified. The resulting kind is a
> value.
>
> The DWARF is ill-formed if the resulting value is in implicit
> storage and the resulting implicit storage is smaller than the
> register size.
>
> *The value of this register need not be representable as a generic
> type. For example vector registers are often larger than the generic
> type. The implicit storage containing the value will be the bit
> contents of the register.**

**Ben: I don't think we should include this. See above**
> an initial stack comprising the location
> description of the current CFL (see Chapter 3 DWARF Expressions).
>

**Ben: I tried to solve this:
[For further discussion...] This has limited usefulness as the DWARF
expression E can only produce values up to the size of the generic
type. This is due to not allowing any operations that specify a type
in a CFI operation> expression. This makes it unusable for registers
that are larger than the generic type. However, expression(E) can be
used to create an implicit location description of any size.**

Change the paragraph that begins:

> A Common Information Entry holds information that is shared among
> many Frame Description Entries.

to:

> A Common Information Entry (CIE) holds information that is shared
> among many Frame Description Entries (FDE).

In the subsection on CIE, in item 1 (length), change "...of the address
size" to "...of the address size specified in the address_size field."

In item 2 (CIE_id), add the following:

> In the 32-bit DWARF format, the value of the CIE id in the CIE
> header is 0xffffffff; in the 64-bit DWARF format, the value is
> 0xffffffffffffffff.

In item 3 (version), add the following:

> The value of the CIE version number is 4.

**[For further discussion...] Should this be increased to 5?
Ben: What has changed?**

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

In Section 7.4.2 Call Frame Instructions, Change the following
sentence to "The DWARF operations that can be used in E have the
following restrictions:".

In the first bullet item, add the following opcodes to the list:
`DW_OP_fbreg`, `DW_OP_implicit_pointer`,
`DW_OP_aspace_deref_type`. Change "operators are not allowed in an
operand of these instructions" to "operations are not allowed".

**Ben: assuming the operator name changes from the address space proposal.**

Change the second bullet to:

> `DW_OP_push_object_location` is not allowed because there is no object
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

In Section 7.4.2.1 Row Creation Instructions add headers like were
done in operation headers:


> 1.  `DW_CFA_set_loc (generic address)`
> 2.  `DW_CFA_advance_loc (opcode_encoded delta)`
> 3.  `DW_CFA_advance_loc1 ( ubyte delta)`
> 4.  `DW_CFA_advance_loc2 ( uhalf delta)`
> 5.  `DW_CFA_advance_loc4 ( uword delta)`

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

In Section 7.4.2.3 "Register Rule Instructions
to:

> 1.  `DW_CFA_undefined ( ULEB R)`
>
>     The DW_CFA_undefined instruction takes a single ULEB operand
>     that represents a register number <span
>     style="background-color:green">specified by R</span>. The
>     required action is to set the rule for the specified register to
>     “undefined.”
>
> 2.  `DW_CFA_same_value( ULEB R)`
>
>     The DW_CFA_same_value instruction takes a single ULEB operand
>     that represents a register number<span
>     style="background-color:green"> specified by R</span>. The
>     required action is to set the rule for the specified register to
>     “same value.”
>
> 3. `DW_CFA_offset(opcode_encoded R, ULEB B)`
>
>     The `DW_CFA_offset` instruction takes two operands: a register
>     number <span style="background-color: green;">R </span>(encoded
>     with the opcode) and an ULEB constant representing a factored
>     <span style="background-color: green;">displacement
>     B</span>. The required action is to change the rule for the
>     register <span style="background-color: green;">specified by R
>     to be an offset(B * data_alignment_factor)</span> rule.
>
> 4.  `DW_CFA_offset_extended(ULEB R, ULEB B)`
>
>     The `DW_CFA_offset_extended` instruction takes two ULEB operands
>     representing a register number <span style="background-color:
>     green;">R </span>and a factored <span style="background-color:
>     green;">displacement B</span>.
>
>     *This instruction is identical to DW_CFA_offset<span
>     style="background-color: green;">,</span> except for the
>     encoding and size of the register operand.*
>
> 5.  `DW_CFA_offset_extended_sf(ULEB R, signed_LEB128 B)`
>
>     The `DW_CFA_offset_extended_sf` instruction takes two operands: an
>     ULEB value representing a register number <span
>     style="background-color: green;">R</span> and a <span
>     style="background-color: green;">LEB</span> factored
>     <span style="background-color: green;">displacement
>     B</span>.
>
>     *<span style="background-color: green;">This instruction is
>     identical to DW_CFA_offset_extended, except that B is
>     signed.</span>*
>
> 6.  `DW_CFA_val_offset(ULEB R, ULEB B)`
>
>     The `DW_CFA_val_offset` instruction takes two ULEB operands
>     representing a register number <span style="background-color:
>     green;">R </span>and a factored <span style="background-color:
>     green;">displacement B</span>. The required action is to change
>     the rule for the register indicated by <span
>     style="background-color: green;">R</span> to be a
>     val_offset(<span style="background-color: green;">B</span> *
>     data_alignment_factor) rule.
>
> 7. `DW_CFA_val_offset_sf(ULEB R, LEB128 B)`
>
>     The `DW_CFA_val_offset_sf` instruction takes two operands: an
>     <span style="background-color: green;"> ULEB</span> value
>     representing a register number <span style="background-color:
>     green;">R</span> and a <span style="background-color:
>     green;">signed LEB128</span> factored <span
>     style="background-color: green;">displacement B</span>. <span
>     style="background-color: green;">
>
>     *This instruction is identical to DW_CFA_val_offset, except that
>     B is signed.</span>*

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
