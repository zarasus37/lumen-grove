# Lumen Grove — design

**One thumb. One grid. Grow light until the dark gives up.**

Lumen Grove is a mobile arcade-defense hybrid: a real-time lane defense on a
6×7 grid, wrapped in the instant-restart loop of a hyper-casual score chaser,
with the base-building brain of a strategy game underneath.

---

## 1. The synthesis map

The brief was "pull the very best aspect of the most successful games into
one". Every mechanic below is borrowed deliberately, and each one is a
mechanic that has demonstrably worked in a game with tens of millions of
downloads.

| Borrowed from | The aspect taken | Where it lives in Lumen Grove |
| --- | --- | --- |
| **Plants vs. Zombies** | Plant before the wave, then defend | 6.5s **Prepare** phase (income ×2) → **Night** → **Dawn** |
| **Plants vs. Zombies** | Zombies chew through your line | Blight stops in a cell with a plant and eats it; the plant dies, the lane opens |
| **Plants vs. Zombies** | Lose the house = lose the run | Blight reaching the Seedheart row costs one of three hearts |
| **Clash of Clans** | Your base is a *place*, and layout is strategy | The Seedheart anchors a fixed grid; where you build decides what fires |
| **Clash of Clans** | Growth is a meta ladder you spend between runs | **Grove Sparks** buy six permanent upgrades |
| **Clash of Clans** | Bigger base = bigger income | The heart pays more Bloom per second for every rooted plant |
| **Candy Crush** | Cascades are the dopamine | A kill ripples light to every rooted neighbour; kills chain into a multiplier |
| **Vampire Survivors** | Power-ups stack, numbers go up | Light Motes drop from kills and stack **DMG / RATE / BLOOM** bonuses |
| **Flappy Bird / Geometry Dash** | One number, instant retry | One giant **GROVE SCORE**; death → one tap → **REGROW** → new run |
| **Geometry Dash** | The run is short and you want another | A full run is ~8–12 minutes; the score is a single chasing number |
| **Subway Surfers / Temple Run** | The agent is always moving, always readable | The field is live in real time; nothing waits for your turn |
| **Angry Birds** | Physics of impact, clear trajectories | Bolts visibly travel, home, and land; impacts flash and shake |
| **Monument Valley / Alto's Odyssey** | Beauty as the reason to keep playing | Dusk palette, additive glow, drifting fireflies, distant spires — zero image assets |
| **Monument Valley** | Calm, readable, uncluttered | One glyph per entity, one colour per tier, nothing on screen without a job |
| **Stardew Valley** | A daily ritual | The sky is seeded by the calendar date, shared by every player that day |
| **Bestiary / Idle games** | Escalating numbers | Blight health climbs to a 30× ceiling, then matches your own power |

### What was deliberately *not* taken

* **No ads, no timers, no energy.** Nothing in the game asks you to come back.
* **No gacha, no loot boxes.** Every upgrade is bought with earnt currency.
* **No lives system.** One run, one score, one retry button.
* **No randomness in the rules.** The same seed always produces the same
  night, so a bad score is your play, not the shuffle.

---

## 2. The one idea

Most of these games are separate games with separate rules. Lumen Grove
fuses them through **two axes that every other system hangs off**:

> **Is it rooted?** — connected to the Seedheart through a path of planted
> cells. Rooted plants fire and regenerate. Unrooted plants are dark, dead
> weight. Break the link and everything above it goes dark with it.
>
> **How high is it?** — tier equals height above the heart, plus any
> promotion you paid for. Row 6 is a Sapling (tier 1); row 0 is a Crownlight
> (tier 7).

That single tension is where the design lives:

* **Tall plants are strong but far away.** The Seedheart is at the bottom.
  The strongest grove is the most exposed one.
* **Front-line plants are weak but they are the wall.** The tier-1 Sapling
  at row 6 is the cheapest thing you can build and the first thing the
  Blight eats. It buys the seconds your tall towers need.
* **Rooting forces commitment.** A plant in column 0 is only rooted if you
  paid for a path to reach it. Scattered towers are decorative.

Two ways to make a plant stronger, both costing Bloom: **build it higher**
(free of structural cost, but far from the heart) or **tend it** (Bloom, but
it can sit on the front line). That is the core decision, repeated.

---

## 3. Systems

### Economy

