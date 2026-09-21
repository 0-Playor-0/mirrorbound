# How Mirrorbound works

A tour of the backend, the AI, and the reasoning behind both.

---

## 1. The one rule

> **Python owns truth. Phaser displays truth. AI suggests intents. The game validates and executes.**

Everything else follows from this.

The server runs the whole simulation at **60 Hz** and is the only thing that decides what is true. The browser client sends **intents** — a movement vector, an attack flag, an ability slot, an aim direction — and draws whatever snapshot comes back. It runs no physics, resolves no collisions, and rolls no dice.

This is not an anti-cheat posture. It is what makes the AI honest. The Twin and the boss go through **exactly the same intent contract as the player**. They cannot teleport, cannot swing through a wall, cannot see through fog, and cannot act faster than the rules allow. When the Twin does something clever, it did it with the same verbs you have.

```
┌──────────┐   intents (WS)   ┌────────────────────────────────────┐
│  Client  │ ───────────────► │           Session @ 60 Hz          │
│ (Phaser) │ ◄─────────────── │  input → simulate → snapshot       │
└──────────┘   snapshots      └───────────────┬────────────────────┘
                                              │ emits frozen events
                                              ▼
                                      ┌───────────────┐
                                      │   Event bus   │
                                      └───────┬───────┘
                          ┌───────────────────┼───────────────────┐
                          ▼                   ▼                   ▼
                  ┌───────────────┐  ┌────────────────┐  ┌──────────────┐
                  │ Player model  │  │   Prediction   │  │   Heatmaps   │
                  │  (9 traits)   │  │ (Markov 1..3)  │  │  (decaying)  │
                  └───────┬───────┘  └────────┬───────┘  └──────┬───────┘
                          └──────────┬────────┴─────────────────┘
                                     ▼
                      ┌──────────────────────────────┐
                      │  Twin controller (utility AI)│
                      │  Mirror boss AI              │
                      └──────────────┬───────────────┘
                                     │ intents, same contract as the player
                                     ▼
                              back into the Session
```

---

## 2. The simulation loop

`api/session.py` owns a tick. Each one, in order:

1. **Drain input.** Whatever intents arrived over the WebSocket since last tick.
2. **Step the agents.** The Twin and each enemy controller produce an intent — but only if their decision cadence says it is time. The Twin re-decides a few times a second, not 60; deciding every frame produced a companion that twitched between plans and never committed to any of them.
3. **Apply intents** through the ordinary movement and combat code.
4. **Simulate.** Movement, collision, separation, projectiles, hitbox resolution, damage, deaths, pickups, room state.
5. **Emit events.** Everything meaningful becomes a frozen record on the bus.
6. **Snapshot.** Serialise the world and push it to every connected client.

The clock is fixed at `SIM_HZ = 60` (`game/core/clock.py`). Fixed rather than wall-clock so the same seed and the same inputs produce the same run, every time.

### Determinism

Randomness comes from a `DeterministicRNG` seeded per run, which can `spawn(label)` independent sub-streams. That matters: if room generation and combat crits drew from one shared stream, adding a single decorative bush would change every crit for the rest of the run. Named sub-streams keep them independent.

Combined with the frozen event log, this gives **JSONL replay**: a run can be recorded and played back exactly, which is the only practical way to debug an AI that learns.

---

## 3. The event bus

The substrate the whole AI sits on. Events are recursively frozen (`MappingProxyType`) at emission, so nothing downstream can rewrite history — not the player model, not the predictor, not a test.

This sounds fussy for a three-day build. It caught real bugs. When an agent can mutate the record of what happened, a learning system built on that record produces results you cannot reproduce or reason about, and you will not notice until the behaviour is subtly wrong.

A representative event:

```python
state.emit(
    "PLAYER_ATTACKED",
    action_token=token,
    tags=weapon.get_tags(),
    weapon=weapon.id,
    position=player.position.to_dict(),
    facing=facing.to_dict(),
    comboStep=player.combo_step,
    targets=[e.id for e in hits],
    hitCount=len(hits),
    nearestEnemyDistance=self._nearest_enemy_distance(state, player.position),
)
```

Note `nearestEnemyDistance`. The event carries the context the learner needs, at the moment it was true. Reconstructing "how far away was the nearest enemy when they swung?" after the fact is impossible once everything has moved.

---

## 4. The player model

