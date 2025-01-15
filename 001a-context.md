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
> DWARF expressions and location descriptions (see Section 2.6) are
> evaluated within a context provided by the debugger or other DWARF
> consumer. The context includes the following elements:
> 
> 1. Required result kind
> 
>     The kind of result required -- either a location or a
>     value -- is determined by the DWARF construct where the expression
>     is found.
> 
>     [non-normative] _For example, DWARF attributes with exprval class
>     require a value, and attributes with locdesc class require a
>     location description (see Section 7.5.5)._
> 
> 2. Initial stack
> 
>     In most cases, the DWARF expression stack is empty at the
>     start of expression evaluation. In certain circumstances,
>     however, one or more values are pushed implicitly onto the
>     stack before evaluation of the expression starts (e.g.,
>     `DW_AT_data_member_location`).
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
>     [non-normative] _Many programming environments support the concept of
>     independent threads of execution, where the process and its address
>     space are shared among the threads, but each thread has its own stack,
>     program counter, and possibly its own block of memory for thread-local
>     storage (TLS). These threads may be implemented in user-space or with
>     kernel threads, or by a combination of the two._
>
>     The current thread identifies a current thread of execution. When
>     debugging a multi-threaded program, the current thread may be
>     selected by a user command that focuses on a specific thread, or it
>     may be selected automatically when the running thread stops at a
>     breakpoint.
>     
>     If there is no running process (or an image of a process, as from
>     a core file), there is no current thread. A single-threaded
>     program (as distinguished from a multi-threaded program
>     with only a single thread) might not have a current thread.
>     
>     It is required for operations that provide access to thread-local
>     storage.
> 
>     [non-normative] _For example, the `DW_OP_form_tls_address` operation
>     requires a current thread._
> 
> 6. Current call frame
> 
>     The current call frame identifies an active invocation of a
>     subprogram in the current thread. It is identified by its address
>     on the call stack (see Section 3.3.5.2). The address is referred
>     to as the frame base or the call frame address (CFA). The call
>     frame information is used to determine the base addresses for the
>     call frames of the current thread’s call stack (see 6.4 Call Frame
>     Information).
> 
>     When debugging a running program or examining a core file, the
>     current frame may be the topmost (most recently activated) frame
>     (e.g., where a breakpoint has triggered), or may be selected by a
>     user command to focus the view on a frame further down the call
>     stack. The current frame provides a view of the state of the
>     running process at a particular point in time.
>     
>     It is required for operations that use the contents of registers
>     (e.g., `DW_OP_reg*`) or frame-local storage (e.g., `DW_OP_fbreg`) so
>     that the debugger can retrieve values from the selected view of
>     the process state.
>     
>     If there is no running process (or image of a process), there is no
>     current call frame.
>     
>     The current frame must be an active call frame in the current
>     thread.
> 
> 7. Current lane
>
>     [non-normative] _On SIMD (Single-Instruction Multiple-Data Stream)
>     and SIMT (Single-Instruction Multiple-Thread) architectures,
>     fine-grained parallel execution can be achieved by dispatching a
>     single instruction across multiple data streams (e.g., a vector or
>     array). Some parallel programming models allow for the
>     vectorization of loops using SIMD instructions. These parallel
>     streams can be considered fine-grain threads of execution, or
>     lanes, where each lane typically shares a common stack, program
>     counter, and register file._
> 
>     [non-normative] _In SIMT architectures, control flow may diverge
>     through the use of predication, where each instruction executes
>     only in certain lanes. Some SIMT architectures, however, provide
>     separate stacks and register files for each lane, and the parallel
>     streams of execution would be better represented as threads
>     (above)._
> 
>     The current lane is a SIMD/SIMT lane identifier. This applies to
>     source languages that are implemented using a SIMD/SIMT execution
>     model. These implementations map source language vectorized
>     operations to SIMD/SIMT lanes of execution (see Section 3.3.5.4).
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
>     If there is no running process, there is no current lane.
>     If the current running program is not using a SIMD/SIMT
>     execution model, the current lane is always 0.
> 
> 8. Current program counter (PC)
> 
>     The current program counter identifies the current point of
>     execution in the current call frame of the current thread and/or lane.
>     
>     The PC of the top (most recently activated) call frame is the
>     program counter for the current thread. The call frame information
>     is used to obtain the value of the return address register to
>     determine the PC of the other call frames (see 6.4 Call Frame
>     Information).
>     
>     It is required for the evaluation of value lists and location lists
>     to select amongst multiple program location ranges.
>     
>     If there is no running process, there is no current program
>     counter. 
>
>     [non-normative] _When evaluating value lists and location lists
>     when no current pc is available, only default location
>     descriptions are used._
> 
> 9. Current object
> 
>     The current object is a data object described by a data object entry
>     (see Section 4.1) that is being inspected. When evaluating
>     expressions that provide attribute values of a data object, the
>     containing debugging information entry is the current object. When
>     evaluating expressions that provide attribute values for a type
>     (e.g., `DW_AT_data_location` for a `DW_TAG_member`), the
>     current object is the data object entry (if there is one) that
>     referred to the type entry (e.g., via `DW_AT_type`).
>     
>     A current object is required for the `DW_OP_push_object_address`
>     operation and by some attributes (e.g.,
>     `DW_AT_data_member_location` and `DW_AT_use_location`) where the
>     object's location is provided as part of the initial stack.
> 
> [non-normative] _A DWARF expression for a location description may be
> able to be evaluated without a thread, call frame, lane, program
> counter, or architecture context element. For example, the location of a
> global variable may be able to be evaluated without such context,
> while the location of local variables in a stack frame cannot be
> evaluated without additional context._


