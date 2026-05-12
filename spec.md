# Slither Arena — Specification

A single-file, browser-based game. No build step, no dependencies. Open `snake-io.html` in any modern browser.

---

## 1. Overview

**Goal:** Grow the longest snake by eating orbs while forcing rival snakes to crash into your body.

**Genre:** Real-time arcade / `.io`-style massively-single-player-with-bots.

**Platform:** HTML5 Canvas + vanilla JavaScript. Runs on desktop (mouse + keyboard) and mobile (touch).

---

## 2. Core Loop

1. Player enters a name and clicks **PLAY**.
2. Spawns at a random point inside a circular arena alongside 14 AI bots.
3. Mouse position relative to the screen center sets the heading; the snake turns smoothly toward it.
4. Player collects food orbs to grow and rack up score; optionally boosts to outrun or cut off enemies.
5. Death occurs when the player's **head** touches another snake's **body**, or when the player exits the arena boundary.
6. On death, the snake's body converts to food, a death screen shows final stats, and the player can replay.

---

## 3. World

| Parameter | Value | Notes |
|---|---|---|
| Arena shape | Circle | Centered at world origin `(0, 0)` |
| `WORLD_R` | 2200 | Arena radius in world units |
| Wall behavior | Instant death | Crossing the boundary kills the snake |
| Background | Dark navy `#0a0e1a` with faint grid | Grid step 80px, parallax with camera |
| Boundary visual | Red-pink glowing ring | `rgba(255,80,120,0.45)` with shadow blur |

**Coordinate system:** world-space origin `(0, 0)` is the arena center. The camera follows the player's head with `lerp` smoothing (factor `0.15` per frame). All entities render via `worldToScreen(x, y)`.

---

## 4. Entities

### 4.1 Snake

| Field | Type | Meaning |
|---|---|---|
| `name` | string | Display label (≤14 chars for player) |
| `isPlayer` | bool | Distinguishes the human snake |
| `segs` | `{x, y}[]` | Body segments, index 0 is the head |
| `angle` | number | Current heading (radians) |
| `targetAngle` | number | Desired heading; `angle` lerps toward it |
| `speed` | number | Current per-frame movement (px) |
| `boost` | bool | Whether the snake is sprinting |
| `length` | number | Logical length (float for partial growth) |
| `color`, `darkColor` | hex | Fill + outline from a paired palette |
| `score`, `kills` | int | Stats |
| `dead` | bool | If true, snake is removed from simulation |
| `partialGrow` | number | Float accumulator for fractional boost drain |
| `wander`, `mood` | number | Bot AI state |

**Tuning:**

| Parameter | Value |
|---|---|
| `BASE_SPEED` | 2.4 px/frame |
| `BOOST_SPEED` | 4.6 px/frame |
| `TURN_RATE` | 0.07 rad/frame (max angular delta) |
| `SEG_SPACING` | 6 px between body samples |
| `HEAD_R_BASE` | 8 px |
| Head radius | `HEAD_R_BASE + min(length, 200) * 0.08` |
| Initial player length | 10 segments |
| Initial bot length | 10 + random(0..40) |
| Min length when boosting | 12 (below this, boost is disabled) |
| `BOOST_DRAIN` | 0.04 length per frame while boosting |

**Movement model:**
- Each frame, `angle` rotates toward `targetAngle` clamped by `TURN_RATE`.
- Head moves forward by `speed` along `angle`.
- New head position is `unshift`ed onto `segs`; tail segments are trimmed so `segs.length === floor(length)`.
- This produces the smooth, "noodle"-like body familiar to slither.io players (segments are samples, not a fixed grid).

