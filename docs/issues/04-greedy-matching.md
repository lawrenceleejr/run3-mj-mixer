# Single-use greedy matching: failed seeds leave the partner pool, and match quality depends on draw order (depletion concentrated in the tails)

**Branch:** both (`match.Matcher.run`: `i = self.draw()` consumes the seed; on failure it is never restored. `library.py` zeroes the seed's budget on failure.)

1. **Failed seeds are lost as partners.** A drawn seed that finds no partner within the cut is consumed, although it may be an acceptable *partner* for a later seed: the pT window (relative to the seed) and the η condition (`cand.eta` vs `seed.partner_eta`) are not symmetric, so "cannot find" does not imply "cannot be found". Failures cluster where the index is sparse, i.e. at high pT, so the hemispheres discarded this way are preferentially the rare high-HT ones the analysis needs most. Fix: on failure mark the row `not drawable` but keep `available` for partner searches (still single-use).

2. **Draw order sets match quality.** Early draws take the best partners; late draws get worse ones or fail. The `match_distance` tail, the unmatched set and their kinematic bias (which QA reports as matched-vs-unmatched means) are functions of the RNG seed, not of physics. Options: draw sparse regions first (descending pT, or ascending local density), or a two-pass scheme; and quantify the RNG-seed dependence of the final spectra as a systematic.

3. **Bootstrap claim.** "Each source event lands in at most one pseudo-event, so the output is statistically independent and can be bootstrapped directly" is approximate: which partner a seed gets depends on what earlier seeds consumed, so pairs are coupled through the pool. Fine to bootstrap, but note it.
