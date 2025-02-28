# Chapter 2: General Description

## <span class="del">2.5 DWARF Expressions [Now Chapter 3]</span>

## <span class="del">2.6 Location Descriptions [Now Chapter 3]</span>

## <span class="add">2.5 Values and Locations [NEW]</span>

<span class="add">A DWARF expression is evaluated in a context that determines whether
its result is expected to be a value or a location. Expressions that are
expected to produce a location are called "location descriptions."</span>

<span class="add">Values on the stack are typed, and can represent a value of any
supported base type of the target machine, or of the generic type, which
is an integral type that has the size of an address in the default
address space on the target machine, and unspecified signedness.</span>

<span class="add">*The generic type is the same as the unspecified type used for stack
operations defined in DWARF Version 4 and before.*</span>

<span class="add">*Debugging information must provide consumers a way to
find the location of program variables, determine the bounds of dynamic
arrays and strings, and possibly to find the base address of a
subroutine’s stack frame or the return address of a subroutine.
Furthermore, to meet the needs of recent computer architectures and
optimization techniques, debugging information must be able to describe
the location of an object whose location changes over the object’s
lifetime.*</span>

<span class="add">Information about the location of program objects is provided by
location descriptions and location lists.</span>

<span class="add">A **location description** is a DWARF expression yielding a
single location. These are sufficient for describing the
location of any object as long as its lifetime is either
static or the same as the lexical block that owns it,
excluding any prologue or epilogue ranges, and it does not
move during its lifetime. As the value of an attribute, a
location description is encoded using class `locdesc`.</span>

<span class="add">A **location list** describes objects that have a limited
lifetime or change their location during their lifetime. A
location list is a list of location descriptions, each
associated with a range of program counters. Location
lists are described in Section 3.17. As the value of an
attribute, a location list is encoded using class
`loclist` (which serves as an index into a separate
section containing location lists).</span>

<span class="add">A location list may have overlapping PC ranges, and thus
may yield more than one location. In these cases, the
object value stored in each location must be the same
(except for uninitialized/undefined parts of the value).</span>

<span class="add">*A location list that yields multiple locations can be
used to describe objects that reside in more than one
piece of storage at the same time. An object may have more
than one location as a result of optimization. For
example, a value that is only read may be promoted from
memory to a register for some region of code, but later
code may revert to reading the value from memory as the
register may be used for other purposes. For the code
region where the value is in a register, any change to the
object value must be made in both the register and the
memory so both regions of code will read the updated
value.*</span>

<span class="add">*When given multiple locations, a consumer can read the
object’s value from any of those locations (since they all
refer to storage that has the same value), but must write
any changed value to all the locations.*</span>

<span class="add">DWARF can describe the location of program objects in
several kinds of storage. The location identifies a
specific bank of storage, and provides a (zero-based) bit
offset relative to the start of that storage.</span>

<span class="add">A storage bank is a linear stream of bits of finite size.
The ordering of bits within a storage bank uses the bit
numbering and direction conventions that are appropriate
to the current language on the target architecture. An
offset may not exceed the size (in bits) of the storage
bank.</span>

<span class="add">DWARF can describe five kinds of storage banks:</span>

- <span class="add">Memory storage.
  Corresponds to the target architecture memory address
  spaces. The size of a memory storage bank is determined
  by the size of the address space.</span>

- <span class="add">Register storage.
  Corresponds to the target architecture registers. Each
  register is a separate storage bank, and the size of the
  storage bank is the size of the register.</span>

- <span class="add">Undefined storage.
  Indicates no value is available and therefore cannot be
  read or written. The size of an undefined storage bank
  is limited to the size of the largest address space or
  register on the target architecture.</span>

- <span class="add">Implicit storage.
  Corresponds to fixed values that can only be read. The
  size of an implicit storage bank is determined by the
  type of the value or the size of the constant block used
  to define the implicit storage, and is limited to the
  size of the largest address space or register on the target
  architecture.</span>

- <span class="add">Composite storage.
  Allows a mixture of these where some bits come from one
  storage bank and some from another storage bank, or from
  disjoint parts of the same storage bank. The size of a
  composite storage bank is the sum of the sizes of the
  composite parts, and is limited to the size of the
  largest address space or register on the target
  architecture.</span>

<span class="add">An implicit conversion between a memory location and a value may happen
during the execution of any operation or when evaluation of the
expression is completed. If a location is expected, but the result is
a value, the value is implicitly treated as a memory address in the
default address space, and converted to a memory location. If a value
is expected, but the result is an addressable memory location in the
default address space, the address is implicitly converted to a value
of the generic type.</span>

# Chapter 3: DWARF Expressions

DWARF expressions describe how to compute a value or
specify a location. They are expressed in
terms of DWARF operations that operate on a stack <span class="del">of values.</span>
<span class="add">Each element on the stack may be
either a value or a location.</span>

A DWARF expression is encoded as a stream of operations,
each consisting of an opcode followed by zero or more literal
operands. The number of operands is implied by the opcode.

<span class="del">In addition to the
general operations that are defined here, operations that are
specific to location descriptions are defined in
Section {locationdescriptions}.</span>

<span class="add">The result of a DWARF expression is the value or location on the top
of the stack after evaluating the operations.</span>

<span class="add">Values on the stack are typed, and can represent a value of any
supported base type of the target machine, or of the generic type,
which is an integral type that has the size of an address on the
target machine, and unspecified signedness.</span>

<span class="add">*The generic type is the same as the unspecified type used for stack
operations defined in DWARF Version 4 and before.*</span>


## <span class="del">2.5.1</span> 3.1 DWARF Expression Evaluation Context

DWARF expressions <span class="del">and location descriptions (see Section
{locationdescriptions})</span> are evaluated within a
context provided by the debugger or other DWARF consumer. 
The context includes the following elements:

1. Required result kind
    
    The kind of result required -- either a location or a value -- is 
    determined by the DWARF construct where the expression is found.
    
    *For example, DWARF attributes with `exprval` class
    require a value, and attributes with `locdesc` class require 
    a location <span class="del">description</span> (see Section {classesandforms}).*
    
1. Initial stack
    
    In most cases, the DWARF expression stack is empty at the start
    of expression evaluation. In certain circumstances, however, one
    or more values are pushed implicitly onto the stack before evaluation
    of the expression starts (e.g., `DW_AT_data_member_location`).
    
1. Current compilation unit
    
    The current compilation unit is the compilation unit debugging
    information entry that contains the DWARF expression being evaluated.
    
    A current compilation unit is required for operations that reference
    debug information associated with the same compilation unit, including
    indicating if such references use the 32-bit or 64-bit DWARF format.
     
    *For example, the `DW_OP_constx` and `DW_OP_addrx` operations
    require the address size, which is a property of the compilation unit.*
    
    *Note that this compilation unit might not be the same as the
    compilation unit determined from the loaded code object corresponding
    to the current program location. For example, the evaluation of the
    expression E associated with a `DW_AT_location` attribute of the debug
    information entry operand of the `DW_OP_call<n>` operations is evaluated
    with the compilation unit that contains E and not the one that contains
    the `DW_OP_call<n>` operation expression.*
    
1. Target architecture
    
    The target architecture is typically provided by the object file
    containing the DWARF information. It may also be refined by instruction
    set identifiers in the line number table.
    
    The target architecture is required for operations that specify
    architecture-specific entities.
    
    *Architecture-specific entities include DWARF register identifiers,
    DWARF address space identifiers, the default address space, and the
    address space address sizes.*
    
1. Current thread
    
    *Many programming environments support the concept of independent
    threads of execution, where the process and its address space are shared
    among the threads, but each thread has its own stack, program counter,
    and possibly its own block of memory for thread-local storage (TLS). 
    These threads may be implemented in user-space or with kernel threads,
    or by a combination of the two.*
    
    The current thread identifies a current thread of execution. When
    debugging a multi-threaded program, the current thread may be selected
    by a user command that focuses on a specific thread, or it may be selected
    automatically when the running thread stops at a breakpoint.
    
    *If there is no current process (or an image of a process, as from
    a core file), there is no current thread.*
    
    A current thread is required for the `DW_OP_form_tls_address` operation 
    (see Section {stackoperations}) which provides access to 
    thread-local storage.
    
