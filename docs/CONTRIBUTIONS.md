# Who built what

Mirrorbound was built by five people between **18 and 21 September 2026** — 134 commits across all branches, 102 of them on `main`.

Commit hashes are given throughout so any of this can be checked against the history.

---

## The team

| | Commits | Owned |
|---|---|---|
| **Ash Vyagni** (`ashvyagni`) | 16 | Server foundation, deployment, repo |
| **Ujesha Sivaramakrishnan** | 10 | The probabilistic player model |
| **Logesh** (`medriid`) | 58 | Art and the art pipeline |
| **Siddharth D** | 18 | Client/server integration, campaign, UI |
| **Ojas Kulkarni** | 32 | Twin decision-making, the Mirror boss, combat geometry |

---

## Ash Vyagni — the foundation

Scaffolded the project and wrote the layer everything else stands on, mostly on day one.

- Deterministic game core and project scaffold
- Game state and entity definitions
- Movement and collision systems
- Combat foundation — weapons and abilities
- The enemy system: archetypes and basic AI
- The dungeon room system
- **The twin entity and the agent integration boundary** — the contract that made the AI work possible without the AI being able to cheat
- The game loop and WebSocket server
- First client/server snapshot integration
- Dark-fantasy atmosphere and visual polish
- Production deployment (Render/Vercel), Dockerfile
- `AGENTS.md` and the build brief

The agent boundary deserves particular credit. Getting that interface right on day one — AI proposes, game disposes — is why the Twin never became a cheating NPC.

---

## Ujesha Sivaramakrishnan — the player model

Wrote the statistical machinery the whole premise depends on.

- **Probabilistic player model**: traits plus n-gram / Markov prediction
- **Spatial heatmaps** — combat, retreat, dodge, spell, melee, high-risk and death zones
- **Pattern detection** — `DETECTED` / `LOST` events layered on top of prediction
- Recursive **event immutability**, real Markov pruning, the telemetry→trait pipeline
- Decay calibration, back-off thresholds, and a **determinism proof**

Later, on their own fork:

- Fed three traits that had never moved — `preferred_range`, `risk_tolerance`, `defensive_tendency` — which had been sitting at 0.50 with zero confidence for entire runs (`525b5ca`)
- Added `healthFraction` to `PLAYER_ATTACKED`, which the contract documented but the game had never actually sent
- Fixed a **sign error in the Twin's retreat learning** (`545e220`): the code fed raw health into `risk_tolerance` where the comment said cautious, so retreating at high health read as *reckless*. Every other signal in the file uses 1.0 for reckless; only this one was inverted, and it inverted the Twin's whole retreat behaviour downstream.

The immutability work is the unglamorous piece that made the rest trustworthy.

---

## Logesh (`medriid`) — the art and its pipeline

The largest commit count on the project, and the reason it does not look like a three-day build.

- **90+ hand-drawn sprite sheets**, sliced into 156 atlases
- The Python atlas build pipeline, generating TypeScript frame metadata so the renderer never hardcodes a sheet size
- Enemy families — idle, walk, alert and attack sheets per archetype
- The **canvas HUD**, drawn in-engine rather than the DOM
- Vitals: level, health and mana, including the Twin's own
- **Floor composition** — composing tiles instead of stamping one texture
- Weapon art in every hand that swings one, and the spell effects
- The frost staff's beam, drawn at the length it actually reaches
- Village buildings, villagers, props, doors, ability icons
- Campaign area walking, per-save twin slots
- An engineering log tracking the art work

---

## Siddharth D — integration at scale

Took a pile of separate systems and made them one game.

- **Server-driven world renderer**, entity views, HUD and menus
- **Rewrote the session loop** with contracts, room flow and replay
- The **campaign** — villages and dungeons, vendors, checkpoints
- Potions, Heal and Shield, damage mitigation
- The Warden's gate, acolyte and swarm rooms, with tests
- Respec, per-room enemy art, and a tutorial fight that actually teaches
- Character, map, dialogue, naming and rebinding screens
- **One keybinding table** shared by Phaser and React, so rebinds cannot disagree
- Integrated Logesh's 94 sheets, the Mirror sheets and the corrupted arsenal
- **Ported Ujesha's pattern detection onto main's server**
- Checkpoints that load, and an opening that is not a dead end
- Contracts export, replay tooling, documentation

---

## Ojas Kulkarni — the Twin, the Mirror, and combat correctness

### The Twin's decision-making

The core of the adaptive-companion premise.

- **The utility AI evaluator** (`2adc8f4`) — the scoring system the Twin decides with. Fifteen candidate intents scored against situation and learned style, rather than a behaviour tree. This is what lets style shift behaviour smoothly instead of requiring a new branch per personality.
- **Closed the `AgentObservation` ↔ `PlayerModelPipeline` loop** (`c9efdc5`) — wired what the game observes into what the model learns and back into what the Twin decides. Before this the pieces existed but did not feed each other.
- **Throttled decision-making** to a few times a second (`4c42cd7`) — deciding every frame produced a companion that twitched between plans and never committed to any.
- **Decision momentum** from `seconds_since_decision`, and a real `combo_dependency` trait (`ae78911`) — hysteresis so the Twin sticks with a plan.
- **Wired `preferred_range` and `spell_preference` into `decide()`** (`3ce1aa7`) — the traits existed but were not actually reaching the decision.
- **Situational focus-fire** (`f144de7`) — the Twin joins the fights worth joining instead of always starting its own, and stops crowding the enemy you are already on.
- **Autonomous weapon-switching** (`bf2a942`) and **building its own arsenal from the run** (`0cb6ca4`) — it picks up weapons it passes and chooses between them.

