# Orb Arena — Specification

A browser-based, P2P-multiplayer arena game where colored circles hunt each other. Single static HTML page, no build step, no backend server.

---

## 1. Overview

**Goal:** Be the largest (or fastest) orb in the arena. Eat smaller orbs to grow or gain speed; avoid being eaten by bigger ones.

**Genre:** Real-time arcade / `.io`-style multiplayer.

**Platform:** HTML5 Canvas + vanilla JavaScript + WebRTC (via PeerJS). Runs entirely in the browser; one player hosts the session, others join by ID.

---

## 2. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Markup | HTML5 | Single file `index.html` |
| Styling | CSS3 | Embedded `<style>` block; flexbox for menu/HUD |
| Logic | Vanilla JS (ES2020+) | No framework, no bundler |
| Rendering | Canvas 2D | One main canvas, plus a minimap canvas |
| Networking | WebRTC DataChannels via [PeerJS](https://peerjs.com/) | CDN-loaded; uses PeerJS's free signaling server for ID brokering |
| Hosting | Static file | Open `index.html` directly, or serve from any static host (GitHub Pages, file://, etc.) |

**Why PeerJS:** WebRTC requires a signaling step to exchange SDP/ICE between peers. PeerJS provides a free public broker that assigns each peer a short ID and forwards offers/answers — no custom server needed for a hackathon-scale build.

---

## 3. Core Loop

1. Page load → **Main Menu** appears.
2. Player types a name, picks a color, and chooses **Host** or **Join**.
   - **Host:** receives a short peer ID; shares it with friends (copy button).
   - **Join:** types the host's peer ID and connects.
3. Once the host clicks **Start**, the simulation begins for all connected peers.
4. Each entity (player or AI bot) is a circle that moves around the arena.
5. When two circles touch, the larger one consumes the smaller. On consumption, the eater rolls a 50/50: **grow** (radius +) or **speed boost** (permanent speed +).
6. Death returns you to a death overlay with a **Respawn** button (rejoins the live session with a fresh small orb).
7. Host leaving ends the session for everyone (joiners see "Host disconnected").

---

## 4. World

| Parameter | Value | Notes |
|---|---|---|
| Arena shape | Square | Axis-aligned |
| `WORLD_W`, `WORLD_H` | 2100, 2100 px | Top-left origin `(0, 0)`, bottom-right `(2100, 2100)` |
| Wall behavior (players) | Hard clamp | Player orb's position is clamped to the arena; velocity component into the wall is zeroed (orb stops sliding into the wall) |
| Wall behavior (AI) | Reflect | AI orbs bounce off walls: position is clamped, then the velocity component normal to that wall is **negated** (elastic bounce, no energy loss) |
| Background | Dark slate `#0e1320` with faint grid | Grid step 100 px, parallax with camera |
| Boundary visual | Bright cyan border `#3df0ff` with subtle glow | 4 px stroke |

**Coordinate system:** world-space `(0, 0)` is the top-left of the arena. The camera follows the local player's orb with `lerp` smoothing (factor `0.12` per frame). All rendering goes through `worldToScreen(x, y)`.

---

## 5. Entities

All living things in the arena are **Orbs** — there is no separate "food" concept. Small AI orbs serve as both food and threats.

### 5.1 Orb

| Field | Type | Meaning |
|---|---|---|
| `id` | string | Unique ID (peer ID for players, `bot_<n>` for AI) |
| `kind` | `"player" \| "ai"` | Drives input source |
| `name` | string | Display label (≤14 chars; bots use generated names) |
| `x`, `y` | number | World position (center) |
| `vx`, `vy` | number | Velocity components (px/frame) |
| `r` | number | Radius (drives mass & hitbox) |
| `speed` | number | Max movement speed (px/frame) |
| `color` | hex | Fill color |
| `darkColor` | hex | Outline color (derived: HSL lightness − 25%) |
| `alive` | bool | False after being eaten |
| `score` | int | Session points (persists across respawns; see §5.6) |
| `aiState` | object? | Bot-only: `{ wander, target, mood }` |

### 5.2 Tuning

| Parameter | Value |
|---|---|
| `START_RADIUS` | 16 px |
| `MAX_RADIUS` | 120 px |
| `START_SPEED` | 2.6 px/frame |
| `MAX_SPEED` | 6.0 px/frame |
| `GROW_AMOUNT` | +3 px radius per eat |
| `SPEED_AMOUNT` | +0.25 px/frame per eat |
| `GROW_VS_BOOST` | 50/50 random roll on eat |
| `EAT_RATIO` | 1.10 (size gate: eater must be ≥10% larger than victim) |
| Eat distance | `d < big.r + small.r * 0.5` (overlap-based: triggers once ~25% of the small orb's diameter is inside the big one) |
| `BOT_COUNT` | 30 (host-side only) |
| `BOT_RESPAWN_FRAMES` | 240 (~4 s @ 60 fps) |
| `AI_SPAWN_RADIUS` | 12 (baseline; actual spawn radius is `12 ± AI_SPAWN_RADIUS_JITTER`) |
| `AI_SPAWN_RADIUS_JITTER` | 2 (small random variation around baseline so bots have enough size diversity to consume each other; range = [10, 14]) |
| `AI_SPAWN_SPEED` | 2.4 (fixed; all bots spawn at this speed, slower than player `START_SPEED`) |
| `SAFE_SPAWN_RADIUS` | 250 px (no other live orb may be within this distance of a respawn point) |
| `SAFE_SPAWN_MAX_TRIES` | 30 (attempts to find a clear point before fallback) |
| `POINTS_PER_EAT` | +5 (awarded to the eater on consumption) |
| `POINTS_PER_DEATH` | −2 (deducted from the victim; clamped at 0) |
| `WIN_SCORE_DEFAULT` | 500 (default; host can override before starting — see §7.2) |
| `WIN_SCORE_MIN` | 20 |
| `WIN_SCORE_MAX` | 1000 |
| `WIN_SCORE_STEP` | 10 |

### 5.3 Movement model

- Player input produces a **desired velocity** vector (see §6).
- AI input produces velocity via `botThink()` (see §9).
- Each frame: `x += vx`, `y += vy`. Speed is capped at `orb.speed`.
- Wall resolution depends on `kind`:
  - **Player:** position is clamped to `[r, WORLD_W - r]` (and same for Y). Velocity component on the clamped axis is zeroed — the player simply stops against the wall.
  - **AI:** position is clamped, **and** the velocity component normal to the wall it hit is negated (`vx = -vx` for left/right walls, `vy = -vy` for top/bottom). The bot now travels in the reflected direction. The bot's internal `aiState.wander` heading is also updated to match the new velocity angle so its wander behavior doesn't immediately steer it back into the wall.

### 5.4 Consumption

When orbs A and B overlap (`distance(A, B) < bigger.r + smaller.r * 0.5`, i.e. ~25% of the smaller's diameter is inside the bigger) **and** the bigger one is at least `EAT_RATIO` × the smaller's radius (size gate):
- B is marked `alive = false`.
- `A.score += POINTS_PER_EAT` (default +5).
- `B.score = max(0, B.score - POINTS_PER_DEATH_ABS)` (default −2; clamped at 0 — score never goes negative).
- A rolls `Math.random() < 0.5`:
  - **Grow:** `A.r = min(A.r + GROW_AMOUNT, MAX_RADIUS)`.
  - **Boost:** `A.speed = min(A.speed + SPEED_AMOUNT, MAX_SPEED)`.
- The host checks `A.score >= WIN_SCORE` and, if so, ends the session (see §5.6).

If both orbs are within `EAT_RATIO` of *each other's* radii (i.e. nearly equal size), neither eats — they pass through.

### 5.5 Respawn (safe spawn)

Both player respawns (after clicking **Respawn** on the death overlay) and bot respawns (after `BOT_RESPAWN_FRAMES`) use the same **safe-spawn** algorithm, executed on the host:

```
function findSafeSpawn():
  for i in 1..SAFE_SPAWN_MAX_TRIES:
    x = random(START_RADIUS, WORLD_W - START_RADIUS)
    y = random(START_RADIUS, WORLD_H - START_RADIUS)
    if no live orb has distance((x,y), orb.pos) < SAFE_SPAWN_RADIUS:
      return (x, y)
  // Fallback: no clear point found in 30 tries (arena is crowded).
  // Pick the candidate that maximizes the distance to the nearest live orb.
  return argmax_over_candidates(min_distance_to_any_live_orb)
```

Properties:
- The new orb starts at `START_RADIUS` and `START_SPEED` — all eat-progress is reset.
- **Score persists across respawns** (it's a session-long stat, not a per-life stat). Only `r` and `speed` reset.
- The fallback (max-min distance) guarantees the function always returns a point, even in a fully crowded arena.
- Algorithm runs on the host. For joiner respawns, the host computes the spawn and includes the new position in the next `state` broadcast.

### 5.6 Session score & victory

The session score is the central competitive metric. Each player and bot has a `score` field, updated by the host:

| Trigger | Effect |
|---|---|
| Eat another orb | `eater.score += POINTS_PER_EAT` (default +5) |
| Be eaten | `victim.score = max(0, victim.score − 2)` |
| Respawn | Score is **not** reset — it carries forward |

**Victory condition:** the first orb (player **or** bot) whose `score` reaches the session's `WIN_SCORE` wins. The host picks `WIN_SCORE` in the lobby before starting (see §7.2); default 500 (= 100 net eats with no deaths).

When the host detects a winner:
1. Host broadcasts `{ type: "victory", winnerId, winnerName, finalScores: [...] }`.
2. The simulation **freezes** — no more movement, eating, or input is processed.
3. All peers transition to the **Victory** UI state (§7): a full-screen overlay showing the winner's name + color, the final ranked scoreboard, and (host only) a **Start New Session** button.
4. **Start New Session** broadcasts `{ type: "reset" }`. All scores reset to 0, all orbs respawn via `findSafeSpawn()`, and the game returns to the **Playing** state.

Bots winning is allowed (and possible if humans avoid each other). If a bot wins, the overlay shows the bot's generated name and color so it still feels like a real opponent.

---

## 6. Controls

The local player chooses **one** of two input modes (auto-detected: keyboard wins if any movement key is pressed; mouse otherwise).

### 6.1 Mouse mode (default)

- Cursor position sets a **target world point**.
- Desired velocity = `normalize(target - orb.pos) * orb.speed`.
- Orb moves toward the cursor until the cursor is inside the orb (then it stops).

### 6.2 Keyboard mode

| Keys | Direction |
|---|---|
| `W` or `↑` | Up |
| `S` or `↓` | Down |
| `A` or `←` | Left |
| `D` or `→` | Right |

- Multiple keys combine for diagonal movement (normalized so diagonals aren't faster).
- Releasing all keys → velocity ramps to zero over ~10 frames (linear decay).

### 6.3 Name input

- Pre-game: typing name + `Enter` advances. Empty name → auto-generated `Player###`.

---

## 7. Main Menu / UI States

| State | Trigger | UI |
|---|---|---|
| `Menu` | Page load | Title, name input, color picker grid, **Host** / **Join** buttons |
| `Hosting` | Click **Host** | Show generated peer ID + **Copy** button; **Win score** input (see §7.2); list of connected joiners; **Start Game** button |
| `Joining` | Click **Join** | Input field for host ID + **Connect** button; status line ("Connecting…", "Waiting for host to start…"); shows the host's chosen win score once received |
| `Playing` | Host clicks **Start** | Overlay hidden, simulation runs |
| `Dead` | Local orb eaten | Overlay with stats (current score, eats, time alive) and **Respawn** button — score persists, just costs 2 points |
| `Victory` | Any orb's score reaches `WIN_SCORE` | Full-screen overlay: winner name + color, final ranked scoreboard, **Start New Session** (host) / "Waiting for host…" (joiners) |
| `Disconnected` | Host disconnects (joiners) or PeerJS error | Overlay with "Host disconnected" + **Back to Menu** |

### 7.2 Host: win score selector

In the **Hosting** state, the host sees a numeric input labelled **Target score** with:
- Default value: `WIN_SCORE_DEFAULT` (500).
- Range: `WIN_SCORE_MIN`..`WIN_SCORE_MAX` (20..1000), step `WIN_SCORE_STEP` (10).
- Quick-pick presets shown as buttons next to the input: **50** (short), **100** (standard), **200** (long), **500** (marathon).
- Out-of-range input is clamped on blur; non-numeric input falls back to the default.

The host's chosen value becomes the session's `WIN_SCORE`. It is:
- Re-broadcast to all joiners whenever it changes (via the `lobby` message, see §8.2), so joiners' menus stay in sync.
- Locked-in at the moment **Start Game** is clicked, and included in the `start` message payload.
- Carried forward into a new session when the host clicks **Start New Session** on the Victory overlay — the host may also change it before clicking. The new value is included in the `reset` message.

### 7.3 Color picker

A 12-color palette grid. Each swatch is a 40×40 button; selected swatch has a white outline. Colors are vivid and saturated so orbs stay distinguishable on the dark background.

Default palette (HSL `60% / 60%`):
`#ff5b5b`, `#ff9b3d`, `#ffd83d`, `#9fff3d`, `#3dff8a`, `#3dffd2`, `#3dc9ff`, `#3d7bff`, `#a13dff`, `#ff3df0`, `#ff3d8a`, `#ffffff`.

If two players pick the same color, the second player gets a thin white inner ring on their orb to keep them distinguishable.

---

## 8. Networking — P2P Topology

**Authoritative host model** (star topology):
- The **host** runs the full simulation: AI, collisions, eat rolls, food respawn.
- **Joiners** send only their input (cursor position or key state) to the host.
- The host broadcasts the world snapshot to all joiners ~20× per second.

### 8.1 Connection lifecycle

1. Host clicks **Host**:
   - `new Peer()` → PeerJS assigns a short ID (e.g. `gentle-fox-42`).
   - ID is shown in the menu with a **Copy** button.
   - Host listens on `peer.on('connection', ...)`.
2. Joiner clicks **Join** and types the host's ID:
   - `new Peer()` → gets own ID.
   - `peer.connect(hostId)` → opens a `DataConnection`.
   - On `open`, joiner sends `{ type: 'hello', name, color }`.
3. Host accepts: registers the joiner, sends `{ type: 'welcome', selfId, lobby: [...] }`, broadcasts updated lobby to everyone.
4. Host clicks **Start**: broadcasts `{ type: 'start' }`; everyone hides the menu and begins rendering.

### 8.2 Message protocol (JSON over DataChannel)

| From → To | `type` | Payload | Frequency |
|---|---|---|---|
| Joiner → Host | `hello` | `{ name, color }` | Once on connect |
| Joiner → Host | `input` | `{ mode: "mouse"\|"keys", mx, my, keys: {w,a,s,d} }` | Every frame (~60/s); coalesced if channel is busy |
| Joiner → Host | `respawn` | `{}` | On clicking Respawn |
| Host → Joiner | `welcome` | `{ selfId, lobby }` | Once on accept |
| Host → All | `lobby` | `{ players: [{id, name, color}], winScore }` | On lobby change OR when host edits win score |
| Host → All | `start` | `{ winScore }` | On host clicking Start (locks the session win score) |
| Host → All | `state` | `{ t, orbs: [{id, kind, x, y, r, color, name, alive, score, speed}] }` | 30 Hz |
| Host → All | `event` | `{ kind: "eat", eaterId, victimId, roll: "grow"\|"boost" }` | On event |
| Host → All | `victory` | `{ winnerId, winnerName, winnerColor, finalScores: [{id, name, score}] }` | Once when an orb reaches `WIN_SCORE` |
| Host → All | `reset` | `{ winScore }` | When host clicks **Start New Session** from the Victory overlay (may carry a new `winScore` if host changed it) |
| Host → All (minus source) | `notice` | `{ text, color }` | On player join / leave; rendered as a transient toast in the bottom-left of every recipient's screen |

**State message format:** orbs array is sent in full each tick (small N, simple to reason about). Each orb entry: `[id, x|0, y|0, r|0, score, alive ? 1 : 0]` — positions rounded to integers to keep payloads small. With ~38 orbs total (8 players + 30 bots) × ~22 bytes each = ~835 B/tick × 20 Hz ≈ 17 KB/s/joiner. Acceptable.

### 8.3 Client-side interpolation (joiners)

Joiners receive `state` snapshots at `STATE_HZ` (30 Hz) but render at ~60 fps. Snapping orb positions directly produces visible stepping. Instead, joiners apply the snapshot to **target** positions (`targetX`, `targetY`, `targetR`) and the render loop lerps current positions toward target every frame:

```
LERP = 0.32  // ~3 render frames to converge on a fresh target at 60 fps
o.x += (o.targetX - o.x) * LERP
o.y += (o.targetY - o.y) * LERP
```

Teleports (orb respawn, or any single-snapshot position jump >200 px) bypass the lerp and snap directly — lerping across the arena would look like a glitch.

Note: there is no input prediction for the local orb. Joiners send input to the host and wait for it to come back authoritatively. RTT latency is visible on the local orb but the interpolator smooths it into continuous motion.

### 8.4 Disconnects

- Joiner closes tab → host's `conn.on('close')` removes them from the simulation; their orb is converted to a small ambient orb.
- Host closes tab → joiners' `conn.on('close')` triggers the **Disconnected** state.

### 8.5 Limits

- Recommended max players: **8**. Beyond that, payload sizes and host CPU become noticeable on consumer hardware.
- **Late join is supported.** A peer that runs `peer.connect(hostId)` after the host clicks **Start Game** receives `welcome` + immediate `start` from the host, and an orb is spawned for them via `findSafeSpawn()` on the next host tick. Everyone in the session sees a `notice` toast naming the joiner.
- No reconnection logic — if a peer drops, they must re-join (which works the same way as a fresh late join).

---

## 9. AI Bots

Bots exist **only on the host**; joiners only see them through `state` messages. Each non-player orb runs `botThink()` once per host-tick.

### 9.1 Decision priority (highest first)

0. **Bounce cooldown.** If `aiState.bounceTimer > 0`, decrement it and follow the rebound direction stored in `aiState.wander` — ignore chase/flee/wander entirely. Set to `AI_BOUNCE_FRAMES` (30) by `moveOrb` whenever a bot hits a wall, so chase logic can't immediately pin the bot back into the wall.
1. **Flee.** Scan all orbs within `AI_VISION_R`. If any has `r > self.r` (strictly bigger), steer directly away from the nearest one.
2. **Chase.** Scan all orbs within `AI_VISION_R`. If any has `r < self.r` (strictly smaller), steer toward the nearest one.
3. **Wander.** Jitter `aiState.wander` by `±0.05 rad/frame` and steer along it.

The AI decision uses a **simple size comparison**, not `EAT_RATIO`. Bots will pursue rivals they cannot yet eat (size diff <10%) because the visible hunting feels right, and the actual consumption is still gated by `EAT_RATIO` in §5.4 — same-size bots collide harmlessly. Without this looseness, freshly-spawned bots (all in the 10–14 range) rarely find a target that clears the 10% size gate and just wander.

### 9.2 Smooth turning

In chase/flee mode, `aiState.wander` is **lerped** toward the target angle (not snapped), clamped to `AI_MAX_TURN` rad/frame. At 0.06 rad/frame (~3.4°/frame, ~206°/sec) a full 180° turn takes ~0.5 s. Without this clamp, bots used to whip their heading instantly when a target came into range, which looked twitchy.

Note: bots do **not** actively avoid walls. They bounce off them physically (§5.3), which is enough to keep them inside the arena and produces a more lively, chaotic motion than steering avoidance.

### 9.3 Bot tuning

| Parameter | Value |
|---|---|
| Initial radius | `AI_SPAWN_RADIUS` ± `AI_SPAWN_RADIUS_JITTER` (12 ± 2 px, range [10, 14]) |
| Initial speed | `AI_SPAWN_SPEED` (2.4 px/frame, fixed — slower than `START_SPEED`) |
| `AI_VISION_R` | 350 px |
| `AI_MAX_TURN` | 0.06 rad/frame (smooth-turn cap) |
| `AI_BOUNCE_FRAMES` | 30 (~0.5 s rebound cooldown after hitting a wall) |
| Wander jitter | ±0.05 rad/frame |
| Respawn delay | 240 frames |

Bots cannot use mouse/keys — their `vx, vy` are set directly by AI.

---

## 10. Collisions

Each host frame, in order:

1. **Movement integration.** All orbs advance by velocity, clamped to arena bounds.
2. **Pairwise consumption pass.** For each pair `(a, b)` where both are alive:
   - Compute `d = distance(a, b)`.
   - If `d < max(a.r, b.r) + min(a.r, b.r) * 0.5` (overlap test) **and** `max(a.r, b.r) ≥ min(a.r, b.r) * EAT_RATIO` (size gate):
     - The larger eats the smaller (see §5.4).
3. **Bot respawn check.** Any dead bot whose `deathFrame + BOT_RESPAWN_FRAMES ≤ now` respawns via `findSafeSpawn()` (§5.5) with default starting stats.
4. **Player respawn check.** Any joiner that sent a `respawn` message (and the host's own player on local click) is respawned via `findSafeSpawn()` (§5.5) with default starting stats.

Pair iteration is `O(N²)` — fine for N ≤ ~30. No spatial hash needed at hackathon scale.

---

## 11. Rendering

Every peer renders independently from its local copy of the orb list (authoritative on host, snapshot+interpolation on joiners).

**Frame structure:**
1. Clear with background color.
2. Draw faded grid (parallax with camera).
3. Draw arena boundary rectangle.
4. Draw all orbs in ascending `r` order so big ones overlap small ones.
5. Draw minimap (top-down arena overview, local player highlighted yellow).
6. Update HUD via DOM (every 6th render frame to throttle reflow).

**Sim/render decoupling:** the simulation runs on `setInterval(simTick, 1000/60)`, NOT on `requestAnimationFrame`. Rendering runs on its own `rAF` loop and is purely a consumer of the simulation state. This means a heavy overlay (e.g. the dead overlay used to use `backdrop-filter: blur(8px)` and slash the rAF rate) cannot slow down the simulation or the host's 20 Hz state broadcasts — joiners stay smooth even if the host's render thread is loaded.

The dead overlay is also rendered without `backdrop-filter` and with `pointer-events: none` on its outer wrapper (only the card itself captures clicks), so the game stays visible behind it and clicks pass through to the canvas.

Caveat: when the host's tab is **backgrounded**, browsers throttle *both* `setInterval` and `rAF` to ~1 Hz. The only true fix for that is moving the sim to a Web Worker (workers aren't throttled). Not currently implemented — see §14.

**Orb rendering:**
- Filled circle in `color` with a 2 px outline in `darkColor`.
- Inner highlight: small white-translucent circle offset toward upper-left (gives a "bubble" feel).
- Name label drawn above the orb in white with a black stroke, scaling with `r`.
- Local player's orb has a soft glow (`shadowBlur = 12`).

**HUD elements (HTML overlay, not canvas):**
- Top-left: your radius, speed, **score** (with a progress bar showing `score / WIN_SCORE` — the session target chosen by the host).
- Top-right: live leaderboard (top 6 by score, you highlighted yellow). Each row shows colored swatch + name + score.
- Bottom-right: 160×160 minimap canvas.
- Bottom-left: connection status indicator (green = connected; yellow = lagging if no `state` in 500 ms; red = disconnected).

---

## 12. File Layout

```
project/
├── index.html        # markup + styles + main game script
├── peerjs.min.js     # vendored OR CDN-loaded: https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js
└── spec.md           # this file
```

Everything (CSS, JS, palettes) lives in `index.html`. PeerJS is the only external dependency and can be inlined for fully-offline play (LAN parties without internet still need a signaling server, though — see §14).

---

## 13. Browser Support

| Feature | Min support |
|---|---|
| Canvas 2D | All modern browsers |
| `requestAnimationFrame` | All modern browsers |
| WebRTC DataChannel | Chrome 56+, Firefox 52+, Safari 11+, Edge 79+ |
| `navigator.clipboard.writeText` (for Copy ID button) | All modern browsers (HTTPS or `file://`) |

Tested target: latest Chrome / Edge / Firefox on Windows 11. Mobile is not a target for v1 (touch controls aren't designed).

---

## 14. Non-Goals / Known Limitations

- **No dedicated server.** Host is authoritative; if host quits, the game ends.
- **No NAT traversal guarantee.** WebRTC works through most home NATs via PeerJS's STUN, but symmetric NATs may fail. TURN relays are out of scope.
- **No persistence.** No accounts, no saved scores, no cosmetics.
- **No anti-cheat.** Joiners trust the host; the host trusts joiner input. Acceptable for friend-group play.
- **No sound.** Silent by design.
- **No spectator mode.** Dead players must respawn or leave.
- **Fixed bot count.** 30 bots regardless of human players.
- **No private rooms beyond ID secrecy.** Anyone with your host ID can join; treat the ID like a session password.

---

## 15. Tuning Cheat Sheet

| Symptom | Knob |
|---|---|
| Game feels too slow | `START_SPEED` ↑ |
| Growing too fast | `GROW_AMOUNT` ↓ |
| Sessions end too quickly / slowly | host picks a higher / lower **Target score** in the lobby (default `WIN_SCORE_DEFAULT`) |
| Death penalty feels too harsh / too soft | `POINTS_PER_DEATH_ABS` (default 2) |
| Speed boosts feel pointless | `SPEED_AMOUNT` ↑ or `MAX_SPEED` ↑ |
| Arena feels empty | `BOT_COUNT` ↑ |
| Bots too easy / too hard | `visionR` or AI flee/chase thresholds |
| Bots bounce off walls too predictably | add small random kick to reflected velocity (`±0.2 rad` perturbation on bounce) |
| Joiners jittery | `state` broadcast Hz ↑ (cost: bandwidth) |
| Joiners eat bandwidth | `state` broadcast Hz ↓ + tighter rounding |
| Cannot connect across networks | enable TURN server in PeerJS config |