1. Current call frame
    
    The current call frame identifies an active invocation of a subprogram.
    It is identified by its address on the call stack (see Section 
    {framebase}). The address is referred to as the frame base
    or the call frame address (CFA). The call frame information is used to
    determine the base addresses for the call frames of the current thread’s
    call stack (see Section {callframeinformation}).
    
    *When debugging a running program or examining a core file,
    the current frame may be the topmost (most recently activated) frame 
    (e.g., where a breakpoint has triggered), or may be selected by a user
    command to focus the view on a frame further down the call stack.
    The current frame provides a view of the state of the running process
    at a particular point in time.*
    
    The current call frame (if there is one) must be an active call frame
    in the current call stack.
    
    A current call frame is required for operations that use the contents
    of registers (e.g., `DW_OP_reg<n>`) or frame-local storage (e.g., `DW_OP_fbreg`)
    so that the debugger can retrieve values from the selected view of the
    process state.
    
1. Current lane
    
    *On SIMD (Single-Instruction Multiple-Data Stream) and SIMT
    (Single-Instruction Multiple-Thread) architectures, fine-grained parallel
    execution can be achieved by dispatching a single instruction across
    multiple data streams (e.g., a vector or array). Some parallel programming
    models allow for the vectorization of loops using SIMD instructions.
    These parallel streams can be considered fine-grain threads of execution,
    or lanes, where all lanes typically share a common stack, program counter,
    and register file.*
    
    *In SIMT architectures, control flow may diverge through the use of
    predication, where each instruction executes only in certain lanes. Some
    SIMT architectures, however, provide separate stacks and register files
    for each lane, and the parallel streams of execution may instead be
    represented as threads (above).*
    
    The current lane is a SIMD/SIMT lane identifier. This applies to source
    languages with scalar code that is vectorized by the compiler using a
    SIMD/SIMT execution model. These implementations map vectorized operations
    to SIMD/SIMT lanes of execution (see Section {lanesinsimdvectorization}).
    When debugging a SIMD/SIMT program, the current lane is typically selected
    by a user command that focuses on a specific lane.
    
    The current lane number must be consistent with the value of the `DW_AT_num_lanes`
    attribute of the subprogram corresponding to the current frame and program 
    location. It is consistent if the lane number is greater than or equal to 0 and 
    less than the, possibly default, value of the `DW_AT_num_lanes` attribute.
    
    If the current program is not using a SIMD/SIMT execution model, the current
    lane is always 0.
    
    A current lane is required for the `DW_OP_push_lane` operation (see Section 
    {stackoperations}), which pushes the value of the current lane.
    
1. Current program counter (PC)
    
    The current program counter (PC) identifies the current point of execution
    in the current call frame.
    
    The PC in each call frame is the address of the next instruction to be
    executed in that frame. For the top (most recent) frame on the call stack,
    this is where execution would resume; for frames lower on the stack, it
    is where the callee will return. The call frame information is used to
    obtain the value of the return address register to determine the PC of
    the other call frames (see Section {callframeinformation}).
    
    If there is no current frame, there is no current PC.
    
    The current PC is used during the evaluation of value lists and location 
    lists to select from among multiple program location ranges.
    
    *When evaluating value lists and location lists when no current PC
    is available, only default <span class="del">location descriptions</span>
    <span class="add">value or location list entries</span> may be used.*
    
1. Current object
    
    The current object is a data object described by a data object entry 
    (see Section {dataobjectentries}) that is being inspected. 
    When evaluating expressions that provide attribute values of a data object,
    the containing debugging information entry is the current object. When 
    evaluating expressions that provide attribute values for a type (e.g.,
    `DW_AT_data_location` for a `DW_TAG_member`), the current object is the data
    object entry (if there is one) that referred to the type entry (e.g., 
    via `DW_AT_type`).
    
    A current object is required for the `DW_OP_push_object_address`
    (see Section {stackoperations}) operation and <span class="add">is
    implicitly defined</span> by some attributes (e.g.,
    `DW_AT_data_member_location` and `DW_AT_use_location`) where the
    object's location is provided as part of the initial stack.

*A DWARF expression <span class="del">for a location description</span> may be able to be
evaluated without a thread, call frame, lane, program counter, or architecture
context element. For example, the location of a global variable may be able
to be evaluated without such context, while the location of local variables
in a stack frame cannot be evaluated without additional context.*

## <span class="del">2.5.2 General Operations [REMOVED]</span>

<span class="del">Each general operation represents a postfix operation on
a simple stack machine.
Each element of the stack has a type and a value, and can represent
a value of any supported base type of the target machine.  Instead of
a base type, elements can have a generic type,
which is an integral type that has the
size of an address on the target machine and
unspecified signedness. The value on the top of the stack after “executing” the
DWARF expression is taken to be the result (the address of the object, the
value of the array bound, the length of a dynamic string,
the desired value itself, and so on).</span>

<span class="del">*The generic type is the same as the unspecified type used for stack operations
defined in DWARF Version 4 and before.*</span>

## <span class="del">2.5.2.3</span> 3.2 Stack Operations

The following
operations manipulate the DWARF stack,
<span class="add">and may
operate on both values and locations.</span>
Operations that index the stack assume that the top of the stack (most
recently added entry) has index 0.

Each entry on the stack <span class="del">has an associated type</span>
<span class="add">is either a value (with an associated type) or
a location</span>.

1. `DW_OP_dup`  
    The `DW_OP_dup` operation duplicates the <span class="del">value (including its
    type identifier)</span>
    <span class="add">entry</span> at the top of the stack.
    
1. `DW_OP_drop`  
    The `DW_OP_drop` operation pops the <span class="del">value <span class="del">(including its type
    identifier)</span></span><span class="add">entry</span> at the top of the stack.
    
1. `DW_OP_pick`  
    The single operand of the `DW_OP_pick` operation provides a
    1-byte index. A copy of the stack entry <span class="del">(including its
    type identifier)</span> with the specified
    index (0 through 255, inclusive) is pushed onto the stack.
    
1. `DW_OP_over`  
    The `DW_OP_over` operation duplicates the entry currently second
    in the stack at the top of the stack.
    This is equivalent to a
    `DW_OP_pick` operation, with index 1.
    
1. `DW_OP_swap`  
    The `DW_OP_swap` operation swaps the top two stack entries.
    The entry at the top of the stack <span class="del">(including its type identifier)</span>
    becomes the second stack entry, and the second entry <span class="del">(including
    its type identifier)</span> becomes the top of the stack.
    
1. `DW_OP_rot`  
    The `DW_OP_rot` operation rotates the first three stack
    entries. The entry at the top of the stack <span class="del">(including its
    type identifier)</span> becomes the third stack entry, the second
    entry <span class="del">(including its type identifier)</span> becomes the top of
    the stack, and the third entry <span class="del">(including its type identifier)</span>
    becomes the second entry.
    
1. `DW_OP_deref`  
    <span class="del">The `DW_OP_deref` operation pops the top stack entry and
    treats it as an address. The popped value must have an integral type.
    The value retrieved from that address is pushed,
    and has the generic type.
    The size of the data retrieved from the
    dereferenced
    address is the size of an address on the target machine.</span>  
    <span class="add">The `DW_OP_deref` operation pops a location `L` from the top of the
    stack. The first `S` bits, where `S` is the size (in bits) of an address on the
    target machine, are retrieved from the location `L` and pushed onto the
    stack as a value of the generic type.</span>
    
