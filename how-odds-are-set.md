# How Your Odds Are Set

Every horse's odds are calculated fresh for each race, before betting opens. Here is what happens behind the scenes, and why you can trust the numbers.

## Step 1: Thousands of practice races

When a race is created, ZED runs it 1,000 times as a fast simulation before a single bet is placed. Each of those practice runs uses the same horse ratings, surface, and conditions you see on the Race Card.

For every horse, we tally two things:

* How often it **wins** (finishes 1st).
* How often it finishes in a **paying position** (a Place result).

That frequency is the raw signal behind the price. A horse that wins 200 of the 1,000 runs is roughly a 1-in-5 shot. One that wins 20 is a genuine long-shot.

**These practice runs are throwaway.** They only measure the *shape* of the field, not the outcome. The real race result is decided separately by a provably-fair method with a committed seed, so the actual winner can never leak out of the odds. For the full technical detail, see [Provably Fair](provably-fair.md).

## Step 2: Fair odds, with a floor for long-shots

Turning those frequencies into odds is almost as simple as "1,000 divided by wins." But two problems get in the way:

* A horse that won zero practice runs would break the maths (you cannot divide by zero).
* Every long-shot would collapse onto the exact same price.

So ZED applies a small floor to the rare end of the field. This does two useful things: it spreads outsiders across a natural band of prices instead of stacking them on one number, and it uses how often a horse *placed* (a stronger signal for outsiders than wins) to rank them sensibly.

For favourites and mid-field runners with real win counts, the floor does nothing at all. Their odds come straight from the simulations. The floor only bites when the signal is genuinely thin.

The result is a set of **fair odds**: no margin yet, just the true modelled chances of every horse.

## Step 3: Your posted price

Fair odds are never what you bet at. Before a price reaches the Race Card, ZED applies four adjustments:

1. **House margin.** A small cut taken on the profit portion of the odds. The headline figure is 10% for Win and 20% for Place. This is the house edge (see [Results and Payouts](07-results-and-payouts.md)).
2. **Long-shot smoothing.** Very high odds are gently pulled toward a ceiling, so a single outsider cannot post an absurd price.
3. **Rounding.** Prices round to standard bookmaker steps, always in the house's favour.
4. **Floor and cap.** A minimum just above even money and a maximum ceiling, so no price drops too low or runs away too high.

The number you see on the Race Card is the result of all four steps. And it is fixed: once you place a bet, that is your price, even if the odds move afterward (see [Fixed Odds](05-placing-a-bet.md)).

## Why this is fair to you

* **Odds reflect the real field.** Prices come from simulating the actual horses, ratings, and track conditions, not a guess.
* **Long-shots get real prices.** The floor spreads outsiders across a band instead of punishing them all with one identical number.
* **The margin is transparent and consistent.** What you see is what you are paid. No extra maths at settlement.
* **The result is independent.** The winner is decided provably and separately, never derived from the odds.

## Exotics and multis

Exotic bets (Trifecta, Supafecta) and multi-leg parlays are priced by the same engine. For exotics, the single-horse chances from the simulations are combined into a price for the full finishing order. For a multi, the already-posted odds of each leg are multiplied together. The house margin and caps are then applied on top, so the fairness and transparency carry straight across every bet type.
