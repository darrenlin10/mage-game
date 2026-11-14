# Implementation Plan — Arc 1《云溪》

> Status: **v0.4 draft**. This document says *how* the game is built and in what order.
> *What* is built is defined by [DESIGN.md](DESIGN.md), [STORY_ARC1.md](STORY_ARC1.md), [ELEMENTS.md](ELEMENTS.md) and [CHARACTERS.md](CHARACTERS.md). Where they disagree with this file, they win and this file gets fixed.
> Game terms keep their Chinese names; the English glossary is in DESIGN.md §12.

---

## 1. Ground rules

1. **Every milestone ends playable.** Each one adds a slice you can run and judge, not a layer of plumbing.
2. **Data over code.** Arts (灵能技), beasts, items, events, NPCs, talents and seeds are data files. Adding the 40th event must not require touching engine code.
3. **Text-authored project.** Scenes, resources and scripts are written as text and versioned. Nothing depends on hand-placing things in the editor.
4. **Tested logic.** Rules that can be checked without rendering (damage, rank gap, reactions, chain judging, worldline, saves) have headless tests that run on every change.
5. **No runtime AI.** The game runs offline. All text ships as data.
6. **Chinese first.** Player-facing text is written in Chinese under a translation key; English is added to the same key.
7. **Engine is provisional until Milestone 1.** Milestone 1 is the look test (DESIGN §14.2). If it fails, the engine changes and Milestones 0–1 are redone; everything from §4 onward in this document is written to survive that.

## 2. Prerequisites and who does what

| Need | Why | Who |
|---|---|---|
| Allow the Godot download host in the cloud environment | I cannot fetch the engine today; without it I write code blind | You approve when I request it at Milestone 0 |
| Godot 4 installed on your computer | Judging visuals and game feel needs a real GPU and a person | You |
| Animation and base-body asset packs in anime style, placed in the repo | Movement, dodge, hit and casting animations come from packs | You choose and download, before Milestone 4; I can shortlist candidates and check licences |
| An API key for a 3D generation service, entered through the secure credential form | Faces, hair, outfits and beasts are generated per the briefs in CHARACTERS.md and the bestiary | You, before Milestone 4; I will compare services first |
| Answers to the open questions in DESIGN §16 | Several block specific milestones; each milestone below names what it needs | You |

What I do: all code, data, text in both languages, procedural geometry, shaders, effects, UI, tests. What I cannot do: judge motion and feel in real time, or produce production-quality character models and animation.

## 3. Technology

| Area | Choice |
|---|---|
| Engine | Godot 4, latest stable at Milestone 0; exact version pinned in `project.godot` and recorded here |
| Language | GDScript, statically typed throughout |
| Renderer | Forward+; Compatibility kept working as a low-spec fallback |
| Look | Toon shading, anime style: outlines, banded lighting, flat colour ramps, strong silhouette-first effects |
| Gameplay data | Typed `Resource` classes saved as `.tres` |
| Narrative data | JSON for events, dialogue and quests (bulk-authored, easy to diff and validate) |
| Text | CSV translation tables, one column per language |
| Saves | JSON in `user://saves/`, versioned with migrations. Per slot: one **chapter checkpoint** (written at chapter start, the only point a player can return to) and one **progress file** (written continuously, resume-only). No manual saves |
| Tests | A small in-repo headless test runner (no external add-on needed) |
| CI | A workflow that runs the headless tests and data validators on every push |

## 4. Repository layout

```
project.godot
core/            boot, game state machine, autoloads, save system, settings
systems/
  elements/      elements, statuses, reactions
  arts/          art execution, cooldowns, targeting
  chain/         灵链 patterns, input capture, judging, mastery
  cultivation/   吞吐, 修为, 淬炼, 冲关, 觉醒
  combat/        damage, rank gap, hit reactions, death and downed states
  ai/            beast and companion state machines, steering
  squad/         party, commands, loot sharing
  affinity/      好感度, gifting, 情缘
  events/        event engine, worldline variables, encounter scheduler
  quests/        quest and chapter progression, calendar
  economy/       inventory, currency, 灵晶, shops, crafting
  professions/   the six 副职业
  siege/         潮位, district defence scores, siege director
  world/         terrain streaming, regions, weather, day/night, outposts, rail
  vfx/           layered effect composer, element palettes, seed overlays
data/
  elements/ arts/ seeds/ talents/ beasts/ items/ npcs/   (.tres)
  events/ dialogue/ quests/ chapters/                     (.json)
content/         chapter scripts, set pieces, cutscene timelines
world/           region definitions, landmark placement, Yunxi layout
ui/              HUD, menus, 灵链盘, 吞吐 screen, dialogue, journal
locale/          translation CSVs, fonts
tests/           headless tests and data validators
tools/           build, export and validation scripts
docs/
```

