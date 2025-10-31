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

## Proposal

### Add a new Appendix G

Add a new "Appendix G" Called "DWARF Expressions Semantics
(Informative)" after "Appendix F" moving the subsequent appendices
further down the alphabet.

The exact behavior of some DWARF expression operators is not precisely
defined in the text of standard. Instead, an informal description of
the behavior is given. This could lead to differing interpretations by
different tool developers. As an aid to tool authors, a simple DWARF
expression evaluator has been implemented in OCaml a functional
programming language, so that producers and consumers can compare
their implementation to an objective standard of how the authors
thought that expressions would be processed.

At the time that the standard was published. The this code was tagged
"dwarf-6.0". However, as behavioral semantic questions must be
resolved, this code base will be kept up to date and hosted at: TBD.

To experiment with the semantics of these new DWARF operations and see
how the authors intended them to work you can go to:

https://slinder1.github.io/dwarf-locstack/

This is a JavaScript web page uses a translation of the OCaml formal
specification of the operators to demonstrate how the operators should
function.
