# Allow Location Descriptions on the Stack

As part of its project to improve debugging on heterogeneous architectures,
AMD has proposed a number of DWARF changes. The first of these proposals,
to [Allow Location Descriptions on the Stack][amd],
makes some sweeping changes to the [DWARF-5 specification][dwarf5].
The documents here are an attempt to split this into several
more manageable proposals, in a form acceptable to the DWARF committee.

* Part 1: [Standardize vector registers](001-vector-registers.txt)
* Part 2: [Clarifications for Expression Evaluation](002-clarifications-eval.txt)
* Part 3: [Clarifications for Location Descriptions](003-clarifications-loc.txt)
* Part 4: [Address Spaces](004-address-spaces.txt)
* Part 5: [Locations on the Stack](005-locations-on-stack.txt)
* Part 6: [Deprecate `DW_OP_entry_value`](006-deprecate-entry-val.txt)
* Part 7: [Editorial Reorganization](007-editorial.txt)

[amd]: https://llvm.org/docs/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack.html#a-2-general-description
[dwarf5]: https://dwarfstd.org/Dwarf5Std.php
