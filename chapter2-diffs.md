# Chapter 2: General Description

## <del>2.5 DWARF Expressions [Now Chapter 3]</del>

## <del>2.6 Location Descriptions [Now Chapter 3]</del>

## <ins>2.5 Values and Locations [NEW]</ins>

<ins>As described in Section 2.2, DWARF attributes may have
different classes of values. In many cases, attribute values
directly provide the numeric value of a property such as
bounds of an array, the size of a type, the value of a
named constant in the program, or a string value such as
the name of a variable or type.
In other cases, they provide the *location* of a data
object, such as the address of a variable or common block
(see Section 4.1), instead of directly providing the value
of the object.</ins>

<ins>The attribute value classes *constant* and *string* are used
to provide static values for properties, but often the
values need to be computed dynamically. The classes
*exprval* and *vallist* indicate that the attribute value is
to be computed by evaluating a DWARF expression (described
in Chapter 3).</ins>

<ins>A DWARF expression of class *exprval* can be used to compute
a value that cannot be given statically, such as the upper
bound of a dynamic array.</ins>

<ins>Sometimes, an attribute value may depend on the program
counter, such as the loop vectorization factor. A value list
(see Section 3.18), encoded using class *vallist*, can be
used in this case. A value list is a list of DWARF
expressions, each associated with a range of program
counters.</ins>

<ins>The class *address* is used to provide a static
memory address for an object located in memory, and the
classes *exprloc* and *loclist* indicate that the location
of an object is to be computed by evaluating a DWARF
expression. In these cases, where the DWARF expression is
expected to produce a location, the expression is called a
“location expression.”</ins>

<ins>*(In DWARF 5, the term “location description” was used to
refer both to a sequence of DWARF operations that evaluates
to a location, and to the result of that evaluation. For
DWARF 6, we use the terms “location expression” for the
expression and “location” for the result of
evaluating the expression.)*</ins>

<ins>The class *exprloc* is used to provide a location
expression that yields a single location. This is
sufficient to describe the location of an object whose
lifetime is either static or the same as the lexical block
that owns it (excluding any prologue or epilogue ranges),
and that does not move during its lifetime.</ins>

<ins>A location list (see Section 3.19), encoded using class
*loclist*, describes objects that have a limited lifetime
or that change their location during their lifetime. A
location list is a list of location expressions, each
associated with a range of program counters.</ins>

<ins>A location list may have overlapping PC ranges, and thus may
yield multiple locations at a given PC.
In this case, the consumer may assume that the object value
stored is the same in all locations, excluding bits of the
object that serve as padding. Non-padding bits that are
undefined (for example, as a result of optimization) must be
undefined in all the locations.</ins>

<ins>*A location list that yields multiple locations can be
used to describe objects that reside in more than one
piece of storage at the same time. An object may have more
than one location as a result of optimization. For
example, a value that is only read may be promoted from
memory to a register for some region of code, but later
code may revert to reading the value from memory as the
register may be used for other purposes.*</ins>

<ins>DWARF can describe the location of program objects in
several kinds of storage, such as a memory address space or
a register. Each kind of storage is treated as a linear
stream of bits of finite size, organized into bytes and/or
words. The ordering of bits uses the bit numbering and
direction conventions that are appropriate to the target
architecture and the ABI.
*For example, on a little-endian architecture, bits are
numbered within bytes and words from least-significant
to most-significant (right-to-left), while on a big-endian
architecture, bits are numbered left-to-right.*</ins>

<ins>A location names a block of storage and a
(non-negative, zero-based) bit offset relative to the start
of that storage. It gives the location of the program
object, which occupies a sequence of contiguous bits
starting at that location. The bit offset of a location must
be less than the size of the named block of storage (for a
zero-length block of storage, the offset must be 0).</ins>

<ins>DWARF can describe locations in six kinds of storage:</ins>

  - <ins>Memory.
    Corresponds to the target architecture memory address
    spaces. There is always a default address space, and the
    target architecture may define additional address
    spaces.
    Each memory storage block is the size of the corresponding
    address space.
    *(The offset can be thought of as a byte or word address
    combined with a bit offset within the byte or word.)*</ins>

  - <ins>Registers.
    Corresponds to the target architecture registers.
    Each register storage block is the size of the corresponding
    register.</ins>

  - <ins>Undefined storage.
    Indicates no value is available, as when a variable has
    been optimized out. The location names a block of
    imaginary storage that cannot be read.
    Writing to undefined storage has no effect.
    The size of an undefined storage block is the same as
    that of the largest memory address space or register.</ins>

  - <ins>Implicit storage.
    Corresponds to fixed values that can only be read, as when
    a variable has been optimized out but can nevertheless be
    rematerialized from program context. The location
    names a block of imaginary storage, whose size is
    the same as the fixed value that it holds.</ins>
    
  - <ins>Implicit pointer storage.
    A special form of implicit storage, created by the
    `DW_OP_implicit_pointer` operator (see Section 3.11).
    The location names a block of imaginary storage that
    holds a secondary location. Its size is the size of the
    pointer that was optimized away, but no
    physical pointer is available.</ins>

  - <ins>Composite storage.
    A hybrid form of storage where different pieces of a
    program object map to different locations, as when a
    field of a structure or a slice of an array is promoted
    to a register while the rest remains in memory.
    Composite storage consists of a (possibly empty) series
    of contiguous pieces, and its size is the sum of the
    sizes of the pieces. The maximum size of a block of
    composite storage is the size of the largest address
    space or register.</ins>

...

---

# Chapter 3: DWARF Expressions

DWARF expressions describe how to compute a value or
specify a location. They are expressed in
terms of DWARF operations that operate on a stack of <del>values</del>
<ins>elements, each of which may be
either a value or a location.</ins>

