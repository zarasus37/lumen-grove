# The balance harness

This is the part of Lumen Grove that most templates do not ship: a **rig for
measuring a game's difficulty curve instead of guessing at it.**

A difficulty curve is a curve. It has a shape, a slope and a ceiling, and the
only honest way to place one is to run the real simulation over a grid of
values and read the numbers. Every number quoted in this repository was
produced by the tools in `tools/`, not estimated.

The harness is generic in spirit even though it is bound to Lumen Grove's
rules: the simulation in `src/game/run.ts` has no Phaser import in it, so you
can drive it headlessly at thousands of times real speed and ask it questions a
playtester cannot answer.

---

## The four commands

```bash
npm install

npx tsx tools/simtest.ts              # 43 headless rule checks
npx tsx tools/balance.ts              # sweep every difficulty axis
npx tsx tools/balance.ts ladder 1     # how many runs to max the whole meta tree
npx tsx tools/balance.ts daily        # what a real run scores under each daily modifier
npx tsx tools/balance.ts set hpBase=1.45,hpPerRooted=0.04   # one named scenario

node tools/playtest.mjs http://127.0.0.1:4173   # 43 checks in a real mobile Chromium
node tools/manifest-check.mjs http://127.0.0.1:4173   # PWA installability audit
```

`tsx` is not a dependency — it is fetched on demand by `npx`. The browser
harnesses need `npm install` to have installed Playwright, and the Chromium
binary itself (`npx playwright install chromium`).

Both browser harnesses take a **base URL**, so the identical suite runs against
your local preview and against the deployed site. That is how you check that a
deployment did not break something.

---

## What each tool actually measures

### `simtest.ts` — the rules

Runs the real `Run` class headlessly and asserts the rules hold: planting,
rooting, tiers, the grove breaking when the front line dies, tending, the
affordability rule, the loss condition, determinism, the Grove Path paying
once, the Daily Grove's target and streak, depth stacking, and the mend.

It also runs a **full realistic run** and prints it night by night:

```
n1: plants=10 score=144 hearts=3 breaches=0
n2: plants=28 score=1874 hearts=3 breaches=0
n3: plants=42 score=5454 hearts=3 breaches=0
...
n8: plants=39 score=73570 hearts=3 breaches=0
note: survived 8 nights, score 86962, plants lost 193
```

That output is the curve. Read it as a shape, not as a number.

### `balance.ts` — the difficulty surface

Sweeps one axis of `DIFFICULTY` across a range of values and, for each value,
plays the same three policies at three save states:

| Policy | What it represents |
| --- | --- |
| **careful** | The strategy the game should reward: cover every lane, then promote |
| **no-tend** | The naive baseline: same breadth, never promotes a plant |
| **gappy** | The old strategy: strong in four columns, two lanes left empty |

```bash
npx tsx tools/balance.ts hpBase 1.27,1.35,1.42,1.49
```

```
  value | careful | no-tend | gappy | max careful | max gappy | sparks
  ------+---------+---------+-------+-------------+------------+--------
  1.27  | 11n      | 6n      | 5n    | 14n         | 7n         | 12
  1.35  | 9n       | 6n      | 5n    | 13n         | 7n         | 9
  1.42  | 8n       | 4n      | 4n    | 13n         | 6n         | 6
  1.49  | 7n       | 4n      | 4n    | 13n         | 5n         | 5
```

That is real output from this repository, not an illustration. Read it three
ways:

* **Raising `hpBase` buys you nights.** 1.27 → 1.49 costs the careful player
  four nights (11 → 7) and the maxed grove only one (14 → 13), because the
  blight's health is capped and then matched to a fraction of the player's
  power.
* **The gap survives every value.** Careful beats no-tend by 3 to 5 nights at
  every single setting. That is the invariant.
* **The naive strategy is punished harder.** The gappy policy (strong in four
  columns, two lanes empty) is 5 nights behind careful at 1.27 and 3 behind at
  1.49 &mdash; because a quarter of the blight ignores your grove entirely, so
  an empty lane is a real hole.
