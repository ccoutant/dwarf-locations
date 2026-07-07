# Move rarely used new operators to exteneded operations encoding range

## Problem description

The single byte operation encoding space is getting tight. At last
count we had about 50 single byte operation encodings available.

When I brought up 250924.2: `DW_OP_mod` doesn’t specify which definition
of modulo my intent was to get clarification not introduce a new
operator. It was decided to add a new operator `DW_OP_mod_floor` and
this new operator was assigned encoding 0xb0 in the current DWARF6
draft. As far as I know there is no compelling consumer demand for
`DW_OP_mod_floor`. However I agree that it could be useful. I do not
think that it will be used very frequently.

## Proposed solution:

Before DWARF6 is released and it becomes permanent, I propose moving
`DW_OP_mod_floor` from the single byte operation encoding space to the
extended operation encoding space. This would free up one more single
byte operation encoding.

Furthermore, currently we do not have any extended operations defined
in DWARF6. This would give DWARF consumers an example case which they
could use to test the new DWARF6 extended operation encoding.

## Proposal

In section 8.7.1 in table 8.9 change the Code for `DW_OP_mod_floor` from
0xb0 to 0xde00

## Alternate proposals:

Assign `DW_OP_mod_floor` 0xde03 if there is some reason we don't want to
begin extended opcodes at 3 like we do with regular operation
encodings.

Or assign `DW_OP_mod_floor` 0xde1d so that `DW_OP_mod_floor` is the
extended version of `DW_OP_mod_trunc` whose operation encoding is
0x1d.
