Expression Evaluation Context
=============================

Background
----------

This proposal is editorial in nature: it does not intend to change the
meaning of any DWARF constructs, but merely to clarify aspects of
DWARF expression evaluation that were unclear to the teams implementing
DWARF consumers.

The context within which expression evaluation takes place is not
spelled out in the DWARF document, but mostly left to the reader to
infer based on scattered references throughout the sections on DWARF
expressions. The changes proposed below explicitly define the context
within which expression evaluation takes place.


Proposed Changes
----------------

Insert the following subsection under 2.5 (renumbering subsequent
subsections):

> 2.5.1 DWARF Expression Evaluation Context
> 
> A DWARF expression is evaluated in a context that can include a
> number of context elements. If multiple context elements are
> specified, then they must be self consistent or the result of the
> evaluation is undefined. The context elements that can be specified
> are:
> 
> 1. A current result kind
> 
>     The kind of result required by the DWARF expression evaluation.
>     If specified it can be a location description or a value.
> 
> 2. A current thread
> 
>     The target architecture thread identifier of the source program
>     thread of execution for which a user presented expression is
>     currently being evaluated.
> 
>     It is required for operations that are related to target
>     architecture threads.
> 
>     [non-normative] For example, the `DW_OP_form_tls_address` operation.
> 
> 3. A current lane
> 
>     The 0-based SIMT lane identifier to be used in evaluating a
>     user-presented expression. This applies to source languages that are
>     implemented for a target architecture using a SIMT execution model.
>     These implementations map source language threads of execution to
>     lanes of the target architecture threads.
> 
>     It is required for operations that are related to SIMT lanes.
> 
>     [non-normative] For example, the `DW_OP_push_lane` operation.
> 
>     If specified, it must be consistent with the value of the
>     `DW_AT_num_lanes` attribute of the subprogram corresponding to
>     context’s frame and program location. It is consistent if the value
>     is greater than or equal to 0 and less than the, possibly default,
>     value of the `DW_AT_num_lanes` attribute. Otherwise the result is
>     undefined.
> 
> 4. A current call frame
> 
>     The target architecture call frame identifier. It identifies a
>     call frame that corresponds to an active invocation of a
>     subprogram in the current thread. It is identified by its
>     address on the call stack. The address is referred to as the
>     Canonical Frame Address (CFA). The call frame information is
>     used to determine the CFA for the call frames of the current
>     thread’s call stack (see 6.4 Call Frame Information).
> 
>     It is required for operations that specify target architecture
>     registers to support virtual unwinding of the call stack.
> 
>     [non-normative] For example, the `DW_OP_*reg*` operations.
> 
>     If specified, it must be an active call frame in the current
>     thread. Otherwise the result is undefined.
> 
>     If it is the currently executing call frame, then it is termed
>     the top call frame.
> 
> 5. A current program location
> 
>     The target architecture program location corresponding to the
>     current call frame of the current thread.
> 
>     The program location of the top call frame is the target
>     architecture program counter for the current thread. The call
>     frame information is used to obtain the value of the return
>     address register to determine the program location of the other
>     call frames (see 6.4 Call Frame Information).
> 
>     It is required for the evaluation of location list expressions
>     to select amongst multiple program location ranges. It is
>     required for operations that specify target architecture
>     registers to support virtual unwinding of the call stack (see
>     6.4 Call Frame Information).
> 
>     If specified:
> 
>       * If the current call frame is the top call frame, it must be
>         the current target architecture program location.
> 
>       * If the current call frame F is not the top call frame, it
>         must be the program location associated with the call site
>         in the current caller frame F that invoked the callee frame.
> 
>       * Otherwise the result is undefined.
> 
> 6. A current compilation unit
> 
>     The compilation unit debug information entry that contains the
>     DWARF expression being evaluated.
> 
>     It is required for operations that reference debug information
>     associated with the same compilation unit, including indicating
>     if such references use the 32-bit or 64-bit DWARF format. It can
>     also provide the default address space address size if no
>     current target architecture is specified.
> 
>     [non-normative] For example, the `DW_OP_constx` and `DW_OP_addrx`
>     operations.
> 
>     [non-normative] Note that this compilation unit might not be the
>     same as the compilation unit determined from the loaded code
>     object corresponding to the current program location. For
>     example, the evaluation of the expression E associated with a
>     `DW_AT_location` attribute of the debug information entry operand
>     of the `DW_OP_call*` operations is evaluated with the compilation
>     unit that contains E and not the one that contains the
>     `DW_OP_call*` operation expression.
> 
>     [non-normative] The DWARF expressions for call frame information (see
>     6.4 Call Frame Information) operations are restricted to those that
>     do not require the compilation unit context to be specified.
> 
> 7. A current target architecture
> 
>     The target architecture.
> 
>     It is required for operations that specify target architecture
>     specific entities.
> 
>     [non-normative] For example, target architecture specific
>     entities include DWARF register identifiers, DWARF address space
>     identifiers, the default address space, and the address space
>     address sizes.
> 
>     If specified:
> 
>       * If the current frame is specified, then the current target
>         architecture must be the same as the target architecture of
>         the current frame.
> 
>       * If the current frame is specified and is the top frame, and
>         if the current thread is specified, then the current target
>         architecture must be the same as the target architecture of
>         the current thread.
> 
>       * If the current compilation unit is specified, then the
>         current target architecture default address space address
>         size must be the same as the `address_size` field in the
>         header of the current compilation unit.
> 
>       * If the current program location is specified, then the
>         current target architecture must be the same as the target
>         architecture of any line number information entry (see 6.2
>         Line Number Information) corresponding to the current
>         program location.
> 
>       * If the current program location is specified, then the
>         current target architecture default address space address
>         size must be the same as the `address_size` field in the
>         header of any entry corresponding to the current program
>         location in the `.debug_addr`, `.debug_line`, `.debug_rnglists`,
>         `.debug_rnglists.dwo`, `.debug_loclists`, and
>         `.debug_loclists.dwo` sections.
> 
>       * Otherwise the result is undefined.
> 
> 8. A current object
> 
>     The location description of a program object.
> 
>     It is required for the `DW_OP_push_object_address` operation.
> 
>     [non-normative] For example, the `DW_AT_data_location` attribute
>     on type debug information entries specifies the program object
>     corresponding to a runtime descriptor as the current object when
>     it evaluates its associated expression.
> 
>     The result is undefined if the location description is invalid
>     (see 2.6 Location Descriptions).
> 
> 9. An initial stack
> 
>     This is a list of values that will be pushed on the operation
>     expression evaluation stack in the order provided before
>     evaluation of an operation expression starts.
> 
>     Some debugger information entries have attributes that evaluate
>     their DWARF expression value with initial stack entries. In all
>     other cases the initial stack is empty.
> 
>     The result is undefined if any location descriptions are invalid
>     (see 2.6 Location Descriptions).
> 
> If the evaluation requires a context element that is not specified,
> then the result of the evaluation is an error.
> 
> [non-normative] A DWARF expression for a location description may be
> able to be evaluated without a thread, lane, call frame, program location,
> or architecture context. For example, the location of a global
> variable may be able to be evaluated without such context. If the
> expression evaluates with an error then it may indicate the variable
> has been optimized and so requires more context.
> 
> The DWARF is ill-formed if all the address_size fields in the
> headers of all the entries in the `.debug_info`, `.debug_addr`,
> `.debug_line`, `.debug_rnglists`, `.debug_rnglists.dwo`, `.debug_loclists`,
> and `.debug_loclists.dwo` sections corresponding to any given program
> location do not match.
