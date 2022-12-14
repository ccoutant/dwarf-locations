# Allow Location Descriptions on the Stack

As part of its project to improve debugging on heterogeneous architectures,
AMD has proposed a number of DWARF changes. The first of these proposals,
to [Allow Location Descriptions on the Stack][amd],
makes some sweeping changes to the [DWARF-5 specification][dwarf5].
The documents here are an attempt to split this into several
more manageable proposals, in a form acceptable to the DWARF committee.

* Part 1: [Clarifications for Expression Evaluation](002-clarifications-eval.txt)
* Part 2: [Clarifications for Location Descriptions](003-clarifications-loc.txt)
* Part 3: [Clarifications for CFI](004-clarifications-cfi.txt)
* Part 4: [Address Spaces](005-address-spaces.txt)
* Part 5: [Locations on the Stack](006-locations-on-stack.txt)
* Part 6: [Editorial Reorganization](007-editorial.txt)

Independent issues not part of this series:

* [Correction to `DW_OP_call_ref` and `DW_FORM_ref_addr`](call-ref.txt)
* [Clarifications for `DW_OP_entry_value`](entry-value.txt)

[amd]: https://llvm.org/docs/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack/AMDGPUDwarfExtensionAllowLocationDescriptionOnTheDwarfExpressionStack.html#a-2-general-description
[dwarf5]: https://dwarfstd.org/Dwarf5Std.php