**Boost:**
- Boosting drains `BOOST_DRAIN` length per frame.
- Whenever the accumulator crosses an integer, the snake loses one segment and has a 50% chance to drop a food orb at the tail position (colored by the snake's color).
- This means boost is a real economic tradeoff: spend length to gain speed.

### 4.2 Food

| Field | Type | Meaning |
|---|---|---|
| `x, y` | number | World position |
| `value` | int | Length / score increment (1 for normal, 2 for corpse-drop) |
| `color` | hex | Render color (random for ambient, snake color for drops) |
| `pulse` | number | Phase for pulsing animation |

| Parameter | Value |
|---|---|
| `FOOD_COUNT` | 600 ambient orbs at all times |
| Food radius | `3 + value * 0.7 + sin(pulse) * 0.7` |
| Pickup radius | `head_radius + 4` |
| Score per food | `value * 10` |
| Replenishment | When a food is eaten, a new one spawns at a random point in the arena |

When a snake dies, its body is converted into food orbs of `value = 2`, sampled at every second segment, with jitter `±3 px`.

---

## 5. Controls

| Input | Action |
|---|---|
| Mouse move | Sets `targetAngle` from screen-center to cursor |
| Left mouse down / up | Boost on / off |
| `Space` (down / up) | Boost on / off |
| Touch (single finger) | Steer + boost while held |
| Name input + `Enter` | Submit and start |

Boost is suppressed when `length <= 12` to prevent the player from boosting themselves to instant death.

---

## 6. AI (Bots)

Each non-player snake runs `botThink()` once per frame. Decision priority (highest first):

1. **Wall avoidance.** If distance from center > `WORLD_R - 200`, steer toward the origin.
2. **Threat avoidance.** Look 60 px ahead of the head. Scan all other snakes' body segments (every 3rd segment for performance). If any segment is within an 80 px radius of the look-ahead point, steer 180° away from the nearest threat, weighted by inverse distance.
3. **Food seek.** Scan all foods within 320 px (`visionR`). Steer toward the nearest one.
4. **Wander.** If none of the above, jitter `wander` by `±0.05 rad/frame` and steer toward it.

Bots boost randomly (0.5% chance per frame, only if `length > 25`).

**Respawn:** When a bot dies, it is scheduled to respawn 240 frames (~4 s at 60 fps) later under the same name, with a fresh body and random length 10–40.

---

## 7. Collisions

Each frame, two collision passes run (in order):

### 7.1 Snake-vs-snake (lethal)
For every alive snake `s`:
- Take `head = s.segs[0]`, `rh = radius(s)`.
- For every other alive snake `o`:
  - Iterate `o.segs` from index 1 (skipping `o`'s head — head-on collisions are not awarded), every 2nd segment.
  - If `|head - seg| < (rh + ro) * 0.8`, kill `s` and credit `o` with the kill (`o.kills++`, `o.score += 100`).

This asymmetry — heads kill bodies, bodies don't kill heads — is what makes the "cut off your opponent" strategy work and matches slither.io semantics.

### 7.2 Food pickup
For every alive snake, scan all foods. If `|food - head| < (head_radius + 4)`, the snake absorbs the food: `length += value`, `score += value * 10`, food removed, ambient food count topped up.

---

## 8. Rendering

**Frame structure** (in `render()`):
1. Clear with `#0a0e1a`.
2. Draw faded grid (parallax with camera).
3. Draw arena boundary ring with glow.
4. Draw all food orbs (skip if off-screen).
5. Draw all snakes, sorted by length ascending so larger snakes render on top.
6. Draw minimap (separate canvas).

**Snake rendering** uses two stacked `lineTo` strokes over the segment polyline:
- A wide outline stroke in `darkColor` (`width = 2r + 2`).
- A narrower inner stroke in `color` (`width = 2r - 2`), with shadow glow when boosting.
- White dot markers every 6 segments (scale pattern).
- Head circle on top, with outline.
- Two eye circles offset `±0.7 rad` from heading; pupils shift slightly forward.
- Name label drawn above head with stroke-then-fill for readability.

**HUD elements** (HTML, not canvas):
- Top-left: length, score, kills.
- Top-right: leaderboard (top 8 by score, player highlighted yellow).
- Bottom-right: circular minimap (160×160 canvas).
- Overlay card: pre-game menu and death screen.

**Performance budget:**
- Target 60 fps.
- Food rendering culls off-screen orbs by AABB check.
- Bot threat scans sample every 3rd body segment.
- Leaderboard DOM updates throttled to every 10th frame.

---

## 9. UI States

| State | Trigger | UI |
|---|---|---|
| Menu | Page load | Overlay with title, name input, PLAY button, controls hint |
| Playing | `PLAY` clicked | Overlay hidden, simulation runs |
| Dead | Player head touches body or exits arena | Overlay reappears with final stats and PLAY AGAIN button |

The overlay uses `display: flex` ↔ `display: none` toggling via the `.hidden` class. The menu card HTML is rebuilt on death and again on restart so each life starts fresh.

---

## 10. File Layout

```
C:\Game\
├── snake-io.html    # everything: HTML, CSS, JS, assets-as-code
└── spec.md          # this file
```

No external assets, no fonts to load, no images. The entire game is portable as a single file and runs offline.

---

## 11. Browser Support

| Feature used | Min support |
|---|---|
| Canvas 2D | All modern browsers |
| `requestAnimationFrame` | All modern browsers |
| `backdrop-filter` | Chrome 76+, Safari 9+, Firefox 103+ (graceful fall-back to opaque panels) |
| CSS gradient text (`background-clip: text`) | All modern browsers |
| Pointer + touch events | All modern browsers |

Tested target: latest Chrome / Edge / Firefox on Windows 11. No transpilation or polyfills.

---

## 12. Known Limitations / Non-Goals

- **No networking.** All "other snakes" are local AI bots; there is no multiplayer.
- **No persistence.** Scores, names, and stats reset on page reload.
- **No sound.** Silent by design — no audio assets.
- **No mobile-optimized UI chrome.** Touch works for steer/boost but the menu layout assumes a reasonable viewport.
- **Fixed bot count.** 14 bots regardless of arena population; no difficulty scaling.
- **Simple AI.** Bots do not coordinate, do not target the player specifically, and do not attempt cut-off maneuvers.

---

## 13. Tuning Cheat Sheet

If the game feels off, these are the first knobs to turn (all at the top of the `<script>` block):

| Symptom | Knob |
|---|---|
| Snake feels sluggish | `BASE_SPEED` ↑ |
| Boost feels weak | `BOOST_SPEED` ↑ or `BOOST_DRAIN` ↓ |
| Snake turns too sharply | `TURN_RATE` ↓ |
| Body looks chunky | `SEG_SPACING` ↓ (more samples) |
| Arena feels cramped | `WORLD_R` ↑ |
| Too few/many bots | `BOT_COUNT` |
| Food too sparse | `FOOD_COUNT` ↑ |
| Bots too aggressive at chasing food | reduce `visionR` in `botThink` |
| Bots crash into walls | widen wall threshold (currently `WORLD_R - 200`) |
