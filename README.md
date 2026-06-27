# 🐐 Golden Goat

A browser-based 3D third-person "goat simulator" built with [Three.js](https://threejs.org).
Rampage through a city as a golden goat: headbutt people and objects, steal and drive
cars, launch off ramps, swim in the lake — and in **Boss Mode**, find pistols and take
down a giant farmer end boss.

## ▶ Play

**[Play it here →](https://cb2206.github.io/golden-goat/)**

No install needed — it runs entirely in the browser.

## Controls

**Keyboard**
- `W/S` move/drive · `A/D` turn/steer
- `Space` jump (double-jump; land on a car to steal it)
- `Shift` Mega-Stoß (goat) / Turbo (car)
- `E` enter/exit car · `F` shoot · `Q` swap inventory
- `P` / `Esc` pause

**Touch (phones & tablets):** on-screen joystick + action buttons appear automatically.

## Modes
- **⏱️ Time Trial** — score as much as possible in 120 seconds.
- **👹 Boss Mode** — reach 1000 points to summon the end boss; beat it by ramming or
  shooting it. Drop to 0 points and you lose.

## Tech
- Single file ([`index.html`](index.html)), Three.js via CDN import map, no build step.
- Procedural world: terrain height field, lake, curved roads, dense city with enterable
  houses, traffic, ramps, gas stations, generative music & SFX.

See [`GAME_DESIGN.md`](GAME_DESIGN.md) for the full design & logic reference.

## Run locally
```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

---
🤖 Built with [Claude Code](https://claude.com/claude-code)
