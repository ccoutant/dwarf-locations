# Host a web playground to evaluate DWARF expressions

## Problem Description

With DWARF expressions becoming considerably more complicated with the
structures needed to support GPUs and with the "locations on the
stack" change to DWARF expression evaluation, several members of the
DWARF for GPUs working group felt that it would be helpful to have an
executable, lightweight DWARF expression evaluator. This would allow
both producers and consumers to experiment with DWARF expressions and
thus expedite their learning process. It would also help stakeholders
to share ideas for new operators by defining their semantics in the
evaluator.

To this aim, a
[DWARF expression evaluator](https://github.com/intel/dwarf-evaluator)
implemented in the OCaml functional programming language has been
developed. The evaluator's focus at the current phase is to show how
locations and values are operated on the same evaluation stack and how
locations, in particular composites, can be built. Several examples
accompany the implementation. For the purposes of being concise, the
evaluator at the moment does not define all DWARF operators. We
envision that the implementation would be extended with new cases and
examples as a community effort.

A web playground, generated from the tool, is available at

  https://intel.github.io/dwarf-evaluator/

Here, users can run experiments and see the progression of the DWARF
evaluation stack.  For example, the `DW_OP_pick` example from DWARF
spec D.1.2 is available [here](
https://intel.github.io/dwarf-evaluator/?context=%28%29&input=DW_OP_const4s+1000%0ADW_OP_lit29%0ADW_OP_lit17%0ADW_OP_pick+2).

The composite location example, again from D.1.3, is
[here](https://intel.github.io/dwarf-evaluator/?context=%28%28TargetReg+3+%220123%22%29%0A+%28TargetReg+10+%22abcd%22%29%29&input=DW_OP_composite%0ADW_OP_reg3%0ADW_OP_piece+4%0ADW_OP_reg10%0ADW_OP_piece+2).
The result of evaluation precisely shows, as written in D.1.3, "A
variable whose first four bytes reside in register 3 and whose next
two bytes reside in register 10."

Yet another, and slightly more complicated, composite location example
from D.1.3 is
[here](https://intel.github.io/dwarf-evaluator/?context=%28%0A%28TargetReg+3+%22%5C005%5C000%5C000%5C000%22%29%0A%28TargetReg+4+%22%5C002%5C000%5C000%5C000%22%29%0A%29&input=DW_OP_composite%0ADW_OP_lit1%0ADW_OP_stack_value%0ADW_OP_piece+4%0ADW_OP_breg3+0%0ADW_OP_breg4+0%0ADW_OP_plus%0ADW_OP_stack_value%0ADW_OP_piece+4%0A).
The result again precisely shows the textual explanation from the
spec, that is "the 4 byte value 1 followed by the four byte value
computed from the sum of the contents of r3 and r4."

In [Issue 251120.1](https://dwarfstd.org/issues/251120.1.html) "DWARF
Operation to Create Runtime Overlay Composite Location Description",
there are several examples for which a playground link is given
(search for "Try it!" links).

In [Issue 260227.1](https://dwarfstd.org/issues/260227.1.html)
"Multi-Location Storage", there also exist several examples for which
playground links are given.

The web playground is generated from an evaluator implemented in
OCaml. The tool and the playground was
[presented at FOSDEM 2026](https://fosdem.org/2026/schedule/event/99DLGQ-dwarf-evaluator/).
Those who are interested in seeing more details are invited
to watch the recording of the FOSDEM talk.

## Proposal

We propose that the playground is hosted at DWARF Debugging Standard
Website (<https://dwarfstd.org/>).
