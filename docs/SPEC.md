# Horizon Line - Architecture and Scoping Specification

**Created:** 2026-10-29
**Status:** Draft v1 - design only; nothing implemented or verified
**Platform dependency:** [hyperreal-platform](https://github.com/PCSchmidt/hyperreal-platform) (HRCore, HRScenario, HRReplay, HRAtmosphere, HRGeo, HRRating)
**Companion documents:** [REQUIREMENTS.md](./REQUIREMENTS.md) (user stories, FR IDs, session walkthrough) | [ML.md](./ML.md) | [AGENTS.md](./AGENTS.md) | [SPIKES.md](./SPIKES.md) | [PRIOR_ART.md](./PRIOR_ART.md) - verification ladder with pass/fail criteria

> Items tagged `[VERIFY]` are assumptions not yet checked against the installed engine, a license, or measured data.

---

## 1. Vision

**Pitch.** A competitive precision-rifle sport simulator in which the skill is *reading the environment*. Wind, mirage, light and atmosphere are simulated and rendered from the same state, so the better observer wins.

**Player fantasy.** Dial a rifle, read the flags, grass, mirage and vapor, call a wind hold, and watch a steel plate ring at 1,200 metres - then do it faster and more accurately than the person or ghost you are competing against.

**Design pillars**
1. **Honest wind.** Every visible wind cue is sampled from the same wind field that bends the bullet.
2. **Optics are the interface.** The scope is not a UI overlay; reticle, parallax, magnification, turret clicks and glass behavior are part of the skill.
3. **Calm tension.** Pressure comes from clocks, wind changes and scoring, not from reflexes under fire.
4. **Explainable outcomes.** After every shot the replay shows what the wind did, so misses teach.
5. **Sport, not warfare.** Paper, steel, clay and reactive targets only.

**Non-goals:** human or animal targets, infantry combat, vehicles, destructible worlds, inventory/loot, a realistic ballistic-computer replacement for real-world use, open-world traversal.

---

## 2. Competitive design

### 2.1 Modes

| Mode | Description | Netcode need |
|---|---|---|
| **Stage Series** (PRS-style) | A match of timed stages: different positions (prone, barricade, rooftop, tripod, unsupported), target sets 300-1,200 m, par times, wind that shifts. Score by hits and time. | None (async) |
| **Long Range Ladder** | Single engagement ladder: 600 m to 2,000+ m, one scenario per day (seed). Score by distance-weighted first-round hits. | None (async) |
| **ELR Challenge** | Extreme long-range (1.5-3 km), subsonic transition modeling, large steel. Fewer, harder shots. | None |
| **Spotter-Shooter Duo** | Two players: spotter has the spotting scope, wind kit and range finder; shooter has the rifle optic. Voice or shared annotations. Score as a pair. | Live (phase 2) |
| **Duel** | Two shooters, same targets alternating or simultaneous, first to N points; sees the other's impacts. | Live (phase 2) or async "ghost duel" |
| **Daily Seed** | Same scenario for all players; global leaderboard. | None |
| **Training Range** | Free practice with a ballistics drill library and a post-shot "wind truth" overlay. | None |

### 2.2 Scoring
- Points per target by size/distance (angular-size weighted), first-round-hit bonus, time bonus, penalty for misses on targets with limited rounds.
- **Hit adjudication** is by exact intersection of the simulated trajectory with target geometry (steel plate, paper rings by distance from center), not by hit-scan rays.
- Scores are computed by a **pure function of the replay**, so any score can be re-derived by the server.

### 2.3 Ratings and seasons
Elo/Glicko-2 for duels and duo pairs; leaderboards for seeded modes. Rating calibration uses scripted bots of known skill (Section 11).

### 2.4 Anti-cheat
Because the sim is deterministic and results derive from input logs: (a) server re-simulation; (b) bound checks on dial/turret input rates; (c) detection of overlay or ballistic-solver tool use is *not* attempted - instead the ranked modes either allow the in-game ballistic computer (as a standardized tool) or disable it per mode, and every input is logged; (d) telemetry anomaly screening for implausible wind-call accuracy.

---

## 3. Simulation architecture

### 3.1 Scope of physical models

| Layer | Model | Rationale |
|---|---|---|
| **Projectile (core)** | Modified point-mass (MPM) trajectory with drag by Mach-dependent `Cd(M)` tables (G1/G7 standard curves, or custom drag curves where public data exists) | Standard external-ballistics practice; transparent and verifiable |
| **Optional high-fidelity** | 6-DOF rigid-body with aerodynamic coefficients | Spike S6 decides whether it earns its cost; used for transonic/ELR realism |
| Gravity | Latitude- and altitude-dependent `g` | Small, cheap |
| Atmosphere | ICAO International Standard Atmosphere baseline + user/scenario temperature, pressure, humidity → air density, speed of sound | Drag depends on density and Mach |
| Wind | 3-D, time-varying wind vector field (Section 3.3) | Central mechanic |
| Coriolis / Eötvös | Full vector form using latitude and azimuth | Matters at ELR |
| Spin drift | Empirical/analytic estimate from twist rate, stability factor (Miller) and time of flight | Real, small, learnable |
| Aerodynamic jump | Vertical deflection from crosswind × gyroscopic stability | Small, optional |
| Transonic behavior | Drag rise through Mach ~1.2→0.9 handled by `Cd(M)`; stability warning | Realistic ELR drama |
| Muzzle velocity | Per-shot variation (extreme spread/SD), temperature sensitivity of powder | Gives irreducible dispersion |
| Rifle dispersion | Mechanical accuracy (MOA class), position stability, breathing, pulse, trigger break | Human-factor layer |

### 3.2 Integration
- Fixed internal step **0.25-1.0 ms** (to be set by Spike S1), RK4 or adaptive RK45 with a fixed substep count. At 2 km a flight of roughly 3 s is a few thousand steps, which is trivially cheap even for many projectiles.
- `double` precision; trajectory sampled in a local ENU frame at the muzzle, converted to UE world coordinates only for rendering.
- **Validation:** compare computed drop, drift and time of flight with published reference trajectories for named cartridges (public manufacturer data and published drag-model literature) to stated tolerances; commit the reference tables as test fixtures with citations. Cartridge data is representative of *public* values (e.g. mass, G7 BC, muzzle velocity classes for 6.5 Creedmoor, .308, .300 PRC, .338 Lapua, .50 BMG classes).
- **Do not** use proprietary or restricted drag data. Where only G1/G7 BCs are public, use them.

### 3.3 Wind field - the key system
Requirements: spatially varying, temporally evolving, physically plausible, cheap to sample along a 2 km line, deterministic, and **visible**.

**Model (layered):**
1. **Mean wind** by height: logarithmic boundary-layer profile `u(z) = (u*/κ)·ln(z/z0)` with surface roughness `z0` from landcover; stability-dependent corrections optional.
2. **Mesoscale direction/speed drift** over minutes (slow random walk with bounds), scenario-defined.
3. **Turbulence:** divergence-free curl-noise or spectral (von Kármán/Kaimal-style) gusts, advected by the mean wind (frozen-turbulence hypothesis), with intensity from stability and roughness.
4. **Terrain effects:** speed-up on ridges, sheltering in lees, channeling in valleys, thermal updrafts over sun-heated slopes - approximated from the DEM with a simple diagnostic flow model (not CFD), precomputed per scenario into a coarse 3-D grid plus analytic terms.
5. **Local obstruction:** trees and buildings create wakes - approximated as sheltered zones.

**Sampling:** `Wind(x,y,z,t)` returns a vector; trajectory integration samples it per step (interpolated from the coarse grid + analytic noise).

**Visualization (P1 compliance):**
- Grass, trees, flags, dust, rain/snow and ribbons are driven by the same sampled vector at their positions. A dedicated test (Spike S3) asserts visual motion tracks sim wind.
- A "wind truth" debrief overlay draws the actual wind vectors along the bullet path after the shot.

### 3.4 Mirage and optical atmosphere
- **Heat shimmer / mirage:** a temperature-gradient field near the ground (from solar heating and surface type) produces refraction-driven image displacement and *boil*. Gameplay-relevant: mirage direction and speed are a classic real-world wind indicator; the sim ties shimmer flow to near-ground wind so the cue is honest.
- **Atmospheric extinction:** Koschmieder-style visibility `V ≈ 3.912/β`; haze, humidity, dust and smoke reduce contrast by range.
- **Light:** time of day, sun azimuth, glare onto the lens (sun in the objective) - optical difficulty without affecting ballistics.
- **Refraction near the horizon** affects line-of-sight at extreme range; modelled only if a spike shows it matters for the chosen ranges.

### 3.5 Weapon, position and human factors
- **Rifle:** barrel, twist, sight height, zero distance, ammo SKU. Players choose from a curated loadout list of public cartridge types.
- **Position stability model:** stance defines base sway amplitude and frequency; breath hold reduces sway for a limited time with oxygen-debt-driven tremor increase; pulse adds a small periodic component; support quality (bipod, bag, barricade) damps. Trigger control: break timing relative to the natural respiratory pause.
- **Recoil and recovery:** player can observe the splash/impact through the optic only if the recovery is fast; spotter provides corrections.
- **Fatigue:** across a stage match, a mild stability decay creates pacing decisions.

### 3.6 Targets
- Steel plates and gongs with sound-delay physics; paper with ring scoring; clay discs; reactive targets (falling plates, swingers). **Moving targets** (walkers, swingers) are limited to non-human mechanical rigs on rails.
- **Hit feedback:** visible impact spray/dust/vapor at the target, audio arrival delayed by distance / speed of sound, plate swing physics (cosmetic).
- **Time-of-flight feedback** is realistic: at ~1,000 m the flight is on the order of 1.5 s, so players see the splash only after recovering from recoil - the spotter role matters.

---

## 4. Optics and rendering

### 4.1 Optic model
- **Scope types:** fixed and variable power; first-focal-plane (FFP) and second-focal-plane (SFP) reticles; MIL and MOA turret/reticle systems; click value and total elevation travel limits; parallax adjustment; illumination; zero-stop.
- **Rendering approach:** at high magnification the main camera uses a **narrow FOV** with a scope-mask overlay rather than a second full scene render, reducing the cost of rendering distant geometry twice `[VERIFY perf in S4]`. The reticle is rendered in screen space with exact angular subtensions so that ranging by reticle is valid.
- **Optical imperfections (tunable realism):** chromatic aberration, edge distortion/vignetting, internal reflections, eye-box shadowing/relief, and parallax error when the eye is off-axis.
- **Sighting systems:** iron-sight/red-dot practice views; spotting scope (higher magnification, different reticle); rangefinder (laser with error and reflectivity limits); wind meter (Kestrel-like handheld, local sample only).

### 4.2 Distance rendering
- Targets at 1-3 km must remain resolvable and the terrain between must read correctly under heavy zoom. Strategy: Nanite terrain detail, aggressive mesh LOD for foliage, landscape virtual texturing, high-res target proxies, and **detail locking** along the shot line (extra tessellation/foliage density within a corridor).
- Atmosphere: height fog with scatter driven by HRAtmosphere visibility; sun disk, glare and bloom plausible under magnification.
- Sub-pixel issues: temporal stability under high zoom (shimmer vs. mirage distinction). The temporal anti-aliasing configuration must not erase the mirage signal.
- Target frame budget (to be tested): ≥ 90 FPS in the scoped view at 1080p on the Reference tier; ≥ 60 FPS floor on the Dev laptop with software-Lumen quality `[VERIFY]`.

### 4.3 Environmental fidelity
- **Terrain:** DEM-driven landscape with high-resolution lidar where available; hero ranges authored or photogrammetry-enhanced.
- **Vegetation:** PCG-placed grass/shrub/tree with wind-driven vertex animation sourced from the sim wind vector (Section 3.3).
- **Skies:** physically based sun/sky, volumetric clouds (cloud shadows moving across the range are both beautiful and a thermal/wind cue).
- **Weather:** rain, fog, snow reduce visibility and affect drag marginally (density).

---

## 5. Environment and content

### 5.1 Range types
| Range | Description | Source strategy |
|---|---|---|
| **Basin range** | Open desert/valley, long clear lanes | USGS 3DEP + NAIP imagery; PCG vegetation |
| **Mountain** | Ridge shots, vertical angles, thermals, tricky wind | DEM + terrain wind model |
| **Coastal** | Maritime wind, haze, shoreline | Combine with HROcean for sea surface |
| **Forest edge** | Trees as wind cues and obstructions | Landcover-driven PCG |
| **Urban lanes (sport-only)** | Rooftop positions, industrial yards, steel targets only | Kit-based assets |
| **Arctic/winter** | Snow, low sun, cold-dense air | Tests atmosphere extremes |

### 5.2 Initial content target (vertical slice)
One range (basin), 3 positions, 12 targets from 300-1,400 m, 2 rifles/loadouts, 2 weather presets, 1 daily-seed generator.

### 5.3 Full content target
6 ranges, 8 loadouts, 40 stages, ELR ladder, tutorial curriculum (wind-reading, ranging, positional shooting, spotting).

---

## 6. UX and systems

- **Flow:** Loadout → Briefing (range card, conditions) → Stage → Debrief (shot-by-shot truth overlay) → Leaderboard.
- **Tools:** range card, dope card (player-generated), ballistic computer (optional per mode), notebook for wind calls.
- **Tutorials:** staged curriculum: ranging with the reticle, elevation holds, wind estimation, mirage reading, spotter communication, ELR mechanics.
- **Accessibility:** colorblind-safe reticle options, adjustable reticle contrast, subtitles for spotter voice, control remapping, one-handed input schemes, adjustable time pressure in practice.
- **Controls:** mouse (look), keyboard (position/breath), scroll/keys (turrets), optional controller; support rotary/analog input for fine turret control later.
- **Audio:** supersonic crack followed by muzzle report with distance-correct delay, wind noise on mic/ear, plate ring decay, ambient (insects, birds, distant machinery) tied to environment. Ear-protection toggle changes the mix.

---

## 7. Networking

- **Phase 1:** none. Async leaderboards via verified replay upload.
- **Phase 2:** server-authoritative duel and duo sessions. The server simulates projectiles; clients display. Because flights are ~1-3 s and results are resolved on the server, 50-100 ms latency is acceptable; shot timing uses server-tick stamping (the player's click is recorded with a client tick the server validates within a bound).
- **Shared wind:** server holds the wind field state; clients predict visuals using the same seeded noise so wind visuals do not require per-frame replication, only periodic corrections.
- **Spotter-shooter voice and shared marking:** platform voice service or third-party voice `[VERIFY]`; shared annotations replicated as lightweight events.

---

## 8. AI and ML

| Use | Approach |
|---|---|
| Ghost shooters | Replay playback |
| Scripted bots | Wind-estimation with error models producing skill tiers (novice → elite) |
| Learned wind-reader | Train a model to estimate effective crosswind from rendered or feature observations (grass motion, flag angle, mirage) - a notable ML portfolio piece: "an AI that learns to read wind from the same cues the human sees" |
| Spotter assistant | Optional trained assistant for single-player practice that gives imperfect calls |
| Anomaly detection | Detect improbable wind-call accuracy or perfect-correction patterns in telemetry |
| Evaluation | Bot-vs-bot rating tables to calibrate difficulty and scoring |

---

## 9. Technical architecture

### 9.1 Modules (game-specific)
```
HorizonLine (game module)
 ├─ HLBallistics     MPM/6-DOF integrator, drag tables, atmosphere hooks
 ├─ HLWind           field model, terrain diagnostic flow, sampler, visual drivers
 ├─ HLOptics         scope model, reticle rendering, turret logic, rangefinder, wind meter
 ├─ HLTargets        target types, scoring geometry, reactive behaviors
 ├─ HLMatch          modes, stages, rules, scoring-from-replay
 ├─ HLRange          range loading, landscape/PCG hooks, lane detail locking
 └─ HLUI             HUD, dope cards, debrief overlay
```
Depends on platform: HRCore, HRScenario, HRReplay, HRAtmosphere, HRGeo, HRRating, HRTelemetry, HRAgents.

### 9.2 Data flow
```
Scenario (seed, range, weather, loadout)
   → HLWind builds field → HLBallistics integrates shots (fixed step)
   → HLTargets adjudicates → HLMatch scores
   → HRReplay logs inputs + hashes
   → Render reads sim state (wind/vapor/impact) only
```

### 9.3 Performance considerations
- Ballistics and wind sampling are negligible next to rendering.
- Rendering budget dominated by distant detail and temporal stability under zoom.
- Headless sim build can run thousands of shots per second for validation and bot training.

---

## 10. Content, data and licensing
- Ballistic reference data: publish citations in `docs/SOURCES_AND_LICENSES.md`; include only public/non-proprietary values; do not copy commercial solver tables.
- Terrain/imagery: USGS and Sentinel/NAIP sources and attribution; record hashes in the data manifest.
- Assets: weapon and optic models self-made or CC/Fab licensed; no brand logos or trademarks without a license (generic names).
- Audio: field recordings or CC0.
- Safety/policy: sport framing, no human targets, no instructions for real-world weapon modification, ballistic data limited to what is public.

---

## 11. Verification strategy
See [SPIKES.md](./SPIKES.md). Summary:

| Layer | Evidence |
|---|---|
| Ballistics accuracy | Match published trajectory tables within stated tolerance (e.g. ≤ 0.1 mil drop and ≤ 0.1 mil drift at 1 km for reference cases) |
| Determinism | Identical state hashes across 10,000 headless runs and thread-count changes |
| Wind honesty | Automated test that rendered wind cues correlate with sim wind (S3) |
| Scoring integrity | Replays re-simulate to identical scores; tampered logs rejected |
| Performance | Frame-time percentiles on benchmark ranges (S4) |
| Fun/skill | Owner playtests; skill-gradient test with bots: better wind-readers must score higher |

---

## 12. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Ballistics fidelity disputes (purist community scrutiny) | Medium | Medium | Cite sources; expose model choices; validate against references |
| Wind cues too subtle or too easy → no skill gradient | High | High | Bot skill-gradient test; tune cue strength; difficulty presets |
| Mirage rendering is a visual research problem | Medium | Medium | Spike S5 with a stylized-but-honest fallback |
| Temporal AA destroys shimmer or creates fake shimmer | Medium | Medium | Dedicated AA settings for scoped view |
| Distance rendering cost / instability at 2-3 km | Medium | High | S4 benchmark; detail corridor; accept 1.5 km range limit if needed |
| Determinism of RK integration across builds | Low-Medium | High | Fixed step, `double`, strict FP, S2 |
| Perception of weapon-themed content | Low | Medium | Sport framing; no human targets |
| Hyper-real art volume | Medium | Medium | Data-driven ranges; few hero scenes |
| Duo/duel netcode complexity | Medium | Low (deferred) | Async first |

---

## 13. Milestones (gated)

| Gate | Deliverable | Exit evidence |
|---|---|---|
| H0 | Spikes S1-S2 | Trajectory matches references; deterministic replay |
| H1 | Spikes S3-S5 | Honest wind visuals; frame budget; mirage prototype |
| H2 | Vertical slice | One range, 12 targets, ghost competition, replay, debrief overlay |
| H3 | Bots + skill gradient | Rated bot ladder reproduces expected ordering |
| H4 | Verified async leaderboard | Tampered logs rejected |
| H5 | Content expansion | 3+ ranges, 20+ stages, tutorial |
| H6 | Duo / duel live (optional) | Latency-tolerant tests pass |

---

## 14. Open decisions

| Decision | Default | Resolve by |
|---|---|---|
| MPM vs 6-DOF core | MPM, 6-DOF for ELR only if S6 justifies | S6 |
| Ballistic computer allowed in ranked | Standardized in-game tool permitted in a "tools" division; off in "hardcore" | H2 |
| First hero range | Basin range from public lidar | H0 |
| Voice solution for duo | Defer | H6 |
| Cartridge list | 5-8 public cartridges | H2 |
| Max range supported | 2 km initially, ELR 3 km after S4 | S4 |
