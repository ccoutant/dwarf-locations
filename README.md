# Allow Location Descriptions on the Stack

As part of its project to improve debugging on heterogeneous architectures,
AMD has proposed a number of DWARF changes. The first of these proposals,
to [Allow Location Descriptions on the Stack][amd],
makes some sweeping changes to the [DWARF-5 specification][dwarf5].
The documents here are an attempt to split this into several
more manageable proposals, in a form acceptable to the DWARF committee.

* Part 1: [Clarifications for Expression Evaluation](001-clarifications-eval.txt)
* Part 2: [Clarifications for Location Descriptions](002-clarifications-loc.txt)
* Part 3: [Clarifications for CFI](003-clarifications-cfi.txt)
* Part 4: [Clarifications for Memory Location Descriptions](004-clarifications-mem.txt)
* Part 5: [Locations on the Stack](005-locations-on-stack.txt)
* Part 6: [Editorial Reorganization](006-editorial.txt)
* Part 7: [Generalize Offsetting of Location Descriptions](010-generalize-offsetting.txt)
* Part 8: [Generalize Creation of Undefined Location Descriptions](011-generalize-undefined.txt)
* Part 9: [Generalize Creation of Composite Location Descriptions](012-generalize-composite.txt)

Independent issues not part of this series:

* [Correction to `DW_OP_call_ref` and `DW_FORM_ref_addr`](call-ref.txt)
* [Clarifications for `DW_OP_entry_value`](entry-value.txt)

[amd]: https://llvm.org/docs/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack.html#a-2-general-description
[dwarf5]: https://dwarfstd.org/Dwarf5Std.php
