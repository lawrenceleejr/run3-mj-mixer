# build_file_table reads every slimmed file twice for cutflow[0]; second pass is serial (+ smaller perf wins)

**Branch:** upstream `full-runs`.

`filetable.build_file_table` first calls `scan_n_original(...)` which reads `cutflow[0]` of every file (optionally threaded, with a progress bar), then in the `for p in new:` loop calls `read_cutflow0(url_of[p])` **again** for `n_original_file`, serially and without progress. At ~5 800 files over xrootd this doubles the slow step the docs already warn about, and the second pass is the one without `--workers`. Fix: have `scan_n_original` return the per-file values (it already has them in `values`) and reuse them.

Smaller, easy wins found while reading:

- `gather.read_file_legs`: `np.asarray(arrays[mapping[fld]][entry])` is an awkward per-element `__getitem__` inside a Python loop over requests × fields (~12). Flatten each branch once and slice with the offsets (`ak.to_numpy(ak.flatten(...))`, `ak.num` cumsum), which is pure numpy per request.
- `assemble.load_legs`: same pattern (`a[f"jet_{fld}"][k]` in a Python loop).
- `match.PairWriter.add`: ~26 memmap scalar reads and Python appends per pair; gather the provenance columns in vectorised blocks at `flush()` time instead (`ix[col][np.asarray(seed_ids)]`).
- `gather.scan_legs`: `np.isin(fid, list(owned))` rebuilds the list on every chunk × leg; build the array once.
- `assemble.N_SCAN = 180` is hard-coded; read `mixer.thrust_scan_points` from the config so the recomputed pseudo-event thrust uses the same setting as stage 1a.

For the record, the thrust finder itself is not a bottleneck (~40 µs/event) and was verified against exact enumeration.