| Source | Value |
| --- | --- |
| Starting Bloom | 30 (two Saplings) |
| Heart income | `1.2 + 0.32 × rooted plants` per second |
| Prepare phase | income ×2 |
| Plant cost | `12 × (1 + 0.5 × (height − 1)) × metaMult` — 12 at row 6, 48 at row 0 |
| Tending | `18 × (promotions + 1)` |

The heart pays a trickle even with an empty grove, so the first plant is
always affordable — but it only pays properly once the grove is rooted.

### The Blight

Five types, each unlocking on a set night:

| Type | From night | HP | Speed | Bite | Trait |
| --- | --- | --- | --- | --- | --- |
| Mote | 1 | 4 | 50 | 5 | fast, fragile |
| Crawler | 1 | 14 | 28 | 8 | the baseline |
| Wisp | 3 | 9 | 42 | 6 | fast, erratic |
| Splitter | 5 | 22 | 21 | 9 | bursts into 3 Motes on death |
| Husk | 7 | 48 | 15 | 15 | slow armoured wall |

Two rules make it feel fair rather than random:

1. **The Blight comes for the light you show it.** Spawn lanes are weighted
   by how many plants you have in them. An empty lane is not what kills you;
   a weak grove is.
2. **Everything scales with the night.** HP ×1.42ⁿᵗ (capped at ×30), base HP ×6,
   speed ×(1 + 0.06n), bite ×(1 + 0.08n), budget
   `(8 + 12n + 5.5n^1.9) × (1 + 0.15√rooted)`, a spawn every 0.4s.
3. **A quarter of the Blight ignores your grove.** Those wanderers walk down
   whatever lane they like, so a column you left empty is a hole the heart
   pays for. Covering all six lanes is a decision, not a formality.

### Chains

When a plant kills a Blight, every **rooted orthogonal neighbour** re-fires
immediately. If that shot kills, the chain continues from that plant.

* Chain shots deal `0.5^depth` damage — a flourish, not the main damage.
* Depth caps at 4, and 16 chain shots per kill streak.
* Every kill inside a 1.5s window raises the **combo**, which multiplies the
  Bloom and score payout up to ×4.75.

Capping the chains mattered: uncapped, one kill in a dense grove cascaded
into ~30 extra shots and the difficulty curve flattened out completely.

### Light Motes

14% of kills drop a mote that drifts to the heart and grants a permanent
(for the run) bonus: **+5% damage**, **+4% fire rate**, **+8% Bloom**, or
occasionally **+1 heart**. Tap one to collect it early for +10 Bloom.

Bonuses are **capped** (damage +120%, rate +60%, Bloom +150%). Uncapped they
compounded past ×50 by night 10 and made the whole night curve irrelevant.

### Meta progression — two ladders, one screen

The progression is split into two things, because "leveling up" was
ambiguous when there was only one:

**The Grove Level** is *earned by playing*. It is the number in the middle
of the screen and it never decreases. Each level costs six more lifetime
Sparks than the last — 15 for the second, 59 for the thirteenth — so the top
of the tree is a long climb. Every level grants a permanent bonus:

| | Per Grove Level | Cap |
| --- | --- | --- |
| Bloom income | +7% | +84% |
| Bolt damage | +6% | +72% |

**The Branches** are *bought with Sparks*. Six of them, each with its own
icon, its own level count, and a stated effect at every step:

| Branch | Levels | Cost | Effect per level |
| --- | --- | --- | --- |
| Deep Roots | I/II | 24 | Maximum tier 7 → 8 |
| Keen Light | I/V | 12,19,28,38,52 | +12% bolt damage |
| Long Reach | I/III | 15,27,42 | +18% bolt range |
| Quick Bloom | I/III | 15,27,42 | −14% cost to plant |
| Night Ward | I/II | 22,38 | +1 heart at the start of a run |
| Lucky Sprout | I/III | 18,31,47 | +6% mote drop chance |

The **Grove screen** shows all of it at once: the tree, the level, the
progress bar, the Grove bonus, and every branch with its icon, its level in
Roman numerals (`LEVEL III / V`), a pip bar, the effect you have *now*, and
the effect the next level gives you (`Next level: +48% bolt damage`).

**The tree is the illustration of the level.** It grows through seven named
forms as you climb:

| Grove Level | Form |
| --- | --- |
| 1 | Seed |
| 3 | Sprout |
| 5 | Sapling |
| 7 | Bloom |
| 9 | Elder |
| 11 | Crownlight |
| 13 | First Flame |