## 5. Architecture

### 5.1 Autoloads

| Autoload | Responsibility |
|---|---|
| `Game` | Top-level state machine: boot, menu, playing, dialogue, cultivation, paused |
| `Bus` | Typed signals between systems; systems never reference each other directly |
| `Worldline` | Every flag and variable that records a choice or its consequence; the only writer of that state |
| `Calendar` | Chapter, in-chapter phase, time of day, weather |
| `Saves` | Serialise and restore all registered systems; chapter checkpoint and continuous progress file; migrations |
| `Rng` | Seeded streams per purpose (talent roll, awakening, events) stored in the save, so outcomes are fixed once rolled |
| `Loc` | Thin wrapper over translations: language switch at runtime, missing-key reporting |
| `Settings` | Controls, display, audio, accessibility |

### 5.2 Rules that hold everywhere

- A system exposes `save_state()` and `load_state()` and registers with `Saves`. Nothing else touches disk.
- Anything a choice can change lives in `Worldline`. UI and scenes read it; only event and quest results write it.
- No player-facing string literal in code or scenes. A validator fails the build on one.
- Balance numbers live in one data file (`data/balance.tres`), never inline.
- NPCs use the same art, chain, cultivation and rank rules as the player. "Ordinary people need several per beast" must fall out of their stats, not out of a special case.

### 5.3 Key data definitions

| Definition | Main fields |
|---|---|
| `ElementDef` | id, tier (basic/advanced), social grade, palette, noise and shape set, cultivation rhythm |
| `ArtDef` | id, element, stage (rank + sub-rank), role, cost, cooldown, targeting, effect script, chain pattern, vfx recipe, casting archetype |
| `ChainPattern` | ordered node positions on the unit disc, time budget per node |
| `SeedDef` | id, element, tier, power bonus, coverage, vfx overlay, per-role behaviour modifiers |
| `TalentDef` | id, rarity, modifiers, hooks |
| `BeastDef` | id, grade, stats, behaviours, weaknesses, drops, soul type |
| `NpcDef` | id, element(s), rank, talent, schedule, affinity profile (rate, likes, dislikes, gates, cap), combat kit |
| `ItemDef` | id, kind, effects, gift tags, soul slots |
| `EventDef` (JSON) | id, class (fixed/conditional/random/encounter), conditions, situation text, options (each with requirements, results, worldline writes), cooldown and exclusivity |
| `QuestDef` (JSON) | id, chapter, steps, completion conditions, results |

### 5.3a Character and beast asset pipeline

- **One humanoid skeleton** for every human character. All animation packs are retargeted onto it once.
- **Modular characters**: shared base body; per-character head, hair, outfit and signature item meshes generated or adapted from the briefs, skinned to the shared skeleton.
- **Shared animation set**: locomotion, dodge, hit reactions, downed and rescue, and about ten casting archetypes. Arts pick an archetype; effects tell them apart.
- **Beasts**: meshes generated from the bestiary. Where no fitting animation exists, motion is procedural (gait from foot placement, lunges and charges from code-driven curves).
- **Toon shading unifies the sources**: one character shader and one outline pass, so pack assets and generated assets read as one style.
- **Presentation of death is restrained**: no blood, no lingering shots, no bodies left in view. Characters fall and the camera moves on; most deaths are reported afterwards. Effects and cameras are built to this rule.

### 5.4 Three systems specified up front

These carry the most risk, so their contracts are fixed before any content depends on them.

**灵链 (chain linking)**

