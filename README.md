# DWARF Support for SIMD/SIMT

As part of its project to improve debugging on heterogeneous architectures,
AMD has proposed a number of DWARF changes. The first of these proposals,
to [Allow Location Descriptions on the Stack][amd],
makes some sweeping changes to the [DWARF-5 specification][dwarf5].
The documents here are an attempt to split this into several
more manageable proposals, in a form acceptable to the DWARF committee.

Issues ready to submit to DWARF committee:

* [Expression Evaluation Context](001a-context.md) (Issue [241011.1][241011.1])
* [Locations on the Stack](005-locations-on-stack.md) (Issue [230524.1][230524.1])
* [Clarifications for Memory Location Descriptions](004-clarifications-mem.txt) (Issue [230120.3][230120.3])
* [Clarifications for Location Descriptions](002-clarifications-loc.txt) (Issue [230120.2][230120.2])
* [General Support for Address Spaces](013-generalize-address-spaces.md) ([Original text](013-generalize-address-spaces.orig.txt))

Issues in progress:

* [DWARF Operations to Create Vector Composite Location Descriptions](015-vector-composite-location-descriptions.txt)
* [DWARF Operation to Create Runtime Overlay Composite Location Description](016-overlay-composite-location-descriptions.txt)
* [DWARF Operation to Access Call Frame Entry Registers](017-call-frame-entry-registers.txt)
* [Support for Source Language Optimizations that Result in Concurrent Iteration Execution](020-simd-hardware.txt)
* [Support for Divergent Control Flow of SIMT Hardware](021-divergent-control-flow.txt)
* [Support for Source Language Memory Spaces](022-memory-spaces.txt)

Clarifications and editorial reorganization:

* [Clarifications for Expression Evaluation](001-clarifications-eval.txt)
* [Clarifications for CFI](003-clarifications-cfi.txt)
* [Editorial Reorganization](006-editorial.txt)

Issues superseded by others:

* [Generalize Offsetting of Location Descriptions](010-generalize-offsetting.txt) (Included in Locations on the Stack)
* [Generalize Creation of Undefined Location Descriptions](011-generalize-undefined.txt) (Included in Locations on the Stack)
* [Generalize Creation of Composite Location Descriptions](012-generalize-composite.txt) (Replaced by DW_OP_composite in Locations on the Stack)

Independent issues not part of this series:

* [Clarifications for `DW_OP_entry_value`](entry-value.txt)

[amd]: https://llvm.org/docs/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack.html#a-2-general-description
[dwarf5]: https://dwarfstd.org/dwarf5std.html
[230524.1]: https://dwarfstd.org/issues/230524.1.html
[230120.2]: https://dwarfstd.org/issues/230120.2.html
[230120.3]: https://dwarfstd.org/issues/230120.3.html
[241011.1]: https://dwarfstd.org/issues/241011.1.html
