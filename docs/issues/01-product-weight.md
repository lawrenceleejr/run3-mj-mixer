# xs_weight = w_seed × w_match makes the pseudo-event yield scale as W²/N_MC per region (shape distortion, not a global scale)

**Branch:** upstream `full-runs` (`src/run3_mj_mixer/assemble.py`, `xs_weight = w[pairs["seed_file_id"]] * w[pairs["match_file_id"]]`; README "Weights"; `docs/global-hemisphere-index-plan.md` "Weights").

## Problem

Consider a kinematic region R (say a pT bin) with physical hemisphere weight `W_R = Σ_h w_h` and `N_R` MC hemispheres. Single-use nearest-neighbour matching produces ≈ `N_R/2` pairs inside R, each with weight ≈ `(W_R/N_R)²`, so

```
Σ_R xs_weight ≈ (N_R/2) · (W_R/N_R)² = W_R² / (2 N_R)
```

The pseudo-event yield in a region depends on how many MC hemispheres were *generated* there, not only on the physics. HT-sliced QCD MC oversamples the high-HT tail by orders of magnitude, so the pseudo-event HT / hemisphere-pT spectrum is suppressed by ~1/N_MC exactly where the analysis lives. This is a shape effect; the global rescale the README describes cannot undo it. The "cross-slice pairs are suppressed by the weight ratio" feature the README and QA profile describe is the same artefact, not physics.

In `--mode data` (w ≡ 1) nothing is affected, so a data-driven run will not reveal it, but any MC closure (pseudo-MC vs exactly-6-jet MC) will be wrong.

## Demonstration

Toy with the actual `match.Matcher`: two non-overlapping pT regions with the **same** physical `W = 1000`, region A with 2e4 MC hemispheres, region B with 2e5 (like a low-HT vs high-HT slice), uniform directions, η ~ N(0,1), `max_distance 0.5`, `pt_tolerance 0.10`.

| region | pairs | Σ w_seed·w_match | Σ √(w_seed·w_match) | Σ w_seed | physical W/2 |
|---|---|---|---|---|---|
| A (N_MC = 20 000) | 9 931 | **24.8** | 496.6 | 496.6 | 500 |
| B (N_MC = 200 000) | 99 720 | **2.49** | 498.6 | 498.6 | 500 |

Identical physics, factor 10 difference under the product rule. (The √ variant only agrees because each toy region is a single slice; with mixed slices it is also biased.)

## Suggested fix

The seed is drawn uniformly from the unweighted MC pool, so it must carry `w_seed`. The partner is chosen by geometry from the local *unweighted* MC composition, so its weight must be an importance factor relative to that composition, not a second absolute weight:

1. simplest: `xs_weight = w_seed`; or
2. `xs_weight = w_seed · w_match / ⟨w⟩_window(seed)` where `⟨w⟩_window` is the mean `w_rel` of the (available) candidates in the seed's pT window (the Matcher already has `window_rows`); or
3. pick the partner among the k nearest candidates with probability ∝ `w_rel` and use `xs_weight = w_seed`.

Options 2/3 matter if slice composition correlates with hemisphere properties that are *not* matched on (mass, scalar HT of the hemisphere). Whichever is chosen, add a QA profile: Σ xs_weight vs seed pT compared with Σ w_rel of the seeds in the same bins; they should agree up to the factor from failed seeds.