- Input: hold an art key; the 灵链盘 appears at the reticle; pointer or right-stick movement is captured for linking while camera rotation is suspended.
- A link succeeds when the pointer passes within a tolerance radius of each node in order, each within its time budget. Wrong node, timeout or a hit that breaks poise fails the link.
- On failure: cost is still paid, a short stagger plays, the cooldown does not start.
- Mastery per art (0–5): each level widens tolerance, extends time and auto-completes one leading node, never below two manual nodes.
- An art whose stage is below the caster's current sub-rank in that element is cast with a single press.
- The same judging module is used for learning an art, breaking through (冲关), inscribing and soul-taking.
- Tests: recorded input traces replayed against patterns at each mastery level.

**Layered effects**

- A `VfxRecipe` lists layers (charge, body, trail, impact, ground, light/camera, sound) with parameters.
- `ElementDef` supplies palette and textures to every layer.
- A `SeedDef` overlay may recolour, add a layer, or rescale parameters; it is applied at compose time to every art of that element.
- Layers are pooled. A budget per effect and per frame is enforced; exceeding it degrades particle counts, never drops the gameplay effect.

**Events and worldline**

- The scheduler evaluates eligible events when the player enters a region, changes time of day, finishes a quest step, or on an exploration tick.
- Fixed events fire from quest steps. Conditional events fire when their conditions first become true. Random events are drawn from the eligible pool with weights modified by worldline state.
- Once an event instance is created its content and any rolls are stored in the save. Reloading reproduces it.
- Every option lists its requirements; unavailable options are hidden unless marked "show locked".
- A validator checks every event: all keys translated, all referenced flags declared, no option without a result, no unreachable event.

## 6. Testing and acceptance

- **Headless tests** per system; each milestone lists what must be covered.
- **Data validators**: schema, cross-references, translation coverage in both languages, chain patterns solvable within their time budget.
- **Golden-path script**: from Milestone 5 on, a scripted input run plays the current content end to end without rendering and asserts the expected worldline state.
- **Captures**: for each milestone I produce still frames and frame sequences of the new content for my own checks.
- **Your playtest**: every milestone ends with a short checklist for you to run on your machine. A milestone is done when the tests pass *and* you accept the checklist.

## 6a. Content budget (30-hour target)

Arc 1 targets about 30 hours: roughly 15 of main story and 15 of exploration, cultivation, professions and character lines. Resources are capped per chapter, so chapter length is bounded by content, not by farming.

| Chapter | Main story | Free play | Random events available | Notes |
|---|---|---|---|---|
| 序章 | 1.5 h | 0.5 h | 3 | No combat |
| 一 启灵 | 1.5 h | 1 h | 4 | First training, first contracts |
| 二 修行 | 3 h | 5 h | 14 | The largest chapter; two regions |
| 三 暗流 | 2.5 h | 2.5 h | 8 | First siege |
| 四 溯源 | 2.5 h | 2.5 h | 8 | Deep zones with a lead |
| 五 高考 | 1.5 h | 0.5 h | 3 | Exam events |
| 六 暑假 | 1 h | 3 h | 6 | Catalyst, omens, farewells |
| 终章与尾声 | 1.5 h | — | — | Real time |
| **Total** | **15 h** | **15 h** | **46** | |

Other pools: about 12 fortuitous encounters, 23 spirit seeds (few obtainable in one run), 9 companion personal lines, 5 romance lines, 4 nest lords, 6 professions with three ranks each.

Per-chapter resource caps (crystals, deposits, nests, market stock, allocations) live in `data/chapters/` so the progression curve is tuned in one place.

## 7. Milestones

Each milestone lists: **Goal**, **Build**, **Accept** (what must be true to call it done), and **Needs** (decisions or inputs required first).

### M0 — Project skeleton

**Goal**: a project that starts, switches language, saves, and runs tests.

**Build**
- Godot project with the layout in §4; version pinned.
- Autoloads from §5.1 with real `Saves`, `Loc`, `Settings`, `Rng`; the rest as working stubs.
- Main menu: new game, continue, settings, language. Chinese font bundled.
- Save slots with version field and one migration test. Each slot holds a chapter checkpoint and a progress file; the menu offers "continue" and "restart chapter", never "load earlier".
- Test runner, validators, CI workflow.

**Accept**
- Starts to the menu; language toggles at runtime without restart; a save round-trips.
- Tests and validators run headless and pass; CI is green.

**Needs**: the Godot download host allowed in the environment.

### M1 — Look and combat sample (engine gate)

