# Runeforge forward plan (v0.9.3 → 1.0 and beyond)

## Context

Runeforge is at v0.9.3, live on Vercel, with six playtest rounds behind it. The core loop works: skills, ages, economy, walls/gates, an AI that scouts and sieges, tutorial, mobile, changelog. The user has limited usage right now, so this document is the deliverable: a prioritized roadmap that collects everything already discussed (gear visuals by skill, Bronze counter unit, replays, multiplayer, more maps, hero villagers) plus gaps found in a read-only audit of `index.html` and a quick look at what current RTS and skilling games do well.

Guiding conclusions from the research:
- Small RTS games win on single-player feel, spectacle and reasons to come back, not on multiplayer. Prioritize the solo match feeling great and having variety before any netcode.
- RuneScape's hook is *visible* milestones every 10 levels with cosmetic payoff. Right now skills are numbers in a panel; nothing on the map shows them.
- AoE4's most-loved additions were alternate victory conditions (sacred sites, wonders) and landmarks that make aging up a choice. Runeforge has one win condition and one path per age.

Constraints stay as in CLAUDE.md: one HTML file, procedural art/audio, `PASS_TEAM` gates, rAF never queues while hidden, bump `VERSION` + `CHANGELOG` per visible change.

## Audit findings that shape the plan (all read-only, cite `index.html`)

- Skills: Fishing has no unique payoff; `UNLOCKS.cb[10]='Faster regeneration'` is not implemented (only flat garrison healing, L713). Villager appearance never changes with skill; soldiers only get a chevron at Combat 20/35 (L983).
- Units: Knight has no counter; Ram is uncountered by the AI (AI builds at most one, 25% roll, L840). Hotkey collision: forge `G` vs guard `G` (L452/L460). Tower `desc` says Iron but is Bronze (L463). `BLD_DEF.tc` has dead cost/time fields.
- AI never builds walls/gates, never garrisons villagers (flee is `team===0` only, L717), never retreats, uses stances only for the relic squad.
- Audio: one oscillator voice type, one drone track; ~20 events. Missing unit-trained, selection ack, gate toggle, tower fire, relic captured, UI clicks.
- End screen is one line of text (`endGame` L1528); `G.hist`, `G.gross`, `G.stats` already hold enough for a graph and score.
- Settings: six options; no hotkey rebind, colorblind, UI scale, scroll speed.
- One victory condition (kill TC). One map generator, fixed 96×96, fixed starts, lobby preview does not mark starts/road/relic.
- Saves: one autosave slot plus text export; no named slots.
- Perf: `R.terr` is a 3072² offscreen canvas (~37 MB, risky on Safari/mobile, L860); `render()` allocates and sorts arrays every frame (L1065/L1076); `drawWater` hashes every water tile every frame; `SH.build()` rebuilds a Map each sim step (L582).
- Onboarding: no tech tree, no unit stat/counter card, no objectives panel outside the tutorial.

## Roadmap

Each phase is one or two sessions of work and ships as its own version. Order is by value per unit of effort, with cheap fixes first so each session ends deployed.

### Phase A — v0.9.4 "Polish and fixes" — SHIPPED 2026-09-10 (hotkey left contextual: G trains Guards when a Barracks is selected, else places Forge)
1. Data fixes: guard/forge hotkey collision (move Forge to `F`... check `farm` is `J`; pick a free letter), tower desc, remove dead `tc` cost, implement Combat 10 regen (out-of-combat `hp += dt*(0.5+lv.cb*0.05)` in `updateUnit` when `u.lastHit` older than 6 s; add `u.lastHit` in the hit path, L765).
2. Lobby preview marks both keeps, the road and the Rune Stone (`paintLobbyMap` L1312).
3. End screen: score (kills×, buildings×, age×, skills×), per-resource gathered from `G.stats`, and a small canvas graph of stock over time from `G.hist` (extend the ring to the whole match at 5 s samples, ~360 entries).
4. Audio pass: unit trained, selection ack (short two-note blip per unit type), gate toggle, tower fire, relic captured, UI click; a second music layer that fades in while `G.alert` is fresh (reuse `MUS` scheduler).
5. Perf quick wins: cache the sorted building/unit draw lists per frame only when counts change; precompute water sparkle offsets per tile; reuse `SH` bucket arrays instead of a new `Map`; chunk `R.terr` into 6×6 tile canvases (16 of 384 px) so mobile never allocates 37 MB.

### Phase B — "Skills you can see" — SHIPPED as v0.9.6 on 2026-09-10 (v0.9.5 was the round-seven playtest fixes)
1. Villager tools by task and skill bracket in `drawUnit` (L935): hatchet (wood), pick (stone/ore), rod (fish), hammer (build/smith); head color by bracket: bronze <10, steel 10–29, cyan (rune) 30+. Reuse `AGE_COL` palette.
2. Villager milestone cosmetics: cap/hood at 20 and a cape at 45 in the villager's best skill, colored per skill (`SKILL_COL` if present, else add one). Mirrors RuneScape milestone capes.
3. Soldier armor by Combat bracket layered over age color: breastplate outline at 10, pauldrons at 20 (with existing chevron), plume at 35. Arms cap adds a sword glint.
4. Selection card portraits (`drawCmdIcon`/`iconImg`) pick up the same brackets so card and sprite agree.
5. Skills overlay: add "next milestone" line per skill and a per-villager best-skill badge in the selection card.