`agent/player_model/` — nine traits, each an EWMA over the events that bear on it.

| trait | what moves it |
|---|---|
| `aggression` | attack frequency, closing distance |
| `mobility` | distance covered, dash use |
| `risk_tolerance` | attacking while hurt; retreating early vs. late |
| `preferred_range` | distance to the nearest enemy when you commit to an attack |
| `melee_dependency` | melee-tagged attacks as a share of all attacks |
| `ranged_dependency` | ranged-tagged attacks |
| `spell_dependency` | spell casts |
| `defensive_tendency` | dodges, blocks, retreats, healing |
| `combo_dependency` | how far into combo chains you go |

### Confidence is the interesting part

Each trait carries a sample count and a confidence derived from it:

```python
self.confidence = min(0.999, 1 - math.exp(-self.samples / CONFIDENCE_SATURATION))
```

and confidence decays on a half-life:

```python
self.confidence *= 0.5 ** (elapsed_seconds / CONFIDENCE_HALF_LIFE_SECONDS)
```

Everything that consumes a trait goes through a gate that blends it back toward neutral in proportion to how unsure the model is:

```python
def confident_value(self, name):
    dim = self.get(name)
    return 0.5 + (dim.value - 0.5) * dim.confidence
```

This is what stops the classic adaptive-AI failure. Without it, your first two actions in a run define you: swing twice, and `aggression` is 1.0, and the companion commits hard to a read built on two samples. With it, a raw average of 1.0 at low confidence still reads as roughly 0.55, and the Twin behaves neutrally until the evidence is actually there.

The decay matters for the opposite case. Play cautiously for ten minutes, then change your mind — the old read fades rather than anchoring the rest of the run.

### A rule the model keeps

**If a field is missing, skip the event. Never guess.** A trait fed a default when the real value was unavailable learns the default, and you get a confident, wrong read that is much worse than no read at all.

---

## 5. Prediction and pattern detection

`agent/prediction/` holds order-1, order-2 and order-3 Markov chains over the action stream, with **back-off**: if the order-3 context has not been seen enough times to be worth trusting, fall back to order-2, then order-1. Standard n-gram practice, and the right shape here — long contexts are precise but sparse, short ones are noisy but always available.

`agent/patterns/` sits on top and watches for **repeated sequences**. When one recurs enough to stop looking like coincidence, it is promoted to a named habit and shown in the AI view. This is what lets the game say *"you have done these three things twice"* rather than just showing a probability.

`agent/spatial/` keeps decaying grids: where you fight, where you retreat to, where you dodge, where you cast, where you have died. The boss reads these. Dying repeatedly in one corner teaches something about that corner.

---

## 6. The Twin

`agent/twin/` — two halves.

### controller.py — deciding what to do

A **utility AI**. Each decision, it builds the full candidate set of intents:

`ATTACK` `FLANK` `PROTECT` `RETREAT` `INTERCEPT` `DISTRACT` `ASSIST` `FOLLOW` `EXPLORE` `REPOSITION` `DASH` `SHADOW_DASH` `FLAME_BURST` `FIRE_BURST` `BINDING_NOVA`

and scores every one against the current situation *and* its learned style, then takes the best. Not a behaviour tree, not a state machine — a scored comparison, which is what lets style shift the outcome smoothly rather than requiring new branches.

Two things keep it from being annoying:

- **Hysteresis.** The incumbent intent gets a bonus, so the Twin commits to a plan instead of recomputing a marginally better one every cycle.
- **Posture momentum.** Its aggressive/defensive stance carries over between decisions, so it reads as having a mood rather than a random seed.

Some of the scoring is genuinely situational. `ATTACK` excludes the enemy *you* are already fighting unless focus-firing is actually the right call, so the Twin spreads pressure rather than crowding your target. It weighs whether a fight is worth joining at all instead of always starting its own.

### style.py — becoming like you

Nine style dimensions of its own: `preferred_range`, `aggression`, `mobility`, `risk_tolerance`, `target_preference`, `melee_dependency`, `ranged_dependency`, `defensive_tendency`, `spell_preference`.

They are pulled by two forces:

```python
IMITATION_RATE  = 0.06   # how hard the player's behaviour pulls the twin's style
EXPERIENCE_RATE = 0.12   # how hard the twin's own outcomes pull it
```

Experience pulls roughly twice as hard as imitation. The Twin copies you, but what actually works for it counts for more. Copy a bad habit, and its own results will argue it back out.

