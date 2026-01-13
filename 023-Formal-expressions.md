# Add an appendix that gives a DWARF expression evaluator

## Problem Description

With DWARF expressions becoming considerably more complicated with the
structures needed to support GPUs and with the "locations on the
stack" change to DWARF expression evalution, several members of the
DWARF for GPUs working group felt that it would be helpful to have an
executable, lightweight DWARF expression evaluator. This would allow
both producers and consumers to experiment with DWARF expressions and
thus expedite their learning process. It would also help stakeholders
to share ideas for new operators by defining their semantics in the
evaluator.

To this aim, we present a DWARF expression evaluator implemented in
the OCaml functional programming language. The evaluator's focus at
the current phase is to show how locations and values are operated on
the same evaluation stack and how locations, in particular composites,
can be built. Several examples accompany the implementation. For the
purposes of being concise, the evaluator at the moment does not define
all DWARF operators. We envision that the implementation would be
extended with new cases and examples as a community effort.

The evaluator is currently available at

  https://github.com/barisaktemur/dwarf-locstack/blob/main/dwarf_locstack.ml

For example, the `DW_OP_reg1` operator, which produces a location on
the stack is is implemented as

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

    [Loc (Composite [(4, 6, (Reg 10, 0));
                     (0, 4, (Reg 3, 0))],
          0)]

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

The `DW_OP_pick` example above is available [here](
https://barisaktemur.github.io/dwarf-locstack/?context=%28%29&input=DW_OP_const4s+1000%0ADW_OP_lit29%0ADW_OP_lit17%0ADW_OP_pick+2)

The composite location examples, respectively, are [here](
https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%29&input=DW_OP_composite%0ADW_OP_reg3%0ADW_OP_piece+4%0ADW_OP_reg10%0ADW_OP_piece+2)
and [here](
https://barisaktemur.github.io/dwarf-locstack/?context=%28%0A%28TargetReg+3+%22%5C005%5C000%5C000%5C000%22%29%0A%28TargetReg+4+%22%5C002%5C000%5C000%5C000%22%29%0A%29&input=DW_OP_composite%0ADW_OP_lit1%0ADW_OP_stack_value%0ADW_OP_piece+4%0ADW_OP_breg3+0%0ADW_OP_breg4+0%0ADW_OP_plus%0ADW_OP_stack_value%0ADW_OP_piece+4%0A).

We intent to move the repository and the playground from the personal
github space to an official location, if the proposal is accepted.

## Proposal

### Add a new Appendix G

Add a new "Appendix G" Called "DWARF Expression Evaluation
(Informative)" after "Appendix F", moving the subsequent appendices
further down the alphabet.

As an aid to improve tool authors' understanding of DWARF operator
semantics, an executable DWARF expression evaluator has been
implemented in OCaml, a functional programming language.  The
evaluator can help producer and consumer developers in their
understanding of DWARF expressions by running experiments.

The evaluator is available at

  https://<dwarfstd.org or sourceware.org or a similar URL>

An interactive playground is also provided at

  https://playground-url

At the time the standard is published, the code is tagged
"dwarf-6.0". However, as behavioral semantic questions must be
resolved, the code base will be kept up to date.
