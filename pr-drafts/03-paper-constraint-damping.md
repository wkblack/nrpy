# Normalize constraint adjustments and expose CAHD, CAMD, and CCD options

Use local spacing divided by the Courant factor for Hamiltonian and momentum adjustments, so their dimensionless strengths retain the same local Courant-step interpretation when the Courant factor changes. Give the existing adjustments consistent Python options and runtime names across the shared equations and their consumers:

| Adjustment | Python option | Runtime coefficient | Default |
| --- | --- | --- | --- |
| Hamiltonian | `enable_CAHD` | `CAHD_C` | `0.015` |
| Momentum gradient | `enable_CAMD` | `CAMD_C` | `0.0` |
| Connection | `enable_CCD` | `CCD_C` | `0.0` |

The phi equation gains `-CAHD_C * dsmin / CFL_FACTOR * H`, with the corresponding W/chi chain rules. Momentum adjustment keeps its existing physical lower-index residual, conformal covariant symmetric trace-free derivative, and W weight, with coefficient `CAMD_C * W * dsmin / CFL_FACTOR`. CCD keeps the existing signed term `-CCD_C * C^i * Dbar_j beta^j`; it attenuates its isolated local factor where the coefficient times the shift divergence is positive.

Both Dendro examples accept `--cahd` / `--no-cahd`, `--camd`, and `--ccd`. CAHD stays enabled by default. Enabling CAMD or CCD compiles the term and registers its coefficient; a nonzero runtime value activates it. For example:

```sh
python3 -m nrpy.examples.dendro_bssn --camd --ccd
```

Then set these root TOML keys for the recommended BSSN strengths:

```toml
CAHD_C = 0.015
CAMD_C = 0.0625
CCD_C = 1.0
```

Propagate the interface through BHaH, Dendro, ETLegacy, CarpetX and affected examples, regenerate the affected trusted expressions, and update the current split wiki pages. For fCCZ4, matching only the CCD divergence coefficient to a BSSN value `a` uses `a + 2/3`; this does not equate the full formulations. The isolated flat-space stability estimates in parameter descriptions are not full-system stability guarantees.

This intentionally changes interfaces and CAHD behavior. Replace `enable_YBS_momentum_constraint_adjustment` / `enable_YBS_Gamma_constraint_adjustment`, `--ybs-momentum` / `--ybs-gamma`, and the old coefficient keys with the names above. For unchanged spacing and Courant factor, `CAMD_C = C_YBS_mom * CFL_FACTOR^2`; the former BHaH/ET coefficient maps as `CAHD_C = C_CAHD * CFL_FACTOR^2`. The old softened Dendro BSSN CAHD formula has no single equivalent coefficient on every level: `0.06` corresponds approximately to `0.015` only on the finest level with negligible softening. Checkpoints with the superseded generated parameter-name list require the matching old executable and cannot restore directly into the renamed one.

Depends on the separate CarpetX prefactor repair; that repair preserves the old normalization, while this PR changes it.

Validation: all 17 edited handwritten Python modules pass the required static/owner checks with no Pylint regression; independent manufactured-field and chain-rule checks pass; all 32 formulation/conformal-variable/enable combinations pass actual parameter/TOML checks; cache switching preserves the baseline. Twenty expression-reference files were regenerated through their owners and checked in fresh processes against the reviewed production expressions. Both full Dendro application suites pass 38 checks with W and chi, and fresh evolution repeats reproduce the updated application references exactly at printed precision. Additional BSSN checks cover nonzero CAMD/CCD, exact restore, individual terms, coefficient-mismatch rejection, production-source equivalence, and compiled-out versus zero controls. KB lint and whitespace checks pass.

Performance: four interleaved parent/candidate pairs on a fixed 736-octant, 160,993-point FD6 mesh, one pinned MPI rank and one thread, gave isolated RHS times of 0.1786 ± 0.0069 versus 0.1735 ± 0.0015 seconds and full RK4-step times of 1.3089 ± 0.0339 versus 1.2887 ± 0.0203 seconds (mean ± run-to-run sample SD, four warmup and four timed steps per run). Paired changes were −2.73 ± 3.37 percentage points for the kernel and −1.51 ± 2.35 for the step. These measurements do not resolve a slowdown or establish a speedup. Compiler flags, Dendrolib source and tensor/unzip settings match. All terms are compiled; CAHD is runtime-zero to match the parent physics, CAMD uses the exact old/new coefficient conversion at CFL 0.25, and CCD is one. This is a small local regression benchmark, separate from production overhead measurements.

Limits: BHaH/ETLegacy/CarpetX have owner/code-generation coverage, with corrected CarpetX preamble compilation, but no complete application/thorn builds in this environment. Nonzero CAMD/CCD evolution coverage is BSSN W; fCCZ4 options also have symbolic/registration coverage and default-CAHD application coverage. Python 3.7 syntax parses; execution uses Python 3.12. No long-time/merger validation is claimed. The retained W-weighted CAMD is the reviewed production formulation; ongoing chi-weight experiments are a separate proposed formulation change.