A DWARF expression is encoded as a stream of operations,
each consisting of an opcode followed by zero or more literal
operands. The number of operands is implied by the opcode.

<del>In addition to the
general operations that are defined here, operations that are
specific to location descriptions are defined in
Section {locationdescriptions}.</del>

<ins>The result of a DWARF expression is the value or location on the top
of the stack after evaluating the operations.</ins>

<ins>Values on the stack are typed, and can
represent a value of any supported base type of the target
machine, or of the generic type, which is an integral type
that has the size of an address in the default address space
on the target machine, and unspecified signedness.</ins>

<del>*The generic type is the same as the unspecified type used for stack
operations defined in DWARF Version 4 and before.*</del>

<ins>An implicit conversion between a location and a value
may happen during the execution of any operation or when
evaluation of the expression is completed. If a location is
expected, but the result is a value of integral type, the
value is implicitly treated as a memory address in the
default address space, and converted to a memory location.
If a value is expected, and the result is an addressable
memory location (i.e., if the offset is a multiple of the
byte or word size) in the default address space, the address
is implicitly converted to a value of the generic
type.</ins>

<del>*Debugging information must provide consumers a way to
find the location of program variables, determine the bounds of dynamic
arrays and strings, and possibly to find the base address of a
subroutine’s stack frame or the return address of a subroutine.
Furthermore, to meet the needs of recent computer architectures and
optimization techniques, debugging information must be able to describe
the location of an object whose location changes over the object’s
lifetime.*</del>


## <del>2.5.1</del> 3.1 DWARF Expression Evaluation Context

DWARF expressions <del>and location descriptions (see Section
{locationdescriptions})</del> are evaluated within a
context provided by the debugger or other DWARF consumer. 
The context includes the following elements:

1. Required result kind
    
    The kind of result required -- either a location or a value -- is 
    determined by the DWARF construct where the expression is found.
    
    *For example, DWARF attributes with `exprval` class
    require a value, and attributes with <del>`locdesc`</del> <ins>`exprloc`</ins> class require 
    a location <del>description</del> (see Section {classesandforms}).*
    
1. Initial stack
    
    In most cases, the DWARF expression stack is empty at
    the start of expression evaluation. In certain
    circumstances, however, one or more
    <del>values</del> <ins>entries</ins> are pushed
    implicitly onto the stack before evaluation of the
    expression starts (e.g., `DW_AT_data_member_location`).
    
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
    
    A current thread is required for the <del>`DW_OP_form_tls_address`</del>
    <ins>`DW_OP_form_tls_location`</ins> operation 
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
    is available, only default <del>location descriptions</del>
    <ins>value or location list entries</ins> may be used.*
    
1. Current object
    
    The current object is a data object described by a data object entry 
    (see Section {dataobjectentries}) that is being inspected. 
    When evaluating expressions that provide attribute values of a data object,
    the containing debugging information entry is the current object. When 
    evaluating expressions that provide attribute values for a type (e.g.,
    `DW_AT_data_location` for a `DW_TAG_member`), the current object is the data
    object entry (if there is one) that referred to the type entry (e.g., 
    via `DW_AT_type`).
    
    A current object is required for the
    <del>`DW_OP_push_object_address`</del>
    <ins>`DW_OP_push_object_location`</ins>
    (see Section {stackoperations}) operation and <ins>is
    implicitly defined</ins> by some attributes (e.g.,
    `DW_AT_data_member_location` and `DW_AT_use_location`) where the
    object's location is provided as part of the initial stack.

*A DWARF expression <del>for a location description</del> may be able to be
evaluated without a thread, call frame, lane, program counter, or architecture
context element. For example, the location of a global variable may be able
to be evaluated without such context, while the location of local variables
in a stack frame cannot be evaluated without additional context.*

## <del>2.5.2 General Operations [REMOVED]</del>

<del>Each general operation represents a postfix operation on
a simple stack machine.
Each element of the stack has a type and a value, and can represent
a value of any supported base type of the target machine.  Instead of
a base type, elements can have a generic type,
which is an integral type that has the
size of an address on the target machine and
unspecified signedness. The value on the top of the stack after “executing” the
DWARF expression is taken to be the result (the address of the object, the
value of the array bound, the length of a dynamic string,
the desired value itself, and so on).</del>

<del>*The generic type is the same as the unspecified type used for stack operations
defined in DWARF Version 4 and before.*</del>

## <del>2.5.2.3</del> 3.2 Stack Operations

The following
operations manipulate the DWARF stack,
<ins>and may
operate on both values and locations.</ins>
Operations that index the stack assume that the top of the stack (most
recently added entry) has index 0.

Each entry on the stack <del>has an associated type</del>
<ins>is either a value (with an associated type) or
a location</ins>.

1. `DW_OP_dup`  
    The `DW_OP_dup` operation duplicates the <del>value (including its
    type identifier)</del>
    <ins>entry</ins> at the top of the stack.
    
1. `DW_OP_drop`  
    The `DW_OP_drop` operation pops the <del>value (including its type
    identifier)</del> <ins>entry</ins> at the top of the stack.
    
1. `DW_OP_pick`  
    The single operand of the `DW_OP_pick` operation provides a
    1-byte index. A copy of the stack entry <del>(including its
    type identifier)</del> with the specified
    index (0 through 255, inclusive) is pushed onto the stack.
    
1. `DW_OP_over`  
    The `DW_OP_over` operation duplicates the entry currently second
    in the stack at the top of the stack.
    This is equivalent to a
    `DW_OP_pick` operation, with index 1.
    
