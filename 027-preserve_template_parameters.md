*Preserving Template Parameter Relationships in Nested Instantiations**

**Background**

DWARF does not represent generic template definitions; rather, instead
it represents each specific concrete instantiation of the
template. Currently, when a template is instantiated using a template
parameter from an enclosing class or subprogram, DWARF points the type
of the member directly to the fully resolved concrete
instantiation. Consequently, there is no description of the
relationship between the enclosing template type parameter and the
nested template instantiation. This loss of source-level syntax
prevents debuggers from determining if a nested type was declared
using a generic template parameter or a hardcoded concrete type.

**Proposed Solution**

To resolve this limitation when there is only one instantiation of the
template within the CU, the template instantiation can be expaned
inline. However, when there are many instantiations of the template
embedding the template instantiation inside the class where it is used
could bloat the DWARF. To resolve this problem without bloating the
debug information by duplicating the entire structural definition of
the nested template, we propose using `DW_TAG_template_alias` to
represent dependent nested template instantiations. By artificially
treating the dependent nested instantiation as a template alias, the
alias entry can point directly to the underlying top-level concrete
instantiation, while owning child entries that describe its own
template actual parameters.

In both cases this allows the nested template's
`DW_TAG_template_type_parameter` to bind directly to the enclosing
`DW_TAG_template_type_parameter`, preserving the source-level
syntax.

**Proposed Changes**

In Section 2.22 Template Parameters, insert the following text after
the sentence "A template type parameter entry has a `DW_AT_type`
attribute describing the actual type by which the formal is replaced":

>    To preserve the relationship between an enclosing template's
>    parameters and a nested template instantiation, if the the nested
>    template instantiation is only used internally to the
>    instantiation of the enclosing template or subprogram and its
>    actual parameter is a template parameter of the enclosing
>    template, then the template can be instantiated within the
>    enclosing template. However, if the template instantiation also
>    exists outside of the enclosing template instantiation, then the
>    nested template instantiation may be represented using a
>    `DW_TAG_template_alias` entry. The alias entry refers to the
>    concrete instantiation of the nested template, In both cases the
>    `DW_AT_type` attribute of the nested template's
>    `DW_TAG_template_type_parameter` entry references the
>    corresponding `DW_TAG_template_type_parameter` entry of the
>    enclosing template.

In Section 4.3.7 Function Template Instantiations, insert a new bullet
point to the list of exceptions under Section 4.3.7:

>    4. If a function template is instantiated within the scope of an
>    enclosing template instantiation and its actual parameter is a
>    template parameter of the enclosing template, the nested template
>    instantiation may be represented as a nested template type
>    instantiation or as a `DW_TAG_template_alias` entry that
>    references the concrete instantiation, as described in Section
>    2.22.

In Section 6.7.10 Class Template Instantiations, insert a new bullet
point to the list of exceptions under Section 6.7.10 cross-referencing
the changes:

>    4. If a class template is instantiated within the scope of
>    another template instantiation and its actual parameter is a
>    template parameter of the enclosing template, the nested template
>    instantiation may be represented as nested template type
>    instantiation or as a `DW_TAG_template_alias` entry that
>    references the concrete instantiation, as described in Section
>    2.22.

In Section 6.17 Template Alias Entries, Insert the following
non-normative paragraph after the first non-normative paragraph in
Section 6.17:

>    *`DW_TAG_template_alias` may also be used by producers to
>    describe dependent nested template instantiations. When a nested
>    template is instantiated using a template parameter from an
>    enclosing class or subprogram, representing the nested
>    instantiation as a template alias allows the producer to preserve
>    the source-level linkage to the enclosing template parameters
>    without structurally duplicating the entire concrete
>    instantiation of the nested type.*

In Appendix D.11 Template Examples

Insert a new example demonstrating nested template instantiation
inside an enclosing class between the first example and the second
example.

Nested Class Template Instantiation Example figure TBD

```
// C++ source //
template<typename U> struct t1 {
  U n1;
};

template\<typename T\> struct t2 {
  T m1;
  t1\<T\> m2;
};

int main() {
  t1\<int\> v0;
  t2\<int\> v1;
  return 0;
}
```

DWARF description figure TBD

```
! DWARF description !
1$:  DW_TAG_base_type
       DW_AT_name("int")

! Top-level instantiation of t1<int> (forced by variable v0) !
10$: DW_TAG_structure_type
       DW_AT_name("t1")
11$:   DW_TAG_template_type_parameter
         DW_AT_name("U")
         DW_AT_type(reference to 1$)         ! points to "int"
12$:   DW_TAG_member
         DW_AT_name("n1")
         DW_AT_type(reference to 11$)        ! points to parameter "U"

! Instantiation of t2<int> (forced by variable v1) !
20$: DW_TAG_structure_type
       DW_AT_name("t2")
21$:   DW_TAG_template_type_parameter
         DW_AT_name("T")
         DW_AT_type(reference to 1$)         ! points to "int"
22$:   DW_TAG_member
         DW_AT_name("m1")
         DW_AT_type(reference to 21$)        ! points to parameter "T"

! Internal representation of t1<T> using a template alias !
23$:   DW_TAG_template_alias
         DW_AT_name("t1")
         DW_AT_type(reference to 10$)        ! points to the top-level t1<int>
24$:     DW_TAG_template_type_parameter
           DW_AT_name("U")
           DW_AT_type(reference to 21$)      ! points to enclosing parameter "T"

25$:   DW_TAG_member
         DW_AT_name("m2")
         DW_AT_type(reference to 23$)        ! points to the template alias

! The main function and variables v0 and v1 !
30$: DW_TAG_subprogram
       DW_AT_name("main")
31$:   DW_TAG_variable
         DW_AT_name("v0")
         DW_AT_type(reference to 10$)        ! points to top-level t1<int>
32$:   DW_TAG_variable
         DW_AT_name("v1")
         DW_AT_type(reference to 20$)        ! points to t2<int>
```

Update the existing `consume(wrapper<U> formal)` example and its
explanatory text to demonstrate the new internal representation.

Replace the sentence:

>    There exist situations where it is not possible for the DWARF to
>    imply anything about the nature of the original template.

with:

>    Previously in DWARF5 there were situations where it was not
>    possible for the DWARF to imply anything about the nature of the
>    original template. However, DWARF6 now explicitly states that the
>    concrete type for a template instantiation should be reached
>    indirectly through an enclosing template's type parameter.

Replace Figure D.61 and its accompanying text with the following:

```
! DWARF description !
11$: DW_TAG_structure_type
       DW_AT_name("wrapper")
12$:   DW_TAG_template_type_parameter
         DW_AT_name("T")
         DW_AT_type(reference to "int")
13$:   DW_TAG_member
         DW_AT_name("comp")
         DW_AT_type(reference to 12$)
14$: DW_TAG_variable
       DW_AT_name("obj")
       DW_AT_type(reference to 11$)
21$: DW_TAG_subprogram
       DW_AT_name("consume")
22$:   DW_TAG_template_type_parameter
         DW_AT_name("U")
         DW_AT_type(reference to "int")
```

Replace the text following the example with:

>    In the `DW_TAG_subprogram` entry for the instance of `consume`,
>    `U` is described as `int`. To maintain the relationship between
>    the parameter `U` and the instantiation of `wrapper<U>`, a new
>    type entry internal to the instantiation of `consume` is
>    generated at 23$. The template parameter `T` at 24$ correctly
>    references `U` at 22$, preserving the source-level syntax and
>    preventing the type of `formal` from being indistinguishable from
>    a hardcoded `wrapper<int>`.
