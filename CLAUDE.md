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

## UI design system (v2)

CSS block "Design system v2" at the end of the stylesheet overrides earlier rules; edit there. Tokens: `--font-ui` (Palatino) for titles only, `--font-body` (system sans) for text, `--font-mono` for numbers/timers (`.num` and the top-bar value spans). One button family (`#topbar/#panel/.ov/#resdrop/#tutor button`, 28px) with variants `.primary` (gold, key action), `.danger` (red), `.on` (rune cyan, active state), `.locked`. The command card `#cmds` is a 4-column grid of 56px icon tiles: `btn()` in `refreshPanel` emits `<canvas class="cico" data-ico>` + `.lbl` + `.k` hotkey badge; `iconFor(act)` maps actions to icon kinds; `drawCmdIcon` draws every building/unit/action glyph on an 18-unit grid; `iconImg` caches offscreen canvases; `paintIcons(root)` fills any `canvas.cico`. Selection cards group by type with a count badge and mean-HP bar (`data-type`; click isolates, shift-click removes). Top-bar Idle and Next army are `.res.stat` pills with `.has` / `.hot` / `.alarm` states.

## Mobile / touch

- Canvas is DPR-scaled: backing store `VW*DPR × VH*DPR`, CSS size `VW×VH`; `frame` and `renderBoot` call `ctx.setTransform(DPR,…)`. Use `VW`/`VH` for screen-space math, never `cv.width`.
- `IS_TOUCH` (coarse pointer, or `?touch=1` for desktop testing) adds `body.touch`; `resize()` also sets `body.narrow` (<900px) and `body.short` (<520px). Breakpoints: 900px (compact bar, two-row panel with a horizontally scrolling command strip), 600px (portrait: Arms/speed/pause hidden, Idle/army pills show bare numbers, Menu pinned), 520px height (landscape phones: toolbar moves above the panel).
- Touch handlers live in `bindInput` (the `TT` block): tap = select own unit, or select an own building unless selected villagers have a real job there (foundation, repair/upgrade, or drop-off while carrying), or issue the right-click command when own units are selected; the villager card has a `garrison` tile (act `garrison` → `issueCommand` at the TC centre) since tapping the TC no longer garrisons; one-finger drag pans; two fingers pinch-zoom around the midpoint; hold 450ms shows the tooltip for ~2.6s; `UI.mode==='box'` makes a drag box-select. `#touchbar` (box, multi/shift, attack-move, idle, army, home) shows only on `body.touch.play`. Placement on touch is two taps (`UI.ghostArmed`); `#mode` has a Cancel button.
- Tutorial on touch: `TUT_TOUCH[i]` replaces each step's text (no hotkeys); the card header toggles `#tutor.collapsed`, and on `body.short` screens it auto-collapses 9s after a step starts (`UI.tutCollapse`). Toasts (`#msg`) sit below the card / above the toolbar on phones.
- Edge scrolling is off on touch. `mobiletest.html` (git-ignored) embeds the game in 844×390 and 390×700 frames for testing; frame-scoped `let` state must be read via `contentWindow.eval`.

## Round-six fixes (Claude)

- Walls: with the Palisade ghost live, mouse-drag draws an L-shaped line (`wallLineTiles`, `UI.wallLine`, `placeWallLine`); every foundation without an assigned villager claims the nearest free one (`claimBuilder`); after wall/gate placement `perimeterOpen()` (enemy-side A* to the player TC) drives a sealed/open toast. Farm builders keep Auto and the first builder works the finished farm.
- Input: Esc always cancels placement/modes (also closes the dropdown), other hotkeys are ignored while a ghost is live, Patrol accepts left-click, `cycleIdle` matches the Idle count, dropdown "Send a villager" prefers idle → lowest current skill and names who moved, and explains skill-gated nodes.
- Combat/HUD: Aggressive engage radius 9 tiles (idle 7); `ai.atWalls` needs contact with a wall/gate, `ai.inside` = within 9 tiles of the TC ("Enemy army inside your base!"); both reset when nothing marches; attack banner 6s; "Your army has been lost" when a ≥5 army hits zero (`G.milPeak`). Tutorial step 11 also requires Space or Z (`G.tutKeys`).
- Balance: AI villager target from `DIFF.vil` (+2 Bronze, +4 later); home garrison capped at 1.5× wave size before Iron; Iron costs 200 stone; Tower is Bronze; start food 300/400/400; extra tier-0 rock cluster near each start.
- UI: `G.gross` tracks gathered income (deposits, farms, relic) and the dropdown shows gathered vs spent; population copy fixed; Cancel-last is a permanent (locked when empty) tile; badges Q/Y/↵/Del; panel tooltips anchor above the panel; world tooltips auto-hide after 4s idle; dropdown is opaque and hides the tutorial card; new army pill (`#armypill`, dropdown `army`: yours / training / next wave / enemy seen / rally, actions select/train/home).
- Mobile: `#tbs` wraps the pills (built at init) and scrolls under 600px with Menu fixed; touch targets 34/36/60px; `.k` badges and `.kbonly` rows hidden on touch; Skills in the pause menu; canvas touchstart closes dropdown/tooltips; selection card and minimap capped inside the panel; safe-area insets; no start toasts in the touch tutorial.