**Goal**: decide whether Godot is good enough, and whether 灵链 is fun.

**Build**
- A 300 m patch of 竹海 with terrain, bamboo, lighting, day/night.
- Third-person controller: move, sprint, jump, dodge, aim, camera.
- 灵链 as specified in §5.4, with the 灵链盘.
- Fire arts 赤星, 燃犀, 烬华 with full layered effects; 地火 seed overlay on all three; names shown with the seed prefix.
- One beast (灰牙犬) with a simple hunt-and-pounce behaviour; burning and grass ignition.
- A training dummy and a mastery readout to feel the difficulty curve from mastery 0 to 5.
- A placeholder caster model with casting archetype animations.
- Toon shader for characters, terrain and props (banded lighting, outlines), and toon-style effect materials. This is the look being judged.

**Accept**
- You can link and cast all three arts, see mastery change the link, and kill a 灰牙犬.
- Stills and sequences of each art, with and without the seed.
- **Your verdict on two questions**: does it look good enough, and does linking feel good under pressure. A "no" on the first triggers the engine change; a "no" on the second triggers a redesign of §5.4 before anything else is built.

**Needs**: nothing further. Art direction is decided: toon shading, anime style.

### M2 — World foundation

**Goal**: the whole map exists and can be walked.

**Build**
- Chunked heightmap terrain for the 8 km map from a fixed seed; region masks for 云溪, 农田, 竹海, 镜沼, 烬岭外山, 烬岭深处, 沉城, 烛渊, sea.
- Threaded chunk generation, collision, level of detail, vegetation and prop scattering by region.
- Day/night, four weather states, water surface for marsh, river and sea.
- 云溪 blockout: four districts, four ward towers, walls and gates, the station, home, the academy.
- Map screen, compass, fast-travel hooks (disabled).
- Depth boundary markers and the "you should not be here" warning.
- Full save and load of position, time and weather.

**Accept**
- Walk from home to each region edge without a hitch above the frame budget on the target machine.
- Chunk streaming tests; save round-trip anywhere on the map.

### M3 — Aether core

**Goal**: everything a lone aetherist does, for all eight basic elements.

**Build**
- Element framework, statuses (wet, burning, frozen, slowed and so on), the reaction table in DESIGN §3.5, environment reactions.
- The three 见习 arts for 火, 霜, 雷, 风, 岩, 水, 木, 金, each with chain pattern and effect recipe, including their exploration uses (ELEMENTS §4).
- Per-element advancement tracks, 修为, sub-rank breakthrough (冲关), 淬炼, mastery.
- 吞吐 minigame and screen; locations with efficiency and interruption; auto-breathing of the previous sub-rank.
- 灵晶 as items by element and grade; consumption by cultivation.
- Art loadout bar; one-press casting for surpassed stages.
- Debug console to grant elements, ranks and crystals.

**Accept**
- For each element: learn, link, master and use all three arts; trigger each reaction it takes part in; use its middle art on at least two world interactions.
- Tests: reaction matrix, cost and cooldown, breakthrough thresholds, auto-breathing rates.

**Needs**: confirmation of chain difficulty options (open question 10).

### M4 — Beasts and squad

**Goal**: real fights, with allies who can die.

**Build**
- Six 游荡级 species from DESIGN §4.2 with their signature behaviours and counters.
- Rank-gap rule; poise and stagger; weak-point hits.
- Companion AI using the same art and chain rules; four squad commands; reaction-aware targeting.
- Downed state, rescue window, permanent death, recovery in town; loot split by head count.
- Player downed outside the finale: in shallow zones, rescue by a passing hunting party with permanent cultivation loss, a gear item taken or a favour owed (which schedules a follow-up event); in deep zones, death and restart from the chapter checkpoint.
- Spawning by region and 潮位 (static values for now).
- Character and beast models and animation sets integrated.

**Accept**
- Each species can be beaten by its intended counter and punishes ignoring it.
- A scripted test shows three baseline NPCs needed for one same-grade beast, and a well-built player soloing it.
- A companion can be downed, rescued, or lost for good, and the save remembers.

**Needs**: the chosen asset packs in the repo and the generation service key (§2); companion strength target (open question 14).

### M5 — Narrative foundation and the prologue

**Goal**: the game can tell a story. 序章 is playable.

