# Zuma Clone — Game Design Document

**Project:** ball-chain-clone  
**Reference Game:** Zuma Deluxe (PopCap Games, 2003)  
**Document Version:** 1.0  
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
13. [Milestones](#13-milestones)
14. [Open Questions](#14-open-questions)

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

**Key tension mechanic:** The chain continuously accelerates. As balls near the
skull the skull visually "opens," escalating urgency.

---

## 3. Game Objects

### 3.1 The Frog (Shooter)

| Property | Value |
|---|---|
| Position | Fixed at level-defined anchor point (usually center) |
| Rotation | 0–360°, follows mouse cursor / analog stick |
| Current ball | Displayed in frog's mouth |
| Next ball | Displayed on frog's back |
| Swap | Player may swap current ↔ next at any time |

The frog always holds two balls. When the current ball is fired, next ball becomes
current and a new random ball is generated for next.

### 3.2 Ball Chain

- Balls are stored as an ordered list with a `pathT` parameter (0.0 = spawn,
  1.0 = skull) representing normalized distance along the Bézier/polyline path.
- Each ball has: `color`, `pathT`, `radius`, `powerUp | null`.
- The chain advances at a base speed that scales with level number.
- Chain speed increases slightly after each ball group is popped (brief pullback
  first, then resume).
- A **reverse** event can push `pathT` backward temporarily.

### 3.3 Ball Colors

| Stage Range | Colors Available |
|---|---|
| Stages 1–3 | Red, Green, Blue, Yellow (4 colors) |
| Stages 4–6 | + Purple (5 colors) |
| Stages 7+ | + White (6 colors) |

Color count directly controls match difficulty. Fewer colors = easier to form
chains accidentally; more colors = deliberate aim required.

### 3.4 The Skull

- Located at path end (pathT = 1.0).
- Animated: mouth opens wider as the leading ball's `pathT` approaches 1.0.
- Fully open at `pathT >= 0.90` (danger zone — audio cue fires).
- When a ball enters (pathT > 1.0), all remaining balls follow and the player
  loses a life.

### 3.5 Projectile Ball

- Fired from frog at a fixed speed in the aim direction.
- Physics: straight-line travel, no gravity, no arc.
- On collision with the chain, the ball inserts between the two nearest balls
  along the path closest to the collision point.
- On miss: ball disappears off-screen.

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

Power-up balls appear embedded in the chain with a glowing icon. Shooting any
ball in the group containing a power-up collects it, but it only activates if
the group pops (3+ match). Once activated, the effect applies immediately.

| Power-Up | Visual | Effect | Duration |
|---|---|---|---|
| **Slow** | Hourglass / snail | Halves chain speed | 10 s |
| **Reverse** | Arrows pointing back | Pushes chain backward | 8 s |
| **Lightning** | Lightning bolt | Destroys all balls matching the shot color | Instant |
| **Laser** | Laser beam | Next fired ball pierces the entire chain, popping every matching-color ball in its path | 1 shot |
| **Bomb** | Explosion icon | Destroys 7-ball radius around impact | Instant |
| **Accuracy** | Target reticle | Shows exact insertion point and outcome preview | 15 s |
| **Color Change** | Paint swirl | Converts a nearby cluster to a single color (random or aimed) | Instant |

Multiple power-ups can be active simultaneously. Their timers run independently.

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

Firing through a gap in the chain to hit balls further along awards a multiplier:

```
gapMultiplier = 1 + (number of gaps passed through)
```

The projectile must travel through at least one visible gap (ball spacing > ball
diameter) to qualify.

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

- Any ball reaches pathT ≥ 1.0 (enters the skull).
- All remaining chain balls are pulled in instantly.
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
| Testing | Vitest | Co-located with Vite |
| Physics engine | None | All motion is path-driven (no free-body simulation needed) |
| Asset pipeline | SVG sources → PNG textures via build script | Crisp at all resolutions, scriptable, version-controllable |

### 12.2 Module Breakdown

```
src/
├── main.ts               # Entry point, canvas setup, game loop
├── Game.ts               # Top-level state machine (menu / playing / paused / gameover)
├── scenes/
│   ├── MenuScene.ts
│   ├── GameScene.ts      # Core gameplay
│   └── GameOverScene.ts
├── entities/
│   ├── Frog.ts           # Shooter entity
│   ├── Ball.ts           # Single ball data + render
│   ├── BallChain.ts      # Ordered chain, insertion, pop, cascade logic
│   ├── Projectile.ts     # In-flight ball
│   └── Skull.ts          # End-of-path danger object
├── systems/
│   ├── PathSystem.ts     # Bézier / polyline, arc-length parameterization
│   ├── CollisionSystem.ts# Projectile ↔ chain hit detection
│   ├── MatchSystem.ts    # 3+ match detection, cascade resolution
│   ├── ScoreSystem.ts    # All scoring rules
│   └── PowerUpSystem.ts  # Power-up activation / timer management
├── input/
│   └── InputManager.ts   # Mouse, touch, keyboard unified
├── audio/
│   └── AudioManager.ts   # Web Audio API wrapper
├── sprites/
│   ├── SpriteSheet.ts    # Loads spritesheet PNG + JSON atlas, drawSprite() helper
│   └── SpriteAtlas.ts    # Type definitions for atlas manifest
├── data/
│   └── levels/           # Level JSON files (one per level)
│       ├── 1-1.json
│       ├── 1-2.json
│       └── ...
├── ui/
│   ├── HUD.ts            # Score, lives, Zuma bar
│   └── LevelComplete.ts
└── utils/
    ├── Vec2.ts           # 2D vector math
    ├── BezierUtils.ts    # Curve sampling, arc-length LUT
    └── Random.ts         # Seeded RNG
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

### 12.5 Chain Insertion Algorithm

1. Find nearest ball in chain to collision point (Euclidean distance).
2. Determine whether inserted ball goes before or after nearest ball based on
   which side of the ball the projectile came from along the path direction.
3. Insert ball into chain array at computed index.
4. Run `MatchSystem.checkMatches(insertionIndex)`: scan left and right from
   insertion point for consecutive same-color balls.
5. If count ≥ 3, mark group for removal, compute score, schedule cascade check.
6. After removal, close gap: advance chain segment behind gap to meet chain
   segment ahead. Check if new neighbors match → cascade.

### 12.6 Game Loop

```
requestAnimationFrame loop:
  dt = clamp(now - lastTime, 0, 50ms)   // cap dt to avoid spiral of death
  
  InputManager.poll()
  
  if state == PLAYING:
    BallChain.update(dt)      // advance pathT of all balls
    Projectile.update(dt)     // move in-flight balls
    CollisionSystem.check()   // projectile ↔ chain
    SpawnSystem.update(dt)    // add new balls to chain head
    PowerUpSystem.update(dt)  // tick timers
    SkullSystem.check()       // leading ball pathT >= 1.0?
    ScoreSystem.flush()       // commit pending score events
  
  // Three.js handles the render call:
  renderer.render(scene, camera)
```

---

## 13. Milestones

| Milestone | Deliverables |
|---|---|
**MVP goal: one fully playable level with complete gameplay mechanics and polished art.**
Multi-level progression, Gauntlet mode, and audio are post-MVP.

| Milestone | Deliverables | MVP? |
|---|---|---|
| **M0 — Scaffold** | Vite + TypeScript + Three.js project, orthographic scene, game loop, mouse + touch InputManager, SVG sprite pipeline + build script | Yes |
| **M1 — Art Assets** | All SVG sprites: 6 ball colors, 7 power-up icons, frog (idle + shoot), skull (5 states), background, UI icons; spritesheet generated | Yes |
| **M2 — Path & Chain** | PathSystem (Bézier, arc-length LUT), BallChain (movement, spawn, speed ramp), Three.js mesh rendering via spritesheet | Yes |
| **M3 — Shooter** | Frog mesh, aim rotation toward pointer/touch, projectile firing, ball swapping | Yes |
| **M4 — Matching** | Collision detection, insertion algorithm, 3+ match detection, cascade logic | Yes |
| **M5 — Scoring & HUD** | Base score, chain bonus, combo bonus, gap shot bonus, HUD (score, lives, Zuma bar) | Yes |
| **M6 — Win/Lose** | Skull mechanic + animation, level complete screen, game over screen, life loss / retry | Yes |
| **M7 — Power-Ups** | All 7 power-up types, chain embedding, activation effects | Yes |
| **M8 — Polish (MVP)** | Particle effects on pops, camera shake on skull hit, responsive canvas scaling, mobile QA | Yes |
| **M9 — Multi-Level** | Level JSON format, 13+ level definitions, stage progression, save state | Post-MVP |
| **M10 — Game Modes** | Adventure mode, Gauntlet (Practice + Survival), rank system | Post-MVP |
| **M11 — Audio** | BGM loop, SFX, Zuma vocal cue via Web Audio API | Post-MVP |

---

## 14. Open Questions

1. **Art assets:** SVG sprites authored in-project, rasterized to a spritesheet at
   build time. All entities render via `SpriteSheet.draw()`. (Resolved.)

2. **Mobile support priority:** Full touch support is in scope. Touch controls implemented in M0 (`InputManager`) and tested throughout. Three.js renders to a `<canvas>` element which works natively on mobile. (Resolved.)

3. **Dual-track implementation:** Should both chains share a single `BallChain`
   manager or have two independent instances? (Recommend two instances.)

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