## Start areas and placement safety

- `STARTS` are (14,14) and (81,81); the guaranteed clearing is 11 tiles (water only beyond 13), the tier-0/1/2 patches sit on its rim, and a 2-wide grass corridor is carved from each start toward the map center on both axes (8–26 tiles out) so every base has at least two exits besides the road.
- `sealRatio(type,tx,ty,team)` = reachable tiles after / before a hypothetical footprint. Player: refused below 0.75 (`wouldSeal`), orange "narrows your exit" hint below 0.9 on the ghost. AI `pickSpot` skips road tiles and any spot below 0.8; house/farm/forge anchors come from `aiRing(tc,rMin,rMax)`. `aiRescue()` (every 10s) demolishes the building whose removal most restores reach when the AI's reachable area falls under 350 tiles.

## Resource cluster thinning

After terrain noise and the road, `genWorld` runs a thinning pass before nodes are created: a clearing noise (`n1`/`n3`, threshold 0.38) turns resource tiles more than 19 tiles from a start back to grass, then a 4-connected flood fill finds every forest/stone/ore component; any component over 18 tiles or wider/taller than 6 gets grass lanes every 5 tiles (random offset per component) along its long axis, both axes if over 60 tiles, never within 14 tiles of a start. Typical result: ~1750 nodes (was ~3000), largest cluster under 80 tiles (was 400–1350), walkable area up ~30%. Test harness pattern: extract `genWorld` with node and count components/reach.

## Exhaustible mines

`depleteNode(n,def,team)` runs whenever a node hits 0. `exhaustible(n)` is stone/ore with `tier<3`. `n.dep` counts depletions; `EXHAUST_P=[0,0,0.3,0.6,1]` indexed by `dep` gives the chance the vein ends for good, decided by a hash of node index, `dep` and `G.seedNum` (deterministic per seed). Exhausted nodes get `n.ex=true`, `regrow=0`, stay `amt=0` (so `pass` and `findNode` already treat them as gone), draw as 55%-alpha rubble, tooltip "Exhausted for good"; the owner sees a toast plus an "Exhausted" float when the tile is visible. Stone/Ore dropdowns show "N ran dry · M low-tier left". Saves store `[amt, regrow, dep, ex]` per node (older 2-field rows load fine). Wood, fish and tier 3–4 veins are always renewable. Low-tier stone/ore `amt` was raised ~15% to compensate.

## Phase A polish (v0.9.4)

- Combat 10 regen is real: `updateUnit` heals `0.5+0.05*cb` HP/s when not garrisoned and `G.time-u.lastHit>6` (`lastHit` is set in `attack`).
- `G.hist5` samples every 5 s for the whole match (`{t,w,s,o,f,m,e}`, capped at 1440 = 2 h) and is saved with the game; `drawEndGraph` plots it on `#end-graph`. `endGame` shows a score (kills×10 + built×5 + age×200 + avgSkill×20 + gathered÷20 + win speed bonus) and a `#end-grid` of totals from `G.gross`.
- `paintLobbyMap` overlays keeps 1/2, a Rune Stone diamond at the map centre and a legend strip.
- Audio: `setSel` plays a per-type acknowledgement for own entities; tower shots, Rune Stone captured/lost, and every `<button>` click have sounds; `MUS.battle` is a filtered 55 Hz sawtooth layer that `MUS.setBattle(on)` fades in while `G.alert` is under 10 s old or the AI is `inside`/`atWalls` (checked once per second in `update`).
- Perf: terrain lives in `R.chunks` (12-tile, 384 px canvases; `R.ctxFor(x,y)` returns a context pre-translated so `drawTile` keeps absolute coordinates); `render` draws only visible chunks. `R.sparks` precomputes water sparkle positions. `SH` uses a generation-stamped array of reusable buckets. `render` reuses `R.bldList` / `R.unitList`.
- Process rule from the user: **after every roadmap phase, run the three Claude-in-Chrome playtesters** before moving on.

