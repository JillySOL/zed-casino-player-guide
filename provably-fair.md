# Provably Fair

_How race outcomes are committed, run, and verified._

## Overview

Every race on this platform is _provably fair_: the outcome is determined by a cryptographic random number generator that is committed to before the race runs and revealed afterwards. Any player can independently replay the simulation to confirm that the published result is the only result the committed seed could have produced.

This document walks through the full lifecycle of a race, from seed generation, through Monte Carlo odds derivation, the betting window, the canonical race simulation, and the post-race reveal, and explains exactly which inputs are frozen at each step so the result cannot be retroactively altered.

> **One-sentence summary:** the server publishes SHA‑256(serverSeed) before betting opens, runs the race using that seed, and then reveals the seed so anyone can re-derive the finish order.

## 1. Race creation & participant snapshots

Races are created on a schedule, well in advance of when they run. At the moment a race is created, every participant slot is filled with an immutable _snapshot_ of the horse and its connections. From that point on, nothing about the participant can change.

What gets frozen per participant:

* **DNA snapshot:** the horse's `speed`, `endurance`, `acceleration`, `agility`, `consistency`, `gateSpeed`, `finishKick`, `trackPreference`, the full `waves` array (DNA wave functions used for in-race variation), and the horse's `formCurve` at the time of snapshot.
* **Surface affinity:** separate ratings for polytrack, tapeta, cushion, and prefabricated surfaces. These are read from the horse's DNA row at snapshot time.
* **Gate assignment:** the starting gate number for this race.
* **Jockey snapshot:** id, name, and the exact list of augment template ids equipped at snapshot time.
* **Trainer snapshot:** same shape: id, name, augment template ids.

Track surface, condition, theme, variant, and the round's position within the season are also fixed on the race record. Because all of this is committed to storage at creation time, a horse that later gets re-trained or has its augments swapped does **not** affect any race that was already scheduled. The historical simulation still uses the DNA and augment effects that existed when the race was created.

## 2. Server seed generation & hash commitment

The provably-fair core uses two pieces of data per race:

* `serverSeed`: 32 cryptographically random bytes produced by the operating system's CSPRNG (`crypto.randomBytes(32)`), encoded as a 64-character hex string. This is the secret.
* `serverSeedHash`: the SHA‑256 digest of `serverSeed`, also hex-encoded. This is the public commitment.

The hash is computed and stored on the race record **before the canonical simulation runs**. Because SHA‑256 is a one-way function, publishing the hash reveals nothing about the seed itself, but anyone holding the hash can later check whether a revealed seed produces that exact digest. If the server tried to swap the seed after the fact, the published hash would no longer match, and verification would visibly fail.

## 3. Monte Carlo odds: 1,000 simulated races

Before betting opens, the server needs to publish prices. It does this by running the same race simulation engine that produces the real result, but **1,000 times**, against a disposable RNG seeded from `Math.random`. The committed `serverSeed` is _not_ used here; the Monte Carlo pass is purely a statistical exercise to estimate each horse's chance of winning and placing.

For each of the 1,000 simulations the engine records:

* Which horse finished **1st** (a _win hit_).
* Which horses finished in the **top 3** (a _place hit_).

Hit counts are converted into decimal odds by inverting the empirical probability, applying the configured house edge, flattening short prices, and clamping to per-market floor and cap values. A small place-informed virtual floor is mixed in so a horse with strong place signal but few outright wins is not assigned an extreme long-shot price. The full pricing pipeline lives in the backend's odds module and is deterministic given the same hit counts.

Once pricing is complete, the resulting `winOdds` and `placeOdds` are written back to each `race_participants` row in a single SQL update. These are the prices players see and bet against.

## 4. The betting window

Once odds are written and the race is published, the betting window opens. The race progresses through the following states:

* `scheduled`: race exists, participants are locked in, but odds may still be pending. No bets accepted yet.
* `open`: odds are published and the betting window is live. Players can place win, place, and exotic bets. The committed `serverSeedHash` is visible on the race's fairness panel from this point on.
* `locked`: the race ticker has reached the lock cutoff (a fixed offset before the scheduled start). The server writes `lockedAt`, emits a `race:locked` event over the socket, and rejects any further bet attempts. No bet can be placed, changed, or cancelled after this point.

The lock timestamp is recorded on the race row, so the closing moment is auditable. Any bet you see settled against this race was, by construction, placed before the lock was applied.

## 5. The canonical simulation

When the race is ready to run, the server constructs the simulation input from the frozen participant snapshots (DNA, surface affinity, augment template ids, gate, etc.) plus the frozen track surface, condition, and season progress. It then performs the following steps, in order:

