# Orbit Breaker

Flick a comet through gravity wells. Single HTML file, no build step, no
dependencies, no external network requests.

## Controls

| Action | Touch | Keyboard |
|---|---|---|
| Aim and fire | Drag back anywhere, release | — |
| Pause | Pause button | `Esc` or `P` |
| Mute | Speaker button | `M` |

Drag *away* from where you want to go, slingshot style — that keeps your thumb
off the part of the arena you're trying to read.

## The hook — why this isn't a slingshot game with gravity in it

Plenty of games let you fling something at targets. The difference here is what
you're rewarded for: **the multiplier comes from completing an orbit.**

Every full lap the comet makes around a planet raises the multiplier on
everything it goes on to shatter. So the good shot isn't the one aimed at a
crystal — it's the one aimed *past* a planet, tight enough to be caught, fast
enough not to fall in, so it wraps round once or twice and then breaks out into
the field with a ×3 on it. The game is named after the strategy.

Three things keep that honest:

- **A lap has to be a real lap.** The comet must stay inside the planet's halo
  for the whole turn and pass genuinely close at some point. Without that rule a
  shot could ricochet off all four walls, technically wind a full turn around a
  distant planet, and bank the multiplier with no gravity involved at all —
  which is exactly what the zero-gravity test build did, and *more often* than
  the real one.
- **The halo shows the rule.** The ring drawn round each planet is precisely the
  radius inside which a lap counts. Nothing hidden.
- **The lap is drawn as it happens** — a bright arc sweeps round the planet as
  the angle accumulates, so a lap that nearly made it is visibly a lap that
  nearly made it.

**Orange planets pull, blue ones push** — and they carry inward and outward
arrows, so they're distinguishable without relying on colour. The core swallows
comets, so gravity is both the tool and the hazard.

## The economy

**Orbs are your lives, and every shot costs one.** You get them back by playing
well:

- clearing a field: **+2**
- shattering four or more crystals in a single shot: **+1**
- a **gold crystal** (a star, not a hexagon — different silhouette, not just a
  different colour): **+1**

Run out and the run ends. The hollow pips in the orb row show the ceiling you
could refill to.

Later fields add armoured crystals that take two hits and deflect the comet,
planets that drift, and pushers.

## Difficulty

Three tiers, tuned around **aim tolerance** — the angular window at the launcher
that still connects with a crystal, roughly `2·atan(r / d)`. At a typical
400-unit shot:

| | Aim window | Crystal | Orbs (max) | Flight | Score |
|---|---|---|---|---|---|
| **Cruise** | ~8.0° | r 28 | 6 (9) | 9.5s | ×0.7 |
| **Drive** | ~6.0° | r 21 | 5 (8) | 8.0s | ×1.0 |
| **Redline** | ~4.3° | r 15 | 4 (6) | 7.0s | ×1.45 |

**Gravity strength is the same on Cruise and Drive.** An earlier version made
the easy tier's gravity weaker, which sounds helpful and isn't: it made the
game's signature mechanic hardest for exactly the players most likely to need it
explained. Ease comes from bigger targets, more orbs and longer flights instead.
Redline gets 1.22× gravity and more planets.

Nothing about the input changes between tiers — same drag, same power curve,
same trajectory preview. Difficulty comes from the world.

Arena and crane geometry are **fixed logical sizes**, not viewport fractions, so
a 9:20 phone and a 16:9 desktop play identically.

Each tier keeps its own best score and deepest field. The game also suggests a
tier: three short runs offers the gentler one, six fields cleared offers the
faster one, each once.

## Playables compliance notes

- **Initial load ~70 KB**, one file. Limit is 30 MB.
- **Zero external requests.** All art drawn procedurally on canvas, all audio
  synthesised with Web Audio. Verified in `test/run.js`.
- **No copyrighted assets** — no image or audio file in the bundle.
- **Scales to 1:1, 16:9 and 9:16.** Screenshots in `test/shots/`.
- **60 fps** at phone and desktop resolutions, measured under load.
- **`firstFrameReady()` then `gameReady()`**, in that order.
- **Pause and mute obeyed immediately.**
- **Progress saved through `saveData` / `loadData`**, localStorage only as fallback.
- **No ads wired up yet.**

## Repo layout

```
index.html                      the whole game
.nojekyll                       serve files as-is
orbit-breaker-playables.zip     bundle for the developer portal
src/body.html                   source of truth
build.js                        wraps src/body.html into index.html
test/run.js                     aspect ratios, external requests, pause, perf
test/sdk.js                     integration against a mocked ytgame SDK
test/gameplay.js                orb economy, gravity differential, per-tier bests
test/probe.js                   measures how often a random shot completes an orbit
test/shot.js                    screenshot capture
```

`node build.js` rebuilds. `node test/gameplay.js 2` runs one section.

### Testing a physics game without debug hooks

The shipped build has no test affordances, so the tests lean on an identity that
can only hold if the economy works: every orb buys exactly one shot, and the run
ends when the last is spent, so on the game-over card

```
shots fired  ===  starting orbs  +  orbs earned
```

That single equation covers per-tier starting orbs, spending, field-clear
refunds, chain bonuses and gold crystals at once.

Gravity itself is tested as a **differential**: the same build with the wells
switched off completed **zero** orbits over 61 shots, while the real build
completed them over the same number. That comparison is also what caught the
wall-bouncing loophole described above — the flawed version scored *better*
without gravity, which is how the bug announced itself.

`test/probe.js` answers a design question rather than a correctness one: what
fraction of *random* shots complete an orbit? It sits around 9–15% per tier,
which was the number used to tune gravity strength, launch speed range and the
length of the trajectory preview. An early build sat at 3%, which would have
made the game's whole premise something most players never discovered.

## Performance notes

Three games in, the same lesson keeps arriving in a new costume: **large
alpha-blended blits and gradient fills are what cost frames**, and each game has
one you don't expect.

- Neon Drift: per-frame `shadowBlur`, then per-frame gradients.
- Tower Tilt: a full-screen radial sky gradient every frame.
- Orbit Breaker: **filtered upscales.** Stretching a small background texture
  over the screen costs per *destination* pixel, and was as expensive as the
  giant semi-transparent planet halos. Caching the background at real canvas
  resolution and blitting it 1:1, and authoring the halo at the exact size it's
  drawn, took 16:9 from 30 to 60 fps.

A bisect harness beat guessing: disable one draw phase at a time and measure.
The first two guesses here were both wrong.