## Round-seven fixes (after Phase A)

- Top bar: `#wavepill`/`#armypill`/`#r-relic` ellipsize; under 1600 px the copy is short (`Inside! ×8`, `At walls ×8`, `Arriving 0:31 ×7`, `Next 6:34`, `✦ held/enemy`, age without the word "Age", clock hidden). `ai.compN` = wave size (fallback: count of enemy attack-movers); the full composition lives in the wave pill tooltip. `widetest.html` (git-ignored) is a 1440×720 iframe harness.
- Leash: `u.leashX/Y` is a persistent home point. Idle scan sets it only when undefined or within 3 tiles; retaliation/response paths reuse it; `giveOrder`/`cmdMove` clear it; the idle scan clears it once the unit is back within 3 tiles. So chained auto-engagements can never drift more than 12 tiles from where the first one started.
- Rune Stone: scouts don't count; `R.pend/R.pendT` require the new owner to hold for 4 s before `R.owner` flips.
- `dropAuto(units,later)` sets `u.autoWas` when the drop came from placement (`tryPlace`, `placeWallLine`); `finishBuild` (no next foundation) and `buildStep` end restore `u.auto`. Manual orders reset `autoWas`.
- Regen is one curve on the existing line in `updateUnit`: `0.25+0.03·cb` after 8 s, or `0.5+0.05·cb` after 6 s at Combat 10+.
- Waves always take idle rams (`att.push` after the slice). Army pill's enemy estimate is at least the visible enemy soldier count. Exhausted rubble is a dark cross; the dropdown counts only explored veins; tooltip drops mining stats when exhausted. Unlock toasts are suppressed for 8 s after an attack and rate-limited to one per 2 s. Gather tick sound plays 1 in 3. `unitAt` radius 22 px. Music no longer stops in `enterPlay`. Panel tooltips refresh text and hide after 7 s. `placeWallLine` reports skipped tiles. End screen counts alive units; graph food is amber and the right axis has ticks.

## Phase B: skills you can see (v0.9.6)