1. `DW_OP_swap`  
    The `DW_OP_swap` operation swaps the top two stack entries.
    The entry at the top of the stack <del>(including its type identifier)</del>
    becomes the second stack entry, and the second entry <del>(including
    its type identifier)</del> becomes the top of the stack.
    
1. `DW_OP_rot`  
    The `DW_OP_rot` operation rotates the first three stack
    entries. The entry at the top of the stack <del>(including its
    type identifier)</del> becomes the third stack entry, the second
    entry <del>(including its type identifier)</del> becomes the top of
    the stack, and the third entry <del>(including its type identifier)</del>
    becomes the second entry.

*Examples illustrating many of these stack operations are
found in Appendix {dwarfstackoperationexamples}.*

## <del>2.5.2.1</del> 3.3 Literal <del>Encodings</del> <ins>and Constant Operations</ins>

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
    size of <del>a machine address</del>
    <ins>the generic type</ins>, is stored.
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
    second operand is a 1-byte unsigned integer that specifies the size of the
    constant value, which is the same as the size of the base type referenced
    by the first operand. The third operand is a
    sequence of bytes of the given size that is
    interpreted as a value of the referenced type.
    
    *While the size of the constant can be inferred from the base type
    definition, it is encoded explicitly into the operation so that the
    operation can be parsed easily without reference to the `.debug_info`
    section.*


## <del>2.5.2.2</del> 3.4 Register Value Operations

The following operations push
<del>a value onto the stack that is either part or all of</del>
the contents of a register <del>or the result of adding the contents of a
register to a given signed offset</del> <ins>onto the stack</ins>.
<del>`DW_OP_regval_type` pushes the contents of a
register together with the given base type.
`DW_OP_regval_bits` pushes the partial contents of
a register together with the generic type.
The other operations
push the result of adding the contents of a register to a given
signed offset together with the generic type.</del>

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

## <del>2.5.2.4</del> 3.5 Arithmetic and Logical Operations

The following provide arithmetic and logical operations.
<ins>The operations in this section take one or two stack operands,
which must be values.</ins>
Operands of an operation with two operands
must have the same type,
either the same base type or the generic type.
The result of the operation which is pushed back has the same type
as the type of the operand(s).

<del>If the type of the operands is the generic type,
except as otherwise specified, the arithmetic operations
perform addressing arithmetic, that is, unsigned arithmetic that is performed
modulo one plus the largest representable address.</del>

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


## <ins>3.6 Context Query Operations</ins>

<ins>The following operations can be used to push a value
or location obtained from the expression evaluation context (see Section 3.1)
onto the stack:</ins>

1. <del>`DW_OP_push_object_address`</del> <ins>`DW_OP_push_object_location`</ins>  
    <del>The `DW_OP_push_object_address` operation pushes the
    address of the object currently being evaluated as part of
    evaluation of a user presented expression (see Section
    {dwarfexpressionevaluationcontext}). This object may correspond to
    an independent variable described by its own debugging information
    entry or it may be a component of an array, structure, or class
    whose address has been dynamically determined by an earlier step
    during user expression evaluation.</del>

    <ins>The `DW_OP_push_object_location` operation pushes the location of the
    current object (see Section 3.1) onto the stack, as part of evaluation of a user
    presented expression.</ins>
    
    <ins>*This object may correspond to an independent
    variable described by its own debugging information entry; or it may be a
    component of an array, structure, or class whose address has been
    dynamically determined by an earlier step during user expression
    evaluation.*</ins>
    
    *This operator provides explicit functionality (especially for
    arrays involving descriptors) that is analogous to the implicit push
    of the base address of a structure prior to evaluation of a
    `DW_AT_data_member_location` to access a data member of a structure. For
    an example, see Appendix D.2.*

    <ins>*In previous versions of DWARF, this operator was named `DW_OP_push_object_address`.
    The old name is still supported in DWARF 6 for compatibility.*</ins>

1. <del>`DW_OP_form_tls_address`</del> <ins>`DW_OP_form_tls_location`</ins>  
    The <del>`DW_OP_form_tls_address`</del>
    <ins>`DW_OP_form_tls_location`</ins>
    operation pops a value from the stack,
    which must have an integral type <del>identifier</del>,
    translates this value into
    <del>an address</del> <ins>a location</ins>
    in the thread-local storage for the current thread
    (see Section 3.1), and pushes the
    <del>address</del> <ins>location</ins>
    onto the stack <del>together with the generic type identifier</del>.
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
    `DW_OP_const<n><x>` operations prior to the
    <del>`DW_OP_form_tls_address`</del>
    <ins>`DW_OP_form_tls_location`</ins>
    operation. Computing the address of the appropriate block can be
    complex (in some cases, the compiler emits a function call to do
    it), and difficult to describe using ordinary DWARF location
    <del>descriptions</del> <ins>expressions</ins>.
    Instead of forcing complex thread-local storage
    calculations into the DWARF expressions, the
    <del>`DW_OP_form_tls_address`</del>
    <ins>`DW_OP_form_tls_location` operation</ins>
    allows the consumer to perform the computation based on the run-time
    environment.*

    <ins>*In previous versions of DWARF, this operator was named `DW_OP_form_tls_address`.
    The old name is still supported in DWARF 6 for compatibility.*</ins>

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
    
1. `DW_OP_push_lane`  
    The `DW_OP_push_lane` operation pushes a lane index value
    of the generic type, which provides the context of the lane in
    which the expression is being evaluated (see 
    Section {dwarfexpressionevaluationcontext} and 
    Section {lowlevelinformation}).

## <del>2.6.1.1.2</del> 3.7 Memory Locations

<ins>A memory location represents the location of a piece or all of an
object or other entity in memory. On architectures that support
multiple address spaces, a memory location identifies
storage associated with the address space.</ins>

