# Zuma Clone — Game Design Document

**Project:** ball-chain-clone  
**Reference Game:** Zuma Deluxe (PopCap Games, 2003)  
**Document Version:** 1.2  
**Date:** 2026-04-03

---

## Table of Contents

1. [Overview](#1-overview)
2. [Core Gameplay Loop](#2-core-gameplay-loop)
3. [Game Objects](#3-game-objects)
4. [Controls](#4-controls)
5. [Level Design](#5-level-design)
6. [Power-Ups and Special Balls](#6-power-ups-and-special-balls)
7. [Scoring System](#7-scoring-system)
8. [Win / Lose Conditions](#8-win--lose-conditions)
9. [Game Modes](#9-game-modes)
10. [Progression System](#10-progression-system)
11. [Audio / Visual Style](#11-audio--visual-style)
12. [Technical Architecture](#12-technical-architecture)
13. [Testing Strategy](#13-testing-strategy)
14. [Debug Mode](#14-debug-mode)
15. [Milestones](#15-milestones)
16. [Open Questions](#16-open-questions)

---

## 1. Overview

A browser-based clone of PopCap's Zuma Deluxe. The player controls a stationary
frog idol that rotates 360° and fires colored balls into a moving chain of balls.
Matching 3+ consecutive same-color balls removes them from the chain. The goal is
to clear the chain before any ball reaches the skull at the path end.

**Target platform:** Web browser (WebGL via Three.js)  
**Proposed tech stack:** TypeScript + Three.js (no physics engine)  
**Target audience:** Casual players familiar with match-3 / marble-shooter genres

---

## 2. Core Gameplay Loop

```
START LEVEL
   │
   ▼
Balls spawn at path entry and travel toward skull
   │
   ▼
Player rotates frog, aims, fires current ball
   │
   ├── Ball hits chain → insertion point determined
   │       │
   │       ├── 3+ same-color neighbors? → POP group → check chain reaction
   │       │       │
   │       │       └── Gap closes: sides match again? → cascade pop
   │       │
   │       └── No match → ball inserted, chain continues
   │
   ├── Ball misses chain → no effect
   │
   ├── Power-up ball collected → apply effect
   │
   └── All balls cleared? → LEVEL COMPLETE
           │
           No → any ball reached skull? → LOSE LIFE → retry / game over
```

**Key tension mechanic:** The chain continuously advances toward the skull.
As the leading ball approaches, the skull visually opens across 5 states,
escalating urgency. A careless insertion near the skull can push the head ball
past `pathT = 1.0` and cost a life.

**Cascade mechanic:** A pop opens a gap. The front segment zips back (away from
skull) to close it. When the gap closes, if the newly adjacent balls form a 3+
match, another pop triggers — and so on. The chain is frozen during zip-back;
the player can still fire into zipping segments. There is no fire lockout at
any point.

---

## 3. Game Objects

### 3.1 The Frog (Shooter)

| Property | Value |
|---|---|
| Position | Per-level `{x, y}` anchor defined in level JSON — may be off-center; independent of spawn point |
| Rotation | 0–360°, instantaneous — follows mouse/touch with no smoothing lag |
| Firing arc | Full 360°, no restriction |
| Current ball | Displayed in frog's mouth |
| Next ball | Displayed on frog's back |
| Queue depth | 3 balls internally (current, next, reserve) — only current and next are displayed |
| Swap | Player may swap current ↔ next at any time |
| Fire animation | Single-frame shoot pose held for fire throttle duration, returns to idle |
| Fire throttle | Fixed constant `FIRE_THROTTLE_MS` (tunable) — resets on each shot |

**Ball queue:** The frog maintains a 3-ball internal queue. Only the current (mouth)
and next (back) balls are shown. When the player fires:
1. Current ball launches as projectile.
2. Next → current (shown in mouth).
3. Reserve → next (shown on back).
4. New ball generated for reserve (not yet visible).

**Smart color generation:** When `ballsRemaining > LOW_BALL_THRESHOLD` (default 10,
counts all unpopped balls including not-yet-spawned), new balls are generated from
the full color set for the level. When `ballsRemaining ≤ LOW_BALL_THRESHOLD`, new
balls are restricted to colors still present among remaining unpopped balls —
ensuring the player always has a useful shot in the final stretch.

**Multiple projectiles:** Multiple balls can be in flight simultaneously. The fire
throttle limits how quickly the player can fire but does not prevent a second
projectile while the first is still travelling. Each projectile resolves
independently on collision.

### 3.2 Ball Chain

- Balls are stored as an ordered list sorted by `pathT` ascending (index 0 = tail
  at spawn, last index = head at skull).
- Each ball has: `color`, `pathT`, `powerUp | null`.
- Balls only move forward because they are pushed from behind — no push means
  stationary. See §12.6 for full movement model.
- **Chain pauses** during any zip-back — spawner stops pushing until all
  gap-closes complete.
- Balls never move toward spawn except during zip-back or Reverse power-up.

### 3.3 Ball Colors

| Stage Range | Colors Available |
|---|---|
| Stages 1–3 | Red, Green, Blue, Yellow (4 colors) |
| Stages 4–6 | + Purple (5 colors) |
| Stages 7+ | + White (6 colors) |

Color count directly controls match difficulty. Fewer colors = easier to form
chains accidentally; more colors = deliberate aim required.

### 3.4 The Skull

- Located at path end (`pathT = 1.0`). Responds to the **absolute leading ball** only.
- 5 visual open states driven by leading ball `pathT`:

| State | pathT range |
|---|---|
| Closed | < 0.75 |
| Quarter open | 0.75 – 0.84 |
| Half open | 0.84 – 0.89 |
| Open | 0.89 – 0.95 |
| Fully open | ≥ 0.95 |

- When the head ball reaches `pathT >= 1.0`: the entire chain accelerates and
  flushes into the skull (toilet-flush animation). All in-flight projectiles
  disappear with animation. Life is decremented. Level restarts.

### 3.5 Projectile Ball

- Fired from the frog's position (not the spawn point) in the aimed direction.
- Straight-line travel, no arc, no gravity.
- Circle-overlap collision detection against all chain balls each tick.
- On miss: disappears off-screen. May be recycled into queue as optimization.
- On game over: all in-flight projectiles play disappear animation and are removed.
- Multiple projectiles can be in flight simultaneously.

---

## 4. Controls

### Mouse / Touch (primary)

| Action | Input |
|---|---|
| Aim | Move mouse / drag touch toward target |
| Fire | Left click / tap |
| Swap balls | Right click / two-finger tap |

### Keyboard (secondary)

| Action | Key |
|---|---|
| Fire | Space |
| Swap balls | Shift / S |
| Pause | Escape / P |

---

## 5. Level Design

### 5.1 Path System

Paths are defined as a sequence of **cubic Bézier curve segments** or a
**polyline** with smooth interpolation. At runtime the path is pre-sampled into
an arc-length-parameterized lookup table so that balls travel at uniform screen
speed regardless of curve curvature.

Each level JSON includes:
- `path`: array of control points
- `spawnInterval`: ms between new balls entering the chain
- `chainLength`: total balls to spawn before the tail enters
- `baseSpeed`: initial chain speed (path-units / second)
- `speedAcceleration`: speed increase per second
- `colors`: array of allowed colors for this level
- `frogAnchor`: `{x, y}` position of frog on screen

### 5.2 Level Progression

| Stage | Levels | New Feature |
|---|---|---|
| 1 | 4 | Tutorial — single S-curve, 4 colors, slow speed |
| 2 | 4 | Tighter curves |
| 3 | 4 | Speed increase |
| 4 | 6 | 5 colors introduced |
| 5 | 6 | Dual-track path (two chains) |
| 6 | 6 | Power-ups appear |
| 7 | 7 | 6 colors introduced |
| 8–12 | 7 each | Compound curves, higher speed, shorter gaps |
| 13 (Final) | 1 | "Space" — very long chain, no visible path, max difficulty |

### 5.3 Dual-Track Levels

The 5th level of each stage group features **two independent ball chains** on
separate paths. Both must be cleared to complete the level. The frog can target
either chain. Each chain has its own skull.

---

## 6. Power-Ups and Special Balls

### 6.1 Embedding and Activation

Power-up balls are embedded directly in the chain as regular balls that also
carry a `powerUp` tag. They look like normal colored balls but display a glowing
icon overlay.

**Activation rules:**
- A power-up activates **only** when its ball is part of a group of 3+ that pops.
- If its group is destroyed by a cascade (not a direct shot), it still activates.
- If its group has fewer than 3 of the same color and doesn't pop, the power-up
  ball stays in the chain — it is never "collected" without a pop.
- Only one power-up per chain group. Groups are never seeded with two power-ups.

**Spawn rules:**
- `powerUpFrequency` (per-level JSON field, default `0.08`) is the probability
  that any given ball spawned into the chain carries a power-up.
- The type is chosen uniformly at random from the enabled set for that level
  (early stages may exclude Laser and Bomb).
- A minimum gap of 8 balls is enforced between consecutive power-up balls so
  they don't cluster.

### 6.2 Power-Up Reference

#### Slow

| Field | Value |
|---|---|
| Duration | 10 s |
| Target | The chain it was embedded in |
| Effect | Multiplies that chain's current speed by `0.5` |
| Stacking | A second Slow resets the 10 s timer; speed multiplier does not compound |
| Expiry | Speed ramps back to normal over 1 s (linear interpolation) |

#### Reverse

| Field | Value |
|---|---|
| Duration | 8 s |
| Target | The chain it was embedded in |
| Effect | Inverts chain velocity — chain moves backward (away from skull) |
| Stacking | A second Reverse resets the 8 s timer |
| Interaction with Slow | Both can be active simultaneously. Reverse takes priority on direction; speed is still halved if Slow is also active |
| Expiry | Chain resumes forward motion at normal speed immediately on expiry |
| Multi-chain | Only the chain containing the power-up ball is affected |

#### Lightning

| Field | Value |
|---|---|
| Duration | Instant |
| Target | All chains |
| Effect | Removes every ball in every chain whose color matches the **fired projectile's color** at the moment of activation |
| No-match case | If no balls of that color exist in any chain, the power-up is consumed with no effect (shows a visual flash but nothing pops) |
| Score | Each removed ball scores 10 pts; chain bonus applies as if they were popped sequentially |
| Cascade | After removal, gaps are closed normally — if closing a gap creates a 3+ match, cascade proceeds as usual |

#### Laser

| Field | Value |
|---|---|
| Duration | 1 shot |
| Target | Whichever chain the next projectile hits |
| Effect | The next fired ball travels through the entire chain along its flight path, popping every ball it passes through whose color matches the laser ball's color |
| Non-matching balls | Passed through without effect; they remain in the chain |
| Insertion | The laser ball is NOT inserted into the chain; it passes through completely |
| Score | Each popped ball scores normally; gap-shot multiplier applies for each gap the ball passes through |
| Expiry | If the player swaps balls after Laser activates, the Laser effect transfers to the new current ball |
| Miss | If the next shot misses the chain entirely, the Laser effect is consumed and lost |

#### Bomb

| Field | Value |
|---|---|
| Duration | Instant |
| Target | The chain it was embedded in |
| Effect | Removes all balls within a radius of 7 ball-widths from the bomb ball's position at the moment of pop |
| Radius measurement | Physical screen distance, not index count — curved paths mean the radius may affect non-contiguous index ranges |
| Score | Each removed ball scores 10 pts + chain bonus |
| Cascade | Gaps close after removal; new matches trigger cascade |
| Multi-chain | Does NOT affect other chains, even if physically nearby |

#### Accuracy

| Field | Value |
|---|---|
| Duration | 15 s |
| Target | Player UI |
| Effect | Renders a preview line showing where the current projectile will land and which ball it will insert next to; also highlights the resulting color group size |
| Visual | Dotted trajectory line from frog to insertion point; affected ball group outlined |
| Stacking | A second Accuracy resets the 15 s timer |
| No gameplay effect | Purely informational; does not change chain or score behavior |

#### Color Change

| Field | Value |
|---|---|
| Duration | Instant |
| Target | A cluster of balls around the activation point |
| Effect | Converts all balls within 5 ball-widths of the power-up ball's position to a single color |
| Color chosen | The color of the ball that completed the 3+ match triggering the pop (the fired ball's color) |
| Result | The converted cluster may immediately form a 3+ match with neighbors, triggering a cascade |
| Multi-chain | Only affects the chain the power-up was embedded in |

### 6.3 Simultaneous Power-Ups

Multiple power-ups can be active at the same time. Rules for simultaneous effects:

| Combination | Behavior |
|---|---|
| Slow + Reverse | Both active: chain moves backward at half speed |
| Two Slows | Timer resets; speed multiplier stays at 0.5 (no compounding) |
| Two Reverses | Timer resets; direction still reversed |
| Slow + Lightning | Lightning pops its targets; Slow continues unaffected |
| Any timed + Instant | Instant resolves immediately; timed continues ticking |

### 6.4 PowerUpSystem Internals

```typescript
interface ActivePowerUp {
  type: PowerUpType;
  chainIndex: number;   // which chain it belongs to (-1 = global)
  remainingMs: number;  // 0 for instant (already resolved)
}

class PowerUpSystem {
  private active: ActivePowerUp[] = [];

  // Called by ChainManager when a group containing a power-up pops.
  activate(type: PowerUpType, chainIndex: number, context: ActivationContext): void;

  // Called each tick. Decrements timers, removes expired effects, returns
  // any speed/direction overrides for ChainManager to apply.
  update(dt: number): PowerUpOverrides;
}

interface PowerUpOverrides {
  // Per-chain speed multipliers (1.0 = normal)
  speedMultipliers: number[];
  // Per-chain direction (-1 = reversed, 1 = forward)
  directions: number[];
  // Whether Accuracy overlay should render
  showAccuracy: boolean;
  // Whether next projectile should be a Laser shot
  laserArmed: boolean;
}
```

`GameLoop.tick()` calls `PowerUpSystem.update(dt)` and passes the returned
`PowerUpOverrides` into `ChainManager.update(dt, overrides)` so chain movement
is adjusted without `ChainManager` knowing anything about power-up state directly.

---

## 7. Scoring System

### 7.1 Base Score

| Event | Points |
|---|---|
| Ball popped (base) | 10 pts each |
| Coin collected | 500 pts |

### 7.2 Chain Bonus

Each successive pop **without missing or the chain stopping** increments the
chain counter `n`. Bonus per pop group:

```
chainBonus = 200 + (n × 10)
```

Examples:
- 10th consecutive group: +300 pts
- 20th consecutive group: +400 pts

Chain counter resets on any miss or when no new pop occurs within 3 s.

### 7.3 Combo (Cascade) Bonus

When popping one group causes a gap that immediately triggers a second (or more)
pop via chain reaction, each additional pop awards:

```
comboBonus = previousGroupScore × 1.5  (cumulative per cascade level)
```

### 7.4 Gap Shot Bonus

A gap shot qualifies when the projectile passes through at least one gap wider
than one ball diameter before hitting the chain. Each qualifying gap adds +1 to
the multiplier:

```
gapMultiplier = 1 + (number of qualifying gaps passed through)
score = (ballsPopped × 10) × gapMultiplier
```

Gap multiplier stacks independently with chain bonus:

```
totalScore = ((ballsPopped × 10) × gapMultiplier) + chainBonus
```

### 7.5 Ace Time Bonus

If a level is cleared in under the "Ace Time" threshold (defined per level in
JSON as `aceTimeSeconds`), a flat bonus is awarded:

```
aceBonus = remainingSeconds × 100
```

### 7.6 Extra Lives

One extra life awarded for every **50,000 points** accumulated.

---

## 8. Win / Lose Conditions

### 8.1 Win

- All balls in the level's chain have been popped.
- The Zuma bar (a progress meter filled by popping balls) reaches 100%.
- Level complete screen shown; score tallied with bonuses.
- Player advances to next level.

### 8.2 Lose (Life)

- Head ball reaches `pathT >= 1.0`.
- Entire chain accelerates and flushes into the skull (toilet-flush animation).
- All in-flight projectiles disappear with animation.
- Player loses 1 life.
- Level restarts from scratch.

### 8.3 Game Over

- All lives exhausted.
- Game over screen shown with final score, high score comparison.
- Option: Continue (if continue tokens available) or Return to Menu.

### 8.4 Lives

- Start: 3 lives (displayed as frog icons, top-left HUD).
- Gain: +1 life per 50,000 points.
- Max: no cap (or configurable; suggest 9 for display purposes).

---

## 9. Game Modes

### 9.1 Adventure Mode

- Linear progression through all stages and levels.
- Persistent save: stores highest completed level, high score.
- Difficulty scales automatically via level definitions.

### 9.2 Gauntlet Mode

- Unlocked after reaching Stage 3 in Adventure.
- Player selects any previously reached level as starting point.

**Gauntlet sub-modes:**

| Sub-mode | Description |
|---|---|
| Practice | Play chosen level repeatedly; no lives lost, focus on score |
| Survival | Level auto-escalates: more colors and faster speed each round; one life only |

**Gauntlet Ranks (Survival):**

| Rank | Unlock Condition | Chain Colors |
|---|---|---|
| Rabbit | Starting rank | 4 |
| Eagle | Clear 7 survival rounds | 5 |
| Jaguar | Clear 14 survival rounds | 6 |
| Sun God | Clear 21 survival rounds | 6 + max speed |

---

## 10. Progression System

### 10.1 Save State (localStorage)

```json
{
  "highScore": 0,
  "adventureProgress": {
    "stage": 1,
    "level": 1
  },
  "gauntletUnlocked": false,
  "gauntletRank": "Rabbit",
  "totalScore": 0
}
```

### 10.2 Difficulty Tuning Knobs (per level JSON)

- `baseSpeed` — starting chain velocity
- `speedAcceleration` — rate of acceleration over time
- `spawnInterval` — time between new balls entering
- `chainLength` — total number of balls
- `colors` — array of allowed ball colors (controls difficulty)
- `powerUpFrequency` — ratio of balls that carry power-ups (0.0–1.0)
- `aceTimeSeconds` — target clear time for ace bonus

---

## 11. Audio / Visual Style

### 11.1 Visual Theme

Inspired by Aztec / Mayan / Incan temple aesthetics:

- Stone-textured frog idol at screen center
- Jungle temple backgrounds: carved stone walls, torches, foliage
- Paths rendered as stone channels / grooves carved into the temple floor
- Ball designs: stone spheres with color markings / glyphs
- Skull: carved stone with animated jaw, glows red in danger zone
- UI: stone tablet frames, carved-stone font

For our clone, we can use simplified flat/vector art with the same thematic cues
(earthy palette: terracotta, jade, obsidian, gold).

### 11.2 Color Palette

| Ball Color | Hex (approximate) |
|---|---|
| Red | `#C0392B` |
| Green | `#27AE60` |
| Blue | `#2980B9` |
| Yellow | `#F1C40F` |
| Purple | `#8E44AD` |
| White | `#ECF0F1` |

### 11.3 Audio Design

- **BGM:** Loop of tribal / Mesoamerican-inspired ambient track (flutes, drums)
- **Ball pop SFX:** Crisp stone-crack sound; pitch rises slightly with chain combo
- **Skull warning:** Deep drum beat when leading ball enters danger zone
- **Power-up collect:** Short chime / glyph-activation sound
- **Level complete:** Triumphant horn fanfare
- **Game over:** Low, slow drum sequence
- **Zuma vocal cue:** Voice shout "Zumaaa!" on especially large chain clears (5+
  balls in a single cascade)

All audio can be implemented via Web Audio API with OGG / MP3 fallback.

---

## 12. Technical Architecture

### 12.1 Technology Stack

| Layer | Choice | Rationale |
|---|---|---|
| Language | TypeScript | Type safety, good IDE support |
| Renderer | Three.js (WebGL) | Hardware-accelerated 2D via orthographic camera; built-in sprite, texture, and scene-graph support |
| Build tool | Vite | Fast HMR, simple config; Three.js tree-shakes well |
| Unit testing | Vitest | Co-located with Vite; fast, native ESM |
| Coverage | Vitest + V8 provider | Target ≥ 90% line/branch coverage on all logic modules |
| E2E testing | Playwright | Cross-browser, headless; tests major gameplay flows |
| Physics engine | None | All motion is path-driven (no free-body simulation needed) |
| Asset pipeline | SVG sources → PNG textures via build script | Crisp at all resolutions, scriptable, version-controllable |

### 12.2 Module Breakdown

**Architectural rule: game logic must have zero Three.js / DOM dependencies.**
Only `Renderer`, `InputManager`, and `SpriteSheet` are allowed to import Three.js
or reference browser globals. Every other module is a pure TypeScript class or
function that can be imported directly in Vitest without a DOM environment.

```
src/
├── main.ts                    # Entry point — wires logic + renderer + input
├── GameLoop.ts                # Deterministic tick engine (see §12.6)
├── Game.ts                    # Top-level state machine (menu/playing/paused/gameover)
│
├── logic/                     # Pure game logic — NO Three.js, NO DOM
│   ├── ChainManager.ts        # Owns all BallChain instances for a level; collision dispatch, all-cleared check
│   ├── BallChain.ts           # Single chain: movement, insertion, pop, cascade
│   ├── Ball.ts                # Ball value type (color, pathT, powerUp)
│   ├── MatchSystem.ts         # 3+ match detection, cascade resolution
│   ├── ScoreSystem.ts         # All scoring rules, chain/combo/gap bonuses
│   ├── PathSystem.ts          # Bézier/polyline, arc-length parameterization
│   ├── CollisionSystem.ts     # Projectile ↔ chain hit detection
│   ├── PowerUpSystem.ts       # Power-up activation and timer management
│   ├── SpawnSystem.ts         # Ball spawning schedule
│   ├── FrogState.ts           # Frog rotation, current/next ball state
│   ├── ProjectileState.ts     # In-flight ball position/velocity
│   └── SkullState.ts          # Path-end state and open-amount calculation
│
├── renderer/                  # All Three.js code — never imported by logic/
│   ├── Renderer.ts            # Three.js WebGLRenderer, orthographic camera, scene
│   ├── ChainRenderer.ts       # Syncs BallChain state → Three.js meshes
│   ├── FrogRenderer.ts        # Frog mesh, rotation animation
│   ├── SkullRenderer.ts       # Skull mesh, open-state animation
│   ├── HUDRenderer.ts         # Score, lives, Zuma bar overlays
│   └── ParticleRenderer.ts    # Pop particle effects
│
├── input/
│   └── InputManager.ts        # Mouse, touch, keyboard → normalized InputState
│
├── audio/
│   └── AudioManager.ts        # Web Audio API wrapper
│
├── sprites/
│   ├── SpriteSheet.ts         # THREE.Texture atlas loader + createMesh()
│   └── SpriteAtlas.ts         # Atlas manifest type definitions
│
├── debug/
│   ├── DebugOverlay.ts        # On-screen panel (see §14)
│   ├── DebugConfig.ts         # Runtime-editable settings store
│   └── StepController.ts     # Pause/step/advance controls for game loop
│
├── data/
│   └── levels/
│       ├── 1-1.json
│       └── ...
│
└── utils/
    ├── Vec2.ts
    ├── BezierUtils.ts
    └── Random.ts              # Seeded, replaceable RNG (critical for test replay)

tests/
├── unit/                      # Vitest — pure logic tests, no DOM
│   ├── logic/
│   │   ├── ChainManager.test.ts
│   │   ├── BallChain.test.ts
│   │   ├── MatchSystem.test.ts
│   │   ├── ScoreSystem.test.ts
│   │   ├── PathSystem.test.ts
│   │   ├── CollisionSystem.test.ts
│   │   ├── PowerUpSystem.test.ts
│   │   ├── SpawnSystem.test.ts
│   │   └── FrogState.test.ts
│   └── utils/
│       ├── Vec2.test.ts
│       └── BezierUtils.test.ts
│
├── integration/               # Vitest — multi-system scenarios, headless
│   ├── GameLoop.test.ts       # Step-through simulation tests (see §13.3)
│   └── LevelCompletion.test.ts
│
└── e2e/                       # Playwright
    ├── gameplay.spec.ts       # Fire ball, match, score update
    ├── powerups.spec.ts       # Each power-up activates and applies correctly
    ├── skull.spec.ts          # Ball reaches skull → life lost
    ├── levelcomplete.spec.ts  # Chain cleared → level complete screen
    └── debug.spec.ts          # Debug panel opens, settings take effect
```

### 12.3 Sprite Pipeline

All visual assets are authored as SVG and rasterized to a single PNG spritesheet
at build time. The Canvas renderer draws from this spritesheet exclusively —
no raw SVG is rendered at runtime.

#### Source SVG layout

```
assets/
└── sprites/
    ├── balls/
    │   ├── ball-red.svg
    │   ├── ball-green.svg
    │   ├── ball-blue.svg
    │   ├── ball-yellow.svg
    │   ├── ball-purple.svg
    │   └── ball-white.svg
    ├── powerups/
    │   ├── powerup-slow.svg
    │   ├── powerup-reverse.svg
    │   ├── powerup-lightning.svg
    │   ├── powerup-laser.svg
    │   ├── powerup-bomb.svg
    │   ├── powerup-accuracy.svg
    │   └── powerup-colorchange.svg
    ├── frog/
    │   ├── frog-idle.svg
    │   └── frog-shoot.svg       # 1-frame shoot pose
    ├── skull/
    │   ├── skull-closed.svg
    │   ├── skull-quarter.svg
    │   ├── skull-half.svg
    │   ├── skull-open.svg
    │   └── skull-full.svg
    ├── ui/
    │   ├── life-icon.svg
    │   ├── zumabar-fill.svg
    │   └── zumabar-bg.svg
    └── bg/
        └── temple-bg.svg        # Tileable or full-scene background
```

#### Build script (`scripts/build-sprites.ts`)

Uses the `sharp` npm package (or `resvg-js`) to:

1. Render each SVG at a configurable base resolution (default: 2× for HiDPI).
2. Pack all rendered PNGs into a single `public/spritesheet.png` using a
   simple bin-packing algorithm (or `spritesmith`).
3. Emit `public/spritesheet.json` — a TexturePacker-compatible atlas manifest:

```json
{
  "frames": {
    "ball-red": { "frame": { "x": 0,  "y": 0,  "w": 64, "h": 64 } },
    "ball-green": { "frame": { "x": 64, "y": 0,  "w": 64, "h": 64 } },
    "frog-idle":  { "frame": { "x": 0,  "y": 64, "w": 96, "h": 96 } }
  },
  "meta": {
    "image": "spritesheet.png",
    "size": { "w": 512, "h": 512 },
    "scale": 2
  }
}
```

4. Script runs as a Vite plugin hook (`buildStart`) so `vite build` and
   `vite dev` both regenerate the sheet when SVGs change.

#### Runtime usage with Three.js (`SpriteSheet.ts`)

The spritesheet PNG is loaded once as a `THREE.Texture`. For each named frame,
a `THREE.MeshBasicMaterial` is created with the shared texture and a UV offset /
repeat that maps to that frame's region. Each game object owns a
`THREE.Mesh` (with `PlaneGeometry`) or a `THREE.Sprite`, updated each frame by
setting its `position` and `rotation` in world space.

```typescript
class SpriteSheet {
  private texture: THREE.Texture;
  private atlas: Atlas;

  async load(atlasUrl: string): Promise<void> { ... }

  // Returns a material configured for the named frame.
  // Caller owns the Mesh; call updateUVs() if the sprite frame changes.
  createMaterial(name: string): THREE.MeshBasicMaterial { ... }

  // Create a ready-to-add Mesh for a named sprite.
  createMesh(name: string): THREE.Mesh { ... }
}
```

- All ball, frog, skull, and UI objects are `THREE.Mesh` nodes in a single
  orthographic scene.
- The camera is a `THREE.OrthographicCamera` sized to the logical canvas
  resolution (e.g. 960 × 540), so world units == pixels.
- No perspective; z-ordering is handled by `mesh.renderOrder` or `mesh.position.z`.
- No physics engine is used — ball positions are computed entirely from path
  parameterization (`pathT`) and chain logic, then written directly to
  `mesh.position`.

#### SVG design conventions

- Balls: 64 × 64 px viewBox, circular with Aztec/Mayan glyph overlay, stone
  texture achieved with SVG filters (`feTurbulence` + `feComposite`).
- Frog: 96 × 96 px viewBox, top-down view, mouth pointing right (rotation
  applied at runtime).
- Skull: 80 × 80 px viewBox, 5 frames covering open states (closed → fully open).
- All SVGs use a consistent earthy palette (see §11.2).
- No external fonts or linked resources inside SVGs (fully self-contained).

---

### 12.4 Path Representation

Paths are stored as a flat array of points with a `type` flag per segment:

```typescript
type PathSegment =
  | { type: "line"; p0: Vec2; p1: Vec2 }
  | { type: "cubic"; p0: Vec2; p1: Vec2; p2: Vec2; p3: Vec2 };

interface LevelPath {
  segments: PathSegment[];
  // Pre-computed at load time:
  arcLengthLUT: { t: number; arcLen: number }[];
  totalLength: number;
}
```

`PathSystem.tFromArcLength(len)` maps a physical distance along the path to the
normalized parameter `t ∈ [0, 1]`, enabling uniform-speed ball movement.

### 12.5 ChainManager API

`ChainManager` is the sole interface the rest of the game uses for anything
chain-related. It owns 1–2 `BallChain` instances depending on the level definition
and routes all calls to the correct one.

```typescript
class ChainManager {
  private chains: BallChain[];

  constructor(paths: LevelPath[], rng: Random) { ... }

  // Advance all chains by dt milliseconds.
  update(dt: number): void;

  // Test a projectile against all chains. Returns the hit chain + insertion
  // index, or null if no hit. GameLoop calls this; never calls BallChain directly.
  checkCollision(projectile: ProjectileState): HitResult | null;

  // Insert a ball into the chain identified by the HitResult.
  insert(hit: HitResult, ball: Ball): void;

  // True when every chain has been fully cleared.
  allCleared(): boolean;

  // The leading ball's pathT across all chains (used for skull animation).
  maxPathT(): number;

  // Apply a power-up effect to the relevant chain(s).
  applyPowerUp(effect: PowerUpEffect): void;
}

interface HitResult {
  chainIndex: number;   // which chain was hit
  insertIndex: number;  // position within that chain's ball array
  side: "before" | "after";
}
```

`BallChain` itself remains a single-responsibility class with no knowledge of
other chains. `ChainManager` is the only place multi-chain logic lives, keeping
both classes easy to test independently.

### 12.6 Chain Movement Model

#### Orientation

```
SPAWN POINT  →  →  →  →  →  →  →  SKULL
   pathT=0   [tail]  ...  [head]   pathT=1
             (back)       (front)
```

- **Front / head** — the leading ball, highest `pathT`, closest to the skull.
- **Back / tail** — the last spawned ball, lowest `pathT`, closest to spawn.
- Balls travel from `pathT=0` toward `pathT=1` (spawn → skull).
- The chain array is sorted by `pathT` ascending: `index 0 = tail`, `last index = head`.

#### Ball representation

Each ball is an independent object with its own `pathT`. There is no shared
chain velocity — a ball only moves because something behind it pushes it.

```typescript
interface Ball {
  color: BallColor;
  pathT: number;        // 0.0 = spawn point, 1.0 = skull
  powerUp: PowerUpType | null;
}
```

#### Push-from-back movement

The spawner pushes the tail ball forward by `pushSpeed * dt` each tick. A ball
passes that push to the ball in front of it only when the gap between them
reaches `minSpacing` (= 1 ball diameter in path units). If a gap exists ahead
of a ball, that ball **does not move** — there is nothing in front to push
against and nothing behind has reached it yet.

```
tick():
  // Push tail forward from spawner
  balls[0].pathT += pushSpeed * dt

  // Propagate push through the chain toward the head
  for i from 1 to lastIndex:
    gap = balls[i].pathT - balls[i-1].pathT
    if gap <= minSpacing:
      // In contact — push propagates
      balls[i].pathT = balls[i-1].pathT + minSpacing
    else:
      // Gap exists ahead — this ball and everything beyond it is stationary
      break
```

Key consequences:
- A **compressed segment** (no gaps within it) moves entirely at `pushSpeed`.
- A **free-floating front segment** (gap behind it) is **stationary** — it
  receives no push and does not move toward the skull on its own.
- The front segment being stationary is **good for the player** — the leading
  ball stops advancing toward the skull while the gap exists.

#### Gap definition

A gap exists between balls `i` and `i+1` when:

```
balls[i+1].pathT - balls[i].pathT > minSpacing
```

(where `i+1` is closer to the skull / head side)

Gaps arise from two sources:
1. **Spawn gaps** — the spawner emits balls at intervals; newly spawned balls
   have not yet pushed into contact with the rest of the chain. These are
   uncommon in practice but are treated identically to pop gaps — Case 2
   applies if the balls on either side happen to be the same color.
2. **Pop gaps** — when a group is removed, the front segment is no longer in
   contact with the back segment and becomes stationary.

#### Gap close rules

A gap closes in exactly **two** cases:

**Case 1 — Pop.**
When a group of 3+ same-color balls is removed, a gap opens. The front segment
is now stationary. The **front segment rushes backward** (toward the spawn
point, away from the skull) at `gapCloseSpeed` to meet the stationary back
segment.

**Case 2 — Color match across a gap.**
After every state change (insertion, pop, movement tick), every gap in the
chain is checked. If the balls on **either side** of a gap are the same color,
the gap triggers a close — the **front segment rushes backward** at
`gapCloseSpeed` to meet the back segment, identical to Case 1. This applies
to all gap types including spawn gaps.

This check runs across **all gaps** every tick while any gap-close is in
progress, not just the most recently created gap.

#### Front segment zip-back

When a gap-close triggers, the front segment moves **backward** (decreasing
`pathT`) at `gapCloseSpeed` — a fixed constant, always the same speed
regardless of chain speed, level, or gap size. This is the "zip back" the
player sees — the leading balls visibly pulling away from the skull toward the
spawn side. This is the primary moment of relief after a successful shot.

`gapCloseSpeed` is much faster than `pushSpeed` to feel snappy. Suggested
default: `gapCloseSpeed = 8 × pushSpeed` (tunable per level via `gapCloseSpeed`
field in level JSON).

```
gapClose tick():
  frontSegmentHead.pathT -= gapCloseSpeed * dt
  // propagate backward through front segment (maintain minSpacing between all balls)
  for each ball in front segment (head → tail direction):
    maintain minSpacing from ball ahead
```

The back segment does **not** change speed during a gap close — it continues
being pushed at `pushSpeed` from the spawner as normal.

When the front segment's tail ball reaches `minSpacing` of the back segment's
head ball, the gap is closed. The front segment stops its backward motion and
resumes being pushed forward at `pushSpeed` by the back segment.

**No minimum distance from skull** — zip-back applies even when the front
segment is very close to `pathT=1.0`. The front segment zips back as far as
needed to meet the back segment, even if it starts adjacent to the skull.

**Reverse power-up interaction** — zip-back still applies when Reverse is
active. If a gap-close triggers during Reverse, the front segment zips back
at `gapCloseSpeed` regardless of the chain's current direction. Both motions
are additive: the front segment moves backward both from the Reverse effect
and from the zip-back simultaneously.

#### Multiple simultaneous gaps

Multiple gaps can exist and close simultaneously. Each gap-close is tracked
independently — each has its own front segment zipping backward at
`gapCloseSpeed`. A ball segment can only belong to one gap-close at a time.
If a segment is already zipping and a second gap-close triggers on the same
segment, the new close takes over (resets the zip).

```typescript
interface GapCloseState {
  frontSegmentStartIndex: number;  // index of tail ball of the front segment
  backSegmentEndIndex: number;     // index of head ball of the back segment
  closing: boolean;
}
```

---

### 12.7 Chain Insertion Algorithm

#### Step 1 — Find the hit ball

The projectile is a moving circle. Each tick, test it against every ball in
every chain. The **first ball whose circle overlaps the projectile circle** is
the hit ball.

#### Step 2 — Determine insert side (before or after the hit ball)

**Primary rule:** Project the projectile's velocity vector onto the path tangent
at the hit ball's `pathT`.

- Dot product **> 0** (projectile travelling in same direction as chain, spawn→skull)
  → insert **before** the hit ball (between hit ball and skull).
- Dot product **< 0** (projectile coming from skull side)
  → insert **after** the hit ball (between hit ball and spawn).

**Fallback (dot product ≈ 0, near-perpendicular hit):** Use nearest neighbor.
Whichever of the hit ball's two neighbors is physically closer to the collision
point — insert between the hit ball and that neighbor.

```
SPAWN ──── [A] ──── [HIT] ──── [B] ──── SKULL

Projectile from spawn side  →  insert between HIT and B  (before HIT)
Projectile from skull side  →  insert between A and HIT  (after HIT)
Near-perpendicular          →  insert toward nearest neighbor
```

#### Step 3 — Insert and push forward

The inserted ball takes the `pathT` of the hit ball. The hit ball and the
**entire front segment ahead of it** (toward skull) are shifted forward by
`minSpacing` to make room.

```
Before:  ... [A] ──── [HIT] ──── [C] ──── [D=head] ──── SKULL

After:   ... [A] ──── [NEW] ──── [HIT] ──── [C] ──── [D=head] ──── SKULL
                           all shifted → toward skull
```

**Critical rule: balls never move toward the spawn point from insertion.**
The back segment (spawn side of the insertion point) is completely untouched.
Balls only ever move toward spawn via zip-back or the Reverse power-up — never
from insertion.

**Skull danger:** if shifting the front segment forward pushes the head ball to
`pathT >= 1.0`, the skull condition triggers immediately — a game over results.
Careless insertion near the skull is punished.

The push propagates through the **entire front segment** — every ball from the
hit ball to the head is shifted by `minSpacing`. It stops at any gap, since
balls beyond a gap are not in contact.

#### Step 4 — Match check

Immediately after insertion, `MatchSystem` scans outward from the inserted
ball's index in both directions, counting consecutive same-color balls
(including the inserted ball).

- Count **≥ 3** → group pops → gap opens → zip-back triggers (Case 1).
- Count **< 3** → no pop. Run full-chain gap color-match scan (Case 2) in case
  the insertion created a same-color match across an existing gap elsewhere.

The match check is synchronous — it happens in the same tick as the insertion,
with no delay.

---

### 12.8 Game Loop

The loop is split into two independent concerns: **simulation** (pure, deterministic)
and **rendering** (Three.js, side-effectful). This separation makes every logic
path unit-testable without a browser.

#### GameLoop.ts — the tick engine

```typescript
interface TickInput {
  aimAngle: number;       // radians from frog to pointer
  fire: boolean;          // true for exactly one tick when fire button pressed
  swap: boolean;          // true for exactly one tick when swap pressed
}

interface GameState {
  chains: ChainManager;
  frog: FrogState;
  projectiles: ProjectileState[];  // multiple in-flight simultaneously
  score: ScoreState;
  powerUps: ActivePowerUp[];
  skull: SkullState;
  fireThrottleMs: number;          // countdown to next allowed shot
  ballsRemaining: number;          // total unpopped balls (in-chain + unspawned)
  rng: Random;                     // seeded — same seed == same game
  tickCount: number;
}

class GameLoop {
  // Advance the simulation by exactly `dt` milliseconds.
  // Has NO side effects outside of returning the new state.
  tick(state: GameState, input: TickInput, dt: number): GameState { ... }

  // Step exactly one fixed tick (FIXED_DT = 16.67ms).
  // Used by tests and the debug StepController.
  step(state: GameState, input: TickInput): GameState {
    return this.tick(state, input, FIXED_DT);
  }
}
```

- `tick()` is a **pure function** (no global state, no DOM).
- All randomness flows through `state.rng` (seeded `Random`), so any game
  can be replayed deterministically from `(seed, inputLog)`.
- The RAF loop calls `tick()` with real `dt`; tests call `step()` with a fixed dt.

#### Browser RAF loop (`main.ts`)

```
requestAnimationFrame loop:
  dt = clamp(now - lastTime, 0, 50ms)     // cap to avoid spiral of death

  if !debugConfig.paused:
    input   = InputManager.poll()
    state   = gameLoop.tick(state, input, dt)

  renderer.sync(state)                     // write state → Three.js meshes
  renderer.render()                        // three.js renderer.render(scene, cam)
  debugOverlay.draw(state)                 // no-op unless debug mode enabled
```

**tick() ordering within a frame:**

```
1. Decrement fireThrottleMs
2. Handle fire input (if fireThrottleMs <= 0, launch projectile, reset throttle)
3. Handle swap input
4. Move all in-flight projectiles
5. Collision detection: each projectile vs all chains
   → on hit: insert ball, run match check, trigger zip-back if pop
6. if chains.anyZipping():
     chains.updateZip(dt)       // advance zip-back animations
     chains.checkGapClose()     // fire match check on any newly closed gaps
     // spawner paused — no push
   else:
     chains.updatePush(dt)      // normal push-from-back movement
     SpawnSystem.update(dt)     // emit new balls at tail
7. PowerUpSystem.update(dt)
8. SkullSystem.check()          // leading ball pathT >= 1.0?
9. ScoreSystem.flush()
```

---

## 13. Testing Strategy

### 13.1 Philosophy

- **Logic first, render never.** All game rules live in `src/logic/` with zero
  Three.js or DOM dependencies. Any unit test can import and exercise them without
  a browser environment.
- **Tests encode the rules.** Each mechanic in §2–§8 of this document has a
  corresponding test. If a rule isn't tested, it isn't implemented.
- **Coverage gate: ≥ 90% lines and branches** on all files under `src/logic/`
  and `src/utils/`. Enforced in CI; PRs fail if coverage drops below threshold.
- **Deterministic replay.** The seeded `Random` class and pure `GameLoop.tick()`
  mean any test can reproduce an exact game sequence by supplying a seed and
  input log.

### 13.2 Unit Tests (Vitest)

Each `logic/` module has a co-located test file in `tests/unit/logic/`. Tests
cover every public method and every branch condition.

**Representative test cases per module:**

| Module | Key test cases |
|---|---|
| `BallChain` | Insert ball at head/middle/tail; correct pathT ordering maintained; gap closes after pop; speed ramp applied correctly |
| `MatchSystem` | 3-match pops; 2-match doesn't pop; cascade after gap close; cascade terminates when no new match; single-color chain cleared entirely |
| `ScoreSystem` | Base 10 pts/ball; chain bonus formula at n=1,10,20; combo multiplier per cascade level; gap shot multiplier; ace time bonus; extra life thresholds |
| `PathSystem` | Arc-length LUT monotonically increasing; `tFromArcLength` round-trips; position at t=0 == path start; position at t=1 == path end; uniform speed despite curvature |
| `CollisionSystem` | Direct hit registers; near-miss doesn't register; insertion index correct for front/back of nearest ball; two-chain level targets correct chain |
| `PowerUpSystem` | Each power-up activates on pop; effect applied; timer expires correctly; multiple simultaneous power-ups each expire independently |
| `SpawnSystem` | Balls spawn at correct interval; correct color distribution; spawn stops after `chainLength` balls |
| `FrogState` | `aimAngle` updated from input; fire produces projectile with correct velocity; swap exchanges current/next; next ball replaced after fire |

### 13.3 Integration / Step-Through Tests (Vitest)

`tests/integration/GameLoop.test.ts` drives the full simulation headlessly using
`GameLoop.step()`. These tests verify multi-system interactions that can't be
checked by unit-testing a single module.

**Pattern:**

```typescript
it("clears a 3-ball chain in 1 shot", () => {
  const state = buildState({
    chain: [ball("red"), ball("red"), ball("red")],
    frog: { currentBall: ball("red"), aimAngle: angleToChain },
    seed: 42,
  });

  let s = state;
  s = gameLoop.step(s, { fire: true, swap: false, aimAngle: s.frog.aimAngle });
  // advance enough ticks for projectile to travel and collide
  for (let i = 0; i < 30; i++) s = gameLoop.step(s, noInput);

  expect(s.chain.balls).toHaveLength(0);
  expect(s.score.total).toBe(30 + 200); // 3×10 + chain bonus at n=1
});
```

**Key step-through scenarios:**

- Single shot clears 3-ball chain; score correct.
- Shot inserts into middle; no match; chain continues.
- Cascade: popping group A creates gap → group B match → second pop.
- Power-up embedded in chain; popping the group activates it.
- Skull reached: life decremented, state reset to level start.
- Full level: all balls cleared → `state.phase == "levelComplete"`.
- Seeded replay: same `(seed, inputLog)` produces identical final `state`.

### 13.4 End-to-End Tests (Playwright)

Playwright tests run the built app in a headless Chromium browser. They drive
input via mouse events and assert on visible DOM/canvas state (or data attributes
written by the renderer for testability).

**Test files:**

| File | Scenarios covered |
|---|---|
| `gameplay.spec.ts` | Page loads; frog visible; click fires ball; score increments on match |
| `powerups.spec.ts` | Each of the 7 power-ups appears in chain, is collected, effect visible |
| `skull.spec.ts` | Letting chain reach skull loses a life; HUD life count decrements |
| `levelcomplete.spec.ts` | Clearing the chain shows level-complete overlay |
| `debug.spec.ts` | `?debug=1` URL param shows debug panel; speed slider changes chain speed |

**Testability hooks for Playwright:**

The renderer writes key game state to `data-*` attributes on the canvas wrapper:

```html
<div id="game-root"
  data-score="1240"
  data-lives="3"
  data-phase="playing"
  data-chain-length="14">
  <canvas id="game-canvas"></canvas>
</div>
```

Playwright assertions use these instead of pixel-sniffing:

```typescript
await expect(page.locator("#game-root")).toHaveAttribute("data-lives", "2");
```

### 13.5 Coverage Configuration

`vitest.config.ts`:
```typescript
coverage: {
  provider: "v8",
  include: ["src/logic/**", "src/utils/**"],
  exclude: ["src/renderer/**", "src/debug/**", "src/sprites/**"],
  thresholds: {
    lines: 90,
    branches: 90,
    functions: 90,
  },
  reportsDirectory: "coverage",
}
```

---

## 14. Debug Mode

Debug mode is activated by the URL parameter `?debug=1` or by pressing
`` ` `` (backtick) at runtime. It adds an on-screen panel and enables
step-through controls without affecting the production build.

### 14.1 Debug Panel (`DebugOverlay.ts`)

A semi-transparent HTML overlay (positioned over the canvas) with:

| Control | Type | Effect |
|---|---|---|
| **Pause / Resume** | Button | Freezes `GameLoop.tick()` |
| **Step** | Button | Advances exactly one tick (`FIXED_DT`) while paused |
| **Step N** | Number + Button | Advances N ticks in one click |
| **Chain speed** | Slider (0.1× – 5×) | Multiplies `baseSpeed` at runtime |
| **Spawn interval** | Slider (100ms – 5000ms) | Overrides level spawn rate |
| **Ball colors** | Checkboxes | Restrict active color set |
| **Force power-up** | Dropdown + Button | Injects chosen power-up into next ball |
| **God mode** | Checkbox | Skull never triggers game over |
| **Show path** | Checkbox | Renders the Bézier path curve |
| **Show pathT** | Checkbox | Renders each ball's `pathT` value as text |
| **Show hitboxes** | Checkbox | Renders collision radii |
| **Seed** | Text input | Set `state.rng` seed; restart level to take effect |
| **Export state** | Button | Copies `JSON.stringify(state)` to clipboard |
| **Import state** | Button | Paste JSON to restore exact game state |

### 14.2 Step Controller (`StepController.ts`)

When paused, every call to the RAF loop skips `gameLoop.tick()` unless a step
was requested. The renderer still runs each frame so the canvas stays live.

```typescript
class StepController {
  paused = false;
  private pendingSteps = 0;

  requestStep(n = 1): void { this.pendingSteps += n; }

  // Called by main.ts each RAF frame instead of tick() directly
  maybeTick(state: GameState, input: TickInput): GameState {
    if (!this.paused) return gameLoop.tick(state, input, realDt);
    if (this.pendingSteps > 0) {
      this.pendingSteps--;
      return gameLoop.step(state, input);
    }
    return state;  // frozen
  }
}
```

### 14.3 DebugConfig (`DebugConfig.ts`)

A singleton store of all runtime-overridable values. Systems read from it when
the value is set, falling back to the level JSON default otherwise.

```typescript
interface DebugConfig {
  enabled: boolean;
  paused: boolean;
  speedMultiplier: number | null;
  spawnInterval: number | null;
  restrictColors: BallColor[] | null;
  godMode: boolean;
  showPath: boolean;
  showPathT: boolean;
  showHitboxes: boolean;
  forcedPowerUp: PowerUpType | null;
}
```

Debug mode is excluded from production bundles via Vite's `import.meta.env.DEV`
guard — the `debug/` modules tree-shake out entirely in production builds.

---

## 15. Milestones

| Milestone | Deliverables |
|---|---|
**MVP goal: one fully playable level with complete gameplay mechanics and polished art.**
Every milestone ships with passing unit tests. Coverage gate (≥90%) enforced from M2 onward.

| Milestone | Deliverables | MVP? |
|---|---|---|
| **M0 — Scaffold** | Vite + TypeScript + Three.js, orthographic scene, deterministic `GameLoop`, seeded `Random`, mouse + touch `InputManager`, Vitest + Playwright configured, CI coverage check | Yes |
| **M1 — Art Assets** | All SVG assets (6 balls, 7 power-ups, frog, skull ×5, bg, UI icons); Three.js asset loading pipeline | Yes |
| **M2 — Path & Chain** | `PathSystem` (Bézier, arc-length LUT) + unit tests; `BallChain` (movement, spawn, speed ramp) + unit tests; chain rendered in Three.js | Yes |
| **M3 — Shooter** | `FrogState` + unit tests; `ProjectileState` + unit tests; `CollisionSystem` + unit tests; frog mesh + aim rotation | Yes |
| **M4 — Matching** | `MatchSystem` (3+ match, cascade) + unit tests; step-through integration tests for single-shot clear and cascade scenarios | Yes |
| **M5 — Scoring & HUD** | `ScoreSystem` (all bonus types) + unit tests; HUD renderer (score, lives, Zuma bar) | Yes |
| **M6 — Win/Lose** | `SkullState` + unit tests; skull animation; level-complete and game-over screens; life loss / retry flow + integration tests | Yes |
| **M7 — Power-Ups** | `PowerUpSystem` (all 7 types) + unit tests; chain embedding; activation effects; integration tests for each power-up | Yes |
| **M8 — Debug Mode** | `DebugOverlay`, `StepController`, `DebugConfig`; all debug controls functional; Playwright `debug.spec.ts` | Yes |
| **M9 — E2E & Polish** | Full Playwright suite; particle effects; camera shake; responsive canvas; mobile QA; coverage report ≥ 90% confirmed | Yes |
| **M10 — Multi-Level** | Level JSON format, 13+ levels, stage progression, save state | Post-MVP |
| **M11 — Game Modes** | Adventure mode, Gauntlet (Practice + Survival), rank system | Post-MVP |
| **M12 — Audio** | BGM loop, SFX, Zuma vocal cue via Web Audio API | Post-MVP |

---

## 16. Open Questions

1. **Art assets:** SVG sprites authored in-project, rasterized to a spritesheet at
   build time. All entities render via `SpriteSheet.draw()`. (Resolved.)

2. **Mobile support priority:** Full touch support is in scope. Touch controls implemented in M0 (`InputManager`) and tested throughout. Three.js renders to a `<canvas>` element which works natively on mobile. (Resolved.)

3. **Dual-track implementation:** `ChainManager` owns all `BallChain` instances
   for a level and is the sole interface for collision dispatch, insertion, power-up
   application, and all-cleared checks. `BallChain` stays single-responsibility.
   (Resolved — see §12.5.)

4. **Difficulty balancing:** Exact `baseSpeed` and `spawnInterval` values need
   empirical playtesting — initial numbers in level JSONs are estimates.

5. **Continues / payment:** Free continues or limited? (Out of scope for MVP;
   default to no limit.)

6. **Network features:** High score leaderboard? (Out of scope for MVP; suggest
   localStorage only.)

7. **Level editor:** Built-in path editor tool for designers? (Out of scope for
   MVP; use JSON directly.)

8. **License / naming:** Avoid "Zuma" in the final product name to prevent
   trademark issues. Current repo name "ball-chain-clone" works as a codename.

9. **Physics engine:** All ball motion is path-driven so a physics engine is not
   needed for core gameplay. Potential use cases if added later: projectile
   ricochets, particle debris on pop, environmental elements. Candidates if
   needed: Rapier (Rust/WASM, excellent perf) or Matter.js (pure JS, simpler).
   Current plan: no physics engine. (To discuss.)
