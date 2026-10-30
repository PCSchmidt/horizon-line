# Horizon Line - Requirements, User Stories and Session Walkthrough

**Status:** Draft v1 - design only
**Companion to:** [SPEC.md](./SPEC.md). Priorities: **P0** first playable headless/greybox slice; **P1** first complete game; **P2** later.

## 1. User stories and acceptance scenarios

### US-1 (P0) - Shoot a stage by reading the wind
As a competitor, I estimate range and wind, dial or hold, and engage steel targets within a time limit.
1. **Given** a seeded stage, **when** I open the briefing, **then** I see the range card (target distances, sizes, par time, position) and weather summary but not the wind field.
2. **Given** I range a target with the reticle, **when** I use correct angular size, **then** my range estimate is within 2% of truth.
3. **Given** I fire a shot, **when** the bullet flight completes, **then** impact (hit/miss, point of impact) is shown with realistic delay and the sound arrives after the flight plus propagation delay.
4. **Given** I miss, **then** the shot is logged in the shot log and the debrief shows the wind the bullet experienced.

### US-2 (P0) - Wind cues are honest
1. **Given** a scenario, **when** the wind changes, **then** grass, flags, dust and mirage motion change consistently with the sim wind at their positions.
2. **Given** an automated test, **when** cues are captured, **then** correlation with sim wind meets the S3 threshold.

### US-3 (P0) - Replay and verify
1. **Given** a recorded run, **when** re-simulated, **then** every shot's impact and the final score match.
2. **Given** a tampered input log, **when** verified, **then** it is rejected with a typed error.

### US-4 (P1) - Learn through a tutorial curriculum
Staged lessons: reticle ranging, elevation holds, wind estimation, mirage reading, positions, spotter calls. Each lesson has a pass condition.

### US-5 (P1) - Daily seed and leaderboard
A seeded scenario is the same for everyone; verified scores appear on a leaderboard.

### US-6 (P1) - Duo mode with a spotter
Spotter and shooter share annotations; score is for the pair.

### US-7 (P1) - Review after the match
Shot-by-shot truth overlay: wind vectors along each trajectory, my dope vs ideal dope, error attribution (range error, wind error, human dispersion).

### US-8 (P1) - Optional AI spotter and coach
1. **Given** AI assistance is enabled in Practice, **when** I request a wind call, **then** the spotter returns a call with stated uncertainty derived only from what a human spotter could observe.
2. **Given** I complete a stage, **when** I open the coach, **then** it explains misses using replay tools and cites shot IDs.
3. **Given** AI features are disabled, the game is fully playable.

### US-9 (P1) - Accessibility
Reticle colors and thickness adjustable, subtitles, remappable controls, adjustable time pressure in practice.

### US-10 (P2) - Live duel
Two shooters on the same targets with a server-authoritative sim.

## 2. Functional requirements

### Ballistics and environment
- **FR-001 (P0)** Trajectories are deterministic: same build + scenario + inputs ⇒ identical impacts and state hashes.
- **FR-002 (P0)** Projectile model: modified point-mass with drag from public Cd(Mach) tables or G1/G7 BC; ISA atmosphere with temperature, pressure, humidity; gravity; Coriolis/Eötvös; spin drift; aerodynamic jump; muzzle-velocity variation.
- **FR-003 (P0)** Reference trajectories for named public cartridges match published tables within tolerance (S1).
- **FR-004 (P0)** Wind field: log-law mean profile, mesoscale drift, turbulence, terrain effect; sampled per integration step.
- **FR-005 (P0)** Every visible wind cue is driven by the sampled sim wind (P1 honesty pillar).
- **FR-006 (P1)** Mirage/shimmer is driven by temperature gradient and near-ground wind.
- **FR-007 (P1)** Visibility/haze follows an extinction model.

### Optics and handling
- **FR-020 (P0)** Reticle angular subtensions are exact; MIL and MOA systems supported.
- **FR-021 (P0)** Turret clicks have correct click value and travel limits; zero is set at a stated distance.
- **FR-022 (P1)** FFP and SFP reticles; parallax adjustment; scope imperfections tunable.
- **FR-023 (P0)** Position stability model (sway, breathing, pulse, support quality) affects point of aim.
- **FR-024 (P1)** Rangefinder with error/reflectivity limits; handheld wind meter samples only the local wind.

### Targets and scoring
- **FR-030 (P0)** Hit adjudication by exact swept intersection of the trajectory with target geometry.
- **FR-031 (P0)** Score is a pure function of the replay.
- **FR-032 (P1)** Targets: steel plates, paper, clay, mechanical movers on rails; no human or animal targets.

### Competition and data
- **FR-040 (P0)** Scenario, action and replay formats follow platform DATA_CONTRACTS.
- **FR-041 (P1)** Server re-simulation verifies leaderboard submissions.
- **FR-042 (P1)** Ranked divisions: with and without the in-game ballistic computer.

### AI (see [ML.md](./ML.md) and [AGENTS.md](./AGENTS.md))
- **FR-050 (P0)** Scripted bots with wind-estimate error models define skill tiers.
- **FR-051 (P1)** Learned wind estimator from visual cues with published evaluation.
- **FR-052 (P1)** Optional spotter/coach agents are role-limited (no access to true wind) and fully optional.

## 3. Non-functional requirements
- **NFR-001** Scoped view ≥ 90 FPS at 1080p on the Reference tier (target; S4).
- **NFR-002** Headless sim ≥ 10,000 shots/s on the Dev machine.
- **NFR-003** A stage lasts 5-12 minutes.
- **NFR-004** Single-player is fully offline.
- **NFR-005** Ballistic reference data uses public values only and is cited.
- **NFR-006** Every learned model has a model card and reproducible training.

## 4. Session walkthrough (Stage Series, ~10 minutes)

| Time | Experience | Systems |
|---|---|---|
| 0:00 | Briefing: "Stage 3: Barricade, 6 steel plates, 480-950 m, 120 s par, gusty crosswind." Range card shown. Player picks loadout (6.5-class cartridge, 100 m zero, 0.1 mil clicks). | Scenario, FR-020/021 |
| 1:00 | Player reads dope card from practice. Looks at flags and grass through the spotting scope; notes mirage boil flowing left at the 700 m mark. | FR-004-006 |
| 1:30 | Timer starts. Ranges plate 1 with the reticle (420 mils tall target → 2% error). Dials elevation, holds 0.6 mil right. | FR-020 |
| 2:00 | Breath settles; fires. Sees vapor off the plate after ~0.9 s and hears the ring ~2.8 s after the shot. Hit. | FR-002, FR-030 |
| 3:00 | Gust arrives mid-flight on plate 4: shot lands 0.4 mil left. Player corrects for the second shot. | Wind field |
| 7:00 | Time expires with 5 of 6 plates hit. | Scoring |
| 8:00 | Debrief: wind vectors along each trajectory, error attribution ("wind estimate off by 1.5 m/s on shot 4"). Replay saved; coach agent offers to review (optional). | US-7, US-8 |

## 5. Traceability
FRs map to spikes S1-S10 in [SPIKES.md](./SPIKES.md) and automated tests; a `docs/TRACEABILITY.md` is created at implementation start.