1. Generate `serverSeed` via `crypto.randomBytes(32)`.
2. Compute `serverSeedHash = SHA‑256(serverSeed)`.
3. Construct the canonical RNG: `createRng(serverSeed, "", 0)` (empty client seed, nonce of zero).
4. Run the race simulation **exactly once** with this RNG. Every random decision the simulation makes (wave phases, stamina rolls, jostling, augment triggers) is drawn from this single deterministic sequence.
5. Persist the result: `segments` (the per-section animation payload), `simPositions` (finish position, finish time, distance from winner, and section times per horse), and `replayData` (an animation-only sidecar containing augment trigger windows).
6. Persist `serverSeed` and `serverSeedHash` on the race row, mark `simStatus = completed`, and stamp `simCompletedAt`.

Because the simulation is deterministic, the same inputs always produce the same outputs. The race that animates in your browser and the finish positions shown on the leaderboard come from **exactly one** execution of the engine, the one seeded by `serverSeed`.

## 6. Animation & finish positions

Playback during the race is driven by `segments`, the per-section payload written when the canonical sim completed. It contains everything the client needs to render the race in real-time: position tracks, speed traces, and section boundaries. Augment trigger windows live in `replayData` as an animation sidecar. They're cosmetic markers, since any speed effect from an augment is already baked into `segments`.

Finish positions, finish times, distance from winner, and per-section times are written to each `race_participants` row from `simPositions` when the race is finalized. These are the values that drive payouts, leaderboards, and bet settlement.

## 7. Post-race reveal & verification

Once the race has finished, the server's secret seed becomes public. The race's fairness panel exposes both fields:

* `serverSeedHash`: what was committed before the race ran. Always present from open onwards.
* `serverSeed`: the actual seed, revealed only after the race is complete.

To verify a race yourself:

1. Read the published `serverSeedHash` and `serverSeed` from the race's fairness panel.
2. Hash the revealed seed with SHA‑256. The digest must equal the previously published hash. If it doesn't, the seed was tampered with.
3. Construct the same RNG locally: `HMAC‑SHA256(serverSeed, ":0")` as the base, then `SHA‑256(base || uint32BE(counter))` for each draw. (Empty client seed, nonce zero.)
4. Feed it the same simulation input (the frozen DNA, augments, gate, surface, condition, and season progress) and run the simulation. The finish order, finish times, and section times you compute must match the published result exactly.

The built-in _Verify_ tab on each race's fairness modal does this for you on the server, and lets you supply alternate seeds and nonces to see what the race _would_ have been under different inputs.

## 8. What this gives you, and what it doesn't

The provably-fair construction guarantees a specific set of properties. It's worth being precise about which:

* **Pre-commitment is real.** The published hash binds the server to one specific seed. Any attempt to change the seed after the fact is detectable by hashing the revealed value.
* **The result is deterministic.** Given the same seed and the same participant snapshots, the simulation always produces the same finish order. There is no hidden tie-breaker.
* **Snapshots can't be back-edited.** DNA, augment loadouts, gate, and track conditions are written at race creation and read directly by the simulation. A re-balance of an augment template tomorrow does not change yesterday's race.
* **Odds are independent of the canonical RNG.** The Monte Carlo pass that prices each horse uses a separate, non-committed RNG. The odds you bet against don't leak any information about the committed seed.

It does _not_ guarantee:

* **That you'll win.** Fairness is about the integrity of the draw, not the expected value of any particular wager. The house edge is baked into the published odds.
* **That the underlying simulation model is correct.** The model is what the model is. Provably fair means "demonstrably the same model for everyone, with the same seed, every time," not "physically accurate horse racing."

## Glossary

* **Server seed:** a 32-byte random value generated by the server per race. Kept secret until the race finishes; the hash of it is the commitment.
* **Server seed hash:** SHA‑256 of the server seed, published before the race runs. The commitment that binds the server to one specific outcome.
* **Client seed:** an optional value the client can mix into the RNG. For the canonical race this is the empty string; the verification UI lets you supply your own to explore alternate sequences.
* **Nonce:** a counter that lets the same seed produce many independent sequences. The canonical race always uses nonce 0; other values are useful in the Verify tab.
* **DNA snapshot:** the full set of attributes for a horse (speed, endurance, acceleration, etc., plus waves and form curve), frozen onto the race participant row at the time the race is created.
* **Monte Carlo odds:** prices derived from 1,000 trial simulations of the race using a throw-away RNG. Win and place hit rates are converted into decimal odds with house edge, floors, caps, and flattening applied.
* **Canonical simulation:** the single, provably-fair run of the simulation that determines the actual race result. Uses the committed server seed, empty client seed, and nonce zero.
* **Segments:** the per-section animation payload (positions, speeds, section boundaries) written from the canonical sim and used by the client to render playback.
* **Replay data:** animation-only sidecar containing augment trigger windows. Cosmetic; speed effects are already baked into segments.

_This page is a standalone reference. It does not require a session and does not pull live race data. Every claim here describes the deterministic pipeline that produces every race on the platform._
