# Fix CarpetX C++ parameter loads and CAHD prefactors

With CAHD enabled, CarpetX currently emits an incomplete scalar expression (`C_CAHD * ;`) and empty operands in the SIMD expression. Emit `C_CAHD * CFL_FACTOR__ignore_repeats_Carpet_timeref_factors * dsmin` in both paths, using the minimum axis spacing, and register the CFL parameter that the generated code reads. Preserve the existing CAHD normalization and coefficient default.

Cast the common SIMD and CAHD parameter lookups to `const CCTK_REAL *`, as required by the `const void *` return type in the [Cactus parameter API](https://www.einsteintoolkit.org/referencemanual/ReferenceManual.html#CCTK_ParameterGet). Add an inexpensive parameter-registration assertion to the defining module's existing validation loop and correct the affected documentation.

Validation: compilation of the actual scalar preamble fails on the parent. After the fix, the scalar and SIMD preambles compile with the real API return type, both with and without CAKO, and give the expected coefficient for unequal axis spacings. The required module static/owner checks, existing trusted-expression comparisons, KB lint, and whitespace checks pass, with no Pylint score regression. Full CarpetX thorn compilation remains untested because this environment does not contain the CarpetX/Cactus build system.

The existing SSL-specific SIMD preamble defect remains documented separately; these compilation cases keep SSL off.

A runtime comparison for the repaired CAHD path is unavailable: the actual parent does not compile with CAHD enabled, and this environment has no complete CarpetX thorn build. The existing expression normalization is preserved; no performance equivalence is claimed from the scalar/SIMD fixtures.
