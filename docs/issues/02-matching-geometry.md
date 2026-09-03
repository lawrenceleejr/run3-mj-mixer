# Matching admits 0.5 rad acoplanarity + 10% pT imbalance: pseudo-events carry a fake transverse imbalance; the metric mixes units and is one-sided in η

**Branch:** both (`library.py`, `match.py`; `main` `stitch.py`).

## Fake transverse imbalance

A pair is accepted when the candidate's directed φ is within `max_distance` (default 0.5 rad, in quadrature with Δη) of the direction opposite the seed and its pT is within ±10% of the seed's. Nothing constrains the vector sum. For two hemispheres of pT each:

| pT (GeV) | Δφ off back-to-back | |Σ p_T| from Δφ alone | from 10% pT alone |
|---|---|---|---|
| 300 | 0.1 / 0.3 / 0.5 rad | 30 / 90 / 148 GeV | 30 GeV |
| 500 | 0.1 / 0.3 / 0.5 rad | 50 / 149 / 247 GeV | 50 GeV |
| 1000 | 0.1 / 0.3 / 0.5 rad | 100 / 299 / 495 GeV | 100 GeV |

Real QCD 6-jet events are balanced to jet resolution (tens of GeV). The pseudo-events therefore have a MET / acoplanarity / "HT-balance" distribution that differs from data, and any analysis variable sensitive to the whole-event momentum sum (pairing-based masses, MET-based cleaning, ΔΦ between jets across hemispheres) inherits it. The mixer already computes and stores each hemisphere's `pt_par` / `pt_perp` in the thrust frame but does not use them in matching.

Suggested: match on the thrust-frame components (require `pt_perp` compatibility, not just |pT|), tighten the φ window to something set by jet resolution, or rotate (see below). Add a closure quantity: |Σ p_T| of pseudo-events vs exactly-6-jet MC / data.

## "No rotation" costs library size for no physics gain

The design forbids any transform, so a partner must happen to point back at the seed in detector φ. QCD is azimuthally symmetric; rotating the partner hemisphere about the beam axis so its thrust direction is exactly anti-parallel to the seed's is a symmetry of the physics (detector φ non-uniformities in scouting jets are small and can be checked). The standard hemisphere-mixing method (arXiv:1712.02538) combines hemispheres in the thrust frame for this reason. Rotation would (a) remove the fake imbalance above and (b) make the whole pT window eligible instead of a Δφ ≲ 0.5 rad slice of it, i.e. ~2π/Δφ ≈ 10× more candidates for the variables that do carry physics (η, mass, structure).

## Metric details

- Euclidean distance adds radians and η units unscaled. Standardise (divide by the resolution or the sample RMS of each coordinate), or state the intended relative weighting explicitly.
- The η matching is one-sided: `cand.eta ≈ seed.partner_eta` but nothing requires `seed.eta ≈ cand.partner_eta`. The partner's internal structure is correlated with where *its* lost partner pointed. Add `(seed.eta − cand.partner_eta)²` to the metric (or match on η_sum and η_diff).
- The hemisphere "eta" is `asinh(pz/pT)` of a massive 3-jet system, i.e. pseudorapidity. The additive longitudinal-boost variable is rapidity `y = atanh(pz/E)`; for hemisphere masses comparable to pT the two differ appreciably. Use rapidity for `eta`/`partner_eta` if the intent is to preserve the event's boost.
