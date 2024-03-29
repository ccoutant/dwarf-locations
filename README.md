# DWARF Support for SIMD/SIMT

As part of its project to improve debugging on heterogeneous architectures,
AMD has proposed a number of DWARF changes. The first of these proposals,
to [Allow Location Descriptions on the Stack][amd],
makes some sweeping changes to the [DWARF-5 specification][dwarf5].
The documents here are an attempt to split this into several
more manageable proposals, in a form acceptable to the DWARF committee.

Issues ready to submit to DWARF committee:

* [Locations on the Stack](005-locations-on-stack.md)
* [General Support for Address Spaces](013-generalize-address-spaces.md)

Issues in progress:

* [General Support for Address Spaces](013-generalize-address-spaces.orig.txt) (Original)
* [Generalize Offsetting of Location Descriptions](010-generalize-offsetting.txt)
* [Generalize Creation of Undefined Location Descriptions](011-generalize-undefined.txt)
* [Generalize Creation of Composite Location Descriptions](012-generalize-composite.txt)
* [Support for Vector Base Types](014-vector-base-types.txt)
* [DWARF Operations to Create Vector Composite Location Descriptions](015-vector-composite-location-descriptions.txt)
* [DWARF Operation to Create Runtime Overlay Composite Location Description](016-overlay-composite-location-descriptions.txt)
* [DWARF Operation to Access Call Frame Entry Registers](017-call-frame-entry-registers.txt)
* [Support for Source Languages Mapped to SIMT Hardware](018-simt-hardware.txt)
* [Support for Source Language Optimizations that Result in Concurrent Iteration Execution](020-simd-hardware.txt)
* [Support for Divergent Control Flow of SIMT Hardware](021-divergent-control-flow.txt)
* [Support for Source Language Memory Spaces](022-memory-spaces.txt)

Clarifications and editorial reorganization:

* [Clarifications for Expression Evaluation](001-clarifications-eval.txt)
* [Clarifications for Location Descriptions](002-clarifications-loc.txt)
* [Clarifications for CFI](003-clarifications-cfi.txt)
* [Clarifications for Memory Location Descriptions](004-clarifications-mem.txt)
* [Editorial Reorganization](006-editorial.txt)

Independent issues not part of this series:

* [Correction to `DW_OP_call_ref` and `DW_FORM_ref_addr`](call-ref.txt)
* [Clarifications for `DW_OP_entry_value`](entry-value.txt)

[amd]: https://llvm.org/docs/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack.html#a-2-general-description
[dwarf5]: https://dwarfstd.org/Dwarf5Std.php
