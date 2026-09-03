# Physics audit of run3-mj-mixer (September 2026)

Scope: the fork's `main` (commit 2bd51c2, identical to upstream `main`) and the
upstream rewrite on `jslawless/run3-mj-mixer` branch `full-runs` (af39163,
18 commits ahead). The rewrite supersedes `mix.py` / `stitch.py`; findings are
labelled by the branch they apply to. Paste-ready GitHub issue bodies for the
upstream tracker are in `docs/issues/` (one file per finding, same numbering).

Numerical checks run for this audit (synthetic inputs, code executed as-is):

| check | result |
|---|---|
| `transverse_thrust` (180-point scan + parabolic refine) vs exact enumeration over the 2^(n-1) sign partitions, 200k 5-jet events | T_scan ≤ T_exact everywhere; max deficit 2.6e-5, mean 1.5e-9. Correct. |
| `match.Matcher.find_partner` vs brute force over the full index (`library.py` metric), n = 2e3 … 2e5, max_distance 0.5 and 0.1, 17k queries | 0 mismatches. Exact in practice. |
| Matcher throughput at 2e6 rows | 0.54 ms per draw at the start of the run. |
| Product weight toy (two regions with identical physical density, N_MC differing by 10x) | Σ(w_seed·w_match) differs by 10x between the regions; Σ w_seed reproduces the physical yield in both. See finding 1. |

## Findings

### 1. `xs_weight = w_seed × w_match` distorts the pseudo-event spectrum (full-runs) — high

`assemble.py` sets `xs_weight = w_rel[seed] * w_rel[match]`. In a kinematic
region R with physical hemisphere weight `W_R = Σ w` and `N_R` MC hemispheres,
single-use matching produces ~`N_R/2` pairs, so

    Σ_R xs_weight ≈ (N_R/2) · (W_R/N_R)² = W_R² / (2 N_R)

The yield depends on how many MC hemispheres were generated in the region, not
only on the physics. HT-sliced QCD MC oversamples the tails by orders of
magnitude, so the pseudo-event HT/pT spectrum is suppressed in exactly the
region the analysis cares about, and no global rescale fixes it. The
"cross-slice pairs are suppressed by the weight ratio" behaviour noted in the
README is the same artefact.

Toy (actual `Matcher`, two non-overlapping pT regions, same W, N_MC = 2e4 vs 2e5):
Σ(w_seed·w_match) = 24.8 vs 2.49; Σ w_seed = 497 vs 499; physical W/2 = 500.

Fix options: (a) `xs_weight = w_seed`, the partner being sampled from the local
MC composition; (b) `w_seed · w_match / ⟨w⟩_window(seed)`; (c) choose the partner
among the k nearest with probability ∝ w. Add a QA profile of
Σ xs_weight vs seed pT against Σ w_rel of the seeds.

### 2. Matching tolerances manufacture transverse imbalance; metric issues (both branches) — high

Accepted pairs may be up to `max_distance` = 0.5 rad off back-to-back and 10%
off in pT. Two 500 GeV hemispheres 0.5 rad off give |Σ p_T| = 247 GeV
(2·pT·sin(Δφ/2)); a 10% mismatch alone gives 50 GeV. Real QCD 6-jet events are
balanced to resolution. The pseudo-events therefore have a MET / acoplanarity
distribution unlike data, and the seed's own `pt_par`/`pt_perp` (computed and
stored) are not used for the match.

Also: the Euclidean metric adds radians and η units unscaled; the η match is
one-sided (`cand.eta ≈ seed.partner_eta`, nothing requires
`seed.eta ≈ cand.partner_eta`); the hemisphere "eta" is the pseudorapidity of a
massive system, whereas rapidity is the additive boost variable. The
"no rotation" rule forces detector-φ coincidence; QCD is azimuthally symmetric,
and rotating the partner about the beam axis to be exactly anti-parallel would
both remove the fake imbalance and enlarge the usable library by ~2π/Δφ.
Recommend a closure quantity: |Σ p_T| of pseudo-events vs 6-jet MC.

### 3. Topology and extrapolation assumptions of 5→6 (both branches) — high