`SKILL_COL` (per-skill colour), `BRACKET`/`bracketOf(L)` (bronze <10, steel 10–29, rune 30+, with `col`/`dark`), `bestSkill(u)` (villager: best of wc/mi/fi/co/sm; soldier: cb) and `toolFor(u)` (task → axe/pick/rod/hoe/hammer + the skill whose level colours it; idle → best skill's tool) sit next to `SKILL_NAME`. In `drawUnit` the villager case draws a cape (best skill ≥45, `SKILL_COL`), the tool with a bracket-coloured head, and a pointed hood (best skill ≥20) instead of the straw hat. Guards/rangers/knights get a bracket-coloured breastplate arc at Combat 10, shoulder pauldrons at 20, a red plume at 35 (knights already have one), and a weapon glint when `team.arms>=2`; this block sits just before the chevron code. `drawPortrait` reuses `drawUnit`, so cards match. The single-unit info shows "● Skill L · bracket tools/armour · hood/cape · next look at N"; the Skills overlay explains the milestones.

## Round-eight fixes (after Phase B, v0.9.7)

- Stuck detection: besides the per-frame `stuckT`, `updateUnit` samples net displacement every second (`u.nx/ny/netT/mvT`); a unit that tried to move for >0.6 s but netted under 0.3×speed detours and drops its path; after three such seconds a gatherer re-targets (`findNode` 22 tiles), an Auto villager clears its task, anyone else is nudged to the nearest passable tile. Fixes the head-on deadlock where separation shoving hid the stall.
- Waves: `u.waveId` tags wave members; `marching` counts attack-movers plus any tagged unit in an attack; `ai.compN` follows the live count; stragglers sent home clear the tag. Rams are trained when the AI knows of player walls/gates/towers, on odd waves, or a 15% roll.
- Auto: `issueCommand` records `wasAuto` before `dropAuto` and sets `u.autoWas` on right-click build/repair/upgrade orders; `finishBuild` restores Auto (and clears the resume task) when no next foundation is nearby. Auto villagers' tier preference dropped from 650 px to 220 px and search radius to 30/60 tiles.
- Walls: `placeWallLine` skips road tiles (toast suggests a gate); `tryPlace` for a gate on an own palisade refunds it (full if unbuilt, half if built), kills it silently (`b.silent`) and places the gate; Esc grace after a drag; gate HP 380.
- Idle count matches `cycleIdle` (includes stopped move tasks). Idle-scan soldiers with a leash more than 3 tiles away walk back to it. Rune Stone toasts have a 45 s cooldown (`R.msgT`). `#mode` is pointer-events none except its button. `,` skips scouts. C with no villagers selected explains itself. Let out is a permanent locked/unlocked TC slot.
- Gear contrast: capes are 20 px wide with a light edge, breastplates are filled ellipses with a highlight, the Arms glint is a pulsing 4-point star. Card line: villagers "bronze tools, hood"; soldiers "plume, pauldrons" etc. End graph: ore `#c9603a`, food `#ffd166`.

## Phase C: counters and a smarter enemy (v0.10.0)

- `UNIT_DEF.spear` (Spearman, Bronze, 40f/30w, key Y at a Barracks; Y otherwise keeps its old meaning). Damage modifiers in `attack`: spear ×1.6 vs knight, ×1.4 vs ram, ×0.7 vs guard; tower ×1.5 and guard ×1.2 vs ram. Sprite = guard body, long spear with leaf head, buckler; icon `case 'spear'`; included in the armour-bracket and card-line type lists. Barracks tier-1 upgrade renamed **Drill Hall** (was "Garrison", which clashed with the villager Garrison tile).
- AI army: trains spears when it can see player knights (up to 40% of its army), alternates guard/spear in Bronze (every third unit), guard/spear/ranger in Iron.
- AI villagers flee to their own TC like Auto player villagers (`u.auto||u.team===1`, TC looked up by `u.team`; scouts don't trigger it).
- AI palisade ring: at Iron with wood >220 and fewer than 3 pending builds, `ai.ring` plans a 9-tile square around the TC; road tiles become gates (or two gates toward the player if no road crosses); each think places up to 2 gates then up to 5 walls, skipping tiles with `sealRatio(...,1)<0.8`; `ai.ringDone` when the plan is exhausted.
- AI retreat: `ai.waveHp0`/`ai.compN0` recorded at launch; if the marching wave's HP falls under 40% before it is `inside`, everyone walks home (`waveId` cleared), the player gets "The enemy army is retreating!" if any of their soldiers is visible to the AI, and the next wave is pulled to ≤75 s away.

## What's New screen

`VERSION`, `CHANGELOG` (newest first: `{v,d,t,items[]}`) and `ROADMAP` (`{t,n,soon?}`) live just above `toggleSkills`. `showNews(fromGame)` renders `#news` (title button `#b-news`, pause-menu `#m-news`, hash `#news`), marks `runeforge_seen_ver`, and `refreshNewsDot` shows the cyan dot on the title button until the current version has been opened. **Bump `VERSION` and add a `CHANGELOG` entry with every user-visible change.**

## Suggested next (not done)

- Playtest farms / rams / gates / scouts in a real match (Chrome playtesters were Claude-in-Chrome; not re-run here).
- (done) Top bar: `nowrap` children, `#word` hidden under 1560px, buttons trimmed under 1400/1240px.
- (done) Enemy fog: `G.exploredE` / `G.visibleE` are the AI's own maps, filled in `updateVision`. `aiSees(x,y)` / `aiKnows(x,y)` gate threat detection, wave targets (unknown TC → march to the player's start), and straggler targets. Saved as `exploredE`.
- (done) Every right-click job in `issueCommand` goes through `giveOrder`. `updateUnit` pops the next queued order when `task` is null (before Auto re-picks). A gather with queued orders ends after one delivered load (`gatherStep` deposit) so `gather → build → gather` works.
- Playtests are run by three parallel Claude-in-Chrome subagents (first-timer / builder / rusher) in their own tabs with `?bg=1`; they cannot see each other's tabs.
