# BSSN Application Wiring

> BSSN equations, CAHD and slow-start-lapse terms, block traversal, native `BSSN_GR` differences, YBS options, and `q1` defaults in `Dendro_NRPy_BSSN`. · Status: provisional
> Up: [Dendro](index.md)

## Summary

`nrpy.examples.dendro_bssn` generates `Dendro_NRPy_BSSN` with code-generation conformal-factor selection (`W` default or `chi`), centered finite differences, slow-start lapse (SSL), spatially constant default `eta = 1`, Dendro-scaled CAHD, Brown's covariant `lambdaU` adjustment, and separate `Ricci_eval_order_N` and `rhs_eval_order_N` kernels. Runtime TOML keys override `eta` and damping strengths, while TwoPunctures parameter rules and gravitational-wave/ADM/VTU outputs are shared with fCCZ4 in [Initial Data, Wave Extraction, And Output Files](initial-data-and-wave-diagnostics.md).

## Detail

### CAHD, Slow-Start Lapse, Initialization, And Block Traversal

Each generated project writes `pars/bssn.toml` (generated defaults) and `pars/q1.par.lowres.toml` (equal-mass TwoPunctures BBH parameters with end time `1000000`, no iteration cap, and native scaling at base cadence `80`, identical across BSSN and fCCZ4 directories; see [q1 versus generated defaults](#q1-versus-generated-defaults), [Runtime Parameter Keys](runtime-parameters.md), and [Production Validation And Deferred Checks](production-validation-and-deferred-checks.md)).

CAHD (`CAHD_C`, default `.015`, `h` the smallest block axis spacing, `CFL` from `BSSN_CFL_FACTOR`, factors from `W = exp(-2 phi)` and `chi = exp(-4 phi)`) and SSL (`BSSN_SSL_H` default `0.6`, `BSSN_SSL_SIGMA` default `20`, RK stage time `t`, `W = cf` or `sqrt(cf)`) add:

```text
W_rhs     += 2 CAHD_C W H h / CFL,
chi_rhs   += 4 CAHD_C chi H h / CFL,
alpha_rhs -= W SSL_h exp(-t^2 / (2 SSL_sigma^2)) (alpha - W).
```

`CAHD_C` is also its TOML key. Writing `SSL_h` or `SSL_sigma` instead of `BSSN_SSL_H` or `BSSN_SSL_SIGMA` triggers an unread-key warning. The former `BSSN_CAHD_C` key is superseded. KO uses constant `KO_DISS_SIGMA` (`enable_CAKO=False`), Brown's covariant term enters `lambdaU` once, and `register_CFunction_BSSN_constraints` emits `BSSN_constraints_order_<fd_order>` (`fd_order` in `4, 6, 8`).

Initial octrees use Dendro-GR's analytic `punctureDataPhysicalCoord` seed (`MIT` provenance) for mesh construction only. Single-rank `--tpid` solves for `u` and writes spectral coefficients; fresh evolution interpolates full `psi = psi_background + u` (`alpha = W = psi^(-2)` for both `W` and `chi` builds) on the initial mesh and again after initial remeshing:

```text
TwoPunctures -> ADM_to_BSSN -> zip/exchange/unzip -> physical boundary
             -> initial_data_lambdaU -> zip -> interior-node floor/projection
```

`twopunctures` and `ADM_to_BSSN` write block interiors only (`zip` reads interiors; unzip + boundary fill supply ghost padding for `initial_data_lambdaU`). During evolution, after one halo exchange and unzip, `Ctx` traverses blocks once (`exterior physical padding` -> `Ricci_eval_order_N` -> `rhs_eval_order_N` -> physical-face outgoing radiation BCs from [Octree Grid, AMR, And Time Stepping](grid-amr-and-time-stepping.md)). After initialization, each RK stage, and AMR transfer, `alpha` is floored at `CHI_FLOOR` and conformal factor at `sqrt(CHI_FLOOR)` (`W`) or `CHI_FLOOR` (`chi`) before `enforce_detgbar_equals_detghat_trAzero`; `W` and `chi` checkpoints carry distinct formulation tags, and constraint slots write `MD0..MD2`, `M_CONSTRAINT`, and `LAMBDA_CONSTRAINT` ([Constraints And Diagnostic Norms](constraints-and-diagnostic-norms.md)).

Evidence (descriptive behavior): `Dendro/general_relativity/rhs_eval.py`; `Dendro/CodeParameters.py::Q1_TOML_PARAMETER_NAMES`; `Dendro/solver_context.py`; `Dendro/main_cpp.py`.

### Generated Dendro_NRPy_BSSN Versus Native CPU BSSN_GR

Sharing a TOML file does not make generated `Dendro_NRPy_BSSN` and native CPU `BSSN_GR` identical (especially when native `RIT_ETA_*` or `CAKO` is active):

| Choice | Generated `Dendro_NRPy_BSSN` | Native CPU `BSSN_GR` |
| --- | --- | --- |
| Evolved conformal factor | `W` default; `--conformal-factor chi` at generation | `chi` |
| TwoPunctures initial lapse | Always `alpha = (psi_background + u)^(-2) = W`; rejects `TPID_REPLACE_LAPSE_WITH_SQRT_CHI = false`; warns on `INITIAL_LAPSE` / `TPID_INITIAL_LAPSE_PSI_EXPONENT` | Needs `TPID_REPLACE_LAPSE_WITH_SQRT_CHI = true` to override `INITIAL_LAPSE = 2` |
| Shift damping (`eta`) | Constant `eta` (default `1`, overridden by `ETA_CONST`, `2.0` in `q1.par.lowres.toml`) | Radial RIT profile `(RIT_ETA_CENTRAL - RIT_ETA_OUTER) exp(-(r / RIT_ETA_WIDTH)^4) + RIT_ETA_OUTER` (`2.0`, `0.25`, `40.0`); ignores `ETA_CONST` |
| Gamma-driver `B^i` | `d_t beta^i = B^i + beta^j Dhat_j beta^i`, `d_t B^i = beta^j Dhat_j B^i + (3/4)(d_t Lambdabar^i - beta^j Dhat_j Lambdabar^i) - eta B^i`; wavelet test scales `betU` by `4/3` | `d_t beta^i = (3/4) B^i + advection`, `d_t B^i = d_t Gt^i - eta B^i + lambda[2] beta^j d_j B^i - lambda[3] beta^j d_j Gt^i` (`B` is `(4/3) betU` only when `eta` matches) |
| KO dissipation | Reads `KO_DISS_SIGMA` (`0.4` in sample when KO on; `CAKO` off) | `KO_DISS_SIGMA` when CAKO off; `sqrt(chi)`-weighted gauge/non-gauge CAKO when on |
| Puncture tracker | Single explicit step `-dt beta^i(x_prev)` after step/remesh; no stored velocity | Heun predictor-corrector on `dx/dt = -beta(x)` with stored velocity (Euler on step 0) |
| Time integrator & CFL | RK4 only (`BSSN_RK_TYPE` unread); CFL uses `min(dx, dy, dz)` | `BSSN_RK_TYPE` selects RK3/RK4/RK5/multistep; CFL uses X-domain width |
| Omitted-key defaults | `BSSN_BH{1,2}_CONSTRAINT_R = 1.0`; `BSSN_SCALE_VTU_AND_GW_EXTRACTION = true` | `5.0` constraint radii; `false` scaling switch |

Evidence (descriptive behavior): `dendro_bssn.py`; `BSSN_gauge_RHSs.py`; `Dendro/solver_context.py`; `Dendro-GR rhs.cpp`; `Dendro-GR bssneqs_SSL_HD_dxsq.cpp`.

### Optional Yo et al. adjustments

Both examples compile CAHD by default (`--cahd`); `--no-cahd` omits it. They also accept default-off `--ccd` and `--camd` (KO stays controlled in-script via `enable_KreissOliger_dissipation=True` and runtime `KO_DISS_SIGMA`, with no `--ko` CLI flag):
- `--ccd` ([Yo, Baumgarte, and Shapiro](https://arxiv.org/pdf/gr-qc/0209066v2) Eq. (45); [Yo, Lin, and Cao](https://arxiv.org/pdf/1205.5111v2) Eq. (47)): adds the Gamma-constraint adjustment to connection and shift-driver RHSs with runtime `CCD_C` (default `0`; recommended `1` in BSSN alongside Brown's term; matching that divergence coefficient in fCCZ4 uses `5/3`; see [YBS Gamma-Constraint Adjustment](../../equations/general-relativity/ybs-gamma-constraint-adjustment.md) and [fCCZ4 Implementation](../../equations/general-relativity/fccz4-implementation-and-validation.md)).
- `--camd` (Yo, Lin, and Cao Eq. (56)): adds the symmetric trace-free gradient of the lower conformal momentum residual to `a_rhsDD` scaled by `CAMD_C * min(abs(dx), abs(dy), abs(dz)) * W / BSSN_CFL_FACTOR` (`CAMD_C` default `0`, recommended `.0625`, no added gridfunctions; see [YBS-MOM](../../equations/general-relativity/ybs-momentum-damping.md#strength-and-timestep-bound)).

Evidence (descriptive behavior): `dendro_bssn.py::parse_args`; `dendro_fccz4.py::parse_args`; `Dendro/general_relativity/rhs_eval.py`; `Dendro/CodeParameters.py`.

The Python options are `enable_CAHD`, `enable_CAMD`, and `enable_CCD`.
`enable_CAMD` replaces `enable_YBS_momentum_constraint_adjustment`, while
`enable_CCD` replaces `enable_YBS_Gamma_constraint_adjustment`. The old
`--ybs-momentum` / `--ybs-gamma` flags and `C_YBS_mom`, `YBS_chi`,
`BSSN_CAHD_C` parameter keys are superseded; no aliases are provided.
The runtime names are `CAHD_C`, `CAMD_C`, and `CCD_C`. Compiling CAMD or CCD
registers a zero default coefficient; enabling a nonzero term requires both
the generation flag and its runtime strength. Generated TOMLs print each
registered parameter description above its key.

For the same spacing and W weight, `CAMD_C = C_YBS_mom * CFL_FACTOR^2`.
The old BHaH CAHD coefficient similarly maps by `CAHD_C = C_CAHD * CFL_FACTOR^2`.
The former softened Dendro BSSN formula instead depends on the global
timestep and level, so it has no single coefficient conversion at every
spacing. The new local-spacing prescription intentionally replaces it.
Checkpoint metadata stores the generated parameter-name list: checkpoints
using the superseded names do not satisfy the new list and are rejected.

For example, generate both optional terms with `--camd --ccd`, then set
the following root TOML keys for the recommended BSSN strengths:

```toml
CAHD_C = 0.015
CAMD_C = 0.0625
CCD_C = 1.0
```

### q1 versus generated defaults

Printed `--tpid` and MPI commands pass `pars/q1.par.lowres.toml` rather than `pars/bssn.toml` (whose defaults `"TwoPunctures alpha=W, and eta=1 enabled"` are described in generator stdout):

| Key | Generated `pars/bssn.toml` | `pars/q1.par.lowres.toml` |
| --- | --- | --- |
| `ETA_CONST` | `1.0` | `2.0` |
| `TPID_FILEPREFIX` | `tp` | `tp_q001` |
| `CAHD_C` | `.015` | `.015` (also the fCCZ4 default) |
| `BSSN_ELE_ORDER` | `--fd-order` value | `6` |

Running the printed commands therefore uses `eta = 2.0`, `CAHD_C = .015`, and order `6` in both BSSN and fCCZ4; running `pars/bssn.toml` alone leaves unset keys at host fallbacks (no horizon finder, constant wavelet tolerance, no Nyquist refinement, 1 VTU field per group; see [Runtime Parameter Keys](runtime-parameters.md)).

Evidence (descriptive behavior): `nrpy/examples/q1.par.lowres.toml`; `Dendro/param_toml.py::generate_default_parfile`; `Dendro/main_cpp.py::output_main_cpp`.

## Sources

- [dendro_bssn.py](../../../nrpy/examples/dendro_bssn.py) - `parse_args`, `main`
- [dendro_fccz4.py](../../../nrpy/examples/dendro_fccz4.py) - `parse_args`, `main`
- [q1.par.lowres.toml](../../../nrpy/examples/q1.par.lowres.toml) - packaged BBH parameter file
- [rhs_eval.py](../../../nrpy/infrastructures/Dendro/general_relativity/rhs_eval.py) - `register_CFunction_rhs_eval`
- [BSSN_constraints.py](../../../nrpy/infrastructures/Dendro/general_relativity/BSSN_constraints.py) - `register_CFunction_BSSN_constraints`
- [ADM_to_BSSN.py](../../../nrpy/infrastructures/Dendro/general_relativity/ADM_to_BSSN.py) - `register_CFunction_ADM_to_BSSN`
- [initial_data_lambdaU.py](../../../nrpy/infrastructures/Dendro/general_relativity/initial_data_lambdaU.py) - `initial_data_lambdaU`
- [twopunctures.py](../../../nrpy/infrastructures/Dendro/general_relativity/twopunctures.py) - TwoPunctures interpolation
- [floor_the_lapse_and_conformal_factor.py](../../../nrpy/infrastructures/Dendro/general_relativity/floor_the_lapse_and_conformal_factor.py) - `CHI_FLOOR` kernel
- [enforce_detgbar_equals_detghat_trAzero.py](../../../nrpy/infrastructures/Dendro/general_relativity/enforce_detgbar_equals_detghat_trAzero.py) - algebraic projection kernel
- [solver_context.py](../../../nrpy/infrastructures/Dendro/solver_context.py) - `Ctx::initialize`, block traversal, `Ctx::evolve_excision_centers`
- [main_cpp.py](../../../nrpy/infrastructures/Dendro/main_cpp.py) - `output_main_cpp`
- [CodeParameters.py](../../../nrpy/infrastructures/Dendro/CodeParameters.py) - `Q1_TOML_PARAMETER_NAMES`, `output_toml_bindings`
- [param_toml.py](../../../nrpy/infrastructures/Dendro/param_toml.py) - `generate_default_parfile`
- [BSSN_gauge_RHSs.py](../../../nrpy/equations/general_relativity/BSSN_gauge_RHSs.py) - `BSSN_gauge_RHSs`
- [dendro_application_check.py](../../../nrpy/examples/tests/dendro_application_check.py) - `Leg.run_negatives`
- [Yo, Baumgarte, and Shapiro, arXiv:gr-qc/0209066v2](https://arxiv.org/pdf/gr-qc/0209066v2) - Eq. (45)
- [Yo, Lin, and Cao, arXiv:1205.5111v2](https://arxiv.org/pdf/1205.5111v2) - Eqs. (47), (56)
- [Dendro-GR rhs.cpp](https://github.com/paralab/Dendro-GR/blob/master/BSSN_GR/src/rhs.cpp) - native CPU `eta` and CAKO dispatch
- [Dendro-GR TwoPunctures.cpp](https://github.com/paralab/Dendro-GR/blob/master/BSSN_GR/src/TwoPunctures.cpp) - native `TPID_REPLACE_LAPSE_WITH_SQRT_CHI`
- [Dendro-GR parameters.cpp](https://github.com/paralab/Dendro-GR/blob/master/BSSN_GR/src/parameters.cpp) - native `RIT_ETA_*` and CAKO defaults
- [Dendro-GR bssngr_main.cpp](https://github.com/paralab/Dendro-GR/blob/master/BSSN_GR/src/bssngr_main.cpp) - native integrator and CFL spacing
- [Dendro-GR bssneqs_SSL_HD_dxsq.cpp](https://github.com/paralab/Dendro-GR/blob/master/BSSN_GR/src/bssneqs_SSL_HD_dxsq.cpp) - native `B_rhs`
- [Dendro-GR grUtils.cpp](https://github.com/paralab/Dendro-GR/blob/master/BSSN_GR/src/grUtils.cpp) - native `computeBHLocations`
- [Dendro-GR eta_RIT.inc.cpp](https://github.com/paralab/Dendro-GR/blob/master/BSSN_GR/src/eta_RIT.inc.cpp) - native radial `eta` profile

## See Also

- Depends on: [BSSN Family](../../equations/general-relativity/bssn-family.md)
- Contrasts with: [fCCZ4 Application Wiring](fccz4-application-wiring.md)
- See also: [Initial Data, Wave Extraction, And Output Files](initial-data-and-wave-diagnostics.md)