1. `DW_OP_deref_size`  
    <span class="del">The `DW_OP_deref_size` operation behaves like the
    `DW_OP_deref`
    operation: it pops the top stack entry and treats it as an
    address. The popped value must have an integral type.
    The value retrieved from that address is pushed,
    and has the generic type.
    In the `DW_OP_deref_size` operation, however, the size in bytes
    of the data retrieved from the dereferenced address is
    specified by the single operand. This operand is a 1-byte
    unsigned integral constant whose value may not be larger
    than the size of the generic type. The data
    retrieved is zero extended to the size of an address on the
    target machine before being pushed onto the expression stack.</span>  
    <span class="add">
    The `DW_OP_deref_size` takes a single 1-byte unsigned integral operand
    that specifies the size `S`, in bytes, of the value to be retrieved.
    The size `S` must be no larger than the size of the generic type. The
    operation behaves like the `DW_OP_deref` operation: it pops a location
    `L` from the stack. The first `S` bytes are retrieved from the
    location `L`, zero extended to the size of the generic type, and
    pushed onto the stack as a value of the generic type.
    </span>
    
1. `DW_OP_deref_type`  
    <span class="del">The `DW_OP_deref_type` operation behaves like the `DW_OP_deref_size` operation:
    it pops the top stack entry and treats it as an address.
    The popped value must have an integral type.
    The value retrieved from that address is pushed together with a type identifier.
    In the `DW_OP_deref_type` operation, the size in
    bytes of the data retrieved from the dereferenced address is specified by
    the first operand. This operand is a 1-byte unsigned integral constant whose
    value which is the same as the size of the base type referenced
    by the second operand.
    The second operand is an unsigned LEB128 integer that
    represents the offset of a debugging information entry in the current
    compilation unit, which must be a `DW_TAG_base_type` entry that provides the
    type of the data pushed.</span>  
    <span class="add">The `DW_OP_deref_type` operation takes two operands. The first operand
    is a 1-byte unsigned integer that specifies the size `S` (in bytes) of the type
    given by the second operand. The second operand is an unsigned LEB128
    integer that represents the offset of a debugging information entry in
    the current compilation unit, which must be a `DW_TAG_base_type` entry
    that provides the type `T` of the value to be retrieved. The size `S`
    must be the same as the size of the base type represented by the
    second operand. This operation pops a location `L` from the stack. The
    first `S` bytes are retrieved from the location `L` and pushed onto the
    stack as a value of type `T`.</span>

    *While the size of the pushed value could be inferred from the base
    type definition, it is encoded explicitly into the operation so that the
    operation can be parsed easily without reference to the `.debug_info`
    section.*
    
1. `DW_OP_xderef`  
    The `DW_OP_xderef` operation provides an extended dereference
    mechanism. The entry at the top of the stack is treated as an
    address. The second stack entry is treated as an “address
    space identifier” for those architectures that support
    multiple
    address spaces.
    Both of these entries must have integral type<span class="add">s</span> <span class="del">identifiers</span>.
    The top two stack elements are popped,
    and a data item is retrieved through an implementation-defined
    address calculation and pushed as the new stack top together with the
    generic type <span class="del">identifier</span>.
    The size of the data retrieved from the
    dereferenced
    address is the size of the generic type.
    
1. `DW_OP_xderef_size`  
    The `DW_OP_xderef_size` operation behaves like the
    `DW_OP_xderef` operation. The entry at the top of the stack is
    treated as an address. The second stack entry is treated as
    an “address space identifier” for those architectures
    that support
    multiple
    address spaces.
    Both of these entries must have integral type<span class="add">s</span> <span class="del">identifiers</span>.
    The top two stack
    elements are popped, and a data item is retrieved through an
    implementation-defined address calculation and pushed as the
    new stack top. In the `DW_OP_xderef_size` operation, however,
    the size in bytes of the data retrieved from the
    dereferenced
    address is specified by the single operand. This operand is a
    1-byte unsigned integral constant whose value may not be larger
    than the size of an address on the target machine. The data
    retrieved is zero extended to the size of an address on the
    target machine before being pushed onto the expression stack together
    with the generic type <span class="del">identifier</span>.
    
1. `DW_OP_xderef_type`  
    The `DW_OP_xderef_type` operation behaves like the `DW_OP_xderef_size`
    operation: it pops the top two stack entries, treats them as an address and
    an address space identifier, and pushes the value retrieved. In the
    `DW_OP_xderef_type` operation, the size in bytes of the data retrieved from
    the dereferenced address is specified by the first operand. This operand is
    a 1-byte unsigned integral constant whose value
    <span class="del">value which</span> is the same as the size of the base type referenced
    by the second operand. The second
    operand is an unsigned LEB128 integer that represents the offset of a
    debugging information entry in the current compilation unit, which must be a
    `DW_TAG_base_type` entry that provides the type of the data pushed.
    
1. `DW_OP_push_lane`  
    The `DW_OP_push_lane` operation pushes a lane index value
    of the generic type, which provides the context of the lane in
    which the expression is being evaluated (see 
    Section {dwarfexpressionevaluationcontext} and 
    Section {lowlevelinformation}).

*Examples illustrating many of these stack operations are
found in Appendix {dwarfstackoperationexamples}.*

## <span class="del">2.5.2.1</span> 3.3 Literal <span class="del">Encodings</span> <span class="add">and Constant Operations</span>

The following operations all push a value onto the DWARF stack.
Operations other than `DW_OP_const_type` push a value with the
generic type, and if the value of a constant in one of these
operations is larger than can be stored in a single stack element,
the value is truncated to the element size and the low-order bits
are pushed on the stack.

1. `DW_OP_lit0`, `DW_OP_lit1`, ..., `DW_OP_lit31`  
    The `DW_OP_lit<n>` operations encode the unsigned literal values
    from 0 through 31, inclusive.
    
1. `DW_OP_const1u`, `DW_OP_const2u`, `DW_OP_const4u`, `DW_OP_const8u`  
    The single operand of a `DW_OP_const<n>u` operation provides a 1,
    2, 4, or 8-byte unsigned integer constant, respectively.
    
1. `DW_OP_const1s`, `DW_OP_const2s`, `DW_OP_const4s`, `DW_OP_const8s`  
    The single operand of a `DW_OP_const<n>s` operation provides a 1,
    2, 4, or 8-byte signed integer constant, respectively.
    
1. `DW_OP_constu`  
    The single operand of the `DW_OP_constu` operation provides
    an unsigned LEB128 integer constant.
    
1. `DW_OP_consts`  
    The single operand of the `DW_OP_consts` operation provides
    a signed LEB128 integer constant.
    
1. `DW_OP_constx`  
    The `DW_OP_constx` operation has a single operand that
    encodes an unsigned LEB128 value,
    which is a zero-based
    index into the `.debug_addr` section, where a constant, the
    size of a machine address, is stored.
    This index is relative to the value of the
    `DW_AT_addr_base` attribute of the associated compilation unit.
    
    *The `DW_OP_constx` operation is provided for constants that
    require link-time relocation but should not be
    interpreted by the consumer as a relocatable address
    (for example, offsets to thread-local storage).*
    
1. `DW_OP_const_type`  
    The `DW_OP_const_type` operation takes three operands. The first operand
    is an unsigned LEB128 integer that represents the offset of a debugging
    information entry in the current compilation unit, which must be a
    `DW_TAG_base_type` entry that provides the type of the constant provided. The
    second operand is 1-byte unsigned integer that specifies the size of the
    constant value, which is the same as the size of the base type referenced
    by the first operand. The third operand is a
    sequence of bytes of the given size that is
    interpreted as a value of the referenced type.
    
    *While the size of the constant can be inferred from the base type
    definition, it is encoded explicitly into the operation so that the
    operation can be parsed easily without reference to the `.debug_info`
    section.*


## <span class="del">2.5.2.2</span> 3.4 Register Value Operations

The following operations push
<span class="del">a value onto the stack that is either part or all of</span>
<span class="add">all or part of</span>
the contents of a register <span class="del">or the result of adding the contents of a
register to a given signed offset</span> <span class="add">onto the stack</span>.
<span class="del">`DW_OP_regval_type` pushes the contents of
a
register together with the given base type.
`DW_OP_regval_bits` pushes the partial contents of
a register together with the generic type.
The other operations
push the result of adding the contents of a register to a given
signed offset together with the generic type.</span>

1. `DW_OP_regval_type`  
    The `DW_OP_regval_type` operation
    pushes
    the contents of
    a given register interpreted as a value of a given type. The first
    operand is an unsigned LEB128 number,
    which identifies a register whose contents is to
    be pushed onto the stack. The second operand is an unsigned LEB128 number
    that represents the offset of a debugging information entry in the current
    compilation unit, which must be a `DW_TAG_base_type` entry that provides the
    type of the value contained in the specified register.
    
