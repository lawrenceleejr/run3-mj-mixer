# Grid ring termination uses cell_w, but the φ cell width is 2π/ceil(2π/cell_w) < cell_w

**Branch:** upstream `full-runs`, `match.Grid` / `Matcher.find_partner`. Low severity.

`Grid.__init__` sets `n_phi = ceil(2π/cell_w)` and `phi_w = 2π/n_phi`, so `phi_w ≤ cell_w` (e.g. `cell_w = 0.5` → `phi_w = 0.483`). `find_partner` stops expanding when `best_d <= r * cell_w`. After gathering rings 0..r, an ungathered row is guaranteed at least `r` whole cells away along *some* axis, i.e. at least `r·phi_w` along φ, which is less than `r·cell_w`. A candidate with `r·phi_w < d < best_d ≤ r·cell_w` can therefore be missed, so the "provably exact" argument in the docstring does not hold as written. One-line fix: `covered = r * min(g.phi_w, g.cell_w)` (also for the `covered > max_distance` break, or keep `max_ring` computed from the smaller width).

In 17 000 random queries against brute force (n = 2e3 … 2e5, `max_distance` 0.5 and 0.1) no mismatch occurred, so the effect is rare in practice; this is about the guarantee, and it is cheap to close.