<ins>In contexts that expect a location, a value of the generic type
will be implicitly converted to a memory location in the default
address space.</ins>

<ins>The following operations push memory locations onto the stack:</ins>

1. `DW_OP_addr`  
    The `DW_OP_addr` operation has a single operand that encodes
    a machine address and whose size is the size of an address
    on the target machine.
    <ins>The value of this operand is treated as an address
    in the default address space and the corresponding memory location is
    pushed onto the stack.</ins>
    
1. `DW_OP_addrx`  
    The `DW_OP_addrx` operation has a single operand that
    encodes an unsigned LEB128 value,
    which is a zero-based index into the `.debug_addr` section,
    where a machine address is stored.
    This index is relative to the value of the
    `DW_AT_addr_base` attribute of the associated compilation unit.
    <ins>The address obtained is treated as an address in the default address
    space and the corresponding memory location is pushed onto the stack.</ins>
    
1. `DW_OP_fbreg`  
    The `DW_OP_fbreg` operation provides a
    signed LEB128 <ins>byte</ins> offset <ins>`B`</ins>
    from the <del>address</del> <ins>location</ins>
    specified by the location <del>description</del> <ins>expression</ins> in the
    `DW_AT_frame_base` attribute of the current function
    (see Section 3.1).
    <ins>The frame base location, offset by `B` bytes,
    is pushed onto the stack.</ins>
    
    *This is typically a stack pointer register plus or minus some offset.*

1. `DW_OP_breg0`, `DW_OP_breg1`, ..., `DW_OP_breg31`  
    The single operand of the `DW_OP_breg<n>` operations provides a signed
    LEB128 <del>offset from the contents of the specified register</del>
    <ins>byte offset. The contents of the specified register (0–31) are
    treated as a memory address in the default address space. The offset is
    added to the address obtained from the register and the resulting memory
    location is pushed onto the stack.</ins>
    
1. `DW_OP_bregx`  
    <del>The `DW_OP_bregx` operation provides the sum of two values specified
    by its two operands. The first operand is a register number
    which is specified by an unsigned LEB128number. The second operand is a signed LEB128 offset.</del>  
    <ins>The `DW_OP_bregx` operation has two operands. The first
    operand is a register number which is specified by an unsigned LEB128
    number. The second operand is a signed LEB128 byte offset. It is the same as
    `DW_OP_breg<n>` except it uses the register and offset provided by the
    operands.</ins>


## <del>2.6.1.1.3</del> 3.8 Register Locations

<del>A register location description consists of a register name
operation, which represents a piece or all of an object
located in a given register.</del>

*Register location descriptions describe an object
(or a piece of an object) that resides in a register, while
the opcodes listed in Section {registervalues}
are used to describe an object (or a piece of
an object) that is located in memory at an address that is
contained in a register (possibly offset by some constant).
<del>A register location description must stand alone as the entire
description of an object or a piece of an object.</del>*

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
    registers, numbered from 0 through 31, inclusive. <del>The object
    addressed is in register _n_.</del>
    <ins>A location that names the storage associated with the designated register,
    with an offset of 0, is formed and pushed on the stack.</ins>
    
1. `DW_OP_regx`  
    The `DW_OP_regx` operation has a single
    unsigned LEB128 literal
    operand that encodes the name of a register.
    <ins>A location that names the storage associated with the designated register,
    with an offset of 0, is formed and pushed on the stack.</ins>

*These operations name a register <del>location</del>,
<ins>not the contents of the register</ins>. To
fetch the contents of a register, it is necessary to use
one of the register based addressing operations, such as
`DW_OP_bregx` (Section 3.7),
<ins>the register value operation
`DW_OP_regval_type` (Section 3.4),
or a `DW_OP_reg` operation followed by a derefencing operation
(Section 3.13)</ins>.*


## <del>2.6.1.1.1</del> 3.9 Undefined Locations

<del>An empty location description consists of a DWARF expression containing
no operations. It represents a piece or all of an object that is present
in the source but not in the object code (perhaps due to optimization).</del>

<ins>An undefined location represents a piece
or all of an object that is present in the source but not in
the object code (perhaps due to optimization). An undefined
location cannot be read from or written to.</ins>

1. <ins>`DW_OP_undefined`</ins>  
<ins>The `DW_OP_undefined` operation pushes an undefined
location with an offset of 0 onto the stack.</ins>

2. <ins>A DWARF expression containing no operations or
that leaves no elements on the stack also produces an undefined location.
*This is for compatibility with earlier versions of DWARF.*</ins>


## <del>2.6.1.1.4</del> 3.10 Implicit Locations

*DWARF location <del>descriptions</del> <ins>expressions</ins>
are intended to yield the **location** of a
value rather than the value itself. An optimizing compiler may perform a
number of code transformations where it becomes impossible to give a
location for a value, but it remains possible to describe the value
itself. Section 3.8 describes operators that
can be used to describe the location of a value when that value exists
in a register but not in memory. The operations in this section are used
to describe values that exist neither in memory nor in a single
register.*

An implicit location <del>description</del>
represents a piece or all
of an object which has no actual location but whose contents
are nonetheless <del>either known or known to be undefined</del>
<ins>known</ins>.

The following DWARF operations may be used to specify a value
that has no location in the program but is a known constant
or is computed from other locations and values in the program.

1. `DW_OP_implicit_value`  
    The `DW_OP_implicit_value` operation specifies an immediate value
    using two operands: an unsigned LEB128 length, followed by a
    sequence of bytes of the given length that contain the value.
    <ins>A location `L` is formed for a block of implicit
    storage which contains the given byte sequence.
    The offset of `L` is set to 0, and `L` is pushed onto the stack.</ins>
    
