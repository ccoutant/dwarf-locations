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

> ### 2.5.1 DWARF Expression Evaluation Context
> 
> A DWARF expression is evaluated within a context provided by the debugger
> or other DWARF consumer.
> 
> 1. Required result kind
> 
>     The kind of result required -- either a location description or a
>     value -- is determined by the DWARF construct where the expression
>     is found.
> 
> 2. Initial stack
> 
>     In most cases, the DWARF expression stack is empty at the
>     start of expression evaluation. In certain circumstances,
>     however, one or more values are pushed implicitly onto the
>     stack before evaluation of the expression starts (e.g.,
>     `DW_AT_data_location`).
> 
> 3. Current compilation unit
> 
>     The current compilation unit is the compilation unit debugging
>     information entry that contains the DWARF expression being
>     evaluated.
>     
>     It is required for operations that reference debug information
>     associated with the same compilation unit, including indicating
>     if such references use the 32-bit or 64-bit DWARF format.
>     
>     [non-normative] _For example, the `DW_OP_constx` and `DW_OP_addrx`
>     operations require the address size, which is a property of the
>     compilation unit._
>     
>     [non-normative] _Note that this compilation unit might not be the
>     same as the compilation unit determined from the loaded code
>     object corresponding to the current program location. For
>     example, the evaluation of the expression E associated with a
>     `DW_AT_location` attribute of the debug information entry operand
>     of the `DW_OP_call*` operations is evaluated with the compilation
>     unit that contains E and not the one that contains the
>     `DW_OP_call*` operation expression._
>     
>     [non-normative] _The DWARF expressions for call frame information (see
>     6.4 Call Frame Information) operations are restricted to those that
>     do not require a current compilation unit._
> 
> 4. Target architecture
> 
>     The target architecture is typically provided by the object file
>     containing the DWARF information. It may also be refined by
>     instruction set identifiers in the line number table.
>     
>     It is required for operations that specify architecture-specific entities.
>     
>     [non-normative] _Architecture-specific entities include DWARF
>     register identifiers, DWARF address space identifiers, the default
>     address space, and the address space address sizes._
> 
> 5. Current thread
> 
>     The current thread identifies a current thread of execution. When
>     debugging a multi-threaded program, the current thread may be
>     selected by a user command that focuses on a specific thread, or it
>     may be selected automatically when the running thread stops at a
>     breakpoint.
>     
>     If there is no running process, the current thread is not
>     available.
>     
>     It is required for operations that are related to target
>     architecture threads.
> 
>     [non-normative] _For example, the `DW_OP_form_tls_address` operation
>     requires a current thread._
> 
> 6. Current lane
> 
>     The current lane is a SIMD/SIMT lane identifier. This applies to source languages that are
>     implemented using a SIMD/SIMT execution model.
>     These implementations map source language threads of execution to
>     lanes of the target architecture threads.
>     When debugging a SIMD/SIMT program, the current lane is typically
>     selected by a user command that focuses on a specific lane.
>     
>     It is required for operations that are related to SIMD/SIMT lanes.
>     
>     [non-normative] _For example, the `DW_OP_push_lane` operation pushes
>     the value of the current lane._
>     
>     The current lane must be consistent with the value of the
>     `DW_AT_num_lanes` attribute of the subprogram corresponding to
>     context’s frame and program location. It is consistent if the value
>     is greater than or equal to 0 and less than the, possibly default,
>     value of the `DW_AT_num_lanes` attribute.
>     
>     If there is no running process, the current lane is not available.
>     If the current running program is not using a SIMD/SIMT
>     execution model, the current lane is always 0.
> 
> 7. Current call frame
> 
>     The current call frame identifies an active invocation of a
>     subprogram in the current thread. It is identified by its address on
>     the call stack. The address is referred to as the Canonical Frame
>     Address (CFA). The call frame information is used to determine the
>     CFA for the call frames of the current thread’s call stack (see 6.4
>     Call Frame Information).
> 
>     When debugging a running program, the current frame may be the
>     topmost frame (e.g., where a breakpoint has triggered), or may be
>     selected by a user command to focus the view on a frame further down
>     the call stack. The current frame provides a view of the state
>     of the running process at a particular point in time.
>     
>     It is required for operations that use the contents of registers
>     (e.g., `DW_OP_reg*`) or frame-local storage (e.g., `DW_OP_fbreg`) so
>     that the debugger can retrieve values from the selected view of
>     the process state.
>     
>     If there is no running process, the current call frame is not available.
>     
>     The current frame, when available, must be an active call frame in
>     the current thread.
> 
> 8. Current program counter (PC)
> 
>     The current program counter identifies the current point of
>     execution in the current call frame of the current thread.
>     
>     The PC of the top call frame is the target
>     architecture program counter for the current thread. The call
>     frame information is used to obtain the value of the return
>     address register to determine the PC of the other
>     call frames (see 6.4 Call Frame Information).
>     
>     It is required for the evaluation of expression lists
>     to select amongst multiple program location ranges.
>     
>     If there is no running process, the current program counter is not
>     available. When evaluating expression lists when no current pc is
>     available, only default location descriptions are used.
> 
> 9. Current object
> 
>     The current object is a data object described by a data object entry
>     (see Section 4.1) that is being inspected. When evaluating
>     expressions that provide attribute values of a data object, the
>     containing debugging information entry is the current object. When
>     evaluating expressions that provide attribute values for a type
>     (e.g., `DW_AT_data_location` for a `DW_TAG_data_member`), the
>     current object is the data object entry (if there is one) that
>     referred to the type entry (e.g., via `DW_AT_type`).
>     
>     A current object is required for the `DW_OP_push_object_address`
>     operation and by some attributes (e.g.,
>     `DW_AT_data_member_location`) where the object's location is
>     provided as part of the initial stack.
> 
> 
> [non-normative] _A DWARF expression for a location description may be
> able to be evaluated without a thread, lane, call frame, program
> counter, or architecture context element. For example, the location of a
> global variable may be able to be evaluated without such context. If the
> expression evaluation requires any missing context elements, it may
> indicate the variable has been optimized and so requires more context._
> 
> The `address_size` fields must match in the headers of all the entries
> in the `.debug_info`, `.debug_addr`, `.debug_line`, `.debug_rnglists`,
> `.debug_rnglists.dwo`, `.debug_loclists`, and `.debug_loclists.dwo`
> sections corresponding to any given program counter.
