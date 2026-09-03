# Physics assumptions of the 5→6 extrapolation: only 3+3 topologies are produced, no closure test exists

**Branch:** both. `shard.py --index-n-jets 3` / `stitch.py` `n_jets == 3`; `qa.py` enforces `Hemisphere_n_jets == [3, 3]`.

## Only 3+3 pseudo-events

Real ≥6-jet events, split on their transverse thrust axis, are (3,3), (4,2), (5,1) or (6,0). The pipeline indexes only 3-jet hemispheres of 5-jet events, so every pseudo-event is 3+3 by construction (after re-splitting on the recomputed axis a few migrate, but the parentage is always 3+3). The background model cannot populate the (4,2)/(5,1) part of the 6-jet phase space at all. Either

- build (4,2) pseudo-events as well (4-jet hemispheres from 4+1 events paired with 2-jet hemispheres from 3+2 events) with a defensible relative normalization, or
- define the signal/control regions in data as events whose thrust split is 3+3, so model and data are the same population, or
- demonstrate that the analysis observables are insensitive.

## The extrapolation assumption is untested

The 3-jet hemispheres come from events whose other side had 2 jets. The method assumes their internal structure (mass, pT sharing, η spread, sub-leading jet pT) at fixed (pT, η) is the same as for 3-jet hemispheres in 3+3 events. Global properties (HT, ISR activity, pileup) correlate both sides of an event, so this is not guaranteed. The plan doc lists closure plots as "the user's own task"; nothing in the repo produces them. Suggested minimum:

1. **MC closure:** pseudo-events from 5-jet QCD MC vs exactly-6-jet QCD MC (njets after re-split, HT, leading/sixth jet pT, hemisphere masses, |Σ p_T|, the analysis discriminants), with the weight fix from the product-weight issue.
2. **Data-only analogue at lower multiplicity:** the code is already parameterised by `n_jets` / `--index-n-jets`; run 3-jet data events (2+1 split) → 2-jet hemispheres → 4-jet pseudo-events and compare to real 4-jet data. Same extrapolation structure (k, k−1) → (k, k), fully data-driven, no signal contamination worry.
3. **Trigger / slimmer threshold sculpting:** a parent event passes the HT trigger and the slimmer cuts using its *discarded* 2-jet hemisphere; the pseudo-event's HT is `HT_A + HT_B`, which does not inherit that acceptance. Require pseudo-event HT well above the trigger plateau and apply the analysis-level jet/HT selection identically to pseudo-events and data; check the HT distribution near threshold in the closure.
4. **Data conditions:** pairs mix pileup, eras, JEC versions and prescales; and event-level quantities (MET, MET filters, trigger bits) cannot be built for a pseudo-event, so the analysis must not use them or must model them.