The same confidence gate applies here, and its own style feeds back into the utility scores — so a Twin that has learned you fight at range will score `FLANK` and `REPOSITION` differently from one that has learned you brawl.

It also picks up weapons it passes and builds its own arsenal over a run.

---

## 7. The Mirror

The final boss reads the same player model, and it is deliberately given a **longer memory** than the Twin. The Twin should feel responsive to how you are playing *now*; the boss should feel like it has been watching the whole time.

Two behaviours worth calling out:

- **Probing.** When the model's confidence is too low to have a real read, the boss does not guess — it probes. It tests you at specific ranges to generate the samples it lacks, then commits to an attack type based on what it learned. An enemy that gathers information is a more interesting enemy than one that guesses.
- **Commitment.** Once it has a read, it commits to an attack type rather than re-deciding mid-approach, which is what makes its attacks readable enough to counter — and therefore fair.

---

## 8. Combat geometry

Worth a section because it is the part most likely to be quietly wrong, and was.

**Hit radius is not physics radius.** Every enemy has a `size` (how much room its body takes up, for separation and walls) and a `hit_radius` (how big a target it is). These were one number, authored against round procedural blobs. Once the art became hand-drawn sheets — a Husk Scarab drawn 96 units across, a Hollow Archer 28 — one number could not be both. Widening `size` to match the art would have made scarabs shove each other apart from twice the distance, which is the opposite of a swarm.

Measured before the split: a straight shot registered on **27%** of a scarab's visible width, **35%** of a hound's, **48%** of the Mirror's. Arrows passed through drawn bodies and missed.

**A swing hits a body, not a point.** The melee arc test measured *distance* to the target's near edge and then measured *angle* to its centre — so for the angle check every enemy was a point. The tell: with bare hands (a 97° arc) every archetype cut off at exactly 48° either side regardless of size. A body now contributes its angular width, `asin(r/d)`:

| | scarab | brute | hound | skeleton | archer |
|---|---|---|---|---|---|
| before | 48° | 48° | 48° | 48° | 48° |
| after | **101°** | **100°** | **98°** | **68°** | **64°** |

**Props block what they draw.** Village buildings were all placed on one hardcoded collision radius — a flagpole and a two-storey house on the same number — so you could walk into the houses and bounce off thin air beside the banner. Each kind is now sized from its drawn art and scaled per instance.

---

## 9. Testing

**467 server tests, 54 client tests.**

The ones that matter most are not unit tests of individual functions but **adaptation proofs**: run a scripted player profile through the pipeline and assert the model ends up where it should. A melee brawler must read as melee; a kiting caster must read as ranged; the Twin of an early retreater must want to retreat *more* than the Twin of a late one.

Tests are written against the **source of truth rather than the chosen number** wherever possible. The collision tests measure the sprite atlases directly and assert the radius matches the drawn art — so re-cutting a sheet at a different size fails in the test suite rather than silently in the game.

And every fix ships with a check that it actually catches the bug: revert the fix, and named tests must fail. A regression test that passes against the broken code is not a regression test.

---

## 10. The client

TypeScript, React 19 for menus and overlays, Phaser 4 for the world, Vite for the build.

- **Snapshot-driven.** Entity views reconcile against each snapshot; nothing is simulated locally.
- **Art pipeline.** 156 atlases built from source sheets by a Python script, with generated TypeScript modules exporting frame metadata — so the renderer never hardcodes a frame name or a sheet size.
- **One keybinding table.** Movement and combat are sampled by Phaser; menus and world keys are handled by React. Both read the same table, so rebinding "attack" and rebinding "inventory" go through one place and cannot disagree.
- **Canvas HUD.** Drawn in-engine rather than in the DOM, so it sits in the same visual language as the art.

---

## 11. Things that are still true and not ideal

Being straight about it:

- `defensive_tendency` still correlates heavily with attack frequency — it moves, but it is not fully independent of `aggression`.
- `risk_tolerance` from attacks looks only at your health, not at how many enemies are near. Most fights start healthy, so ordinary play reads slightly cautious.
- `preferred_range` uses the *nearest* enemy, not the one you are actually engaging.
- Trait confidence tracks sample count only. It does not yet account for how *consistent* you have been.
- Prediction accuracy is not scored against outcomes — a 70%-confidence guess is not yet verified to be right about 70% of the time.
