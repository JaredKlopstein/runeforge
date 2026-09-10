# Runeforge — Ages of the Skilled

A complete Age-of-Empires-style skirmish in **one HTML file**. Procedural Canvas 2D art, WebAudio, no build step.

**Play:** https://runeforge-six.vercel.app · **Tutorial:** https://runeforge-six.vercel.app/#tutorial

Deploys automatically from `main` via Vercel. The in-game **What's New** screen (title menu) carries the version, changelog and roadmap.

```bash
python3 -m http.server 8765 --bind 0.0.0.0
# open http://127.0.0.1:8765/index.html
```

## Play

Title → **Tutorial** for a guided first game, or **New Skirmish** → pick a seed and hostility → **March to War**. Destroy the enemy Town Center.

Hotkeys (also in the in-game pause menu):

- WASD / edge pan, wheel zoom, left-drag select, right-click command
- `.` idle villager · `,` select army · Shift+right-click queues
- Build (villagers selected): H house, E store, J farm, B barracks, L wall, O gate, T tower, G forge
- Train: V villager, N scout, G guard (barracks selected), R ranger, M ram, K knight
- C auto · Z attack-move · X stop · P pause · F speed · Esc menu

Hold the **Rune Stone** in the map center for a food trickle and bonus XP.

## Layout

| file | |
|---|---|
| `index.html` | the entire game |
| `CLAUDE.md` | architecture and session handoff for coding agents |

Settings and Continue saves live in `localStorage` (`runeforge_cfg`, `runeforge_save`).
