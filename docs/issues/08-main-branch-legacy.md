# main carries physics/perf defects that are fixed only on full-runs — merge or mark superseded

**Branch:** `main` (both repos; the fork's `main` is identical to upstream `main`).

If anyone still runs `run3-mj-mixer` / `run3-mj-stitch` from `main`, these apply:

1. **Per-file `n_original`** (`mix.py`, `weight = lumi*xs_pb/cutflow[0]` of *this file*): every weight is inflated by the slice's file count, non-uniformly across slices, so relative slice normalisation is only right when every job takes the same number of files per slice. `full-runs` fixed this with a slice-wide sum.
2. **Stochastic-rounding budgets at lumi = 1** (`library.py addHemisphere`): with `w = xs_pb/n_file` the high-HT slices have `w ≪ 1`, so almost all their hemispheres get budget 0 and are never used. The mixing throws away exactly the MC statistics generated for the tails, and the resulting weight-1 sample has enormous tail fluctuations. `full-runs` replaced this with single use + weights (see the separate product-weight issue for what is still wrong there).
3. **Hemisphere membership re-derived in float** (`stitch.load_library`: `proj = pt*cos(phi - phi_t)` with float32-stored `thrust_axis_phi`, compared against the mixer's float32 `px*cos_t + py*sin_t`). Jets near the splitting plane can be assigned differently from the stored `Hemisphere_n_jets`; if both legs are affected the 6-jet event gets wrong `hemisphere` flags silently, otherwise `np.stack` raises at the end of the run. `full-runs` records `jet_mask`.
4. **`HemisphereLibrary.findPartnerHemisphere` is O(n) Python per query** (re-`np.asarray` of three Python lists and a Python generator over all hemispheres for `exclude_event_id`), so a run is O(n²) with large constants. Fine for the 1e5-hemisphere per-job libraries it was written for, not beyond.
5. README `pytest` instructions with no tests in the tree.

Suggest merging `full-runs` into `main` (or archiving `main`'s pipeline) so the branch people clone does not carry these.
