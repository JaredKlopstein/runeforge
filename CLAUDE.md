# Runeforge — agent notes

Single-file browser RTS. **All game code, art, UI, and audio live in `index.html`.** Do not add a build step, bundler, CDN, or external image/audio files. Art is procedural Canvas 2D. Sound is WebAudio.

Run: `python3 -m http.server 8765 --bind 0.0.0.0` from this directory, then open `http://127.0.0.1:8765/index.html`. Hard-refresh after edits.

This file is the handoff from a Grok session (2026-09-09) that continued Claude session `b9b033c9-cd3d-4eb3-bc76-273aff702e9e`. Claude built the original playable game and two playtest/fix rounds. Grok resumed mid round-three freeze fix and then added presentation, art, units, and QoL.

## Constraints (do not break)

- One self-contained HTML file. No `import`, no spritesheets, no Google fonts.
- Map is `N=96`, tile `TS=32`. Pathfinding uses A* with a per-frame budget (`PC.budget=12`) plus flow fields for groups of 5+.
- `pass(x,y,team)` is team-aware because **open friendly gates are walkable**. Cache keys in `findPath` / `getFlow` include `PASS_TEAM`. Toggling a gate calls `invalidatePaths()`.
- Hidden-tab playtest mode is `?bg=1` (Worker ticks the sim). `requestAnimationFrame` must **not** queue while hidden or the tab freezes when shown. See `schedule()` / `frame()` / visibility handler at the bottom of the script.
- Settings persist in `localStorage` key `runeforge_cfg`. Autosave every 30s to `runeforge_save`.

## Screen flow

`UI.screen`: `title` → `lobby` / `settings` / `howto` → `game` (`pause`, `skills`). `document.body.play` shows the HUD.

- Title: procedural dusk canvas (`renderBoot`) + HTML menu. Music (`MUS`) starts on first click.
- Lobby (`#lobby`): seed, hostility (`DIFF`: Squire / Lord / Warlord), map preview, **March to War**.
- Hash shortcuts: `#lobby` `#settings` `#howto` `#play` (skips tutorial).

## Units (`UNIT_DEF`)

| type | from | age | notes |
|---|---|---|---|
| villager | tc | 0 | Auto gather/build/smith/farm. Manual order turns Auto off (`dropAuto`). |
| scout | barracks | 0 | `sight:12`. Auto-searches fog (`autoScout` / `pickScoutTile`). Flees fights. Reports enemy buildings once (`b._spotted`). Gold on minimap. |
| guard | barracks | 0 | melee line |
| ranger | barracks | 1 | ranged; half damage to buildings |
| ram | barracks | 1 | 3.2× vs buildings, 0.35× vs units; idles onto enemy buildings |
| knight | barracks | 2 | cavalry |

Starting player villagers spawn **Auto ON**. New trained villagers follow `team.autoNew`.

## Buildings (`BLD_DEF` + `UPG` two tiers)

tc, house, store, **farm** (J), barracks, forge, wall (L), **gate** (O), tower.

- **Gate**: open (default) = owner walkable, enemies blocked. Locked = nobody. Select gate + O (no villagers selected) toggles. Construction 5 / 14 upgrades match walls.
- **Farm**: one villager, food goes straight to stock, fishing XP. Upgrades Orchard / Granary Farm. Auto vils sow when food is low.
- Walls autotile from neighbors (`isWallish`).
- Rally: right-click empty ground with a building selected; flag is drawn on the map.

## Map extras

- **Rune Stone** (`G.relic`) near map center. Exclusive control (units within ~4.2 tiles, no enemies) → food trickle +12% XP (`addXp`). Drawn as a glowing menhir.
- Resource tiers sit farther from starts / toward center.

## Commands / QoL

- `.` cycle idle villagers (also click **Idle N** in the top bar).
- `,` select all military.
- Shift+right-click **queues** orders (`giveOrder` / `u.orders`, max 10).
- Patrol, Aggressive / Hold stance on military.
- Esc after placing a building has a short grace so it does not open the pause menu.
- Wheel zoom is slow (`~5%` per typical notch, delta-scaled). Pivot stays under the cursor. Range `cam.z` 0.4–2.2.

## Combat / sim notes

- Combat resolves on `COMBAT_TICK` 0.6s.
- XP curve: `XP_TABLE` step is `50 * 1.1^(L-1)` and **×2 from level 8 up**.
- `G.stats` tracks killed / lost / built / resources; shown on the end screen.
- Corpses fade (`G.corpses`). Dust puffs while moving.

## AI (`aiThink`)

Same rules as the player: economy, houses, barracks, farms, forge, towers, scouts, rams. Waves store `ai.comp` for the HUD. Difficulty from lobby sets first-wave time, wave extra, and villager target.

## Pathfinding / gates

`PASS_TEAM` is set in `updateUnit`, `setGoal`, `cmdMove`, `separate`, `spawnSpot`. Do not call `pass()` for a unit without that team, or friendlies will treat their own open gates as walls (or enemies will walk through yours).

## Frame loop

If `!UI.ingame`, only `renderBoot`. Else update + `render`. Worker (`bg=1`) calls `frame()` only while `document.hidden` and must not `schedule()` rAF in that state.

## What Grok changed (this session)

1. Finished round-3 freeze: rAF pending-flag was a no-op because Worker-driven `frame()` re-queued. Worker frames no longer schedule rAF while hidden.
2. Auto no longer steals after a haul; manual orders turn Auto off; builders resume gather; Esc grace after place; army HUD is **arrival** ETA; starting vils Auto ON; slower XP after 8; short skill floats.
3. Title / lobby / settings / pause tablet, Palatino titles, boot music, difficulty, localStorage autosave / Continue.
4. Art pass: terrain, nodes, units, buildings, HUD icons (still Canvas, no bitmaps).
5. Gates, scouts, farms, rams, rune stone, queues, patrol, stances, idle/army hotkeys, wall autotile, scout reports, wave composition, end stats, slower zoom.

## Suggested next (not done)

- Playtest farms / rams / gates / scouts in a real match (Chrome playtesters were Claude-in-Chrome; not re-run here).
- Top bar is tight at 1440px with Idle + Arms wrapping.
- Enemy fog is player-only (`G.explored` / `G.visible` are team 0).
- Shift-queue does not apply to gather/build the same way as move (move/attack-move go through `giveOrder`; some right-click jobs still assign `u.task` directly).
- No git repo in this folder.