1. `DW_OP_stack_value`  
    The `DW_OP_stack_value`
    operation specifies that the object
    does not exist in memory but its value is nonetheless known
    and is at the top of the DWARF expression stack. <del>In this form
    of location description, the DWARF expression represents the
    actual value of the object, rather than its location.
    The `DW_OP_stack_value` operation terminates the expression.</del>
    <ins>The value `V` on top of the stack is popped, and
    a location `L` is formed for a block of implicit storage
    containing the value `V`, represented using the encoding and byte order of
    the value's type.
    The offset of `L` is set to 0 and `L` is pushed onto the stack.</ins>


## <ins>3.11 Implicit Pointer Locations [NEW]</ins>

*An optimizing compiler may eliminate a pointer, while still
retaining the <del>value</del> <ins>object</ins> that the
pointer addressed. `DW_OP_implicit_pointer` allows a
producer to describe <del>this value</del> <ins>the location
(or value) of the latter object</ins>.*

<ins>An implicit pointer location is used to describe</ins>
a pointer <ins>object `P`</ins> that cannot be represented as a
real pointer, even though the <ins>location
or</ins> value <ins>of the object</ins> it would point to can be
described. <del>In this form of location description, the
DWARF expression</del> <ins>It</ins> refers to a debugging
information entry that represents the <del>actual value of the</del>
object <ins>`V`</ins> to which the pointer would
point. Thus, a consumer of the debug information would be
able to show the value of the dereferenced pointer, even
when it cannot show the value of the pointer itself.

The following DWARF operation is used to create an implicit
pointer location:

1. `DW_OP_implicit_pointer`  
    
    <del>The `DW_OP_implicit_pointer` operation has two
    operands: a reference to a debugging information entry
    that describes the dereferenced object's value, and a
    signed number that is treated as a byte offset from the
    start of that value. The first operand is a 4-byte
    unsigned value in the 32-bit DWARF format, or an 8-byte
    unsigned value in the 64-bit DWARF format (see Section
    {32bitand64bitdwarfformats}) that is used as the offset
    of a debugging information entry in the `.debug_info`
    section of the current executable or shared object file.
    The second operand is a signed LEB128 number.</del>

    *<del>The debugging information entry referenced by a
    `DW_OP_implicit_pointer` operation is typically a
    `DW_TAG_variable` or `DW_TAG_formal_parameter` entry whose
    `DW_AT_location` attribute gives a second DWARF expression or a
    location list that describes the value
    of the object, but the
    referenced entry may be any entry that contains a `DW_AT_location`
    or `DW_AT_const_value` attribute (for example, `DW_TAG_dwarf_procedure`).
    By using the second DWARF expression, a consumer can
    reconstruct the value of the object when asked to dereference
    the pointer described by the original DWARF expression
    containing the `DW_OP_implicit_pointer` operation.</del>*

    <ins>The `DW_OP_implicit_pointer` operation has two operands.
    The first operand is a reference to a debugging
    information entry `D`. It is a 4-byte unsigned value in
    the 32-bit DWARF format, or an 8-byte unsigned value in
    the 64-bit DWARF format (see Section
    {32bitand64bitdwarfformats}) that is used as the offset
    of the debugging information entry `D` in the
    `.debug_info` section of the current executable or
    shared object file. The second operand is a signed
    LEB128 number that is treated as a byte offset `B`.</ins>

    <ins>The debugging information entry `D` must contain a
    `DW_AT_location` attribute or a `DW_AT_const_value`
    attribute. In the first case, the `DW_AT_location`
    attribute is evaluated to obtain a location; in the
    second case, an implicit location is formed to hold the
    constant value. The byte offset `B` is added to that
    location and the resulting location is used as the
    location of the object `V`.</ins>

    <ins>A location `L` is formed for a block of implicit
    pointer storage, which describes the location of the
    pointer object `P`. The implicit pointer storage
    represents the location of the object `V`, and has the
    size of the pointer that was optimized away. The offset
    of `L` is set to 0, and `L` is pushed onto the
    stack.</ins>

    *<ins>The debugging information entry `D` is typically a
    `DW_TAG_variable` or `DW_TAG_formal_parameter` entry
    whose `DW_AT_location` attribute evaluates to the
    location of the object `V`, but it may be any entry that
    contains a `DW_AT_location` or `DW_AT_const_value`
    attribute (for example, `DW_TAG_dwarf_procedure`). The
    location of `V` would typically be an implicit value, a
    register, or a composite location (an optimized-out
    pointer to a memory location could be described more
    simply as an implicit value).</ins>*


## <del>2.6.1.2</del> 3.12 Composite Locations

<del>A composite location description describes an object or
value which may be contained in part of a register or stored
in more than one location. Each piece is described by a
composition operation, which does not compute a value nor
store any result on the DWARF stack. There may be one or
more composition operations in a single composite location
description. A series of such operations describes the parts of a value
in memory address order.</del>

<del>Each composition operation is immediately preceded by a simple
location description which describes the location where part
of the resultant value is contained.</del>

<ins>A composite location represents the location of an object or
value that does not exist in a single block of contiguous
storage *(e.g., as the result of compiler optimization where
part of the object is promoted to a register)*. Its storage consists
of a (possibly empty) sequence of pieces, where each piece
maps a fixed range of bits from the object onto a
corresponding range of bits at a new (sub-)location.</ins>

<ins>The pieces of a block of composite storage are contiguous, so that
the size of the composite storage is the sum of the sizes
of the individual pieces, and each piece covers a range of
bits immediately following the previous piece.
The maximum size of a block of composite storage is
the size of the largest address space or register.</ins>

*<ins>Typically, the size of a composite storage is the same as
that of the object it describes. If the composite storage
is smaller than the object, the remaining bits of the object
are treated as undefined.</ins>*

