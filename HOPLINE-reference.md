# HOPLINE — Project Reference

A voxel "cross the lanes" arcade game built as a **single self-contained HTML file** with [Three.js](https://threejs.org) (r128, loaded from a CDN). Working title: **HOPLINE**, token ticker concept: **$HOP**. Intended for submission to the **Oryth Gamecup**, with a coin/token to launch alongside.

This document is the source of truth for how the game works, what's tunable, and what's planned. Keep it with the project; re-share it with Claude to resume work with full context.

---

## 1. At a glance

- **Genre:** endless lane-crosser (Crossy-Road lineage) with a distinct identity, juice, and a greed/combo economy.
- **Tech:** one `.html` file. Three.js r128 global build via `cdnjs`. No build step, no dependencies, no server. Runs by double-clicking the file or hosting it statically.
- **Rendering:** orthographic, mirrored isometric camera. Z is up. Everything is built from `BoxGeometry` / `CylinderGeometry` voxels.
- **Persistence:** browser `localStorage` (wrapped so it degrades to in-memory if blocked, e.g. inside a sandboxed iframe).
- **Controls:** keyboard only — Arrow keys / WASD to hop, `Esc` to pause.

### Coordinate system (important)
- `Z` is **up** (`camera.up = (0,0,1)`).
- Forward / distance progress is **+Y** (rows are indexed along Y: row `i` sits at world `y = i * TILE`).
- Lateral movement is along **X** (columns `MIN_COL..MAX_COL`).
- Vehicles, logs and trains travel along **X**.
- The camera is **mirrored** (sits on the −X side) and zoomed in via an orthographic frustum (`VIEW`).

---

## 2. How to run / iterate

- Open the `.html` directly in any modern browser. Audio starts on first key/click (browser autoplay policy).
- It is a single file: all CSS, HTML and JS are inline. The only external request is the Three.js CDN script (and a Google Font, with a system fallback) — both degrade gracefully offline except 3D won't render without Three.js.
- To host: drop the file on any static host (GitHub Pages, Netlify, itch.io, etc.).

---

## 3. Game modes

| Mode | How it starts | Seed | Character/abilities | Score goes to |
|---|---|---|---|---|
| **Endless** | "Play" button / `Enter` | Fresh random each run | Selected character + its ability | Personal best |
| **Daily Challenge** | "★ Daily Challenge" button | Deterministic from today's date (`YYYYMMDD`) | **Locked to base Chick, no ability** (fairness) | Local daily leaderboard |

The HUD shows a `★ DAILY` label during daily runs. Retrying preserves the current mode.

---

## 4. Core systems

### 4.1 World generation (rows)
- The world is an infinite stream of **rows**, generated ahead of the player and culled behind.
- `AHEAD = 26` rows generated ahead, `BEHIND = 24` kept behind (large so the low, zoomed camera never reveals empty space).
- Each row is one of four **types**, chosen by weighted random in `chooseType(i)`:
  - **grass** — safe; may hold tree obstacles + decorative flowers.
  - **road** — cars and trucks travel along X.
  - **water** — round logs travel along X; you must ride them or you drown.
  - **rail** — a crossing signal lamp + a periodic full-width train.
- Weights shift with **difficulty** and **biome** (see below). Run-length guards prevent too many identical hazard rows in a row.
- Ground strips are very wide (`STRIP_W`) and lane markings/ties tile across the whole width (`DASH_HALF`) so nothing stops short of the viewport.

### 4.2 Biomes (zones)
- The world is divided into **biome zones** by row index. The **opening zone is long** (`FIRST_LEN = 75` rows) so the first biome lasts a while; afterwards a new biome every `ZONE_LEN = 50` rows.
- `BIOMES` cycle: **Sunset Isles → Neon City → Autumn Woods → Frostpeak → (repeat)**.
- Each biome overrides the **palette** (grass/road/water/rail/dash/tie colors + tree colors) and tunes **hazard weights** (`w`) and **lane speed** (`spd`). Examples:
  - *Neon City* — dark roads, neon dashes, more/faster traffic.
  - *Autumn Woods* — warm palette, orange/red trees, calmer, more grass.
  - *Frostpeak* — icy palette, more (and faster) trains.
- Crossing into a new biome shows an "Entering <name>" popup. Biome boundaries are **deterministic** (pure function of row index), so Daily runs stay fair.

### 4.3 Difficulty
- `difficulty()` = `clamp((furthest - 6) * 0.006, 0, 1.15)` — a gentle, slow ramp that only reaches max around ~200 rows. The first few rows are guaranteed safe.
- Difficulty raises lane speeds, narrows vehicle gaps, and shifts row weights toward hazards. It is driven by **distance only** (`furthest`), independent of the combo/coin bonus.

### 4.4 Player & movement
- Grid-based hopping with a short jump arc, squash/stretch, and facing that follows actual world-movement direction (so it reads correctly under the mirrored camera).
- **In-air immunity:** collisions are only evaluated while settled or on landing — never mid-hop. (This fixed an early bug where jumping past a car still killed you.)
- On water, the player **rides** the log under them (`RIDE_Z` height) and drifts with it; hopping snaps to the grid. Landing where no log is = drown.
- Obstacles (trees, the rail lamp) register blocked columns and stop movement into them.

### 4.5 Collisions & death
- **crash** (car/truck), **train**, **splash** (water), **eagle** (idle too long).
- The **eagle** enforces forward progress: if you don't reach a new furthest row within a shrinking time limit, a red vignette warns you, then the eagle grabs you. This is the anti-camping pressure.
- Death triggers screen shake, a debris/splash particle burst, and a death animation (flatten / sink-with-ripple / eagle carry).

### 4.6 Near-miss system (juice + skill reward)
- A genuine near-miss is rewarded with brief **slow-mo**, **screen shake**, a **whoosh**, a spark burst, **bonus points**, and a floating popup.
- **Exploit guard:** only fires when a vehicle is **closing on you** (`(player.x − v.x) * dir > 0`) and the lane is genuinely fast (`speed > 52`). This kills the "tail a slow car and farm points" exploit while keeping real close calls.
- The train awards a bigger near-miss ("CLOSE CALL").

### 4.7 Coins, combo & score
- **Coins** spawn across the world, weighted toward danger: worth **1 in grass**, **3 on roads/rails**. You collect by landing on their tile.
- **Combo multiplier:** consecutive forward hops build a combo; the multiplier is `min(8, 1 + floor(combo/4))` (×2, ×3 … ×8). It drops if you stall for `COMBO_WINDOW` seconds or move backward. Shown under the score, pulsing on increase.
- The multiplier scales **coin** and **near-miss** points.
- **Score model:** `score = furthest (distance) + bonus`, where `bonus = (coins + near-misses) × combo`. Distance still drives difficulty and the eagle; bonus is the skill/greed layer.
- Coins also accumulate into a persistent **bank** (`coinsLife`) shown on the menu — the currency for the shop.

### 4.8 Characters, abilities & shop
- Characters are **meaningful choices, not skins** — each has one ability. The shop spends banked coins to unlock them permanently.

| Character | Ability | Effect | Cost |
|---|---|---|---|
| Chick | — | Balanced starter (free) | 0 |
| Frog | Magnet | Auto-grabs coins from adjacent tiles | 60 |
| Duck | Golden Touch | Coins worth ×2 | 60 |
| Pig | Combo Keeper | Combo window lasts twice as long | 70 |
| Cat | Nine Lives | Survive one car/train hit per run (brief invuln) | 90 |
| Axolotl | Waterproof | Never drowns | 100 |
| Robot | Overclock | Longer slow-mo, +1 on every near-miss | 110 |

- The menu shows each character's ability and an **Unlock · N 🪙** button when locked (with a "not enough" shake).
- **Daily runs ignore abilities** (locked to base) for leaderboard fairness.
- *Balance note:* Waterproof is deliberately priced high; for competitive integrity the fix is the daily lock, not just price.

### 4.9 Daily leaderboard (local)
- Each daily run is recorded under a per-day key and ranked. The board panel shows today's top runs, today's best, and an all-time daily record. Game-over shows your **rank today**.
- This is intentionally **local-only scaffolding** — see the migration plan in §7.

### 4.10 Audio (fully synthesized, no asset files)
- **SFX:** Web Audio oscillator/noise blips for hop, crash, splash, train, eagle, coin, whoosh, score, select, beep.
- **Music:** a small look-ahead **step sequencer** plays one of **three looping chiptune tracks** (Sunny / Breezy / Island — different progressions, BPM, lead waveform). A **different track is chosen each time you start a level** (avoids repeating the previous one). An intensity layer thickens as you go further.
- **Two independent mutes:** SFX (🔊) and Music (🎵), each persisted. Audio unlocks on first interaction.

### 4.11 Day/night, particles, camera
- **Day/night:** a continuous cycle interpolates a CSS gradient sky (behind a transparent canvas) + a sun/moon disc + fog and light colors through Tropic Noon → Mango Sunset → Neon Night → Cotton Dawn. It's cosmetic and independent of biomes.
- **Particles (`FX`):** a fixed pool of reusable quads for landing dust, near-miss sparks, splash, and crash debris. Uses cosmetic RNG so it never affects determinism.
- **Camera:** mirrored isometric orthographic, eased follow of player Y and partial X, plus a decaying shake offset.

---

## 5. Determinism & the seeded RNG (Daily)

Randomness is split into **two streams**:

- **World RNG** (`RNG`, via `rand` / `randi` / `choice` and the `RNG()<…` calls in generation) — used **only** for layout: row types, lane directions/speeds, vehicle colors, tree/flower placement, coin placement. In Daily mode this is a seeded `mulberry32(todaySeed)`; in Endless it's `Math.random`.
- **Cosmetic RNG** (`mrand`, plus direct `Math.random` in FX/shake/music/rail re-arm timers) — never seeded, so particles and effects don't desync the layout.

Because rows generate strictly in index order and the world RNG is consumed only by generation, the **same seed produces the same course**. Caveat: moving-object *timing* depends slightly on framerate, so two players on the same seed get identical **layout and placement** (what matters for fairness) but not pixel-identical car positions over time. Frame-perfect parity would require a fixed-timestep simulation — a later option if the competition demands it.

---

## 6. Tunable constants (where to change feel)

Near the top of the script:

| Constant | Meaning |
|---|---|
| `TILE` | World units per grid tile (42) |
| `MIN_COL` / `MAX_COL` | Lateral play bounds |
| `VIEW` | Orthographic frustum height — **smaller = more zoomed in** |
| `CAM_OFF` | Camera offset/angle; `MIRROR` flips it to the other side |
| `AHEAD` / `BEHIND` | Rows generated ahead / kept behind |
| `STRIP_W` / `DASH_HALF` | Ground width and lane-marking coverage |
| `MOVE_TIME` / `HOP_HEIGHT` | Hop speed and arc |
| `IDLE_BASE` | Base seconds before the eagle (shrinks with difficulty) |
| `RIDE_Z` | How high the player sits on a log |
| `COMBO_WINDOW` | Seconds of forward flow before the combo drops |
| `FIRST_LEN` / `ZONE_LEN` | Opening biome length / subsequent biome length |
| `BIOMES` | Per-biome palette, hazard weights (`w`), speed (`spd`) |
| `CHARACTERS` | Names, builders, `abil`, `desc`, `cost` |
| `difficulty()` | The difficulty ramp curve |

Audio feel lives in the `AudioSys` module (`TRACKS` for music; the SFX methods for sounds).

---

## 7. Leaderboard → real backend (migration plan)

The local board is structured so going global is a contained change, **not** a redesign. You already have:

1. **A stable per-day seed:** `todaySeed = YYYYMMDD`. Send it with every score so the server can group/validate by day.
2. **A fairness-locked ruleset:** Daily forces base character, no abilities, seeded layout.
3. **A single submission point:** `addDailyRun(score)` in `showGameOver()` for daily mode. This is the one place to also `POST` to your API / write to your DB / submit on-chain.
4. **A single render point:** `renderBoard()` reads `getDailyRuns()`. Swap that to fetch from your API.

**To go live:**
- Replace `getDailyRuns()` / `addDailyRun()` with API calls (`GET /daily/:seed`, `POST /daily/:seed { score, signature }`).
- Add identity (wallet address or account id) to each entry so the board can show names/addresses.
- **Anti-cheat is the hard part.** A client-side score is trivially forgeable. Options, increasing in robustness: (a) server-side plausibility checks (max score per elapsed time); (b) submit the full **input replay** + seed and **re-simulate on the server** with a fixed timestep to verify the score (this is why deterministic generation matters); (c) for on-chain rewards, verify the replay in a backend and have it sign/attest results before any payout. Do **not** trust raw client scores for token payouts.

---

## 8. $HOP / token integration hooks (design only)

The economy is already shaped to plug into a token; these are **engineering hooks, not financial or legal advice** — tokenomics, compliance, and launch decisions are yours and your advisors'.

- **In-game coins ↔ $HOP:** banked coins (`coinsLife`) are the natural mirror of an off-chain/on-chain balance. Swap the `localStorage` bank for a wallet balance read.
- **Shop:** character unlocks are a coin sink — the obvious first on-chain spend (or an NFT-gated cosmetic system).
- **Daily rewards:** seeded daily + verifiable replays → a fair, low-cheat surface for token rewards or competitions.
- **Sequencing advice:** ship a genuinely fun game first; let the token ride on it. A fun game survives a rocky token; a token can't save an unfun game.

---

## 9. Known caveats

- **Framerate determinism** (see §5): layout is deterministic; live object timing isn't frame-perfect.
- **Ability balance for competition:** handled by locking Daily to base character. Endless intentionally lets abilities shine.
- **`localStorage`:** wrapped in try/catch; inside a sandboxed preview iframe it falls back to in-memory (best scores/coins won't persist there, but do when the file is opened/hosted normally).
- **Keyboard only:** by design. No touch input yet — desktop-first.
- **Three.js r128:** chosen for the simple global build with no import maps. Avoids r142+ geometry (e.g. no `CapsuleGeometry`).

---

## 10. Roadmap (built vs. planned)

**Built:** core lane-crosser; mirrored zoomed camera; vibrant candy identity + day/night sky; round logs; realistic cars; crossing-signal lamps; pause; juice pass (near-miss, shake, particles, death cam); greed economy (coins + combo multiplier); ability characters + unlock shop; daily seed + local leaderboard; biome zones; 3 random looping tracks + dual mute.

**Planned (rough priority):**
1. **Missions / achievements** — "cross 5 tracks", "60 coins in a run", "hit ×6 combo" — cheap, strong retention; pays coins.
2. **Authored set-pieces** — occasional boss truck, collapsing bridge, parade row — deliberate pattern-breaks that read as hand-designed.
3. **Mascot personality pass** — idle fidgets, glance-before-crossing, panic near trains, celebration on a new best.
4. **Chase-pressure mode** — a rising flood/creature as an alternate, high-urgency mode.
5. **Power-up pickups** — temporary magnet / shield / slow-mo / 2× on the field.
6. **Real leaderboard backend** (§7) + optional on-chain rewards.
7. **Polish** — share-score / "beat your best by X" game-over, control options, tutorial.

---

## 11. Gamecup talking points

- **Not a reskin:** distinct identity (candy → biome journey), a greed/combo skill loop, near-miss risk reward, and authored game-feel — not 1:1 cloned mechanics.
- **Replay value:** combo chasing, risky coins, ability characters that change how you play, biome variety, and a fair daily challenge.
- **Technical story:** a complete, dependency-light single-file game; deterministic daily generation; fully synthesized audio (zero asset weight).
- **Token-ready:** a coin sink (shop), a fair competitive surface (seeded daily + verifiable replays), and a clean migration path to a real backend.

---

*Generated as a living reference. Re-share with Claude to continue development with full context.*