### The Mirror boss

- **Long memory** (`865831f`) — gave the boss its own horizon rather than the Twin's short one. The Twin should feel responsive to how you are playing now; the boss should feel like it has been watching the whole run.
- **Probing** (`d5ba78e`) — when the model's confidence is too low for a real read, the boss tests you at specific ranges to generate the samples it lacks, then commits to an attack type. An enemy that gathers information instead of guessing.
- **Tests for four untested boss counters** (`976b00b`)
- Corrected the disposition naming from Foil to Mirror (`711faf3`)

### Model correctness

- **Trait contamination** (`25e2e5d`) — the player model was scoring *the Twin's* actions as the player's. The Twin's frost staff was being counted as the player's ranged preference, so the system was partly learning from itself. A feedback loop that would have quietly corrupted every trait.
- **Consolidated adaptation proof** (`930165a`) — a test suite that runs scripted player profiles end-to-end and asserts the model lands where it should, rather than unit-testing the arithmetic.

### Content and client

- **The eleven designed creatures, fielded by biome** (`306c459`) — so each area has its own roster rather than recolours.
- **Rendering enemies from Logesh's drawn sheets** (`4861349`), and letting the art HUD be the HUD (`cd52b86`).
- **HUD fixes** (`09a7f13`, `2548ff3`) — map layout, twin scale, Space and the ability keys, minimap legibility, completing the panel border and putting the room into the minimap.

### Session and world integrity

- **A reload rejoins its run** (`e6b7109`) instead of starting a second simulation of the same world.
- **A dead enemy stops being your target**, and runs stop overwriting each other's replay files (`6401bf1`).

### Combat geometry

The part most likely to be quietly wrong, and it was.

- **Split the damage radius from the physics radius** (`9be363d` / `0f91c05`). `size` and `hit_radius` had been one number, authored against round procedural blobs. Once the art became hand-drawn — a Husk Scarab 96 units across, a Hollow Archer 28 — one number could not be both. Measured before the split: a straight shot registered on **27%** of a scarab's visible width, **35%** of a hound's, **48%** of the Mirror's. Widening `size` instead would have made scarabs shove each other apart from twice the distance.
- **A swing hits a body, not a point** (`a5986e6` / `7f2587f`). The arc test measured distance to the target's near edge and then measured angle to its *centre*. The tell: with bare hands every archetype cut off at exactly 48° regardless of size. Bodies now contribute their angular width:

  | | scarab | brute | hound | skeleton | archer |
  |---|---|---|---|---|---|
  | before | 48° | 48° | 48° | 48° | 48° |
  | after | **101°** | **100°** | **98°** | **68°** | **64°** |

- **Extended the split to the four enemies added later** (`109517e`), measured the same way and checked against the nine already in place.
- **Village buildings block what they draw** (`9b87fcf` / `300109f`) — all of them had shared one hardcoded radius, so you could walk into a house and bounce off thin air beside a flagpole. Also made a prop's circle scale with the prop, which the dungeon generator had always done and the village never had.
- **A focused text field owns the keyboard** (`f80af09`) — Phaser's key-capture list called `preventDefault` from a `window` listener that never checked focus, so W, A, S, D, J, Space and the digits could not be typed into the naming field.

### Integration

Built and maintained the `Review` branch that brought the parallel workstreams together (`5ce877a`, `08d79d2`, `5fa1365`), including recovering 13 unmerged commits that would otherwise have been garbage-collected.

---

## Work done in parallel

Three days with five people running in parallel means some problems got found and fixed by more than one person at once. Noting these honestly rather than double-claiming them:

- **The text-field keyboard bug.** Found and fixed independently on both sides — Logesh tied the capture release to the menu-mute flag; Ojas tied it to focus. Main ships the mute-based version.
- **Village building collision radii.** Both arrived at near-identical numbers independently (`hut` 34 vs 35, `hut_big` 52 vs 48, `forge` 32 vs 30). Main ships Logesh's; Ojas's per-instance scale handling and the art-derived test suite were kept on top.
- **The HUD.** Logesh drew it and Siddharth wired it; Ojas fixed the panel border, minimap rendering, twin scale and the ability keybindings against it.
- **Enemy rendering.** Logesh drew the sheets; Ojas wired them into the client and fielded the roster by biome.
- **Pattern detection.** Ujesha wrote it; Siddharth ported it onto main's server layout.

---

## Timeline

| Date | |
|---|---|
| **18 Sep** | Scaffold, game core, player model, the Twin's utility AI — 38 commits |
| **19 Sep** | The big day: art integration, campaign, AI adaptation work, HUD — 63 commits |
| **20 Sep** | Polish, combat geometry, the final feature merge — 31 commits |
| **21 Sep** | Last fixes — 2 commits |