### Phase C — "Counters and a smarter enemy" — SHIPPED as v0.10.0 on 2026-09-11 (Warlord+ tier deferred until the new AI tricks are playtested)
1. New Bronze unit: **Spearman** (barracks, age 1, 40f/30w): 1.6× vs knight/cavalry, 0.7× vs guard; slow. Gives Knights a counter and Bronze a decision. Add to `UNIT_DEF`, `drawUnit`, `drawCmdIcon`, tooltips, AI wave comp.
2. Ram counter: towers get 1.5× vs rams; guards 1.2× vs rams. AI builds rams when it has seen walls (`b._spotted` type wall/gate) and targets the nearest wall segment via `siegeFallback`.
3. AI upgrades: builds a palisade ring with one gate at Iron (`placeWallLine` exists; add a ring helper around `aiRing`), garrisons villagers on raid (drop the `team===0` guard at L717 with a per-team cooldown), retreats a wave that falls under 40% strength to its TC, uses Aggressive stance on waves and Hold on tower guards.
4. Difficulty gets a fourth tier "Warlord+" only after the above so the AI's new tricks have a home.

### Phase D — "Ways to win and reasons to replay" — SHIPPED as v0.11.0 on 2026-09-11 (map size option and UI scale deferred: `N` is a const used ~300 times)
1. Alternate victories in the lobby: **Rune Stone hold** (exclusive control for 6 continuous minutes, HUD timer on `#r-relic`), **Wonder** (Rune age building, 400w/400s/200ore, 8-minute countdown, enemy gets an alert and a marker). Default stays Conquest.
2. Landmark-style age choice: aging up offers two perks (e.g. Bronze: +10% gather XP vs +1 pop per house; Iron: cheaper walls vs faster training; Rune: relic trickle doubled vs knight cost down). Stored on the team, shown in the Age dropdown, AI picks by difficulty.
3. Map variants selectable in the lobby: **Diagonal road** (current), **River crossing** (river with two fords and a bridge tile type), **Islands** (needs fish-heavy economy; land bridge unlocked by building a Dock... scope check: maybe defer), **Central plateau** (ring of stone around the relic with four gaps). Implement as `MAP_STYLES` table of post-processing hooks in `genWorld`, reuse corridor carving and thinning passes. Add map size Small/Normal (72/96) if `N` can be made a `let` set before `genWorld`; audit `N` uses first.
4. Tech tree overlay (from `BLD_DEF`, `UPG`, `UNIT_DEF`, `UNLOCKS` tables; no hand-written content) and a unit stat card with counters in tooltips.
5. Settings: scroll speed, UI scale, colorblind team colors (swap `TEAM_COL` to an orange/blue pair), autosave toggle, hotkey rebind for the ten most-used keys (`KEYS` map read by `bindInput`).
6. Named save slots (3) with timestamps in the pause menu, plus file download/upload via Blob for desktop.

### Phase E — "Heroes and campaign feel" — SHIPPED as v0.12.0 on 2026-09-11
1. Hero villagers: at match start each team gets one named villager with a signature skill (+25% XP in it), a portrait, and a milestone cape. Hero death is a toast, not a loss. Names from a seeded list.
2. Objectives panel for skirmish (not just tutorial): three rotating short goals (reach Mining 10, hold the relic 60 s, kill 5 units) that grant a small XP bonus; keeps solo matches feeling directed.
3. Replay recording: every player command goes through one `issueCommand`/`giveOrder` entry point already; log `{t, cmd}` to `G.replay`, save with the game, and add a "Watch replay" mode that replays commands with input disabled. This requires the sim to be deterministic: audit `Math.random` uses in sim code (particles are fine, combat/AI must use `G.rng`). This is the groundwork for multiplayer and the biggest engineering item; do after D so the command surface has settled.

### Phase F — later "Two-player"
- Lockstep over WebRTC data channel with a tiny signaling relay (Vercel serverless + a room code). Depends on E.3 determinism. Not before 1.0.

## Version 1.0 definition
Phases A through D shipped, three playtest rounds clean, mobile verified on a real phone, README screenshots. Then tag `v1.0.0`.

## Verification pattern per phase
- After every phase, run the three Claude-in-Chrome playtesters (first-timer / builder / rusher) and fix what they find before starting the next phase (user rule).
- `node --check` on the extracted script after every patch.
- Headless world-gen harness (`/tmp/wt.js` pattern) for map changes: component sizes, reach between starts, relic reachability.
- Chrome on `http://jared-mini:8765/index.html?v=<stamp>&tester=<name>#play`; scripted `update(1/30)` loops for sim checks; screenshots for art passes; `mobiletest.html` for phone layouts.
- Three parallel Claude-in-Chrome playtesters after C and D.
- Bump `VERSION`, add `CHANGELOG`, update `CLAUDE.md`, commit, push, confirm `VERSION` on the live site.

## Sources consulted
- Stormgate discussion on indie RTS lessons: https://steamcommunity.com/app/2012510/discussions/0/597410286607101804/
- Indie retention notes: https://www.guardingpearsoftware.com/blog/lessons-from-successful-indie-game-developers-33092
- RuneScape milestone capes: https://runescape.wiki/w/Milestone_capes
- AoE4 landmarks and victory conditions: https://ageofempires.fandom.com/wiki/Landmark , https://realsport101.com/article/age-of-empires-4-skirmish-mode-how-to-win-wonder-landmarks-sacred-sites-win-conditions
