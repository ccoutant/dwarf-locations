# Multi-Location Storage

## Introduction

The current DWARF spec explains how location lists may yield multiple
locations, with the following text in Section 2.5 "Values and
Locations":

> A location list may have overlapping PC ranges, and thus may yield
> multiple locations at a given PC. In this case, the consumer may
> assume that the object value stored is the same in all locations,
> excluding bits of the object that serve as padding. Non-padding bits
> that are undefined (for example, as a result of optimization) must
> be undefined in all the locations.
>
> *A location list that yields multiple locations can be used to
> describe objects that reside in more than one piece of storage at
> the same time. An object may have more than one location as a result
> of optimization. For example, a value that is only read may be
> promoted from memory to a register for some region of code, but
> later code may revert to reading the value from memory as the
> register may be used for other purposes.*

This was sufficient before the locations on the stack proposal was
merged, as back then how to read from or write to multiple locations
only mattered to the consumer when working with the result of fully
evaluating a `DW_AT_location` expression or location list.  Since
locations on the stack ([Issue
230524.1](https://dwarfstd.org/issues/230524.1.html)) was accepted
into the spec, however, the "multiple location" scenario can now
appear in the middle of an expression, but the spec doesn't fully
specify how that works.

This proposal starts closing the gap, by making multi-locations a new
kind of location storage, called multi-location storage, or
*multiloc*, for short.

This is a simple, yet powerful and fundamental change.  With this,
everywhere in the spec that refers to locations, automatically and
transparently works in the multi-location case too.  For example, a
`DW_OP_deref` on a multi-location location works just like with other
locations, it simply reads bytes from it, using the rules for
multi-location storage.  For an object that happens to be live at
multiple places, `DW_OP_push_object_location` simply pushes a multiloc
location.  Et cetera.

Multi-location storage and composite storage become naturally
recursively composable.  I.e.:

- Composites can contain pieces of any storage kind, including
  multilocs and other composites.
- Multilocs can be formed from locations of any storage kind,
  including composites and other multilocs.

More on this below.

The proposal tweaks the location list description to say that the
overlapping PC entries case creates a multiloc location.  This has no
behavior change.  It is just a refactor.

In addition, this proposal defines a new DWARF expression operator,
`DW_OP_multiloc`, which pops two locations from the stack and pushes a
new multi-location location formed from the two locations.  This
operator can be used as an an alternative to a location list with
overlapping PC entries.

For example, to represent the location of an object that is live in
both memory at address 0xf00 and in register 0, we can simply do:

    DW_OP_addr 0xf00
    DW_OP_reg0
    DW_OP_multiloc

As mentioned before, multilocs and composites are fully composable.
As a simple example, consider the location of a variable of a struct
type with three 32-bit fields, whose default location is in memory at
address X:

              +-----------------------------------------------+
    memory:   |    |     a     |     b     |     c     |      |
              +-----------------------------------------------+
              |0   |X                                         |end-of-memory


If the compiler chooses to promote field `b` to `reg1` for some
section of code, while keeping the memory copy live too, we could
describe this as a composite with a four-byte memory piece for `a`
(1), a four-bytes multiloc piece for `b` (2), and another four-byte
memory piece for `c` (3):

Example 1 - composite with multiloc piece:

                        (1)         (2)         (3)
                   +-----------+-----------+-----------+
                   |           | register  |           |
                   |  memory   +-----------+  memory   |
                   |           |  memory   |           |
                   +-----------+-----------+-----------+

The above could be produced with the following expression, for
example, with address X being 0xf00:

    DW_OP_addr 0xf00
    DW_OP_piece 4        (1 - a memory location piece)
    DW_OP_addr 0xf04
    DW_OP_reg1
    DW_OP_multiloc
    DW_OP_piece 4        (2 - a multiloc piece)
    DW_OP_addr 0xf08
    DW_OP_piece 4        (3 - a memory location piece)

Example 2 - multiloc formed from composite and memory (DW_OP_piece)

Alternatively, and isomorphically in the consumer's perspective, we
could represent the object's location with a multiloc formed from a
composite that contains the promoted register, and a memory location,
like so:

                   +-----------+-----------+-----------+
                   | memory(1) |  reg1(2)  | memory(3) |
                   |-----------+-----------+-----------|
                   |             memory(4)             |
                   +-----------------------------------+

    DW_OP_addr 0xf00
    DW_OP_piece 4        (1 - a memory location piece)
    DW_OP_reg1
    DW_OP_piece 4        (2 - a register piece)
    DW_OP_addr 0xf08
    DW_OP_piece 4        (3 - a memory location piece)
    DW_OP_addr 0xf00     (4 - the base memory location)
    DW_OP_multiloc       (create multiloc from memory (4) and the {1, 2, 3} composite)

Example 3 - multiloc formed from composite and memory (DW_OP_overlay):

The same location can be expressed with the overlay operator
introduced in Issue
[251120.1](https://dwarfstd.org/issues/251120.1.html), "DWARF
Operation to Create Runtime Overlay Composite Location Description",
like so:

    DW_OP_addr 0xf00        (4 - the base memory location)
    DW_OP_dup               (1 and 3 - the base for the overlay)
    DW_OP_reg1              (2 - the overlaying register)
    DW_OP_lit4  # offset 4
    DW_OP_lit4  # size 4
    DW_OP_overlay           (create {1, 2, 3} composite [1])
    DW_OP_multiloc          (create multiloc from memory (4) and the {1, 2, 3} composite)

[1] - not strictly accurate, as with overlay, the composite covers the
      whole base memory storage, but similar enough for the
      illustration.

The locations of examples 1 and 2 are isomorphic, because from the
consumer's perspective, there is no difference -- the rules for
multiloc storage and composite storage compose naturally such that
reads and writes to the two locations produce the same results.

## Nesting

Does:

    multiloc(A, multiloc(B, C))

produce:

 - nested multiloc, or
 - flattened multiloc(A, B, C)?

From a semantic standpoint, it makes no difference, thus is it left as
an implementation choice.  A non-normative note is added to inform
implementors.

## Naming

While "multi-location location" is a mouthful, it was picked because
it is accurate.  Experience across many discussion about the topic
with other people shows that the "multiloc" shorthand is the term most
frequently used in practice, easily understood, and that appears
naturally.

In turn, the `DW_OP_multiloc` operator's name follows the pattern of
most other operators that push new locations.  E.g.:

    DW_OP_regX              => push register location
    DW_OP_undefined         => push undefined location
    DW_OP_composite         => push empty composite location
    DW_OP_implicit_pointer  => push implicit pointer location
    DW_OP_implicit_value    => push implicit value location
    DW_OP_mem               => push memory location (new in upcoming address spaces proposal)

Note: The `DW_OP_addr` operator's name made sense in DWARF 5 and
prior, as it pushed an integer value of generic type on the stack,
representing a machine address.  Only in DWARF 6, with locations on
the stack, did it start pushing a memory location.

## DW_OP_call* and loclist with overlapping PCs

What should happen when `DW_OP_call*` calls a procedure whose
`DW_AT_location`'s class is `loclist`, and the location list has
overlapping PC entries at the current PC?  This is a preexisting
problem, mitigated by the fact that producers in practice don't ever
produce overlapping PC ranges.

Is this even allowed today?  It isn't crystal clear in the current
spec whether it is valid at all for a `DW_OP_call*` operator to
transfer control to a location list.  But for the sake of argument,
let's assume it is.

Section 3.15 "Control Flow Operations", point 4 states, quote:

> Values and locations on the stack at the time of the call may be
> used as parameters by the called expression and entries (values or
> locations) left on the stack by the called expression may be used as
> return values by prior agreement between the calling and called
> expressions.

This implies that the caller and the callee are running on the same
DWARF evaluation stack.  If we have multiple matching PC entries, then
the first location list entry will pop arguments and push its return
values/locations, breaking the evaluation of the second matching
location list entry's expression, which would be expecting to find the
caller's pushed arguments at the top of the stack.

### Approach 1, punt it to producers

One possible approach here is to accept that that scenario doesn't
work, making this a producer problem.  If you want overlapping PC
entries, then as a producer you need to make the expressions in the
loclist entries handle the fact that they all run under the same
stack.

One wrinkle here is that given a loclist with overlapping ranges like
e.g.:

    <DIE $1>:
      DW_AT_location:
        loclist:
          0x100..0x200:  # overlap
            DW_OP_reg0
          0x100..0x200:  # overlap
            DW_OP_reg1
          0x300..0x400:
            DW_OP_addr 0xf00

    <DIE $2>:
      DW_AT_location:
        exprloc:
          DW_OP_call <$1>

When this loclist is evaluated as a result of the `DW_OP_call` in DIE
`$2`, with PC=0x300, then it leaves one location at the top of the
stack.  But when it is evaluated for PC=0x100, then it leaves two
separate locations on the stack.

This can be solved by producers using the new `DW_OP_multiloc`
operator instead of overlapping PC ranges.  E.g.:

    0x100..0x200:
      DW_OP_reg0
      DW_OP_reg1
      DW_OP_multiloc
    0x300..0x400:
      DW_OP_addr 0xf00

### Approach 2, automatic multiloc

Alternatively, we could make DW_OP_call* also automatically create
multiloc, same as when the loclist is evaluated as a top level
`DW_AT_location`.

The following approaches #2.1 to #2.4 make that work, in incremental
levels of support for different scenarios.

### Approach 2.1, push multiloc

If the location list has multiple matching entries, then say that each
matching expression is evaluated with the same stack, and that the
result of each expression is bundled into a multi-location.

E.g. with:

    0x100..0x200:
      DW_OP_reg0
    0x100..0x200:
      DW_OP_reg1

Calling this loclist for PC=0x100 would result in the following
entries at the top of the stack:

    multiloc(reg0, reg1)

### Approach 2.2, push multiple multilocs

Approach 2.1 assumes that expressions can only return one location.
This is not currently required by the spec.  A caller and callee pair
could agree that the procedure returns multiple entries on the stack.
This could be addressed by saying that:

 - all matching location list entries must push the same number of
   entries on the stack.

Then the result of each each expression would be bundled into the same
number of multilocs.  E.g. with:

    0x100..0x200:
      DW_OP_reg0
      DW_OP_reg1
    0x100..0x200:
      DW_OP_reg2
      DW_OP_reg3

Calling this loclist for PC=0x100 would result in the following
entries at the top of the stack:

    multiloc(reg0, reg1)
    multiloc(reg2, reg3)

### Approach 2.3, handle merging values too

However, what about returning values instead of locations?  That is
not handled by approach #2.2, but that is actually the only thing that
DWARF procedures could return on the stack.  That could be handled by
replacing the multiloc creation logic with the following logic:

1. After evaluating all loclist entries, compare the stack entries of
   each evaluation at each level on the stack.

1. If all entries are value entries, they must have the same type and
   value.  The result is a single value entry of that type and value.

1. If all entries are convertible to a location, the result is a
   multiloc location formed from the entries converted to locations.

1. Otherwise, error.

E.g., with:

    0x100..0x200:
      DW_OP_lit1
    0x100..0x200:
      DW_OP_lit1

Calling this loclist for PC=0x100 would result in the following single
new entry at the top of the stack:

    value(value=1, type=generic)

### Approach 2.4, handle arguments too, clone => zip

Approach #2.3 does not address passing arguments to the loclist
expressions, though.  To address this, we could make location list
evaluation pass a fresh copy/clone of the caller's stack to each
location list entry's expression, and then after all matching location
list entries are evaluated (each on its own stack copy), zip/merge the
resulting stacks back into one.  I.e., in pseudo-code:

```
  # Clone current stack, once per matching entry, and evaluate
  # location list expression with that cloned stack.
  stacks = {}
  foreach entry in matching_location_list_entries
    stacks.push(current_stack.clone())
    entry.eval(stacks.back())

  # Require that all stacks have the same size after evaluation.
  for s in stacks
    if s.size() != stacks[0].size()
      error()

  # Zip/merge stacks back into one.
  merged_stack = new_empty_stack()
  for i in 0..stacks[0].size() - 1)
    merged_stack.push(merge_stack_entries_at(stacks, i))

  # Continue evaluation with the new stack contents.
  current_stack = merged_stack
```

With the `merge_stack_entries_at` function implementing the rules of
approach #2.3.

### What to do?

This proposal does not (yet) state a preference, leaving it open for
discussion.  The new multi-location storage concept should be able to
be accepted into the spec without addressing `DW_OP_call*`, which is
the same as silently going for approach #1 for now, preserving the
status quo, but with the possibility of avoiding it, with
`DW_OP_multiloc`.

## Proposal

### Section 2.5 refer to new multiloc section
In Section 2.5 Values and Locations, replace:

> A location list may have overlapping PC ranges, and thus may yield
> multiple locations at a given PC. In this case, the consumer may
> assume that the object value stored is the same in all locations,
> excluding bits of the object that serve as padding. Non-padding bits
> that are undefined (for example, as a result of optimization) must
> be undefined in all the locations.
>
> *A location list that yields multiple locations can be used to
> describe objects that reside in more than one piece of storage at
> the same time. An object may have more than one location as a result
> of optimization. For example, a value that is only read may be
> promoted from memory to a register for some region of code, but
> later code may revert to reading the value from memory as the
> register may be used for other purposes.*

With:

> A location list may have overlapping PC ranges, in which case it
> yields a multiple-location location at a given PC.  This describes
> an object live in more than one piece of storage at the same time.
> See Section 3.XX on page NN.

All the deleted text reappears in a new section, detailed below.

Where it reads:

> DWARF can describe locations in six kinds of storage:

Say that DWARF can describe locations in seven kinds of storage,
instead of six:

> DWARF can describe locations in seven kinds of storage:

Append the following entry to the existing storage kinds list, after
the "Composite storage" entry:

> - Multi-location storage. A hybrid form of storage where a single
>   piece of a program object is live at multiple locations. Its size
>   is the size of the smallest of the storages of the multiple
>   underlying locations. Reading from multi-location storage reads
>   from any of the underlying locations. Writing to multi-location
>   storage writes to all the underlying locations.

Add the following non-normative notes after the storage kind list:

> *Composite storage and multi-location storage are orthogonal and
> composable. Composite storage can contain pieces of any storage
> kind, including multi-location. Multi-location storage can be formed
> from locations of any storage kind, including composite.*

> *A consumer may flatten composite storage composed of composite
> pieces, to avoid nesting in its internal representation.  Whether to
> flatton or not is purely an implementation choice with no visible
> semantic difference.  Similarly, a consumer may flatten
> multi-location storage composed of multi-locations.*

### Section 3.xx [NEW]
Add a new section after Section 3.12, “Composite Locations”, called
"Multi-Location Locations (multiloc)", with new text derived from the
text deleted from Section 2.5:

> A multi-location location (a.k.a. multiloc) is used to describe
> objects that reside in more than one piece of storage at the same
> time. An object may have more than one location as a result of
> optimization. For example, a value that is only read may be promoted
> from memory to a register for some region of code, but later code
> may revert to reading the value from memory as the register may be
> used for other purposes.
>
> The consumer may assume that the object value stored is the same in
> all the underlying locations of a multiloc, excluding bits of the
> object that serve as padding. Non-padding bits that are undefined
> (for example, as a result of optimization) must be undefined in all
> the underlying locations.
>
> A location list yields a multiloc when it has overlapping PC entries
> at a given PC.  Each of the matching location list entries
> contributes one location to the single multiloc.  See 3.19 page NN
> (note to editor: xref to "Location Lists").
>
> The following operation may also be used to form a new multiloc
> location and push it onto the stack:
>
> DW_OP_multiloc
>
> The DW_OP_multiloc operation pops two locations from the stack, and
> pushes a new multiloc location formed from the two popped locations.

### Section 3.19 multiloc cross-reference
In Section 3.19 "Location Lists", where it reads:

> The address ranges defined by the bounded location expressions of a
> location list may overlap.  When they do, they describe a situation in
> which an object exists simultaneously in more than one place.

Add a cross reference to the new "Multiple-Locations Location"
section.  For example:

> The address ranges defined by the bounded location expressions of a
> location list may overlap.  When they do, they describe a situation in
> which an object exists simultaneously in more than one place (see
> Section 3.XX on page NN).