1. `DW_OP_regval_bits`  
    The `DW_OP_regval_bits` operation takes a single unsigned LEB128
    integer operand, which gives the number of bits to read. This number must
    be smaller or equal to the bit size of the generic type.  It pops
    the top two stack elements and interprets the top element as an
    unsigned bit offset from the least significant bit end and the
    other as a register number identifying the register from which to
    extract the value.  If the extracted value is smaller than the size
    of the generic type, it is zero extended.

## <span class="del">2.5.2.4</span> 3.5 Arithmetic and Logical Operations

The following provide arithmetic and logical operations.
Operands of an operation with two operands
must have the same type,
either the same base type or the generic type.
The result of the operation which is pushed back has the same type
as the type of the operand(s).

<span class="del">If the type of the operands is the generic type,
except as otherwise specified, the arithmetic operations
perform addressing arithmetic, that is, unsigned arithmetic that is performed
modulo one plus the largest representable address.</span>

Operations other than `DW_OP_abs`,
`DW_OP_div`, `DW_OP_minus`, `DW_OP_mul`, `DW_OP_neg` and `DW_OP_plus`
require integral types of the operand (either integral base type
or the generic type).  Operations do not cause an exception
on overflow.

1. `DW_OP_abs`  
    The `DW_OP_abs` operation pops the top stack entry, interprets
    it as a signed value and pushes its absolute value. If the
    absolute value cannot be represented, the result is undefined.
    
1. `DW_OP_and`  
    The `DW_OP_and` operation pops the top two stack values, performs
    a bitwise and operation on the two, and pushes the result.
    
1. `DW_OP_div`  
    The `DW_OP_div` operation pops the top two stack values, divides the former second entry by
    the former top of the stack using signed division, and pushes the result.
    
1. `DW_OP_minus`  
    The `DW_OP_minus` operation pops the top two stack values, subtracts the former top of the
    stack from the former second entry, and pushes the result.
    
1. `DW_OP_mod`  
    The `DW_OP_mod` operation pops the top two stack values and pushes the result of the
    calculation: former second stack entry modulo the former top of the stack.
    
1. `DW_OP_mul`  
    The `DW_OP_mul` operation pops the top two stack entries, multiplies them together, and
    pushes the result.
    
1. `DW_OP_neg`  
    The `DW_OP_neg` operation pops the top stack entry, interprets
    it as a signed value and pushes its negation. If the negation
    cannot be represented, the result is undefined.
    
1. `DW_OP_not`  
    The `DW_OP_not` operation pops the top stack entry, and pushes
    its bitwise complement.
    
1. `DW_OP_or`  
    The `DW_OP_or` operation pops the top two stack entries, performs
    a bitwise or operation on the two, and pushes the result.
    
1. `DW_OP_plus`  
    The `DW_OP_plus` operation pops the top two stack entries,
    adds them together, and pushes the result.
    
1. `DW_OP_plusuconst`  
    The `DW_OP_plusuconst` operation pops the top stack entry,
    adds it to the unsigned LEB128constant operand
    interpreted as the same type as the operand popped from the
    top of the stack and pushes the result.
    
    *This operation is supplied specifically to be
    able to encode more field offsets in two bytes than can be
    done with
    “`DW_OP_lit<n> DW_OP_plus`”.*
    
1. `DW_OP_shl`  
    The `DW_OP_shl` operation pops the top two stack entries,
    shifts the former second entry left (filling with zero bits)
    by the number of bits specified by the former top of the stack,
    and pushes the result.
    
1. `DW_OP_shr`  
    The `DW_OP_shr` operation pops the top two stack entries,
    shifts the former second entry right logically (filling with
    zero bits) by the number of bits specified by the former top
    of the stack, and pushes the result.
    
1. `DW_OP_shra`  
    The `DW_OP_shra` operation pops the top two stack entries,
    shifts the former second entry right arithmetically (divide
    the magnitude by 2, keep the same sign for the result) by
    the number of bits specified by the former top of the stack,
    and pushes the result.
    
1. `DW_OP_xor`  
    The `DW_OP_xor` operation pops the top two stack entries,
    performs a bitwise exclusive-or operation on the two, and
    pushes the result.


## <span class="add">3.6 General Location Operations</span>

<span class="add">The following operations can be used to push a location onto the stack:</span>

1. `DW_OP_fbreg`  
    The `DW_OP_fbreg` operation provides a
    signed LEB128 <span class="add">byte</span> offset
    from the <span class="del">address</span> <span class="add">location</span>
    specified by the location description in the
    `DW_AT_frame_base` attribute of the current function
    (see Section {dwarfexpressionevaluationcontext}).
    
    *This is typically a stack pointer register plus or minus some offset.*

1. `DW_OP_push_object_address` [Rename to `DW_OP_push_object_location`?]  
    <span class="del">The `DW_OP_push_object_address` operation pushes the
    address of the object currently being evaluated as part of
    evaluation of a user presented expression (see Section
    {dwarfexpressionevaluationcontext}). This object may correspond to
    an independent variable described by its own debugging information
    entry or it may be a component of an array, structure, or class
    whose address has been dynamically determined by an earlier step
    during user expression evaluation.</span>

    <span class="add">The `DW_OP_push_object_address` operation pushes the location of the
    current object (see Section 3.1) onto the stack, as part of evaluation of a user
    presented expression.</span>
    
    <span class="add">*This object may correspond to an independent
    variable described by its own debugging information entry; or it may be a
    component of an array, structure, or class whose address has been
    dynamically determined by an earlier step during user expression
    evaluation.*</span>
    
    *This operator provides explicit functionality (especially for
    arrays involving descriptors) that is analogous to the implicit push
    of the base address of a structure prior to evaluation of a
    `DW_AT_data_member_location` to access a data member of a structure. For
    an example, see Appendix D.2.*

## <span class="del">2.6.1.1.2</span> 3.7 Memory Locations

<span class="add">A memory location represents the location of a piece or all of an
object or other entity in memory. On architectures that support
multiple address spaces, a memory location contains a component that
identifies the address space.</span>

<span class="add">In contexts that expect a location, a value of the generic type
will be implicitly converted to a memory location in the default
address space.</span>

<span class="add">The following operations push memory locations onto the stack:</span>

1. `DW_OP_addr`  
    The `DW_OP_addr` operation has a single operand that encodes
    a machine address and whose size is the size of an address
    on the target machine.
    <span class="add">The value of this operand is treated as an address
    in the default address space and the corresponding memory location is
    pushed onto the stack.</span>
    
1. `DW_OP_addrx`  
    The `DW_OP_addrx` operation has a single operand that
    encodes an unsigned LEB128 value,
    which is a zero-based index into the `.debug_addr` section,
    where a machine address is stored.
    This index is relative to the value of the
    `DW_AT_addr_base` attribute of the associated compilation unit.
    <span class="add">The address obtained is treated as an address in the default address
    space and the corresponding memory location is pushed onto the stack.</span>
    
1. `DW_OP_breg0`, `DW_OP_breg1`, ..., `DW_OP_breg31`  
    The single operand of the `DW_OP_breg<n>` operations provides a signed
    LEB128 <span class="del">offset from the contents of the specified register</span>
    <span class="add">byte offset. The contents of the specified register (0–31) are
    treated as a memory address in the default address space. The offset is
    added to the address obtained from the register and the resulting memory
    location is pushed onto the stack.</span>
    
1. `DW_OP_bregx`  
    <span class="del">The `DW_OP_bregx` operation provides the sum of two values specified
    by its two operands. The first operand is a register number
    which is specified by an unsigned LEB128number. The second operand is a signed LEB128 offset.</span>  
    <span class="add">The `DW_OP_bregx` operation has two operands. The first
    operand is a register number which is specified by an unsigned LEB128
    number. The second operand is a signed LEB128 byte offset. It is the same as
    `DW_OP_breg<n>` except it uses the register and offset provided by the
    operands.</span>

    
