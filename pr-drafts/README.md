# NRPy draft PR package

Saved on William's public fork for review. Upstream submission is on hold until William explicitly requests it. These Markdown files are prepared PR descriptions; this package does not open GitHub pull requests or request review.

The three code branches are already online at the reviewed commits below. The descriptions include the validation, performance results, migration details and remaining coverage limits.

| PR | Description | Branch | Diff | Commit |
| --- | --- | --- | --- | --- |
| 01 | [Pass configured Dendro black-hole spins to TwoPunctures](01-twopunctures-spins.md) | [Code branch](https://github.com/wkblack/nrpy/tree/fix/dendro-twopunctures-spins) | [Reviewed diff](https://github.com/wkblack/nrpy/compare/33df5324da8702047c4b5b40ae13f123ea9a4073...56b789593eeaa17c33060bd6383ef236df7bb29e) | `56b78959` |
| 02 | [Fix CarpetX C++ parameter loads and CAHD prefactors](02-carpetx-cahd.md) | [Code branch](https://github.com/wkblack/nrpy/tree/fix/carpetx-cahd-prefactor) | [Reviewed diff](https://github.com/wkblack/nrpy/compare/33df5324da8702047c4b5b40ae13f123ea9a4073...dbbf7ded098dae78a1ccdc938a2fdcf4d730fd6c) | `dbbf7ded` |
| 03 | [Normalize constraint adjustments and expose CAHD, CAMD, and CCD options](03-paper-constraint-damping.md) | [Code branch](https://github.com/wkblack/nrpy/tree/feature/paper-constraint-damping) | [Reviewed diff](https://github.com/wkblack/nrpy/compare/dbbf7ded098dae78a1ccdc938a2fdcf4d730fd6c...d9f167e8cb4277601506cd07d74ffc756ee2425a) | `d9f167e8` |

## Submission order

The spin fix and CarpetX repair are independent. When upstream submission is authorized, submit those separately. Merge the CarpetX repair before rebasing/submitting the damping normalization against upstream main. The damping branch currently descends from the CarpetX repair, so its linked diff shows only the damping change.

The damping branch retains the qualified W-weighted momentum operator. Proposed chi-weight changes remain separate research and are outside these drafts.

## Qualification

Both full Dendro application suites pass 38 checks with W and chi, and fresh evolution repeats reproduce the updated references at printed precision. Independent equation checks, all 32 parameter/TOML combinations, cache-switch checks, 17 edited Python-module checks, spin forwarding and corrected CarpetX preamble compilation also pass.

The descriptions include the local interleaved performance measurements. They resolve no step slowdown and do not establish a speedup. Full BHaH application and ETLegacy/CarpetX thorn builds, long-time/merger validation and production performance measurements remain outside this local qualification. The original CarpetX CAHD path does not compile, so it has no parent runtime baseline.

Keep this documentation branch separate from the three code branches; its files are preparation material for the eventual PRs.