**Build**
- Dialogue system with choices, conditions, speaker portraits, and gendered address forms.
- Event engine and scheduler (§5.4), `Worldline`, 两界札记, journal.
- Quest and chapter system; `Calendar` with chapter jumps.
- 好感度 with per-NPC profiles; tier display only; gifting.
- NPC schedules in 云溪; shops as stubs.
- Character creation (gender, name, appearance).
- Prologue content: classes, home, the four districts, the tower-at-night event, the eve of 启灵.
- Golden-path script begins here.

**Accept**
- Play the prologue start to finish in both languages.
- Each prologue choice leaves the documented worldline state.
- Validators: no untranslated key, no undeclared flag.

**Needs**: nothing further. The protagonist has a fixed temperament in narration and player-chosen attitude in dialogue.

### M6 — Chapter 1: 启灵

**Goal**: the ceremony and its first decision.

**Build**
- Random awakening with stored rolls; the full talent table including 噬魂; talent effects wired into M3 and M4 systems.
- The 启灵 ceremony sequence, 纪昭's public talent, the report-or-hide decision with all four branches, including the 噬魂 variant.
- 暴露度 and its sources.
- Year-end theory exam driven by prologue study; class placement.
- First cultivation at home, first field training with 方蔚, guild student licence, the 纪昭 attack event.
- The five advanced elements' 见习 arts for 异核 (光, 影, 契, 空, 音).

**Accept**
- A new game can roll every element and talent; each talent demonstrably changes play.
- All four decision branches reach the start of Chapter 2 with distinct resources and flags.
- 纪昭 can end the chapter saved, wounded or dead.

**Needs**: talent odds and reroll policy (open question 15).

### M7 — Economy and professions

**Goal**: ways to earn, craft and spend.

**Build**
- Guild board with generated contracts; shops; black market.
- 灵晶 sources: allocation, market stock, refining 兽核, 探脉 with 灵脉 and deposits.
- Six professions with three ranks each. 铸器师 gated on 金, with secondary-element inputs; 铭文师 inscribing gear and other objects; soul infusion for 噬魂.
- Gear with slots, affixes, 器魂.
- Outposts: repair, upgrade, garrison, fast travel by rail.
- 护道 contracts: train-top defence set piece.
- Hiring a 正阶 to lead a deep expedition.

**Accept**
- Each profession can be taken from 学徒 to 匠师 and produces something used elsewhere.
- A non-金 character can still obtain gear by commissioning it.
- Economy simulation test: resources available per chapter on each route stay within the planned bands.

**Needs**: report-versus-hide gap (open question 17). Resources are capped per chapter (§6a).

### M8 — Chapter 2: 修行

**Goal**: the open game. A full school year.

**Build**
- 竹海 and 镜沼 complete: landmarks, nests, ruins, 灵脉.
- Main line: tower readings, first outpost, the abandoned hideout.
- Random event pool (at least 25) and encounter pool; seeds and single-art variants; seed tiers.
- Nest lords 断尾 and 沉灯.
- Recruitable companions and their personal events; 沈知微, 谢停云, 闻筝 introduced; romance tracks up to "托付".
- Family line: the father's tag.
- Report-line night attack; hide-line "the fight someone saw".
- The two routes (by-the-book and off-campus) both viable.

**Accept**
- Chapter 2 playable to its end on both the report and hide lines.
- A run that only attends school and a run that only goes outside both reach Chapter 3, with measurably different strength and knowledge.

**Needs**: romance exclusivity rules (open question 12).

### M9 — Sieges and Chapter 3: 暗流

**Goal**: the city can be attacked, and defended.

**Build**
- 潮位 per region, driven by story and player actions.
- Siege director: preparation phase, simultaneous district assaults, off-screen resolution from defence scores and NPC strength.
- District damage, closures and rebuilding.
- Faction reputation and reinforcements.
- 烬岭外山; mole investigation with its three clue routes; 杜若 and 渡者; the father's truth.
- 初潮, the archive-room branch, 韩维's three outcomes.
- 头领级 beasts; NPC-only 正阶 arts.

**Accept**
- 初潮 can be won in the district the player holds; other districts resolve consistently with their preparation.
- Each 韩维 outcome sets the flags the finale reads.

**Needs**: rank-gap numbers (open question 13).

