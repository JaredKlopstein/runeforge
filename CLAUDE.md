# Runeforge — agent notes

Single-file browser RTS. **All game code, art, UI, and audio live in `index.html`.** Do not add a build step, bundler, CDN, or external image/audio files. Art is procedural Canvas 2D. Sound is WebAudio.

Run: `python3 -m http.server 8765 --bind 0.0.0.0` from this directory, then open `http://127.0.0.1:8765/index.html`. Hard-refresh after edits.

This file is the handoff between the Claude session `b9b033c9-cd3d-4eb3-bc76-273aff702e9e` and a Grok session (2026-09-09). Claude built the original playable game and two playtest/fix rounds; Grok resumed mid round-three freeze fix and added presentation, art, units, and QoL; Claude then took Grok's open items (below) and continued. The folder is now a git repo — commit after each working change.

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

## Round-four fixes (Claude, after the Grok handoff)

- **Siege fallback**: an attack-move that cannot path (`flowStep`/`setGoal` fail) calls `siegeFallback(u,t)`, which A*-checks the nearest enemy buildings (walls and gates included) and attacks the first reachable one with `resume` set to the march. Throttled by `u.siegeT`.
- **Honest army ETA**: `aiThink` recomputes `ai.marchArrive` from the actual lead marching unit each think and clears it when nobody is marching.
- **Enemy scouting**: `pickScoutTile` uses the unit's own explored map; team-1 scouts always auto-explore. A wave whose target TC is unknown sends the scout toward the player's start and delays 30s, up to three times (`ai.scoutWait`).
- **AI contests the Rune Stone** from the Iron Age with three idle soldiers every two minutes.
- **Seal guard**: `wouldSeal(type,tx,ty,team)` flood-fills reachable tiles from the TC before/after a temporary footprint; `tryPlace` refuses anything but a gate that cuts reach below 60%.
- **Saves**: `SAVE_KEY` is `runeforge_save` plus `_<tester|profile>` from the query string; the meta stores `diff`, the save body stores `diff`, and the Continue button shows difficulty, seed, time and age.
- Upgrades auto-recruit up to two qualified villagers; placement takes the two nearest selected villagers; a finished builder moves to another unfinished foundation within 12 tiles before resuming.
- Stances are two lit buttons (`stance:aggro` / `stance:hold`). Box-select uses unit hitboxes. Tooltips sit under overlays (z 25) and hide while placing. Farm XP floats say "Farm". Rune Stone has a capture ring, glow, top-bar badge (`#r-relic`) and minimap diamond. How to Play is a five-step opening plus a key grid.
- Difficulty: Warlord first army at 6:00 with the first wave capped at 3 units; Lord and Warlord start with 300 food.

## Tutorial mode

Title → **Tutorial** (or `#tutorial`). `startTutorial()` starts a Squire game on seed `tutor` with the starting villagers Auto OFF, first army at 10:00, `ai.soft` (first wave 2 units, later waves one smaller), and `G.tutorial={i,shown,t}`. `TUT_STEPS` is an ordered list of `{title,text,done(),onStart?}`; `tutorTick` (called from `update`) checks `done()` every 0.3s and advances with a flash on the `#tutor` card. Skip / Exit buttons on the card. Step 11's `onStart` pulls the next wave to +40s. The step index is saved (`st.tutorial`) and restored on Continue.

## Round-five fixes (Claude)

- Tutorial: step 11 spawns a scripted 2-guard raid (`G.tutRaidT`, `u.tutRaider`) 30s after the step starts, independent of the AI wave timer; cards say when they complete; the last card is a closing card (Exit ends the tutorial). `showTitle` hides the card.
- Seal check counts own gates as passable even while unbuilt (`SEAL_CHECK` flag read by `gateWalkable`). The build ghost shows "would seal your base" in orange before the click.
- Army ETA: `ai.atWalls` when any marcher is within 4 tiles of a player building → HUD reads "Enemy army at your walls".
- Auto-engaged pursuits (idle scan, retaliation, response-to-attack) carry `leashX/leashY`; a chaser more than 12 tiles from its leash point gives up, blacklists the target for 20s and walks back.
- Scout reports use `G.scoutPing` (Space jumps there after any attack alert), not the attack banner.
- Rune Stone is placed on the grass tile nearest the center that is reachable from the player's TC with at least 5 open neighbours; the AI relic squad gets `relicGuard` + hold stance and is skipped by straggler and siege logic.
- Frame loop caps simulated time at 1.2× game speed per real second and stops simulating 90s after game over. Defeat/victory clears the autosave. New units start with `stance:'aggro'`.

## Resource dropdowns

Top-bar pills with `data-res` (wood/stone/ore/food/pop/age/arms) open `#resdrop` on click (`toggleResDrop`, content from `resDropHtml`, refreshed every second while open, closed by outside mousedown or Esc). Income comes from `G.hist`, a per-second stock sample ring (90 entries) pushed in `update`; `incomePerMin(k)` reads the 30s delta. Actions are `data-ract` buttons routed through `resDropAction` and only call existing commands (gather/build placement/upgrade/smith). Hover tooltips are suppressed on the open pill.

## Suggested next (not done)

- Playtest farms / rams / gates / scouts in a real match (Chrome playtesters were Claude-in-Chrome; not re-run here).
- (done) Top bar: `nowrap` children, `#word` hidden under 1560px, buttons trimmed under 1400/1240px.
- (done) Enemy fog: `G.exploredE` / `G.visibleE` are the AI's own maps, filled in `updateVision`. `aiSees(x,y)` / `aiKnows(x,y)` gate threat detection, wave targets (unknown TC → march to the player's start), and straggler targets. Saved as `exploredE`.
- (done) Every right-click job in `issueCommand` goes through `giveOrder`. `updateUnit` pops the next queued order when `task` is null (before Auto re-picks). A gather with queued orders ends after one delivered load (`gatherStep` deposit) so `gather → build → gather` works.
- Playtests are run by three parallel Claude-in-Chrome subagents (first-timer / builder / rusher) in their own tabs with `?bg=1`; they cannot see each other's tabs.