* **`sparks` is the economy, not the threat.** It falls with the nights because
  a run pays on score and nights survived. If your sweep moves sparks without
  moving nights, you have changed the reward, not the difficulty.

**The invariant: careful must clearly beat no-tend.** If the sweep ever says
otherwise, the tier mechanic has stopped earning its keep and the curve is
wrong — not the numbers, the *design*. This is the single most valuable line
in the tool, because it turns "does my upgrade system matter?" from an
argument into a measurement.

### `balance.ts ladder` — is the meta game too short?

Plays run after run, spends the Sparks the way a player would, and reports how
many runs it takes to max the Grove and every branch.

```bash
npx tsx tools/balance.ts ladder 1     # one run a day
npx tsx tools/balance.ts ladder 3     # three runs a day
```

This exists because of a specific failure. An earlier build paid
`score / 1000 + nights` Sparks against a 66-Spark ladder, so **one good run
maxed the entire meta game** — and every player was done by night three. The
harness caught it by playing the ladder instead of estimating it.

Current output, one run a day:

```
=== ladder: 1 run(s) per day ===
    FINAL: run 23, day 23, grove level 13, branches roots 1/1, light 5/5,
            reach 3/3, quick 3/3, ward 2/2, lucky 3/3
    path done 10/12, daily bonuses 115, total sparks earned 469
```

**23 runs at one run a day to max the Grove and every branch.** A month of
daily play, not a weekend. And there is a regression check in `simtest.ts`
that asserts the ladder stays long, so it cannot silently regress.

### `balance.ts daily` — is the daily target real?

Prints the score a careful run actually produces under each daily modifier, at
a fresh, a mid and a maxed save. The daily target has to be set against these
numbers. A target below the score of a single run is not a challenge — it is a
freebie that every player clears on day one.

```
  modifier          | fresh score | mid score | max score
  STILL NIGHT       |      73293 |   101112 |   269158
  SWARM             |     112047 |   139064 |   289171
  THICK BRAMBLE     |      67287 |    90474 |   164934
  HUNGRY DARK       |      97007 |   121687 |   217784
```

This game's daily band is `45000..100000`, against a fresh careful run of
73,632 — so a new player clears it about seven days in ten and still fails
often enough for it to matter. A band that any single run clears is not a
challenge; a band a maxed grove sails past is not a goal. Both numbers above
come from the same command.

### `playtest.mjs` — the real browser

Drives a real mobile-sized Chromium: boots to the menu, opens the Grove and
the Path, checks that every branch states its next level's effect, reads the
Daily card, starts a run, **plants by tapping the canvas**, waits for the
night, verifies kills score, verifies kills chain into a combo, loses a run
deliberately, restarts, resizes, and asserts the console is clean.

Screenshots land in `playtest/`.

### `manifest-check.mjs` — the installability audit

Asks Chromium itself whether the page is installable (CDP
`Page.getAppManifest` + `Page.getInstallabilityErrors`), fetches every icon,
and hit-tests the game's touch targets to prove an injected floating widget
does not swallow taps.

---

## Three measurement traps this rig exists to catch

These are the bugs that look fine in a spreadsheet and are invisible in
playtesting. All three were real defects in this project.

### 1. A flat run of zeros followed by a cliff

The symptom is a difficulty chart that is flat, flat, flat and then vertical.
The endpoint looks fine; nothing happened for eight nights and then everything
happened at once.

The cause is usually that the enemy **cannot reach the defender at all**. A
tier-7 plant puts out ~138 dps while the authored blight health was 4–48, so
the blight died during its approach and never touched a single plant. The
grove was a spectator. `chewed=0` for nine straight nights was the tell.

**Measure plants chewed and breaches per night, not just the final night.**
A flat run of zeros followed by a cliff is the signature.

### 2. Wave size is not threat