*<ins>In the process of fetching a value from a composite
location, the consumer may need to fetch and assemble bits from
more than one piece.</ins>*

<ins>The following operation is used to form a new, empty, composite location:</ins>

1. <ins>`DW_OP_composite`</ins>  
    <ins>The `DW_OP_composite` operator has no operands. It pushes a new, empty,
    composite location onto the stack, with an offset of 0.</ins>

    <ins>*This operator is provided so that a new series of
    piece operations can be started to form a composite location when
    the state of the stack is unknown (e.g., following a `DW_OP_call*`
    operation), or when a new composite is to be started (e.g., rather
    than add to a previous composite location on the stack).*</ins>

<ins>The following operations are used to build a composite
location in storage order, one piece at a time. Each piece
operation expects a sub-location, `L`, at the top of the
stack, and the composite location under construction, `C`,
in the preceding element on the stack. It pops those two
locations and extends `C` by adding a new piece that maps
the given number of bytes or bits to the sub-location `L`.
The offset of the new location is the same as that of `C`.
The new composite location is pushed onto the stack.</ins>

1. `DW_OP_piece`  
    The `DW_OP_piece` operation takes a
    single operand, which is an
    unsigned LEB128 number.
    The number describes the size, in bytes,
    of the piece of the object
    <del>referenced by the preceding simple location description</del>
    <ins>to be appended to the composite `C`</ins>.

    If <del>the piece is located in a register,</del>
    <ins>the sub-location `L` is a register location,</ins>
    but <ins>the piece</ins> does not occupy the entire register,
    the placement of the piece within that register is defined by the ABI.

    <del>*Many compilers store a single variable in sets of registers,
    or store a variable partially in memory and partially in
    registers. `DW_OP_piece` provides a way of describing how large
    a part of a variable a particular DWARF location description
    refers to.*</del>
    
2. `DW_OP_bit_piece`  
    The `DW_OP_bit_piece` operation takes two operands.
    The first is an unsigned LEB128 number that gives the size, in bits,
    of the piece <ins>to be appended</ins>.
    The second is an unsigned LEB128 number that
    gives the offset in bits
    <del>from the location defined by the preceding DWARF location description</del>
    <ins>to be applied to the sub-location `L`</ins>.
    
    Interpretation of the offset depends on the location <del>description</del>.
    If the location <del>description</del> is
    <del>empty</del> <ins>an undefined location</ins> (see Section {undefinedlocations}),
    the `DW_OP_bit_piece` operation
    describes a piece consisting of the given number of bits whose values
    are undefined, and the offset is ignored.
    If the location is a memory
    <del>address</del> <ins>location</ins> (see Section {memorylocationdescriptions})
    <ins>or a composite location</ins>,
    the `DW_OP_bit_piece` operation describes a
    sequence of bits relative to the location <del>whose address is</del>
    on the top of the DWARF stack using the bit numbering and
    direction conventions that are appropriate to the current
    language on the target system.
    In all other cases, the source of the piece is given by either a
    register location (see Section {registerlocationdescriptions}) or an
    implicit value <del>description</del> <ins>location</ins>
    (see Section {implicitlocationdescriptions});
    the offset is from the least significant bit of the source value.

    *`DW_OP_bit_piece` is
    used instead of `DW_OP_piece` when
    the piece to be assembled into a value or assigned to is not
    byte-sized or is not at the start of a register or addressable
    unit of memory.*
    
    *Whether or not a `DW_OP_piece` operation is equivalent to any
    `DW_OP_bit_piece` operation with an offset of 0 is ABI dependent.*

<del>A composition operation that follows an empty location description indicates
that the piece is undefined, for example because it has been optimized away.</del>

<ins>For compatibility with DWARF Version 5 and earlier, the following
exceptions apply to the piece operations:</ins>

- <ins>If `L` is a non-composite location (or convertible to
one) and is the only element on the stack, the result
is a new composite location with the single piece `L` (as
if `DW_OP_composite DW_OP_swap` had been processed
immediately prior to the piece operation).
*This rule supports the first piece operation in a DWARF 5 expression.*</ins>

- <ins>If a piece operation is processed while the stack is empty, a new
empty composite and an undefined location are pushed implicitly (as if
`DW_OP_composite DW_OP_undefined` had been processed immediately prior
to the piece operation). The result is a composite with a single
undefined piece.
*This rule supports the empty piece operation in DWARF 5 when it is the
first piece of a composite.*</ins>

- <ins>If the top of the stack is a composite location, and is
the only element on the stack, an undefined location is
pushed implicitly (as if `DW_OP_undefined` had been
processed immediately prior to the piece operation),
whereupon the composite on top of the stack is taken as
`C` and the undefined location is `L`. The result is the
addition of an undefined piece to the existing composite
location.
*This rule supports the empty piece operation in DWARF 5 when it is not the
first piece of a composite.*</ins>


## <ins>3.13 Dereferencing Operations [NEW]</ins>

<ins>The following operations are used to dereference a location
on the stack; i.e., to load a value (or another location) stored at that location.
Each operation defines the number of bytes or bits to be loaded.</ins>

<ins>Dereferencing a memory location simply loads a value from memory.
Dereferencing a register location extracts all or some of the bits from the register,
depending on the offset of the location and the size of value to be loaded.
Dereferencing an implicit location loads a value (or, in
the case of an implicit pointer, another location) from
the implicit storage. Attempting to dereference an
undefined location is an error.</ins>

<ins>When dereferencing from a composite location, the value to be loaded
may span more than one of the pieces of the composite, and the
consumer must reassemble the value from the component pieces.</ins>

