# DWARF Support for SIMD/SIMT

As part of its project to improve debugging on heterogeneous architectures,
AMD has proposed a number of DWARF changes. The first of these proposals,
to [Allow Location Descriptions on the Stack][amd],
makes some sweeping changes to the [DWARF-5 specification][dwarf5].
The documents here are an attempt to split this into several
more manageable proposals, in a form acceptable to the DWARF committee.

Issues approved by the DWARF committee:

* [[001a](001a-context.md)] Expression Evaluation Context (DWARF Issue [241011.1][241011.1])
* [[005](005-locations-on-stack.md)] Locations on the Stack (DWARF Issue [230524.1][230524.1])
* Tensor types (DWARF Issue [230413.1][230413.1])
* Add lane support for SIMD/SIMT machines (DWARF Issue [211206.1][211206.1])

Issues submitted to the DWARF committee:

* [[004](004-clarifications-mem.txt)] Clarifications for Memory Location Descriptions (DWARF Issue [230120.3][230120.3])
* [[002](002-clarifications-loc.txt)] Clarifications for Location Descriptions (DWARF Issue [230120.2][230120.2])
* [[016](016-overlay-composite-location-descriptions.md)] DWARF Operation to Create Runtime Overlay Composite Location Description (DWARF Issue [251120.1][251120.1])

Issues in progress:

* [[013](013-generalize-address-spaces.md)] General Support for Address Spaces ([Original text](013-generalize-address-spaces.orig.txt))
* [[005a](005a-misc.md)] Deferred Issues from Tony’s Review of Locations on Stack
* [[015](015-vector-composite-location-descriptions.txt)] DWARF Operations to Create Vector Composite Location Descriptions
* [[017](017-call-frame-entry-registers.txt)] DWARF Operation to Access Call Frame Entry Registers
* [[020](020-simd-hardware.txt)] Support for Source Language Optimizations that Result in Concurrent Iteration Execution
* [[021](021-divergent-control-flow.txt)] Support for Divergent Control Flow of SIMT Hardware
* [[022](022-memory-spaces.txt)] Support for Source Language Memory Spaces
* [[023](023-Formal-expressions.md)] Add an appendix that gives definitions of expressions more precisely

Clarifications and editorial reorganization:

* [[001](001-clarifications-eval.md)] Clarifications for Expression Evaluation
* [[003](003-clarifications-cfi.txt)] Clarifications for CFI

Issues superseded by others:

* [[006](006-editorial.txt)] Editorial Reorganization (Included in Locations on the Stack)
* [[010](010-generalize-offsetting.txt)] Generalize Offsetting of Location Descriptions (Included in Locations on the Stack)
* [[011](011-generalize-undefined.txt)] Generalize Creation of Undefined Location Descriptions (Included in Locations on the Stack)
* [[012](012-generalize-composite.txt)] Generalize Creation of Composite Location Descriptions (Replaced by DW_OP_composite in Locations on the Stack)

Independent issues not part of this series:

* [Clarifications for `DW_OP_entry_value`](entry-value.txt)

[amd]: https://llvm.org/docs/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack.html#a-2-general-description
[dwarf5]: https://dwarfstd.org/dwarf5std.html
[230524.1]: https://dwarfstd.org/issues/230524.1.html
[230120.2]: https://dwarfstd.org/issues/230120.2.html
[230120.3]: https://dwarfstd.org/issues/230120.3.html
[241011.1]: https://dwarfstd.org/issues/241011.1.html
[211206.1]: https://dwarfstd.org/issues/211206.1.html
[230413.1]: https://dwarfstd.org/issues/230413.1.html
[251120.1]: https://dwarfstd.org/issues/251120.1.html
