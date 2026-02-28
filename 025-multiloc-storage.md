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
everywhere in the spec that refers to locations, including DWARF
expression operations, automatically and transparently works in the
multi-location case too without any special casing.

For example:

- A `DW_OP_deref` on a multiloc location works just like with other
  locations, it simply reads bytes from it, using the rules for multiloc
  storage.

- For an object that happens to be live at multiple places,
  `DW_OP_push_object_location` can now push a multiloc location.

  Before, if the object is live in multiple locations, a consumer
  would have to choose one to push.  This meant that if the expression
  that uses `DW_OP_push_object_location` returns that location or a
  location based on it, e.g.:

    DW_AT_data_location:
      DW_OP_push_object_location
      DW_OP_offset 4

  then the resulting location returned back to the consumer is also a
  single location.  If the debugger user writes to that resulting
  location, it misses updating the other locations where the object is
  also live.

- Et cetera.


The proposal tweaks the location list description to say that the
overlapping PC entries case creates a multiloc location.  This can
mostly be seen as a simple refactor, but it also opens the possibility
of roundtripping the multiple locations to
`DW_OP_push_object_location` as a single multiloc.

In addition, this proposal defines a new DWARF expression operator,
`DW_OP_multiloc`, which pops two locations from the stack and pushes a
new multiloc location formed from the two locations.  This operator
can be used as an an alternative to a location list with overlapping
PC entries.

Example 1 - multiloc with two simple locations

For example, to represent the simple, but typical case of the location
of a scalar 32-bit integer `A` that is live in both memory at address
X=0xf00 and in register 0, like so:

    Location 1 (storage=memory, offset=0xf00):

    +----------------...-+
    |    | A     |       |
    +----------------...-+
    0    |X              |end-of-memory

    Location 2 (storage=register 0, offset=0):

    +-------+
    | A     |
    +-------+
    0       |end-of-register

we can simply do:

    DW_OP_addr 0xf00
    DW_OP_reg0
    DW_OP_multiloc

resulting in the following multiloc location, containing the two
original locations:

    Location 3 (storage=multiloc, offset=0):

    +-------+
    | mem   |
    +-------+
    | reg0  |
    +-------+
    0       |end-of-multiloc