1. `DW_OP_deref`  
    <del>The `DW_OP_deref` operation pops the top stack entry and
    treats it as an address. The popped value must have an integral type.
    The value retrieved from that address is pushed,
    and has the generic type.
    The size of the data retrieved from the
    dereferenced
    address is the size of an address on the target machine.</del>  

    <ins>The `DW_OP_deref` operation pops a location `L` from the top of the
    stack. The first `S` bits, where `S` is the size (in bits) of an address on the
    target machine, are retrieved from the location `L` and pushed onto the
    stack as a value of the generic type.</ins>

    <del>If the location on the stack is an implicit pointer (see Section 3.11),
    the location `LP` is dereferenced to obtain the location `LV`.
    In this case, the offset of the location `LP` must be 0.</del>

1. `DW_OP_deref_size`  
    <del>The `DW_OP_deref_size` operation behaves like the
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
    target machine before being pushed onto the expression stack.</del>  
    <ins>The `DW_OP_deref_size` takes a single 1-byte unsigned integral operand
    that specifies the size `S`, in bytes, of the value to be retrieved.
    The size `S` must be no larger than the size of the generic type. The
    operation behaves like the `DW_OP_deref` operation: it pops a location
    `L` from the stack. The first `S` bytes are retrieved from the
    location `L`, zero extended to the size of the generic type, and
    pushed onto the stack as a value of the generic type.</ins>
    
1. `DW_OP_deref_type`  
    <del>The `DW_OP_deref_type` operation behaves like the `DW_OP_deref_size` operation:
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
    type of the data pushed.</del>  
    <ins>The `DW_OP_deref_type` operation takes two operands. The first operand
    is a 1-byte unsigned integer that specifies the byte size `S` of the type
    given by the second operand. The second operand is an unsigned LEB128
    integer that represents the offset of a debugging information entry in
    the current compilation unit, which must be a `DW_TAG_base_type` entry
    that provides the type `T` of the value to be retrieved. The size `S`
    must be the same as the byte size of the base type represented by the
    type `T`. This operation pops a location `L` from the stack. The
    first `S` bytes are retrieved from the location `L` and pushed onto the
    stack as a value of type `T`.</ins>


    <del>*While the size of the pushed value could be inferred from the base
    type definition, it is encoded explicitly into the operation so that the
    operation can be parsed easily without reference to the `.debug_info`
    section.*</del>
    
1. `DW_OP_xderef`  
    The `DW_OP_xderef` operation provides an extended dereference
    mechanism. The entry at the top of the stack is treated as an
    address. The second stack entry is treated as an “address
    space identifier” for those architectures that support
    multiple
    address spaces.
    Both of these entries must have integral type<ins>s</ins> <del>identifiers</del>.
    The top two stack elements are popped,
    and a data item is retrieved through an implementation-defined
    address calculation and pushed as the new stack top together with the
    generic type <del>identifier</del>.
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
    Both of these entries must have integral type<ins>s</ins> <del>identifiers</del>.
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
    with the generic type <del>identifier</del>.
    
1. `DW_OP_xderef_type`  
    The `DW_OP_xderef_type` operation behaves like the `DW_OP_xderef_size`
    operation: it pops the top two stack entries, treats them as an address and
    an address space identifier, and pushes the value retrieved. In the
    `DW_OP_xderef_type` operation, the size in bytes of the data retrieved from
    the dereferenced address is specified by the first operand. This operand is
    a 1-byte unsigned integral constant whose value
    <del>value which</del> is the same as the size of the base type referenced
    by the second operand. The second
    operand is an unsigned LEB128 integer that represents the offset of a
    debugging information entry in the current compilation unit, which must be a
    `DW_TAG_base_type` entry that provides the type of the data pushed.


## <ins>3.14 Offset Operations [NEW]</ins>

<ins>In addition to the composite operations, locations
may be modified by the following operations:</ins>

1. <ins>`DW_OP_offset`</ins>  
    <ins>`DW_OP_offset` pops two stack entries. The first (top of stack)
    must be an integral type value, which represents a signed byte
    displacement. The second must be a location. It forms an updated
    location by adding the given byte displacement to the offset
    component of the original location and pushes the updated location
    onto the stack.</ins>

2. <ins>`DW_OP_bit_offset`</ins>  
    <ins>`DW_OP_bit_offset` pops two stack entries. The first
    (top of stack) must be an integral type value, which represents a signed bit
    displacement. The second must be a location. It forms an updated
    location by adding the given bit displacement to the offset
    component of the original location and pushes the updated location
    onto the stack.</ins>

    <ins>*A bit offset of `N × byte_size` is equivalent to a byte offset of `N`.*</ins>

<ins>The resulting offset must remain valid for the location.</ins>

## <del>2.5.2.5</del> 3.15 Control Flow Operations

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
    top of stack. If the value popped is not <del>the constant</del> 0,
    the <del>2-byte</del> constant operand is the number of bytes of the
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
    <del>add to and/or remove from values on</del>
    <ins>pop elements from the stack and/or push values or locations onto</ins>
    the stack.
    Execution returns
    to the point following the call when the end of the attribute
    is reached. Values <ins>and locations</ins> on the stack at the time of the call may be
    used as parameters by the called expression, and <del>values</del>
    <ins>elements (values or locations)</ins> left on
    the stack by the called expression may be used as return values
    by prior agreement between the calling and called expressions.

## <del>2.5.2.6</del> 3.16 Type Conversions

The following operations provide for explicit type conversion.
<ins>The operand on top of the stack must be a value.</ins>

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
    `DW_TAG_base_type` entry that provides the type to which the value is
    <del>converted</del> <ins>reinterpreted</ins>.
    The type of the operand and result type must have the same size in bits.