### M10 — Chapter 4: 溯源

**Goal**: the deep wilds and the truth.

**Build**
- 烬岭深处, 沉城 and 烛渊外缘, enterable only with a 正阶 lead; stealth around 头领级.
- 沉城 expedition with 白鹭; the control-room records.
- 烛渊 scouting with 沈砚; 衔烛 as a set piece; the 地火 seed in the old nest with its three approaches.
- 渡者's confession; the "who gets the truth" decision and its preparation effects.
- Nest lords 铁脊 and 末班车.
- 闻筝, 禾生 and 杜若 recruitment conditions.

**Accept**
- Entering the deep zones without a lead is survivable only by retreating.
- Each recipient of the truth produces its documented 疏散准备度 and faction changes.

### M11 — Chapters 5 and 6: 高考 and 暑假

**Goal**: the exam, and the last summer.

**Build**
- 文考 generated from the player's own codex; 武考 in 烬岭外山 with scoring, team rules and its three events; results, admission tiers, classmates' destinations.
- Summer: the catalyst for 圆满 by wild and by city routes; omens visible only in the wilds; the five responses to them and their preparation effects.
- Early-train window for sending family ahead; the on-foot departure sequence, with the notified and unnotified variants and an "absent" ending.
- Summer personal events, confession events, last gifts.
- The ticket errand that ends the chapter.

**Accept**
- A player who never leaves the city reaches the finale with no warning given.
- Leaving on foot is playable and can fail; both variants apply the documented affinity changes.

**Needs**: 噬魂 consequences (open question 16).

### M12 — Finale and epilogue: 大潮

**Goal**: the night the city falls, and the reckoning.

**Build**
- Five-phase night director with rising danger and route closures. The finale is its own chapter, so its checkpoint is the start of the night.
- Home-to-station structure; the three approaches to family.
- Hotspots by district; rail repair defence; train departures gated on 护道人; the last-train defence.
- "Hold, don't kill" encounters against 头领级.
- 顾松年's fixed sacrifice; 渡者's request; the assassin and the relic; 无岸's appearance.
- Player death restarts the night from its checkpoint.
- Settlement: named survivors, evacuee count with its breakdown, carry-over record written to the save.
- Epilogue on the train.

**Accept**
- The night completes from every combination of the major worldline branches (automated sweep).
- Outcomes span the intended range: a neglectful run and a thorough run differ visibly in survivors and count.
- The carry-over record contains everything listed in STORY §13.4.

**Needs**: nothing further. Deaths are presented with restraint (§5.3a); only 顾松年's death is fixed.

### M13 — Polish and balance

**Goal**: shippable Arc 1.

**Build**
- Balance pass against the targets set by open questions 13–18.
- Performance pass to the frame target; low-spec renderer check.
- Gamepad, rebinding, accessibility options.
- English review of all text; consistency check against the glossary.
- Audio pass; music if sourced.
- Full playthroughs on at least four distinct routes.

**Accept**
- No blocker bugs on the four routes; frame target held in sieges.
- Both languages complete.

## 8. Risks

| Risk | Effect | Mitigation |
|---|---|---|
| 灵链 is frustrating in real fights | The core mechanic fails | Tested in M1 before anything depends on it; assist options; mastery curve tunable in data |
| Godot's look is not good enough | Engine change | Decided at M1, when only two milestones are at stake |
| No source for character models and animation | Fights look bad however good the effects are | Decision required before M4; placeholder rig until then |
| Content volume (events, dialogue, two languages) | Schedule | Everything is data; validators catch gaps; pools sized to the agreed game length |
| Branch explosion in the finale | Untestable outcomes | Outcomes computed from state, not hand-authored; automated sweep in M12 |
| I cannot judge feel | Tuning stalls | A playtest checklist every milestone; balance in data so you can adjust without me |
| Unlimited in-chapter time breaks the economy | Trivial difficulty | Resources are capped per chapter; economy simulation test in M7 |

## 9. If the engine changes after M1

Kept: all four design documents, the data schemas in §5.3, event, dialogue and quest JSON, translation tables, the balance file, the milestone plan from M2 on.

Redone: M0 and M1 code, the effect layers, the terrain generator.

New requirement: a computer with the engine installed that this session can reach, or you run every build and test yourself.
