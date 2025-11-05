# Add an appendix that gives definitions of expressions more precisely

## Problem Description

With DWARF expressions becoming considerably more complicated with the
structures needed to support GPUs, several members of the DWARF for
GPUs working group felt that it would be helpful to define the DWARF
expression operators in a more formal way, which preferably is also
executable. This would allow both producers and consumers to verify
their implementations with respect to a reference
implementation. However adding these formal definitions of operators
to the text of the spec itself is not in keeping with the style of the
existing standard.

It was decided that a functional programming language would be the
best way to define the semantics of the DWARF operators. This would
allow the structure of the mathematics to be represented in a way that
could be executed by a concise evaluator.

The evaluator's focus at the current phase is to show how locations
and values are operated on the same evaluation stack and how
locations, in particular composites, can be built. To this aim,
several examples accompany the implementation. For the purposes of
being concise, the evaluator at the moment does not define all DWARF
operators. We envision that the implementation would be extended with
new cases and examples as a community effort.

The evaluator is currently available at

  https://github.com/barisaktemur/dwarf-locstack/blob/main/dwarf_locstack.ml

For example, an operator that produces a location on the stack is
`DW_OP_reg1` and is implemented as

```
  | DW_OP_reg1  -> Loc(Reg 1, 0)::stack
```

Here, the operator pushes a location on the stack, indicated by
`Loc(...)::stack`, where the storage is `Reg 1`, meaning register #1,
and the offset is 0.

Another example that pushes a location is `DW_OP_addr`:

```
  | DW_OP_addr(a) -> Loc(Mem 0, a)::stack
```

The operator has the operand `a`. It pushes a location where the
storage is `Mem 0` (the memory storage with the default address space)
and the offset is `a`.

The `DW_OP_plus` operator is implemented as

```
  | DW_OP_plus ->
     (match stack with
      | e1::e2::stack' -> Val((as_value e1) + (as_value e2))::stack'
      | _ -> eval_error op stack)
```

Here, the stack is expected to have two elements, `e1` and `e2`, at
the top, with `stack'` being the rest of the stack. This is indicated
by the pattern `e1::e2::stack'`. The elements are expected to be (or
be convertible to) values. This is accomplished by the application of
`as_value` on the elements. The result of the operation is the sum of
the two values followed by the rest of the stack, indicated by
`Val(... + ...)::stack'`.

The `DW_OP_pick` example from DWARF spec D.1.2 can be written in the
evaluator as

    eval_all [DW_OP_pick 2] [Val 17; Val 29; Val 1000] [];;

and evaluates to

    [Val 1000; Val 17; Val 29; Val 1000]

The composite location example, again from D.1.3, can be implemented
as

    eval_all [DW_OP_composite;
              DW_OP_reg3;
              DW_OP_piece 4;
              DW_OP_reg10;
              DW_OP_piece 2] [] [];;

and evaluates to

    [Loc (Composite [(4, 6, (Reg 10, 0)); (0, 4, (Reg 3, 0))], 0)]

This precisely shows, as written in D.1.3, "A variable whose first four
bytes reside in register 3 and whose next two 7 bytes reside in
register 10."

Yet another, and slightly more complicated, composite location example
from D.1.3 is

    eval_all [DW_OP_composite;
              DW_OP_lit1;
              DW_OP_stack_value;
              DW_OP_piece 4;
              DW_OP_breg3 0;
              DW_OP_breg4 0;
              DW_OP_plus;
              DW_OP_stack_value;
              DW_OP_piece 4]
		     []
			 [TargetReg(3, int_to_data 5);
			  TargetReg(4, int_to_data 2)
			 ];;

This evaluates to

    [Loc (Composite [(4, 8, (ImpData "\007\000\000\000", 0));
                     (0, 4, (ImpData "\001\000\000\000", 0))],
          0)
    ]

and again precisely shows the textual explanation "the 4 byte value 1
followed by the four byte value computed from the sum of the contents
of r3 and r4."

A web-based playground is also available at

  https://barisaktemur.github.io/dwarf-locstack/

Here, users can run experiments with a simplified syntax and also see
the progression of the evaluation stack.

The `DW_OP_pick` example above is available at

  https://barisaktemur.github.io/dwarf-locstack/?context=%28%29&input=DW_OP_const4s+1000%0ADW_OP_lit29%0ADW_OP_lit17%0ADW_OP_pick+2

The composite location examples, respectively, are at

  https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%29&input=DW_OP_composite%0ADW_OP_reg3%0ADW_OP_piece+4%0ADW_OP_reg10%0ADW_OP_piece+2

and

  https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%28TargetReg+3+%22%5C005%5C000%5C000%5C000%22%29%0A%28TargetReg+4+%22%5C002%5C000%5C000%5C000%22%29%0A%29&input=DW_OP_composite%0ADW_OP_lit1%0ADW_OP_stack_value%0ADW_OP_piece+4%0ADW_OP_breg3+0%0ADW_OP_breg4+0%0ADW_OP_plus%0ADW_OP_stack_value%0ADW_OP_piece+4%0A

We intent to move the repository and the playground to
git.dwarfstd.org or sourceware.org, if the proposal is accepted.

## Proposal

### Add a new Appendix G

Add a new "Appendix G" Called "DWARF Expressions Semantics
(Informative)" after "Appendix F" moving the subsequent appendices
further down the alphabet.

The exact behavior of some DWARF expression operators is not precisely
defined in the text of standard. Instead, an informal description of
the behavior is given. As an aid to improve tool authors'
understanding, a DWARF expression evaluator has been implemented in
OCaml, a functional programming language, so that producers and
consumers can compare their implementation to an objective standard of
how the authors thought that expressions would be processed.

The evaluator is available at

  https://<dwarfstd.org or sourceware.org or a similar URL>

An interactive playground is also provided at

  https://<a dwarfstd.org or sourceware.org web URL>

At the time the standard is published, the code is tagged
"dwarf-6.0". However, as behavioral semantic questions must be
resolved, the code base will be kept up to date.
