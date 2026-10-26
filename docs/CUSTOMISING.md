# Making it yours

Lumen Grove is a complete, shippable game. Most licensors of this kind hand
you a folder of spaghetti and a "good luck". This one is built so that the
parts you would want to change are **data, not logic**, and every one of them
lives in a file you can find in about thirty seconds.

This guide is the map of those files.

---

## The rule that makes everything else possible

> **The simulation owns the state. The renderer reads it.**

`src/game/` contains the entire rule set and **imports nothing from Phaser**.
`src/phaser/` draws whatever `src/game/` says. `Run.drainEvents()` is the only
channel from rules to effects, and the renderer never writes game state.

Two things fall out of this:

1. You can run the whole game headlessly at thousands of times real speed —
   that is what the balance harness does (`docs/HARNESS.md`).
2. You can replace the entire renderer without touching a single rule.

If you add a mechanic, put it in `src/game/`. If you add a visual, put it in
`src/phaser/` or `src/ui/`. Do not blur it.

---

## 1. The theme: colour and type

**`src/style.css`** — the entire UI palette, in one `:root` block:

```css
:root {
  --void:  #070b1c;   --deep:  #0d1430;
  --mint:  #7ff0d0;   --mint-dim: #4fbfa2;
  --gold:  #ffd28a;   --rose:  #ff6b81;
  --violet: #a98bff;  --ink:   #e9f1ff;
  --muted: #8ea0c8;
}
```

Change those eight lines and the whole game becomes a different game visually.
The dusk palette (deep navy → mint → gold) is the mood; a warmer or colder
palette is a few minutes' work.

Type is system-stack. Nothing is loaded from a font CDN, so there is no
network dependency and no FOUT.

---

## 2. The plants: flora tiers

**`src/game/content.ts` → `FLORA`**

Ten tiers, each a single line of pure data:

```ts
{ tier: 1, name: 'Sapling', damage: 11, interval: 1.15, bolts: 1, hp: 30,
  color: 0x5fd6a8, core: 0xc7fff0, petals: 0 },
```

| Field | What it does |
| --- | --- |
| `damage` | Damage per bolt |
| `interval` | Seconds between volleys |
| `bolts` | Bolts per volley |
| `hp` | How long it survives being chewed |
| `color` / `core` | The plant's tint and its bright centre |
| `petals` | How many petals the procedural flower grows |

The plant is **drawn from these numbers** — stem, leaves, bulb and a rotating
petal ring, tinted by tier. Change `color` and you have a different species
without touching a renderer.

**The curve is deliberately flat.** A tier 7 plant is strong, but not 40× a
tier 1 plant. If the curve is too steep, one tall tower beats every other
strategy and the interesting decisions disappear. That is why `BASE_MAX_TIER`
is 9 and the in-run cap rises to 10 only with the Deep Roots branch.

To add a tier, append a row and raise `BASE_MAX_TIER`. Then re-run the
harness — `npx tsx tools/balance.ts` — because a new tier changes the
ceiling of what the player can output and therefore the whole difficulty
curve.

---

## 3. The enemies: blight types

**`src/game/content.ts` → `BLIGHTS`**

Five types, unlocked on a set night:

| Type | From night | Trait |
| --- | --- | --- |
| Mote | 1 | fast, fragile, cost 1 |
| Crawler | 1 | the baseline |
| Wisp | 3 | fast, erratic (high sway) |
| Splitter | 5 | bursts into 3 Motes on death |
| Husk | 7 | slow armoured wall, cost 9 |

