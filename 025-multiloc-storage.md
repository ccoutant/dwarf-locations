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
kind of location storage, called *multiloc* storage, short for
multi-location.

This is a simple, yet powerful and fundamental change.  With this,
everywhere in the spec that refers to locations, automatically and
transparently works in the multi-location case too.  For example, a
`DW_OP_deref` on a multiloc location works just like with other
locations, it simply reads bytes from it, using the rules for multiloc
storage.  For an object that happens to be live at multiple places,
`DW_OP_push_object_location` simply pushes a multiloc location.  Et
cetera.

Multiloc storage and composite storage become naturally recursively
composable.  I.e.:

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
new multiloc location formed from the two locations.  This operator
can be used as an an alternative to a location list with overlapping
PC entries.

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

"multiloc" was picked because it is accurate.  It names the storage
for what it is.

In turn, the `DW_OP_multiloc` operator's name follows the pattern of
most other operators that push new locations, of being named by the
storage kind they create.  E.g.:

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

> - Multiloc storage. A hybrid form of storage where a single piece of
>   a program object is live at multiple locations. Its size is the
>   size of the smallest of the storages of the multiple underlying
>   locations. Reading from multiloc storage reads from any of the
>   underlying locations. Writing to multiloc storage writes to all
>   the underlying locations.

Add the following non-normative notes after the storage kind list:

> *Composite storage and multiloc storage are orthogonal and
> composable. Composite storage can contain pieces of any storage
> kind, including multiloc. Multiloc storage can be formed from
> locations of any storage kind, including composite.*

> *A consumer may flatten composite storage composed of composite
> pieces, to avoid nesting in its internal representation.  Whether to
> flatton or not is purely an implementation choice with no visible
> semantic difference.  Similarly, a consumer may flatten multiloc
> storage composed of multiloc locations.*

### Section 3.xx [NEW]
Add a new section after Section 3.12, “Composite Locations”, called
"Multiloc Locations", with new text derived from the text deleted from
Section 2.5:

> A multiloc location is used to describe objects that reside in more
> than one piece of storage at the same time. An object may have more
> than one location as a result of optimization. For example, a value
> that is only read may be promoted from memory to a register for some
> region of code, but later code may revert to reading the value from
> memory as the register may be used for other purposes.
>
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