A bigger wave makes a night *longer*. It does not make it harder. What ends a
run is the **time to kill one enemy against the rate they arrive** — so the
backlog, not the count, is the mechanism. If enemies trickle in one at a time
and each is focus-killed before the next arrives, there is never a backlog and
there is never a defence.

This is why `budgetGrowth` is pinned at `1.0` in `src/core/tuning.ts`: wave
size is a pacing dial, deliberately not a threat dial.

### 3. Scaling the enemy to the player is a trap — if you scale it to 100%

Matching enemy health to the player's output keeps the fight permanent, so the
run never ends; a fully-invested grove hit **94 nights**. The fix is to scale
to a *fraction* of the player's power — the `powerExponent: 0.4` in
`DIFFICULTY` — so progression buys a bounded number of extra nights instead of
a runaway. The same grove that hit 94 nights now holds 13.

---

## The three levers, in the order they actually work

If you take one thing from this repository, take this list. When a game "feels
too easy", the numbers are rarely the problem — **check whether the enemy can
reach the defender.** The fixes, in order of impact:

1. **Multiply the enemy's base health.** Often the whole problem.
2. **Cap the compounded health.** Otherwise the enemy out-scales a player who
   has plateaued, and the only reachable states are "trivial" or "over".
3. **Speed up the spawn rate.** So waves arrive as a stream and a backlog
   forms. This is what actually erodes a front line.
4. **Make some spawns ignore the player's defences.** Otherwise leaving a lane
   empty is mathematically free, and concentrating in four columns plays
   identically to spreading over all six.

And for the meta progression, the same shape of problem:

1. **Scarcity, not a longer ladder.** Cut the kill bounty so the player cannot
   both hold a full grove and promote it. Every night then has a real choice.
   A cheap ladder is the same bug as a cheap income.
2. **A third thing to spend the resource on.** Lumen Grove has "mend the
   heart", priced to climb each night, so it competes with promotion for the
   same scarce currency. Depth comes from competing sinks, not more sinks.
3. **Then a longer ladder**, so promotion is still live late in a run.

The test for all of these is the same: **a run's progression must never
finish before the run does.** The harness asserts it directly —
`"plants are still being promoted late in the run"`.

---

## Where the numbers live

Every difficulty constant is in one object, `DIFFICULTY` in
`src/core/tuning.ts`, and **every one of them is commented with why it is
what it is.** That file is the design. If you change a number, the comment
above it tells you what you are about to break.

```
DIFFICULTY = {
  blightHpMult: 6,      // "the dial that decides whether the game is ever a defence"
  hpCap: 30,            // "without it the only reachable states are trivial or over"
  budgetGrowth: 1.0,    // "wave size is a pacing dial, not a threat"
  powerExponent: 0.4,   // "progression buys a bounded number of extra nights"
  bountyMult: 0.3,      // "Bloom is scarce, so every night is a choice"
  wandererChance: 0.25, // "an unplanted lane is a real hole"
}
```

`src/game/content.ts` holds the other content tables: the ten flora tiers, the
five blight types, the six meta branches with their costs and effect summaries,
and the escalating Grove Level thresholds.

---

## Tuning it for your own game

The harness will not port to a different game unchanged — it calls Lumen
Grove's `Run` class. But it will teach you the shape of the thing, and the
shape is portable:

1. **Keep the simulation free of rendering.** If `src/game/` never imports
   Phaser, you can drive it headlessly. That single architectural rule is what
   makes the whole rig possible.
2. **Put every difficulty constant in one object.**
3. **Author a "careful" policy and a naive one.** Play both. If careful does
   not clearly win, your upgrade mechanic is decoration.
4. **Print the run, night by night.** Not the endpoint. The shape is the
   diagnosis.
5. **Sweep one axis at a time** and read the table, do not move five numbers
   and hope.

---

## Licence

The harness is part of the Lumen Grove source licence — see `LICENSE.md`. You
may use it, modify it, and ship the result commercially. You may not
redistribute it.