## <del>2.5.2.7</del> 3.17 Special Operations

There are these special operations currently defined:

1. `DW_OP_nop`  
    The `DW_OP_nop` operation is a place holder. It has no effect
    on the <del>location</del> stack or any of its
    <del>values</del> <ins>elements</ins>.
    
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
    
    <del>`DW_OP_push_object_address`</del>
    <ins>`DW_OP_push_object_location`</ins>
    is not meaningful inside of this DWARF operation.
    
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
    <del>the standard</del>
    <ins>this specification</ins> for the same purpose.*

## <del>2.5.2</del> 3.18 Value Lists

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

<del>*The DWARF expressions in value list entries, being
expressions and not location descriptions, may not contain
any of the DWARF operations described in Section
{locationdescriptions}.*</del>

The address ranges defined by the bounded expressions of a
value list may overlap. When they do, the meaning is undefined
if the overlapping expressions do not produce the same value.

## <del>2.6.2</del> 3.19 Location Lists

Location lists are used in place of
location <del>descriptions</del>
<ins>expressions</ins> whenever
the object whose location is being described can change location
during its lifetime. 
Location lists are contained in a separate
object file section called `.debug_loclists` or `.debug_loclists.dwo`
(for split DWARF object files).

A location list is indicated by <del>a location or other</del>
<ins>an</ins> attribute whose value is of class `loclist`
(see Section {classesandforms}).

<del>*This location list representation, the `loclist` class, and the
related `DW_AT_loclists_base` attribute are new in DWARF Version 5.
Together they eliminate most or all of the object language relocations
previously needed for location lists.*</del>

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

---

# Chapter <del>5</del><ins>6</ins>: Type Entries

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

2. Otherwise, the value must be a location
<del>description</del> <ins>expression</ins>.
<del>In this case, the
beginning of the containing entity must be byte aligned.</del>
The <del>beginning address</del>
<ins>location of the containing entity</ins>
is pushed on the DWARF stack before the location
<del>description</del> <ins>expression</ins> is
evaluated; the result of the evaluation is the
<del>base address</del> <ins>location</ins> of the member
entry (see Section <del>2.5.1 on page 27</del> <ins>3.1</ins>).

   *The push on the DWARF expression stack of the <del>base address</del> <ins>location</ins> of the containing
construct is equivalent to execution of the <del>`DW_OP_push_object_address`</del>
<ins>`DW_OP_push_object_location`</ins> operation
(see Section <del>2.5.2.3</del><ins>3.6</ins>);
<del>`DW_OP_push_object_address`</del>
<ins>`DW_OP_push_object_location`</ins>
therefore is not needed at the beginning of a location
<del>description</del> <ins>expression</ins>
for a data member. The result of the evaluation is a
location<del>—either an address or the name of a register</del>,
not an offset to the member.*

   <del>*A `DW_AT_data_member_location` attribute that has the form of a location
description is not valid for a data member contained in an entity that is not byte
aligned because DWARF operations do not allow for manipulating or computing bit
offsets.*</del>

...

### Section 6.7.8 Member Function Entries

...

An entry for a virtual function also has a
`DW_AT_vtable_elem_location` attribute whose value contains a
location expression yielding the <del>address</del>
<ins>location</ins> of the slot for
the function within the virtual function table for the
enclosing class. The <del>address</del>
<ins>location</ins> of an object of the enclosing
type is pushed onto the expression stack before the location
<del>description</del> <ins>expression</ins>
is evaluated.

...

## 6.14 Pointer to Member Type Entries

...

The pointer to member entry has a `DW_AT_use_location`
attribute whose value is a location <del>description</del>
<ins>expression</ins>
that computes the address of the member of the class to which the
pointer to member entry points.

*The method used to find the address of a given member of a
class or structure is common to any instance of that class
or structure and to any instance of the pointer or member
type. The method is thus associated with the type entry,
rather than with each instance of the type.*

The `DW_AT_use_location` <del>description</del>
<ins>expression</ins> is used in conjunction with the
location <del>descriptions</del> <ins>expressions</ins> for
a particular object of the given pointer to member type and
for a particular structure or class instance. The
`DW_AT_use_location` attribute expects two <del>values</del>
<ins>elements</ins> to be pushed onto the DWARF expression
stack before the `DW_AT_use_location` <del>description</del>
<ins>expression</ins> is evaluated (see Section
<del>2.5.1</del><ins>3.1</ins>). The first <del>value</del>
<ins>element</ins> pushed is the value of the pointer to member
object itself. The second <del>value</del> <ins>element</ins>
pushed is the <del>base address</del> <ins>location</ins> of
the entire structure or union instance containing the member
whose <del>address</del> <ins>location</ins> is being
calculated.

---

# Chapter <del>6</del><ins>7</ins>: Other Debugging Information

...

## 7.4 Call Frame Information

...

### 7.4.1 Structure of Call Frame Information

...

The register rules are:

...

> expression(E)  
> The previous value of this register is located at the
> <del>address</del> <ins>location</ins>
> produced by executing the
> DWARF expression E (see <del>Section
> 2.5</del> <ins>Chapter 3</ins>).

---

# Chapter <del>7</del><ins>8</ins>: Data Representation

...

### 8.5 Format of Debugging Information

...

### 8.5.5 Classes and Forms

...

- <del>`locdesc`</del> <ins>`exprloc`</ins>  
  A DWARF location <del>description</del> <ins>expression</ins>
  (see Section 2.<del>6</del><ins>5</ins>).
  This is represented as an unsigned LEB128 length,
  followed by a byte sequence of the specified length
  (<del>`DW_FORM_locdesc`</del><ins>`DW_FORM_expression`</ins>)
  containing the location expression.