([click to try example on the dwarf-evaluator playground](https://aktemur.github.io/dwarf-evaluator/?context=%28%29&input=DW_OP_addr+0xf00%0ADW_OP_reg0%0ADW_OP_multiloc%0A))

You can only access as many bits are there are available in the
smallest of the underlying locations, so location 3's multiloc storage
has the size of register 0, 32 bits.

Multiloc storage and composite storage become naturally recursively
composable.  I.e.:

- Composites can contain pieces of any storage kind, including
  multilocs and other composites.
- Multilocs can be formed from locations of any storage kind,
  including composites and other multilocs.

As a simple example, consider the location of a variable of a struct
type with three 32-bit fields, whose default location is in memory at
address X:

              +-----------------------------------------------+
    memory:   |    |     a     |     b     |     c     |      |
              +-----------------------------------------------+
              0    |X                                         |end-of-memory


If the compiler chooses to promote field `b` to `reg1` for some
section of code, while keeping the memory copy live too, we could
describe this as a composite with a four-byte memory piece for `a`
(1), a four-bytes multiloc piece for `b` (2), and another four-byte
memory piece for `c` (3):

Example 2 - composite with multiloc piece

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

([click to try example on the dwarf-evaluator playground](https://aktemur.github.io/dwarf-evaluator/?context=%28%29&input=DW_OP_addr+0xf00%0ADW_OP_piece+4%0ADW_OP_addr+0xf04%0ADW_OP_reg1%0ADW_OP_multiloc%0ADW_OP_piece+4%0ADW_OP_addr+0xf08%0ADW_OP_piece+4))

Example 3 - multiloc formed from composite and memory (DW_OP_piece)

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

([click to try example on the dwarf-evaluator playground](https://aktemur.github.io/dwarf-evaluator/?context=%28%29&input=DW_OP_addr+0xf00%0ADW_OP_piece+4%0ADW_OP_reg1%0ADW_OP_piece+4%0ADW_OP_addr+0xf08%0ADW_OP_piece+4%0ADW_OP_addr+0xf00%0ADW_OP_multiloc))

The locations of examples 2 and 3 are isomorphic, because from the
consumer's perspective, there is no difference -- the rules for
multiloc storage and composite storage compose naturally such that
reads and writes to the two locations produce the same results.

Example 4 - multiloc formed from composite and memory (DW_OP_overlay)

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

([click to try example on the dwarf-evaluator playground](https://aktemur.github.io/dwarf-evaluator/?context=%28%28TargetReg+1+%2201234567%22%29%29&input=DW_OP_addr+0xf00%0ADW_OP_dup%0ADW_OP_reg1%0ADW_OP_lit4%0ADW_OP_lit4%0ADW_OP_overlay%0ADW_OP_multiloc))

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

## Comparison with Issue 251017.1 (Compound Storage as a Generalization of Composite Storage and Multiple Locations)

Instead of a new multiloc storage kind, [Issue
251017.1](https://dwarfstd.org/issues/251017.1.html) proposes making
composite storage itself handle multi-location.  This author argues
that that results in a more complicated model unnecessarily, with the
introduction of transparent vs opaque mappings, with terminology that
isn't obviously tied to multi-location even though that's the ultimate
intent.

The need to distinguish transparent vs opaque mappings itself shows
that the generalization in reality does not generalize.  In the end,
we still have a new kind of storage, just pushed down as a new kind of
compound storage mapping.

In the multiloc storage model, there's a cleaner separation of
concerns and terminology that is more to the point.

`DW_OP_copy` in Issue 251017.1 essentially serves the same purpose as
`DW_OP_multiloc` in this proposal, and they both have the same
interface (pop two locations, push a new one).  How they are designed
under the hood, and how they are specified, is different.

This proposal does not introduce an operator equivalent to Issue
251017.1's `DW_OP_overlay_copy`, for the following reasons:

1. In this author's view, `DW_OP_multiloc` in combination with
`DW_OP_overlay` (from
[251120.1](https://dwarfstd.org/issues/251120.1.html), the overlay
proposal) achieves the same effect, at the cost of two extra bytes in
the DWARF expression bytecode (`DW_OP_dup` + `DW_OP_multiloc`).  It is
not clear whether this warrants a new operator.

1. The intent of this proposal is to add the fundamental building
blocks first.  `DW_OP_overlay_copy` has a dependency on
[251120.1](https://dwarfstd.org/issues/251120.1.html) (DW_OP_overlay)
which this proposal currently does not currently have.

If however, there is desire to add `DW_OP_overlay_copy`, the model
introduced by this proposal accomodates it perfectly:
`DW_OP_overlay_copy` can be simply specified as creating a composite
with a multiloc piece, similar to Example 2.

## Impact of Issue 251017.2 (Incremental Location Lists Using Overlays)

The incremental location list proposal essentially changes locations
lists to be just one long DWARF expression, with an "if (PC in range)
... endif" conditional around each LLE.  If that proposal is accepted,
then location lists with overlapping PCs simply no longer
automatically create a multiloc.  That is essentially a one line
change to the spec.  This would not impact the multiloc storage model,
which would still be used with `DW_OP_multiloc` / `DW_OP_copy` /
`DW_OP_overlay_copy`.

## Proposal

### Section 2.5 refer to new multiloc section
In Section 2.5 Values and Locations, replace:

> <del>A location list may have overlapping PC ranges, and thus may yield
> multiple locations at a given PC. In this case, the consumer may
> assume that the object value stored is the same in all locations,
> excluding bits of the object that serve as padding. Non-padding bits
> that are undefined (for example, as a result of optimization) must
> be undefined in all the locations.</del>
>
> <del>*A location list that yields multiple locations can be used to
> describe objects that reside in more than one piece of storage at
> the same time. An object may have more than one location as a result
> of optimization. For example, a value that is only read may be
> promoted from memory to a register for some region of code, but
> later code may revert to reading the value from memory as the
> register may be used for other purposes.*</del>

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

> - Multiloc storage. A hybrid storage form in which a single portion
>   of a program object is simultaneously live in multiple
>   locations. The size of a multiloc storage is the minimum number of
>   contiguous bits accessible in each of its underlying locations,
>   starting at each location's offset. A read from multiloc storage
>   may use any one of the underlying locations. A write to multiloc
>   storage updates all underlying locations.

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

### Section 8.7.1 add new operator
In Section 8.7.1 (DWARF Expressions, Operator Encodings), add the
following row to Table 8.9 (DWARF operation encodings):

- `DW_OP_multiloc` (no operands)