Only 3-jet hemispheres are indexed, so every pseudo-event is a 3+3 split.
Real ≥6-jet events split on their thrust axis into (3,3), (4,2), (5,1), (6,0);
the model cannot populate the latter. The 3-jet hemispheres come from events
whose other side had 2 jets; whether their internal structure (mass, pT
sharing, η spread) at fixed (pT, η) matches that of hemispheres in 3+3 events
is the core assumption and is untested in the repo. Closure tests are noted as
"the user's task" in the plan doc but nothing in the pipeline produces them.

Recommendations: MC closure (pseudo-events from 5-jet MC vs exactly-6-jet MC,
with the correct weights from finding 1); a data-only analogue at lower
multiplicity (2+1 hemispheres from 3-jet events → 4-jet pseudo-events vs real
4-jet data; the code is already parameterised by `n_jets` / `--index-n-jets`);
either model (4,2) or define the analysis region as 3+3-split events in data
too; verify trigger / slimmer HT threshold sculpting (parents pass the
threshold with their discarded hemisphere; pseudo-events do not inherit that
acceptance); in data, pairs mix pileup conditions and eras (JEC, prescales),
and event-level quantities (MET, trigger bits) cannot be built for a pseudo-event.

### 4. Greedy random-order single-use matching (both branches) — medium

A drawn seed that finds no partner is consumed and never returns to the pool,
although it may be a perfectly good partner for a later seed (the pT window and
η match are not symmetric). Failures concentrate where the index is sparse
(high pT), so this discards the rarest hemispheres preferentially. Match quality
also degrades as the pool drains, so the tail of `match_distance` and the
unmatched set depend on the RNG seed and draw order. Recommend: keep failed
seeds partner-eligible; draw sparse regions first (or process in descending pT);
treat RNG-seed variation as a systematic; note that "pairs are independent, so
bootstrap directly" is approximate because pairs interact through the pool.

### 5. `build_file_table` reads every slimmed file twice (full-runs) — perf, easy

`filetable.py`: `scan_n_original` reads `cutflow[0]` of every file (threaded),
then the per-file loop calls `read_cutflow0(url_of[p])` again for
`n_original_file`, serially. At ~5800 files over xrootd this doubles the slow
step. Return per-file values from the scan and reuse them. Smaller wins:
`gather.read_file_legs` and `assemble.load_legs` index awkward arrays
element-by-element in Python loops (use offsets/flatten); `PairWriter.add` does
~26 memmap scalar reads per pair.

### 6. Grid ring termination bound (full-runs) — low

`Grid.n_phi = ceil(2π/cell_w)` gives `phi_w ≤ cell_w`, but `find_partner` stops
when `best_d ≤ r·cell_w`. Ungathered rows are only guaranteed ≥ `r·phi_w` away
along φ. Use `r·min(phi_w, cell_w)`. No mismatch was observed in 17k random
queries, so the effect is rare, but the exactness proof does not hold as written.

### 7. Tests are referenced but absent (full-runs) — repo hygiene

README, `docs/global-hemisphere-index-plan.md` and `pyproject.toml` describe
`tests/test_match.py`, `test_index.py`, `test_index_build.py` and an oracle
fixture directory, but no `tests/` exists on the branch. `.gitignore` has a
`test*` pattern with `!tests/` negations; check that they were not swallowed.
The exactness claim for the matcher rests on these tests.

### 8. `main` carries defects fixed only on `full-runs` — housekeeping

Per-file `n_original` (weights inflated by the slice's file count,
non-uniformly across slices); stochastic-rounding budgets at lumi = 1 discard
almost all high-HT hemispheres (w ≪ 1); `stitch.load_library` re-derives the
hemisphere mask in float64 from float32 values; `findPartnerHemisphere` is O(n)
Python per query (O(n²) per run); README `pytest` instructions with no tests.
Merge `full-runs` or mark `main` superseded.

## Verified OK

Thrust-axis finder; hemisphere split and four-vector sums; directed-φ query
convention (candidate on the −n_T side with a parallel thrust axis points back
at a +n_T seed); Matcher exactness in practice; `jet_mask` consistency checks;
slice-wide `n_original` on `full-runs`.
