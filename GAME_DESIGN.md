# Golden Goat — Game Design & Logic Reference

> **Single source of truth** for the game's design, rules, and implementation.
> Keep this file **up to date with every change** to the project. When you change
> behavior in `index.html`, update the matching section here in the same task.

---

## 1. Overview

**Golden Goat** is a browser-based 3D third-person "goat simulator" style game.
You control a golden goat (Leonidas) that rampages through a small city, knocking
over people and objects with its horns, driving stolen cars, and — in Boss Mode —
fighting a giant farmer end boss.

- **Engine:** [Three.js](https://threejs.org) r0.161 via CDN ES-module import map.
- **Everything lives in a single file:** [`index.html`](index.html) — HTML/CSS UI +
  one `<script type="module">` with all game code. No build step.
- **Served over HTTP** (ES modules require it). Local: `python3 -m http.server`
  (see [`.claude/launch.json`](.claude/launch.json), port 8724).

---

## 2. Controls

| Key | Action |
|-----|--------|
| `W` / `S` / `↑` `↓` | Move forward / backward (goat) · accelerate / brake (car) |
| `A` / `D` / `←` `→` | Turn (goat) · steer (car) |
| `Space` | Jump (double-jump in air). Land on a car to **carjack** it. |
| `Shift` | **MEGA-STOSS** (goat AoE knockback) · **Turbo** (while driving) |
| `E` | Enter / exit nearest car |
| `F` | Shoot the pistol in the **hand** slot (Boss Mode) |
| `Q` | Swap **hand** ↔ **back** inventory slot |
| `P` / `Esc` | **Pause / resume** |

Horns are **automatic**: any civilian/prop the goat touches while moving is knocked away.

### Pause
`P`/`Esc` or the top-center **⏸ button** toggles pause. While paused the game loop is
frozen (`update` skipped), held inputs are cleared, music is muted, and a pause overlay
shows (tap / click / `P` to resume). The pause button appears only while a round is active.

### Touch controls (phones & tablets)
Auto-enabled on touch devices (`pointer: coarse` / `ontouchstart`; sets `body.touch`,
hides the keyboard hint bar). A **virtual joystick** (bottom-left) drives the
goat/car (mapped to W/A/S/D), and **action buttons** (bottom-right) cover ⤴ jump,
💥 mega/turbo (hold), 🚗 car, 🔫 shoot (hold = full-auto), 🔄 swap. Buttons also respond
to mouse for desktop testing.

---

## 3. Game Modes

Chosen from the start overlay (two buttons). State var `gameMode = 'time' | 'boss'`.

### 3.1 Time Trial (`time`)
- 120-second countdown (`TOTAL_TIME`). Score as many points as possible.
- Ends when timer hits 0 → game-over screen with score + persistent highscore
  (`localStorage['leonidas_highscore']`).

### 3.2 Boss Mode (`boss`)
- **No timer.** Inventory HUD is shown; pistols are spawned in houses.
- Reach **`BOSS_GOAL` = 1000 points** → the **end boss spawns**.
- **Lose** if score drops to **0** (only after you have scored at least once —
  `everScored` guard prevents an instant loss at the start). Triggered by being hit
  by boss projectiles, police, or traffic.
- **Win** by defeating the boss (reduce its HP to 0): ram it with a car or shoot it.
- Win/lose both open the game-over screen with the appropriate title
  (`🏆 BOSS BESIEGT!` / `💀 VERLOREN!`). Losing plays a booming evil laugh.

---

## 4. Scoring

`addScore(base)` applies a **combo multiplier**: `gained = base * (1 + floor(combo/3))`.
`combo` increments on every scoring event and resets after `comboTimer` (2.5 s) of no
scoring. Combo milestones show toasts (8/16/30).

| Event | Base points |
|-------|-------------|
| Goat horn knock (civilian / prop) | 5 each |
| Car runs over person / prop | 8 × count |
| Car rams another car | 8 × 2 |
| Mega-Stoß (per target) | 15 (×count, capped) · 40 if it hits police |
| Smash tree with car | 6 |
| Detonate a gas pump | 20 |

### 4.1 Multi-kill bonus (UT-style)
`registerKills(n)` tracks a **kill streak** within a short rolling window
(`killStreakTimer`, ~1.6 s). Hitting many targets quickly grants escalating bonus
points + announcements (e.g. DOPPEL / MULTI / MASSAKER / UNAUFHALTSAM). The streak
resets when the window lapses.

### 4.2 Penalties
`penalty(amount, text)` subtracts points (clamped at 0), resets combo, flashes red,
shows a toast. In Boss Mode, reaching 0 (after `everScored`) ends the game as a loss.

| Source | Penalty |
|--------|---------|
| Police catches the goat | −20 |
| AI car runs the goat over | −15 |
| Boss tractor 🚜 / bale 🌾 / pitchfork 🔱 / manure 💩 | −90 / −50 / −35 / −25 |
| Break your car on too many trees | −40 |

---

## 5. World Generation

Half-extent `WORLD = 320`. Playable boundary `clampWorld()` keeps entities within
`WORLD * 1.15` so you can roam (almost) to the visible terrain edge.

### 5.1 Terrain (height field)
- `terrainHeight(x,z)` = sum of a few sine/cosine hills (range ≈ 0–20) minus a lake
  basin (`lakeDip`). **Everything sits on the terrain** (goat, cars, people, props,
  trees, buildings, boss).
- Rendered as a displaced `PlaneGeometry` (150×150 segments) with **vertex colors**:
  grass, **sand** near the waterline, rock on peaks.
- `groundAt(x,z) = terrainHeight + rampAddAt` (ramps add height). `roadY` clamps road
  height so roads sit on land.

### 5.2 Water / Lake
- `WATER_LEVEL` (≈ −1.2). One lake (`LAKE = {x,z,r,depth}`), carved by `lakeDip`.
- A large semi-transparent water plane (gently bobbing) + a darker deep plane.
- The lake is positioned **between road grid lines so no road crosses it**; any road
  segment that would pass over water is **clipped** (not rendered) and AI cars turn
  around at the shoreline.
- Goat/car in water (`terrainHeight < WATER_LEVEL`): **float** at the surface, get
  **slowed**, and emit splash particles.

### 5.3 Roads
- Curved network: 5 vertical + 5 horizontal **wavy** polylines. Rendered as ribbon
  meshes (`ribbonGeo`): a dark **asphalt-gray road** strip + a lighter **concrete
  sidewalk** strip + **white dashed** center line (`roadMat`/`walkMat`/`lineMat`).
- `roadSamples[]` (sampled points) used for placement avoidance and AI car paths.
- AI cars follow these polylines (`road.pts`), bouncing at ends and at water.

### 5.4 Buildings & Houses
- Dense grid (~26 spacing, skipped near roads & water). Each lot is either:
  - **Tower** (`addBuilding`): textured box (10 shared window-texture materials, UV
    scaled), roof + optional AC unit. Solid for goat & car (`buildings[]`).
  - **Enterable House** (`addHouse`, ~30% of lots): 4 walls with a **door gap** facing
    the nearest road, floor, pitched roof (hidden while you're inside), interior rug,
    ceiling lamp, shelf, and a knockable table. Wall collision boxes (`allWalls[]`)
    block the goat but leave the doorway open; house footprints (`houseSolids[]`) block
    cars (too big for doors). Pistols may spawn inside (Boss Mode).
- `houses[]` track footprint + roof for the "hide roof when inside" logic.

### 5.5 Props (destructible)
`props[]`. Types: `bench`, `trash`, `hydrant` (street props near roads), `table`
(inside houses), `pump` (gas station — **explodes** when knocked, +20). Knocked props
fly with physics, land, despawn after a few seconds, and respawn.

### 5.6 Trees & decoration
- `trees[]`: goat is **blocked** by trees (can't knock them); a **car can smash** them
  (tree flies, car takes damage). Smashing >5 trees within 20 s **destroys the car**.
- Bushes and drifting clouds for decoration.

### 5.7 Monuments
- **Jump ramps** (`addRamp`, 5 of them): orange wedge with stripes. Registered in
  `ramps[]`; `rampAddAt` adds height in the ramp footprint. Driving/running up and off
  the top **launches** you (real ballistic jump, scaled by speed). One aims at the lake.
- **Gas stations** (`addGasStation`, 2): canopy on pillars, shop (solid), sign, and two
  explosive fuel **pumps**.

---

## 6. Entities

### 6.1 Goat (Leonidas) — `goatState`
- **Gold** low-poly model: spherical body, head pivot (dips for headbutt), horns, ears,
  beard, 4 legs (walk-animated), tail. Fur material is gold/metallic.
- Movement: tank-style (turn + forward). Auto-horns knock nearby civilians/props while
  moving (`speed > 3`, not airborne). Police are immune to plain horns.
- Vertical physics via `groundPhysics`: follows ground, double-jump, ramp launch, water
  float/slow. `carjack` when landing on a car.

### 6.2 Civilians & Police — `people[]`
- **Civilians:** detailed humanoid (hips, torso, head with eyes/nose, hair/hat, arms +
  hands, legs + shoes), wander, knockable by **any** means (horns, car, mega, bullets).
- **Police:** larger, **navy uniform + cap + badge + baton**. They **actively chase** the
  goat and **penalize** it on contact (−20, cooldown). Immune to plain horns; **can** be
  taken out by **Mega-Stoß, car ram, or bullets**.
- Spawning: a base population plus **cluster spawns** (groups standing together) so you
  can mow several at once. Knocked people ragdoll, land, despawn, and respawn to keep the
  world populated.

### 6.3 Cars — `cars[]`
- AI traffic follows road polylines (two lanes). Drivable via `E` (carjack or enter).
- Drivable car physics: accel/brake/steer, turbo (Shift), terrain/ramp/water handling.
- Player car **runs over** people/props, **rams** other cars (they fly), and **smashes**
  trees (damage). Ramming the boss deals damage. AI cars run the goat over (penalty).
- Carjack: jump onto a car → it stops, the driver is **ejected** (GTA-style), enter with `E`.

### 6.4 End Boss (Boss Mode only) — `boss`
- A **giant, detailed, menacing farmer**: oversized body/overalls, plaid shirt, big head
  with **glowing red eyes**, angry brows, beard, straw hat, arms holding a pitchfork.
  Bigger and scarier than a normal person; collision radius `boss.r`.
- **Actively pursues** the goat across the map (moves toward the player every frame,
  only easing very close). `boss.hp` (start 100) shown in the top **HP bar**.
- **Throws projectiles at the player**, aiming at (and **leading**) the goat's position
  with enough range to reach: 🚜 tractor, 🌾 bale, 🔱 pitchfork, 💩 manure
  (`BOSS_AMMO`). Each hit penalizes and knocks the goat back.
- **Defeated** by car rams (−25 HP) or bullets (−8 HP) → win.
- An on-screen **arrow/radar** points toward the boss while it's alive.

### 6.5 Bullets — `bullets[]`
- Fired with `F` from the hand pistol. **Full-auto: hold `F` to keep firing** at
  `SHOOT_RATE` (~0.05 s ⇒ ~20 shots/s), gated by `shootCd`. **Ammo is infinite** once you
  hold a pistol (HUD shows `∞`). Bullets travel straight; hit the boss (−8 HP) and knock
  civilians/police/props. Limited lifetime/range.

### 6.6 Pistol pickups & Inventory — `pickups[]`, `inv`
- Pistols spawn on the floor of ~12 random houses (Boss Mode). Spin/bob; pick up by
  walking over them. Each pistol has **infinite ammo**.
- **Inventory** `inv = { hand, back }`, two slots. Pickups fill an empty slot (hand
  first); a loaded pair replaces the hand. `F` shoots the hand pistol (full-auto), `Q`
  swaps hand ↔ back. Items render on the goat (hand in front, back on the back) and in
  the HUD (`∞` ammo).

---

## 7. Physics summary (`groundPhysics`)

Shared by goat & car each frame for the active entity:
- **Follow ground:** `y = groundAt(x,z)`; in water, float at `WATER_LEVEL` and slow.
- **Jump/air:** gravity integration; land back on ground (`resetJumps` for the goat).
- **Ramp launch:** while on a ramp the climb rate is tracked; leaving the top at speed
  converts that climb into upward velocity (a real jump). Natural hills don't launch.

---

## 8. Camera
- Third-person follow behind the active entity (`updateCamera`). Height is **relative to
  the entity's terrain height**. Distance/height are **larger** (pulled further back) for a
  wider view; the car view is further still; inside a house the camera pulls in close & high.
- Smooth yaw approach to the entity heading; screen shake on impacts (`camShake`).

## 9. Audio & Music
- Tiny WebAudio synth (`tone`, `noise`) for SFX: thud, mega, jump, honk, combo, hurt,
  siren, crash, gunshot, and an **evil booming laugh on losing**.
- Generative background **music** (`Music`) with bass/kick/hat/snare/arp + pad. Intensity
  rises with score momentum, low time, and (Boss Mode) when the boss is active.

## 10. HUD / UI
- Top-left: **score** + combo + (Boss Mode) goal progress. Top-right: timer + best
  (Time mode). Top-center: **boss HP bar** + **boss direction arrow** (Boss Mode).
- Bottom-right: **inventory** (hand/back slots, Boss Mode). Bottom-center: controls help.
- Center: toast messages, red/white screen flash. Start & game-over overlays.

---

## 11. Key constants (quick reference)

| Constant | Value | Meaning |
|----------|-------|---------|
| `WORLD` | 320 | World half-extent |
| boundary | `WORLD*1.15` | Movement clamp |
| `ROAD_W` | 16 | Road width |
| `WATER_LEVEL` | ≈ −1.2 | Water surface height |
| `TOTAL_TIME` | 120 | Time-mode seconds |
| `BOSS_GOAL` | 1000 | Points to spawn the boss |
| boss HP | 100 | Bullet −8, car ram −25 |

---

## 12. Known limitations / ideas
- People/props/cars respawn to keep counts stable; no persistence between runs except
  the highscore.
- No mobile/touch controls. No real "lives" system (points act as health in Boss Mode).
- Future ideas: multiple boss phases, more weapons, ammo shops, minimap, boats.

---

## 13. Changelog (most recent first)
- **Pause + touch + publish:** added pause mode (`P`/`Esc`/⏸ button, freezes loop &
  mutes music); on-screen touch controls (virtual joystick + action buttons) auto-enabled
  on phones/tablets; published to a public GitHub repo + GitHub Pages. Added a custom
  golden-goat icon (`icon.svg` → `apple-touch-icon.png`/`icon-192/512.png`), a web app
  manifest (installable / add-to-home-screen), and an Open Graph / Twitter link-preview
  image (`og.png`). Icons/PNGs are rendered from the SVGs via `qlmanage`.
- **Fix:** entering cars (`E`) used 3D distance, so terrain height (cars sit on the
  ground, the goat's `pos.y` stays 0) pushed every car out of range. Now uses horizontal
  distance with a ~9-unit reach, so you can enter from any side, with some distance.
- **Realism + full-auto update:** pistol has infinite ammo and full-auto fire (hold `F`,
  ~20/s, `∞` in HUD). More realistic art: houses get glazed windows, a door, chimney &
  varied roof colors; cars get a windshield/side glass, bumpers, grille, headlights &
  chrome hubcaps; people get shoulders & ears; roads are asphalt-gray with white dashed
  lines; tower facades have framed/mullioned, varied-lit windows + floor bands.
- **Golden Goat update:** renamed to Golden Goat; gold goat; boss aims at & leads the
  player with proper range; boss bigger/scarier/detailed + actively pursues + direction
  arrow; gray roads that don't cross water; reachable map edge; more people + cluster
  spawns + multi-kill bonus; cars/police/all people knockable by car/bullets/mega;
  evil laugh on losing; camera pulled further back.
- **Boss mode update:** added Boss Mode, inventory (hand/back), pistols in houses,
  bullets, end boss + thrown projectiles, win/lose conditions.
- **World update:** terrain height field, lake/water, jump ramps, gas stations.
- **City update:** denser city, enterable houses with interiors.
- **Core:** goat, third-person camera, auto-horns, mega-stoß, drivable cars, AI traffic,
  police, destructible props/trees, carjacking, combos, time-trial mode, music/SFX.