Each form is a different procedurally drawn SVG — a seed in the soil, a
sprout, a sapling, then a tree that gains height, branches, canopy and
blossoms while the colour climbs from teal through mint to white-gold. The
final form is a radiant tree with a halo and rising embers.

**The economy was rebalanced twice, both times because it was too cheap.**
The first version paid `score / 1000 + nights` Sparks, so a single good run
earned ~275 while the whole ladder cost 66 — the entire meta game was maxed
in one run and "what am I levelling" had no answer. The second paid
`max(1, score / 25000 + nights / 3)` against a 94-Spark ladder, but the
ladder itself was still cheap: the Grove Level cost a flat 8 Sparks a level
while a careful run earned ~16, so everything was maxed in six runs.

Now a run pays `max(1, score / 18000 + nights / 3)`, the Grove Level costs
escalating lifetime Sparks (15 to reach level 2, 444 to reach level 13), and
the six branches together cost 376. Measured by simulation, that is **23
runs to max everything** — a month of daily play, not a weekend.

### In-run level clarity

Tier is the in-run level, and it is now legible at a glance:

* every plant draws a **tier pip bar** beneath it (filled pips = tier);
* planting pops `TIER 4`, tending pops `TIER 5` in gold;
* the damage bar only appears when a plant is actually hurt, so the tier
  pips are never buried;
* **tending is two-step**: the first tap on a plant shows
  `TEND → TIER 5 · 18 ✦` and spends nothing; a second tap on the same
  plant commits it. Tending used to cost 30 and fire on the first tap,
  which made an accidental brush of the screen cost a third of your
  economy — and at 30 it was 4× less Bloom-efficient than planting a new
  plant, so nobody would ever have used it. At 18 a promotion is
  stat-equivalent to planting one row higher, so it is worth doing for the
  better defensive position it keeps.

### The in-run ladder never finishes

The first version of this game let a careful player max every plant by
**night three**, then sit out the rest of the run with 114,616 unspent
Bloom. The in-run progression was over before the game was.

Three things fixed it, and all three are measured, not asserted:

1. **Bloom is scarce.** The kill bounty was cut to 0.3×, so a run cannot
   both hold a 42-bed grove and promote it. Now the player is choosing
   between the two every night.
2. **The ladder is longer.** Ten flora tiers, with an in-run cap of 9
   (10 with the Deep Roots branch). A careful player is still promoting
   plants on the second-to-last night, and a fresh save never maxes every
   plant — 38 of 42 at the end of a good run.
3. **There is a third thing to spend Bloom on.** **Mending the heart** — a
   button in the dock that restores a heart for `180 + 70 × night` Bloom.
   Mend now and keep a heart, or hold the Bloom and promote a plant that
   might stop the next breach on its own. The price climbs every night, so
   a heart saved for later is not free.

---

## 3b. Retention — the reason to come back

A score chase ends. Without a reason to return, the game is a one-afternoon
curiosity. Three interlocking systems:

### The Daily Grove

Every calendar day has **one shared grove and one shared target** — the same
for every player, derived from the date. No server, no account, no network
call: the grove and the target are both hashes of `YYYY-MM-DD`.

* The target is a score between 45,000 and 100,000, set by the date. It is
  calibrated against measured runs: a careful player scores roughly 67k-112k
  on a fresh save, so the daily is a real challenge that a new player clears
  about seven days in ten — and fails often enough to matter.
* Each day also carries a **modifier** that changes the shape of the night:
  `STILL NIGHT` (−15% blight), `SWARM` (+30% blight), `THICK BRAMBLE`
  (+30% blight health), `HUNGRY DARK` (+15% blight, +10% health).
* Beating the target pays **+5 Sparks** and ticks the day.
* A **7-day strip** on the menu shows the last week: mint squares for days
  beaten, red for days attempted and missed, empty for days untouched.
* A **streak** counts consecutive beaten days. It survives until the end of
  the following day, so a streak is never lost at midnight before you have
  had a chance to play.

The daily is the strongest retention hook available to a short-session
score chaser, and it needs no backend.

### The Grove Path

Twelve one-time milestones, each paying Sparks once:

