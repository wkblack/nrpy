# Pass configured Dendro black-hole spins to TwoPunctures

Dendro's generated `--tpid` driver currently assigns zero to both black-hole spin vectors, so nonzero `[BSSN_BH1]` / `[BSSN_BH2]` spin inputs do not reach TwoPunctures. Convert each configured physical spin magnitude and its radian angles to a Cartesian dimensionless vector, `chi = S / M^2`, and forward the vectors in TwoPunctures' smaller/larger-mass order. Equal masses retain BH1 as the smaller-mass entry; omitted spins remain zero.

Update the initial-data documentation to describe the actual input convention.

Validation: the actual emitted C++ spin block fails on the parent and passes with this change for unequal masses, exchanged black-hole labels, equal masses, and zero defaults, checking all six Cartesian components. The edited Python module passes the required single-file static/owner checks; KB lint and whitespace checks pass.

A local integration check also links the emitted handoff into the built Dendro application, runs real single-rank TwoPunctures solves, and verifies all six serialized physical spin inputs for a signed, oblique, unequal-mass case and a zero-spin control. Other application objects come from the locally qualified damping build; the four-case emitted-block check isolates this PR's change.

Performance: the actual emitted handoff took 13.25 ± 1.92 microseconds versus 0.085 ± 0.006 microseconds for the parent's zero assignments (mean ± sample SD, four interleaved startup measurements). This work occurs once at startup. Four interleaved zero-spin control pairs using otherwise identical application objects gave full RK4-step times of 1.2933 ± 0.0183 versus 1.2855 ± 0.0121 seconds; the paired change was −0.60 ± 0.96 percentage points. No step slowdown is resolved. The benchmark main differs only in the reviewed handoff, on a fixed 736-octant FD6 mesh with one pinned rank/thread; this measures setup overhead and preserves the existing evolution path, rather than comparing the cost of physically different spinning initial-data solves.