1. `DW_OP_form_tls_address`  
    The `DW_OP_form_tls_address` operation pops a value from the stack,
    which must have an integral type <span class="del">identifier</span>,
    translates this value
    into an address in the thread-local storage for the current thread
    (see Section {dwarfexpressionevaluationcontext}), and pushes the
    address onto the stack <span class="del">together with the generic type identifier</span>
    <span class="add">as a memory location (which may be an address
    space other than the default)</span>.
    The meaning of the value on the top of the stack prior to this
    operation is defined by the run-time environment.  If the run-time
    environment supports multiple thread-local storage blocks for a
    single thread, then the block corresponding to the executable or
    shared library containing this DWARF expression is used.
    
    *Some implementations of C, C++, Fortran, and other languages,
    support a thread-local storage class. Variables with this storage
    class have distinct values and addresses in distinct threads, much
    as automatic variables have distinct values and addresses in each
    function invocation. Typically, there is a single block of storage
    containing all thread-local variables declared in the main
    executable, and a separate block for the variables declared in each
    shared library. Each thread-local variable can then be accessed in
    its block using an identifier. This identifier is typically an
    offset into the block and pushed onto the DWARF stack by one of the
    `DW_OP_const<n><x>` operations prior to the `DW_OP_form_tls_address`
    operation. Computing the address of the appropriate block can be
    complex (in some cases, the compiler emits a function call to do
    it), and difficult to describe using ordinary DWARF location
    descriptions. Instead of forcing complex thread-local storage
    calculations into the DWARF expressions, the `DW_OP_form_tls_address`
    allows the consumer to perform the computation based on the run-time
    environment.*
    
1. `DW_OP_call_frame_cfa`  
    The `DW_OP_call_frame_cfa` operation pushes the value of the
    current call frame address (CFA), 
    obtained from the Call Frame Information
    (see Section {dwarfexpressionevaluationcontext} and 
    Section {callframeinformation}).
    
    *Although the value of `DW_AT_frame_base`
    can be computed using other DWARF expression operators,
    in some cases this would require an extensive location list
    because the values of the registers used in computing the
    CFA change during a subroutine. If the
    Call Frame Information
    is present, then it already encodes such changes, and it is
    space efficient to reference that.*
    
## <span class="del">2.6.1.1.3</span> 3.8 Register Locations

<span class="del">A register location description consists of a register name
operation, which represents a piece or all of an object
located in a given register.</span>

*Register location descriptions describe an object
(or a piece of an object) that resides in a register, while
the opcodes listed in Section {registervalues}
are used to describe an object (or a piece of
an object) that is located in memory at an address that is
contained in a register (possibly offset by some constant).
<span class="del">A register location description must stand alone as the entire
description of an object or a piece of an object.</span>*

The following DWARF operations can be used to
specify a register location.

*Note that the register number represents a DWARF specific
mapping of numbers onto the actual registers of a given
architecture. The mapping should be chosen to gain optimal
density and should be shared by all users of a given
architecture. It is recommended that this mapping be defined
by the ABI authoring committee for each architecture.*

1. `DW_OP_reg0`, `DW_OP_reg1`, ..., `DW_OP_reg31`  
    The `DW_OP_reg<n>` operations encode the names of up to 32
    registers, numbered from 0 through 31, inclusive. <span class="del">The object
    addressed is in register _n_.</span>
    <span class="add">A location is pushed on the stack for the
    register's storage bank with an offset of 0.</span>
    
1. `DW_OP_regx`  
    The `DW_OP_regx` operation has a single
    unsigned LEB128 literal
    operand that encodes the name of a register.
    <span class="add">A location is pushed on the stack for the
    register's storage bank with an offset of 0.</span>