| Milestone | Goal | Reward |
| --- | --- | --- |
| First Light | Survive Night 1 | 2 |
| Rooted | 5 rooted plants at once | 3 |
| Chain Reaction | Chain 5 kills | 4 |
| The Tallest | Grow a tier 5 plant | 5 |
| Dawn Breaker | Survive Night 5 | 7 |
| Full Bloom | 20 plants in one run | 7 |
| Night Warden | Survive Night 10 | 10 |
| The Grove Path | Reach Grove Level 5 | 10 |
| Deep Roots | Buy the Deep Roots branch | 14 |
| Crownlight | Reach Grove Level 11 | 18 |
| Full Garden | Fill all 42 beds | 20 |
| First Flame | Reach Grove Level 13 | 30 |

100 Sparks across the Path, on top of the 376-Spark branch ladder. The Path
screen shows every milestone with its illustration, its reward, and — for
the next one — exactly how close the last run got (`7 / 10 nights`).

### The near miss

The single strongest retention element in a score game is **always telling
the player what they were one step away from**. The game-over screen
therefore *always* names the next milestone and how close the run got:

> **NEXT ON THE PATH** — Night Warden — Survive Night 10. You reached
> **7 / 10 nights**.

If the run completed a milestone, the line becomes the reward instead.

### Deepen the Grove (prestige)

At Grove Level 11+, the Grove screen offers **DEEPEN THE GROVE**: reset the
Grove Level, every branch, and all Sparks — and take a permanent **Depth**
instead. Each Depth grants +3% Bloom income and +2% bolt damage, forever.

The Path and the daily history survive, because they are achievements, not
currency. This is the long tail: the game never truly ends, it just starts
over deeper.

---

## 4. The difficulty curve (measured, not guessed)

`tools/simtest.ts` runs the simulation headlessly with a careful
always-planting policy. Current numbers, one run, default upgrades:

| Night | Plants | Score | Hearts | Breaches |
| --- | --- | --- | --- | --- |
| 1 | 10 | 144 | 3 | 0 | 0 |
| 3 | 42 | 5,454 | 3 | 0 | 0 |
| 5 | 42 | 23,079 | 3 | 0 | 0 |
| 7 | 42 | 54,795 | 3 | 0 | 9 |
| **8** | 39 | 73,570 | 3 | 0 | 55 |
| — | — | — | **0** | **3 → run over** |

A careful player builds for six nights, then the pressure arrives: the front
line starts being chewed on night 7, and by night 8 the grove is collapsing
(193 plants lost) and the heart is struck. An undefended grove falls in the
first night. That gap — safe if you play well, fatal if you don't — is the
whole game.

**The Blight matches the grove facing it.** Blight health is scaled by the
player's own power (damage multiplier and average tier, to a 0.4 exponent),
so a fully-invested grove meets a proportionally harder night instead of
coasting. Without it the meta progression snowballs and a maxed grove hit
94 nights; with it, the same grove holds 13.

**The curve is measured, not asserted.** `tools/balance.ts` sweeps the
difficulty constants against a careful policy, a naive "never promote" policy
and a "leave two lanes empty" policy, at a fresh, a mid and a fully-invested
save, and reports the nights each policy survives. The careful policy must
stay clearly ahead of the naive one, or the tier mechanic has stopped earning
its keep: right now careful holds 7 nights, never promoting holds 4.

The curve is measured, not asserted. `tools/balance.ts` sweeps the difficulty
constants against a careful policy **and** a naive "spread wide, never
promote" baseline, at a fresh, a mid and a fully-invested save, and reports
the nights each policy survives. The careful policy must stay clearly ahead
of the naive one, or the tier mechanic has stopped earning its keep.

---

## 5. Feel

* **Zero image assets.** Every sprite is drawn procedurally: a radial glow,
  a spark, a ring. Plants are composed from a stem, leaves, a bulb and a
  rotating petal ring, tinted by tier.
* **Zero audio assets.** Every sound is synthesised in WebAudio: an
  ambient pad of four detuned oscillators under a lowpass filter, pentatonic
  plells that climb with the combo, a chain shimmer, a low thud for a lost
  heart, a bell for a mote.
* **Juice.** Screen shake on chains and heart loss, a red flash when the
  heart is struck, particle bursts on planting and killing, floating combo
  text, expanding shockwave rings, and a soft bloom on every plant.
* **Readability.** One glyph per Blight type, one colour per tier, the root
  network drawn as glowing lines so you can *see* what is connected, and
  dormant plants desaturated so a broken grove is obvious at a glance.