Each has `hp`, `speed`, `chew`, `bounty`, `score`, `cost` (how much of the
night's blight-point budget it consumes), `fromNight`, `radius`, `body`,
`eye` and `sway`.

**`cost` is the important one.** A night's threat is a *budget*, and each
enemy spends from it — so a Husk is genuinely expensive and a Mote is nearly
free. Change `cost` and you change the texture of a night, not just its
size.

To add a type: add the row, add the id to `BlightType`, add it to
`BLIGHT_ORDER`, and draw it in `src/phaser/view/field.ts`.

---

## 4. The meta game: branches, levels and the path

**`src/game/content.ts`**

### Branches (six, bought with Sparks)

```ts
{
  id: 'light', name: 'Keen Light', desc: '+12% bolt damage.',
  max: 5, costs: [10, 16, 23, 31, 42],
  summary: (l) => `+${12 * l}% bolt damage`,
}
```

`summary` is what the Grove screen shows for the level you have **and** the
level you are about to buy — so the player always knows what the next level
gives them. `costs` is an array, one entry per level, and the levels are
bought in order.

### Grove Levels (the climb)

```ts
export const GROVE_LEVEL_COSTS = [0, 15, 34, 57, 84, 115, 150, 189, 232, 279, 330, 385, 444];
```

Escalating lifetime Sparks — each level costs six more than the last. This
was flat at one point, and the consequence was that a player maxed the entire
tree in six runs. A flat cost is a cheap ladder is a finished game.

### Grove stages (the illustration)

```ts
export const GROVE_STAGES = [
  { level: 1, name: 'Seed' }, { level: 3, name: 'Sprout' },
  ...
];
```

Seven named forms. Each one has a hand-built procedural SVG in
`src/ui/illustrations.ts` — a seed in the soil, a sprout, then a tree that
gains height, branches, canopy and blossoms as the colour climbs from teal
through mint to white-gold.

### The Grove Path (twelve one-time milestones)

**`src/game/path.ts`** — each milestone is `{ id, name, desc, reward, icon,
value, goal, unit }`, where `value` reads a stat off the finished run. The
Path screen shows every milestone with its illustration, its reward, and —
for the next one — exactly how close the last run got (`7 / 10 nights`).

That "how close did you get" line is the single strongest retention element in
a score game. Keep it.

---

## 5. The daily challenge

**`src/game/daily.ts`**

The daily is derived from the date. No server, no account, no network call:
the grove, the target and the night's modifier are all hashes of `YYYY-MM-DD`.

```ts
export const DAILY_MODS = [
  { name: 'STILL NIGHT',    count: 0.85, hp: 1.0, desc: '...' },
  { name: 'SWARM',          count: 1.30, hp: 1.0, desc: '...' },
  { name: 'THICK BRAMBLE',  count: 1.0,  hp: 1.3, desc: '...' },
  { name: 'HUNGRY DARK',    count: 1.15, hp: 1.1, desc: '...' },
];
```

The target band is a constant in the same file. **Calibrate it against real
runs** — `npx tsx tools/balance.ts daily` prints what a careful run actually
scores under each modifier, at a fresh, a mid and a maxed save. A target below
the score of one run is not a challenge.

---

## 6. The grid, the layout and the heart

**`src/core/tuning.ts`**

```ts
export const DESIGN_W = 720;
export const DESIGN_H = 1280;
export const GRID_COLS = 6;
export const GRID_ROWS = 8;       // rows 0..6 are plantable, row 7 is the heart band
export const HEART_COLS = [2, 3]; // the two cells holding the Seedheart
```

The canvas scales to fit; all layout maths uses these constants. Changing
`GRID_COLS` to 7 gives you seven lanes. **Then re-run the harness** — the
number of lanes changes how much income a full grove generates, which changes
the whole economy.

Tier is derived from height: `heightTier(row) = BASE_ROW - row`. The bottom
row is tier 1 (cheap, and the wall); the top row is the strongest and the most
exposed. That single relationship is the game's core tension, so think
carefully before you break it.

---

## 7. The audio

**`src/audio/synth.ts`** — every sound is synthesised in WebAudio at runtime:

* an ambient pad of four detuned oscillators under a lowpass filter
* pentatonic plells that climb with the combo
* a chain shimmer
* a low thud for a lost heart
* a bell for a mote

There are no audio files, so there is nothing to license and nothing to load.
Rewriting this file is how you rescore the game — see the existing synths for
the patterns.

---

## 8. The art

**`src/phaser/view/`** — every sprite is drawn procedurally:

| File | What it draws |
| --- | --- |
| `textures.ts` | The radial glow, the spark, the ring — generated textures |
| `field.ts` | The plants, the blight, the root network, the projectiles |
| `backdrop.ts` | The dusk sky, the drifting fireflies, the distant spires |
| `fx.ts` | Screen shake, particle bursts, shockwave rings, floating combo text |

**`src/ui/illustrations.ts`** — the SVG illustrations for the tree stages, the
branch icons and the milestone icons.

This is the biggest single advantage of the project: there is **no asset
pipeline**. You can change the whole look of the game from a palette and a
handful of numbers, and your build stays under 400 KB.

---

## 9. The app shell

**`src/app.ts`** wires meta state, the menu → run → game-over loop, and the
save. **`src/ui/controller.ts`** is the DOM HUD: menus, the Grove screen, the
Path screen, the upgrades and the game-over card.

Saves live in `localStorage` under `lumen-grove:meta:v4`. Older versions
migrate forward automatically. **If you change the save shape, bump the
version and add a migration** — or you will break every existing player's
progress on your next release.

---

## 10. The deployment shape

* `npm run build` → `tsc --noEmit && vite build` → `dist/`
* `base: './'` in `vite.config.ts`, so the build works from a sub-path
* `public/sw.js` is the service worker — it makes the game installable and
  playable offline on a phone
* `public/manifest.json` is served as `application/json`, **not**
  `manifest.webmanifest` — some hosts serve unknown extensions as
  `application/octet-stream`, which Firefox and Safari reject
* `tools/make-icons.mjs` rasterises `public/icon.svg` into the PNG sizes phones
  want. Chrome needs a real raster icon to consider the game installable, and
  iOS ignores an SVG `apple-touch-icon`.

After you change anything, run the full gate:

```bash
npm run build
npx tsx tools/simtest.ts
node tools/playtest.mjs http://127.0.0.1:4173
node tools/manifest-check.mjs http://127.0.0.1:4173
```

---

## A short list of things not to do

1. **Do not make blight health scale to 100% of the player's power.** Use a
   fraction (`powerExponent: 0.4`) so progression buys a bounded number of
   extra nights. At full parity a maxed grove hit 94 nights instead of 13.
2. **Do not make wave size the threat.** Wave size is a pacing dial
   (`budgetGrowth: 1.0`). Threat is the arrival *rate* against the time to
   kill one.
3. **Do not make the meta ladder cheap.** A flat Grove Level cost made the
   whole tree finishable in six runs.
4. **Do not give the player a second free income.** Bloom has to stay scarce
   enough that holding a full grove and promoting it are mutually exclusive.
5. **Do not let a run's progression finish before the run does.** The harness
   asserts this directly.
6. **Do not skip the harness.** Every one of those numbers was found by a
   sweep, and every one of them was a bug before it was a feature.

---

*Licensed under the Lumen Grove Source License — see `LICENSE.md`. Ship it
commercially. Do not redistribute the source.*