*These operations name a register <span class="del">location</span>,
<span class="add">not the contents of the register</span>. To
fetch the contents of a register, it is necessary to use
one of the register based addressing operations, such as
`DW_OP_bregx` (Section {memorylocations},
<span class="add">or a register value operation, such as
`DW_OP_regval` (Section {registervalues})</span>.*


## <span class="del">2.6.1.1.1</span> 3.9 Undefined Locations

<span class="del">An empty location description consists of a DWARF expression containing
no operations. It represents a piece or all of an object that is present
in the source but not in the object code (perhaps due to optimization).</span>

<span class="add">An undefined location represents a piece or all of an object that is
present in the source but not in the object code (perhaps due to
optimization).</span>

1. <span class="add">`DW_OP_undefined`</span>  
<span class="add">The `DW_OP_undefined` operation pushes an undefined
location with an offset of 0 onto the stack.</span>

2. <span class="add">A DWARF expression containing no operations or
that leaves no elements on the stack also produces an undefined
location.</span>


## <span class="del">2.6.1.1.4</span> 3.10 Implicit Locations

An implicit location <span class="del">description</span>
represents a piece or all
of an object which has no actual location but whose contents
are nonetheless <span class="del">either known or known to be undefined</span>
<span class="add">known</span>.

The following DWARF operations may be used to specify a value
that has no location in the program but is a known constant
or is computed from other locations and values in the program.

1. `DW_OP_implicit_value`  
    The `DW_OP_implicit_value` operation specifies an immediate value
    using two operands: an unsigned LEB128length, followed by a
    sequence of bytes of the given length that contain the value.
    <span class="add">A location is pushed on the stack for an implicit
    storage bank containing the byte sequence starting with the first byte
    at offset 0 and with an offset of 0.</span>

    
1. `DW_OP_stack_value`  
    The `DW_OP_stack_value`
    operation specifies that the object
    does not exist in memory but its value is nonetheless known
    and is at the top of the DWARF expression stack. <span class="del">In this form
    of location description, the DWARF expression represents the
    actual value of the object, rather than its location.
    The `DW_OP_stack_value` operation terminates the expression.</span>
    <span class="add">A location is pushed on the stack for an implicit storage
    bank containing the value represented using the encoding and byte order of
    the value's type and with an offset of 0.</span>
    
1. `DW_OP_implicit_pointer`  
    *An optimizing compiler may eliminate a pointer, while
    still retaining the value that the pointer addressed.
    `DW_OP_implicit_pointer` allows a producer to describe this value.*
    
    The `DW_OP_implicit_pointer` operation specifies that the object
    is a pointer that cannot be represented as a real pointer,
    even though the value it would point to can be described. In
    this form of location <span class="del">description</span>, the DWARF expression refers
    to a debugging information entry that represents the actual
    value of the object to which the pointer would point. Thus, a
    consumer of the debug information would be able to show the
    value of the dereferenced pointer, even when it cannot show
    the value of the pointer itself.
    
    The `DW_OP_implicit_pointer` operation has two operands: a
    reference to a debugging information entry that describes
    the dereferenced object's value, and a signed number that
    is treated as a byte offset from the start of that value.
    The first operand is a 4-byte unsigned value in the 32-bit
    DWARF format, or an 8-byte unsigned value in the 64-bit
    DWARF format (see Section
    {32bitand64bitdwarfformats})
    that is used as the offset of a debugging information entry
    in the `.debug_info` section of the current executable
    or shared object file.
    The second operand is a
    signed LEB128 number.
    <span class="add">A location is pushed on the stack for an implicit storage
    bank with a size of an address in the default address space and with an
    offset of 0. If the contents of the storage bank are dereferenced, the
    result is the location `L` offset by `B` bytes.</span>
    
    *The debugging information entry referenced by a
    `DW_OP_implicit_pointer` operation is typically a
    `DW_TAG_variable` or `DW_TAG_formal_parameter` entry whose
    `DW_AT_location` attribute gives a second DWARF expression or a
    location list that describes the value of the object, but the
    referenced entry may be any entry that contains a `DW_AT_location`
    or `DW_AT_const_value` attribute (for example, `DW_TAG_dwarf_procedure`).
    By using the second DWARF expression, a consumer can
    reconstruct the value of the object when asked to dereference
    the pointer described by the original DWARF expression
    containing the `DW_OP_implicit_pointer` operation.*

*DWARF location descriptions are intended to yield the **location** of a
value rather than the value itself. An optimizing compiler may perform a
number of code transformations where it becomes impossible to give a
location for a value, but it remains possible to describe the value
itself. Section {registerlocationdescriptions} describes operators that
can be used to describe the location of a value when that value exists
in a register but not in memory. The operations in this section are used
to describe values that exist neither in memory nor in a single
register.*


## <span class="del">2.6.1.2</span> 3.11 Composite Locations

<span class="add">The above kinds of locations are considered "simple" locations.</span>

<span class="del">A composite location description describes an object or
value which may be contained in part of a register or stored
in more than one location. Each piece is described by a
composition operation<span class="del">, which does not compute a value nor
store any result on the DWARF stack</span>. There may be one or
more composition operations in a single composite location
description.</span>

<span class="add">A composite location description describes the location an object or
value which may be contained in zero or more contiguous parts, where
each specifies the location and size of the part. The composite's
storage bank size is the sum of the sizes of the parts. The location of
each of the parts can be any kind of storage bank. For example, each
part could be a piece of a different (or same) register, memory,
implicit or undefined storage bank. A composite location is created by
using one or more composite operations to add each of the pieces.</span>

A series of <span class="del">such</span> <span class="add">piece</span>
operations <span class="add">(`DW_OP_piece` or `DW_OP_bit_piece`)</span>
describes the parts of a value in <span class="del">memory address</span>
<span class="add">storage</span> order.
<span class="add">Each piece operation pops a location `A` from the stack and updates the
composite location `B` in the preceding element of the stack by
appending the new piece described by `A`.</span>

<span class="del">Each composition operation is immediately preceded by a simple
location description which describes the location where part
of the resultant value is contained.</span>

<span class="add">A composite location may be formed from several simple or composite
location parts by the composition operations described in this
section. Each part's location describes the location of one piece of
the object; each composition operation describes which part of the
object is located there.</span>

1. <span class="add">`DW_OP_composite`</span>  
    <span class="add">The `DW_OP_composite` operator has no operands. It pushes a new, empty,
    composite location onto the stack, with an offset of 0.</span>

    <span class="add">*This operator is provided so that a new series of
    piece operations can be started to form a composite location when
    the state of the stack is unknown (e.g., following a `DW_OP_call\*`
    operation), or when a new composite is to be started (e.g., rather
    than add to a previous composite location on the stack).*</span>

1. `DW_OP_piece`  
    The `DW_OP_piece` operation takes a
    single operand, which is an
    unsigned LEB128 number.
    The number describes the size <span class="add">`S`</span>, in bytes,
    of the piece of the object referenced by <span class="del">the preceding simple
    location description</span> <span class="add">the location `A` on the top of the stack</span>.
    If the piece is located in a register,
    but does not occupy the entire register, the placement of
    the piece within that register is defined by the ABI.
    
    *Many compilers store a single variable in sets of registers,
    or store a variable partially in memory and partially in
    registers. `DW_OP_piece` provides a way of describing how large
    a part of a variable a particular DWARF location description
    refers to.*
    
1. `DW_OP_bit_piece`  
    The `DW_OP_bit_piece` operation takes two operands.
    The first is an unsigned LEB128 number that gives the size <span class="add">`S`</span> in bits
    of the piece. The second is an
    unsigned LEB128 number that
    gives the offset in bits from the location defined by
    <span class="del">the preceding DWARF location description</span>
    <span class="add">the location `A` on the top of the stack</span>.
    
    Interpretation of the offset depends on the location <span class="del">description</span>.
    If the location <span class="del">description</span> is
    <span class="del">empty</span> <span class="add">an undefined location</span> (see Section {undefinedlocations}),
    the `DW_OP_bit_piece` operation
    describes a piece consisting of the given number of bits whose values
    are undefined, and the offset is ignored.
    If the location is a memory
    <span class="del">address</span> <span class="add">location</span> (see Section {memorylocationdescriptions}),
    the `DW_OP_bit_piece` operation describes a
    sequence of bits relative to the location whose address is
    on the top of the DWARF stack using the bit numbering and
    direction conventions that are appropriate to the current
    language on the target system.
    In all other cases, the source of the piece is given by either a
    register location (see Section {registerlocationdescriptions}) or an
    implicit value <span class="del">description</span> <span class="add">location</span>
    (see Section {implicitlocationdescriptions});
    the offset is from the least significant bit of the source value.

    *`DW_OP_bit_piece` is
    used instead of `DW_OP_piece` when
    the piece to be assembled into a value or assigned to is not
    byte-sized or is not at the start of a register or addressable
    unit of memory.*
    
    *Whether or not a `DW_OP_piece` operation is equivalent to any
    `DW_OP_bit_piece` operation with an offset of 0 is ABI dependent.*

<span class="del">A composition operation that follows an empty location description indicates
that the piece is undefined, for example because it has been optimized away.</span>

<span class="add">For compatibility with DWARF Version 5 and earlier, the following
additional rules apply to piece operations:</span>

- <span class="add">If a piece operation is processed while the stack is empty, a new
empty composite and an undefined location are pushed implicitly (as if
`DW_OP_composite DW_OP_undefined` had been processed immediately prior
to the piece operation). The result is a composite with a single
undefined piece.</span>

- <span class="add">Otherwise, if the top of the stack `A` is a composite, and is the only
element on the stack (i.e., `B` does not exist), an undefined location
is pushed implicitly (as if `DW_OP_undefined` had been processed
immediately prior to the piece operation), whereupon the composite `A`
becomes `B` and the undefined location is now `A`. The result is the
addition of an undefined piece to the existing composite location.</span>

- <span class="add">Otherwise, if the top of the stack `A` is a location, or convertible
to a location, and the preceding element is not a composite location,
one or more elements below `A` are popped and discarded until the
preceding element `B` is a composite location, or until `A` is the
only element on the stack. If `A` is the only remaining element, a new
empty composite is inserted before it (as if `DW_OP_composite
DW_OP_swap` had been processed immediately prior to the piece
operation), and the result is a new composite location with the single
piece `A`.</span>

  <span class="add">*[This third rule may not in fact be necessary. It covers the case
  where a DWARF5 piece expression left multiple items on the stack.]*</span>

## <span class="add">3.12 Offset Operations [NEW]</span>

<span class="add">In addition to the composite operations, locations
may be modified by the following operations:</span>

1. <span class="add">`DW_OP_offset`</span>  
    <span class="add">`DW_OP_offset` pops two stack entries. The first (top of stack)
    must be an integral type value, which represents a byte
    displacement. The second must be a location. It forms an updated
    location by adding the given byte displacement to the offset
    component of the original location and pushes the updated location
    onto the stack.</span>

2. <span class="add">`DW_OP_bit_offset`</span>  
    <span class="add">`DW_OP_bit_offset` pops two stack entries. The first
    (top of stack) must be an integral type value, which represents a bit
    displacement. The second must be a location. It forms an updated
    location by adding the given bit displacement to the offset
    component of the original location and pushes the updated location
    onto the stack.</span>

    <span class="add">_A bit offset of `n*8` is equivalent to a byte offset of `n`._</span>

<span class="add">The resulting offset must be within range of the location's storage bank.</span>

## <span class="del">2.5.2.5</span> 3.13 Control Flow Operations

The following operations provide simple control of the flow of a DWARF expression.

1. `DW_OP_le`, `DW_OP_ge`, `DW_OP_eq`, `DW_OP_lt`, `DW_OP_gt`, `DW_OP_ne`  
    The six relational operators each:

    -   pop the top two stack values, which have the same type,
        either the same base type or the generic type,
        
    -   compare the operands:  
        `<former second entry>`  `<relational operator>` `<former top entry>`
        
    -   push the constant value 1 onto the stack
        if the result of the operation is true or the
        constant value 0 if the result of the operation is false.
        The pushed value has the generic type.
    
    If the operands have the generic type, the comparisons
    are performed as signed operations.
    
1. `DW_OP_skip`  
    `DW_OP_skip` is an unconditional branch. Its single operand
    is a 2-byte signed integer constant. The 2-byte constant is
    the number of bytes of the DWARF expression to skip forward
    or backward from the current operation, beginning after the
    2-byte constant.
    
1. `DW_OP_bra`  
    `DW_OP_bra` is a conditional branch. Its single operand is a
    2-byte signed integer constant.  This operation pops the
    top of stack. If the value popped is not the constant 0,
    the 2-byte constant operand is the number of bytes of the
    DWARF expression to skip forward or backward from the current
    operation, beginning after the 2-byte constant.
    
1. `DW_OP_call2`, `DW_OP_call4`, `DW_OP_call_ref`  
    `DW_OP_call2`, `DW_OP_call4`, and `DW_OP_call_ref` perform
    DWARF procedure calls during evaluation of a DWARF expression or
    location description.
    For `DW_OP_call2` and `DW_OP_call4`,
    the operand is the 2- or 4-byte unsigned offset, respectively,
    of a debugging information entry in the current compilation
    unit. The `DW_OP_call_ref` operator has a single operand. In the
    32-bit DWARF format,
    the operand is a 4-byte unsigned value;
    in the 64-bit DWARF format, it is an 8-byte unsigned value
    (see Section {32bitand64bitdwarfformats}).
    The operand is used as the offset of a
    debugging information entry in
    the `.debug_info` section of the current executable or shared object file.
        
    *Operand interpretation of
    `DW_OP_call2`, `DW_OP_call4` and `DW_OP_call_ref` is exactly like
    that for `DW_FORM_ref2`, `DW_FORM_ref4` and `DW_FORM_ref_addr`,
    respectively (see Section {attributeencodings}).*
    
    These operations transfer control of DWARF expression evaluation to
    the `DW_AT_location` attribute of the referenced debugging information entry. If
    there is no such attribute, then there is no effect. Execution
    of the DWARF expression of a `DW_AT_location` attribute may
    <span class="del">add to and/or remove from values on</span>
    <span class="add">pop elements from the stack and/or push values or locations onto</span>
    the stack.
    Execution returns
    to the point following the call when the end of the attribute
    is reached. Values <span class="add">and locations</span> on the stack at the time of the call may be
    used as parameters by the called expression, and <span class="del">values</span>
    <span class="add">elements (values or locations)</span> left on
    the stack by the called expression may be used as return values
    by prior agreement between the calling and called expressions.

## <span class="del">2.5.2.6</span> 3.14 Type Conversions

The following operations provide for explicit type conversion.

1. `DW_OP_convert`  
    The `DW_OP_convert` operation pops the top stack entry, converts it to a
    different type, then pushes the result. It takes one operand, which is an
    unsigned LEB128 integer that represents the offset of a debugging
    information entry in the current compilation unit, or value 0 which
    represents the generic type. If the operand is non-zero, the
    referenced entry must be a `DW_TAG_base_type` entry that provides the type
    to which the value is converted.
    
1. `DW_OP_reinterpret`  
    The `DW_OP_reinterpret` operation pops the top stack entry, reinterprets
    the bits in its value as a value of a different type, then pushes the
    result. It takes one operand, which is an unsigned LEB128 integer that
    represents the offset of a debugging information entry in the current
    compilation unit, or value 0 which represents the generic type.
    If the operand is non-zero, the referenced entry must be a
    `DW_TAG_base_type` entry that provides the type to which the value is converted.
    The type of the operand and result type must have the same size in bits.

## <span class="del">2.5.2.7</span> 3.15 Special Operations

There are these special operations currently defined:

1. `DW_OP_nop`  
    The `DW_OP_nop` operation is a place holder. It has no effect
    on the location stack or any of its values.
    
1. `DW_OP_entry_value`  
    The `DW_OP_entry_value` operation pushes
    the value that
    an expression would have had, or a register location would have held,
    upon entering the current subprogram.  It has two operands: an
    unsigned LEB128 length, followed by
    a block containing a DWARF expression or a register location description
    (see Section {registerlocationdescriptions}).
    The length operand specifies the length in bytes of the block.
    If the block contains a DWARF expression,
    the DWARF expression is evaluated as if it had been evaluated upon entering
    the current subprogram.  The DWARF expression
    assumes no values are present on the DWARF stack initially and results
    in exactly one value being pushed on the DWARF stack when completed.
    If the block contains a register location
    description, `DW_OP_entry_value` pushes the value that register
    held
    upon entering the current subprogram.
    
    `DW_OP_push_object_address` is not meaningful inside of this DWARF operation.
    
    *The register location description provides a more compact form for the
    case where the value was in a register on entry to the subprogram.*
    
    *The values needed to evaluate `DW_OP_entry_value` could be obtained in
    several ways. The consumer could suspend execution on entry to the
    subprogram, record values needed by `DW_OP_entry_value` expressions within
    the subprogram, and then continue; when evaluating `DW_OP_entry_value`,
    the consumer would use these recorded values rather than the current
    values.  Or, when evaluating `DW_OP_entry_value`, the consumer could
    virtually unwind using the Call Frame Information
    (see Section {callframeinformation})
    to recover register values that might have been clobbered since the
    subprogram entry point.*
    
1. `DW_OP_extended`  
    The `DW_OP_extended` opcode encodes an extension operation. It has
    at least one operand: a ULEB128 constant identifying the extension operation.
    The remaining operands are defined by the extension opcode, which are named
    using a prefix of `DW_OP_`.
    The extension opcode 0 is reserved.
    
1. `DW_OP_user_extended`  
    The `DW_OP_user_extended` opcode encodes a
    producer
    extension operation.
    It has at least one operand: a ULEB128 constant identifying a
    producer
    extension operation. The remaining operands are defined by the
    producer
    extension. The
    producer
    extension opcode 0 is reserved and cannot be used by any
    producer
    extension.
    
    *The `DW_OP_user_extended` encoding space can be understood to supplement
    the space defined by `DW_OP_lo_user` and `DW_OP_hi_user` that is allocated by
    the standard for the same purpose.*

## <span class="del">2.5.2</span> 3.16 Value Lists

Value lists are used in place of
DWARF expressions whenever the value of an object's attribute
can change during the lifetime of that object.

Value lists are contained in a separate object file section,
along with location lists (see {locationlists}).

A value list is indicated by an attribute whose value is of class
`vallist` (see Section {classesandforms}).

A value list consists of a series of value list entries.
The representation of a value list is the same as for a
location list (see {locationlists}), except
that bounded location description and default location description
entries are understood to provide DWARF expressions that produce
values rather than location descriptions.

<span class="del">*The DWARF expressions in value list entries, being
expressions and not location descriptions, may not contain
any of the DWARF operations described in Section
{locationdescriptions}.*</span>

The address ranges defined by the bounded expressions of a
value list may overlap. When they do, the meaning is undefined
if the overlapping expressions do not produce the same value.

## <span class="del">2.6.2</span> 3.17 Location Lists

Location lists are used <span class="del">in place of</span>
<span class="add">as</span> location descriptions whenever
the object whose location is being described can change location
during its lifetime. 
Location lists are contained in a separate
object file section called `.debug_loclists` or `.debug_loclists.dwo`
(for split DWARF object files).

A location list is indicated by <span class="del">a location or other</span>
<span class="add">an</span> attribute whose value is of class `loclist`
(see Section {classesandforms}).

*This location list representation, the `loclist` class, and the
related `DW_AT_loclists_base` attribute are new in DWARF Version 5.
Together they eliminate most or all of the object language relocations
previously needed for location lists.*

A location list consists of a series of location list entries.
Each location list entry is one of the following kinds:

-   Bounded location description.
    
    This kind of entry provides a
    location description that specifies the location of
    an object that is valid over a lifetime bounded
    by a starting and ending address. The starting address is the
    lowest address of the address range over which the location
    is valid. The ending address is the address of the first
    location past the highest address of the address range.
    When the current PC is within the given range, the location
    description may be used to locate the specified object.
    The location description is valid even if the address range
    includes addresses within a prologue or epilogue range.
    
    There are several kinds of bounded location description
    entries which differ in the way that they specify the
    starting and ending addresses.
    
    The address ranges defined by the bounded location descriptions
    of a location list may overlap. When they do, they describe a
    situation in which an object exists simultaneously in more than
    one place. If all of the address ranges in a given location
    list do not collectively cover the entire range over which the
    object in question is defined, and there is no following default
    location description, it is assumed that the object is not
    available for the portion of the range that is not covered.
    
    In the case of a bounded location description where the range is defined
    by a starting address and either an ending address or a length, a
    starting address consisting of the reserved address value (see Section
    {reservedtargetaddress}) indicates a non-existent range,
    which is equivalent to omitting the description.
    
-   Default location description.
    This kind of entry provides a
    location description that specifies the location of
    an object that is valid when no bounded location description
    applies.
    As with simple location descriptions, the lifetime of a default
    location excludes any prologue or epilogue ranges.
    
-   Base address.
    This kind of entry provides an address to be
    used as the base address for beginning and ending address
    offsets given in certain kinds of bounded location description.
    The applicable base address of a bounded location description
    entry is the address specified by the closest preceding base
    address entry in the same location list. If there is no
    preceding base address entry, then the applicable base address
    defaults to the base address of the compilation unit (see
    Section {fullandpartialcompilationunitentries}).
    
    In the case of a compilation unit where all of the machine
    code is contained in a single contiguous section, no base
    address entry is needed.
    
    If the base address is the reserved target address, either explicitly
    or by default, then the range of any bounded location description
    defined relative to that base address is non-existent, which is
    equivalent to omitting the description.
    
-   End-of-list.
    This kind of entry marks the end of the location list.

A location list consists of a sequence of zero or more bounded
location description or base address entries, optionally followed
by a default location entry, and terminated by an end-of-list
entry.

If there is no current PC (see Section 
{dwarfexpressionevaluationcontext}), only the
default location list entry is used.

Each location list entry begins with a single byte identifying
the kind of that entry, followed by zero or more operands depending
on the kind.

In the descriptions that follow, these terms are used for operands:

-   A counted location description operand consists of
    an unsigned ULEB integer giving the length of the location
    description (see Section {singlelocationdescriptions})
    that immediately follows.
    
-   An address index operand is the index of an address
    in the `.debug_addr` section. This index is relative to the
    value of the `DW_AT_addr_base` attribute of the associated
    compilation unit. The address given by this kind
    of operand is not relative to the compilation unit base address.
    
-   A target address operand is an address on the target
    machine. (Its size is the same as used for attribute values of
    class `address`, specifically, `DW_FORM_addr`.)

The following entry kinds are defined for use in both
split or non-split units:

1. `DW_LLE_end_of_list`  
    An end-of-list entry contains no further data.
    
    *A series of this kind of entry may be used for padding or
    alignment purposes.*
    
    
1. `DW_LLE_base_addressx`  
    This is a form of base address entry that has one unsigned
    LEB128 operand. The operand value is an address index (into the
    `.debug_addr` section) that indicates the applicable base address
    used by subsequent `DW_LLE_offset_pair` entries.
    
1. `DW_LLE_startx_endx`  
    This is a form of bounded location description entry
    (see page {bndlocdesc})
    that has two unsigned LEB128 operands. The operand values are
    address indices (into the `.debug_addr` section). These indicate the
    starting and ending addresses, respectively, that define
    the address range for which this location is valid.
    These operands are followed by a counted location description.
    
1. `DW_LLE_startx_length`  
    This is a form of bounded location description entry
    (see page {bndlocdesc})
    that has two unsigned LEB128 operands. The first value is an address index
    (into the `.debug_addr` section)
    that indicates the beginning of the address range over
    which the location is valid.
    The second value is the length of the range.
    These operands are followed by a counted location description.
    
1. `DW_LLE_offset_pair`  
    This is a form of {bounded location description} entry
    (see page {bndlocdesc})
    that has two unsigned LEB128 operands. The values of these
    operands are the starting and ending offsets, respectively,
    relative to the applicable base address, that define the
    address range for which this location is valid.
    These operands are followed by a counted location description.
    
1. `DW_LLE_default_location`  
    The operand is a counted location description which defines
    where an object is located if no prior location description
    is valid.
    
1. `DW_LLE_include_loclistx`  
    This is a form of list inclusion, that has one unsigned LEB128
    operand.  The value is an index into the `.debug_loclists` section,
    interpreted the same way as the operand of `DW_FORM_loclistx` to find
    a target list of entries, which will be regarded as part of the
    current location list, up to the `DW_LLE_end_of_list` entry.

The following kinds of location list entries are defined for
use only in non-split DWARF units:

1. `DW_LLE_base_address`  
    A base address entry has one target address operand.
    This address is used as the base address when interpreting
    offsets in subsequent location list entries of kind
    `DW_LLE_offset_pair`.
    
1. `DW_LLE_start_end`  
    This is a form of bounded location description entry
    (see page {bndlocdesc})
    that has two target address operands. These indicate the
    starting and ending addresses, respectively, that define
    the address range for which the location is valid.
    These operands are followed by a counted location description.
    
1. `DW_LLE_start_length`  
    This is a form of bounded location description entry
    (see page {bndlocdesc})
    that has one target address operand value and an unsigned LEB128
    integer operand value. The address is the beginning address
    of the range over which the location description is valid, and
    the length is the number of bytes in that range.
    These operands are followed by a counted location description.
    
1. `DW_LLE_include_loclist`  
    This is a form of list inclusion, that has one offset operand.  The
    value is an offset into the `.debug_loclists` section, like the
    operand of `DW_FORM_sec_offset`.  The offset identifies the
    first entry of a location list whose entries are to be regarded as part of
    the current location  list, up to the `DW_LLE_end_of_list` entry.

...

# Chapter <span class="del">5</span><span class="add">6</span>: Type Entries

...

## 6.7 Structure, Union, Class and Interface Type Entries

...

### 6.7.6 Data Member Entries

...

For a `DW_AT_data_member_location` attribute there are two cases:

1. If the value is an integer constant, it is the offset in bytes from the beginning
of the containing entity. If the beginning of the containing entity has a
non-zero bit offset then the beginning of the member entry has that same bit
offset as well.

2. Otherwise, the value must be a location description. <span class="del">In this case, the
beginning of the containing entity must be byte aligned.</span> The <span class="del">beginning
address</span> <span class="add">location of the containing entity</span> is pushed on the DWARF stack before the location description is
evaluated; the result of the evaluation is the <span class="del">base address</span> <span class="add">location</span> of the member
entry (see Section 2.5.1 on page 27).

   *The push on the DWARF expression stack of the <span class="del">base address</span> <span class="add">location</span> of the containing
construct is equivalent to execution of the `DW_OP_push_object_address` operation
(see Section <span class="del">2.5.2.3</span><span class="add">3.6</span>); `DW_OP_push_object_address` therefore is not
needed at the beginning of a location description for a data member. The result of the
evaluation is a location<span class="del">—either an address or the name of a register</span>, not an offset to
the member.*

   <span class="del">*A `DW_AT_data_member_location` attribute that has the form of a location
description is not valid for a data member contained in an entity that is not byte
aligned because DWARF operations do not allow for manipulating or computing bit
offsets.*</span>

...

## 6.14 Pointer to Member Type Entries

...

The `DW_AT_use_location` description is used in conjunction with the location
descriptions for a particular object of the given pointer to member type and for a
particular structure or class instance. The `DW_AT_use_location` attribute expects
two values to be pushed onto the DWARF expression stack before the
`DW_AT_use_location` description is evaluated (see Section <span class="del">2.5.1</span><span class="add">3.1</span>). The
first value pushed is the value of the pointer to member object itself. The second
value pushed is the <span class="del">base address</span> <span class="add">location</span> of the entire structure or union instance
containing the member whose <span class="del">address</span> <span class="add">location</span> is being calculated.

...

# Chapter <span class="del">6</span><span class="add">7</span>: Other Debugging Information

...

## 7.4 Call Frame Information

...

### 7.4.1 Structure of Call Frame Information

...

The register rules are:

...

> expression(E)  
> The previous value of this register is located at
> the <span class="del">address</span> <span class="add">location</span> produced by executing the DWARF
> expression E (see <span class="del">Section 2.5</span> <span class="add">Chapter 3</span>).
