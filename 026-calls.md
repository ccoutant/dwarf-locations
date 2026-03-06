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