### Section 2.5.2.2 Register Values [was 2.5.1.2]

In item 1, `DW_OP_fbreg`, add "(see Section 2.5.1)" at
the end of the first paragraph.


### Section 2.5.2.3 Stack Operations [was 2.5.1.3]

In item 13, `DW_OP_push_object_address`, add "(see Section 2.5.1)" at
the end of the first sentence.

In item 14, `DW_OP_form_tls_address`, change the first sentence to:

> The `DW_OP_form_tls_address` operation pops a value from the stack, which
> must have an integral type identifier, translates this value into an
> address in the thread-local storage for the current thread (see Section
> 2.5.1), and pushes the address onto the stack together with the generic
> type identifier.

In item 15, `DW_OP_call_frame_cfa`, change the first paragraph to:

> The `DW_OP_call_frame_cfa` operation pushes the value of the current
> call frame address (CFA), obtained from the Call Frame Information (see
> Section 2.5.1 and Section 6.4).

In item 16, `DW_OP_push_lane`, change the last sentence of the first paragraph to:

> See Section 2.5.1 and Section 3.3.5.

Remove following two non-normative paragraphs (which duplicate description
added in 2.5.1.)

### Section 2.6.2 Location Lists

Following the first bulleted list (after "End-of-list"), add the following
paragraph:

> If there is no current PC (see Section 2.5.1), only
> default location description entries will apply.


### Section 5.7.6 Data Member Entries

In bullet 2 under `DW_AT_data_member_location`, add "(see Section 2.5.1)"
at the end of the first paragraph.


### Section 5.14 Pointer to Member Type Entries

In the paragraph beginning "The `DW_AT_use_location` description is used
in conjunction...", add "(see Section 2.5.1)" at the end of the second
sentence:

> The `DW_AT_use_location` description is used in conjunction with the
> location descriptions for a particular object of the given pointer to
> member type and for a particular structure or class instance. The
> `DW_AT_use_location` attribute expects two values to be pushed onto
> the DWARF expression stack before the `DW_AT_use_location` description
> is evaluated (see Section 2.5.1). ...


### Section 6.4.2 Call Frame Instructions

Add the following after the first sentence of the second paragraph:

> The DWARF expressions for call frame information are restricted to
> those operations that do not require a current compilation unit (see
> Section 2.5.1).
