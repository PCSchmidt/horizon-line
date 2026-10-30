# Horizon Line - Agent Layer (secondary scope)

**Status:** Draft v1 - design only. Dark Fleet is the primary agent showcase; this title reuses its harness (`HRAgentKit`) where possible. Rules are identical to [Dark Fleet AGENTS.md](https://github.com/PCSchmidt/dark-fleet/blob/main/docs/AGENTS.md): agents act through the same typed, role-limited API as a human, decisions are recorded as inputs, the game is fully playable without them, and in-world text is untrusted.

## 1. Agents

| Agent | Mode | Observations | Tools | Output |
|---|---|---|---|---|
| **AI Spotter** | Practice (and optional unranked duo) | Rendered spotting-scope frames and the handheld wind meter reading; **never** the true wind field | `capture_view`, `measure_angle`, `read_wind_meter`, `estimate_range` | Wind call with uncertainty; range estimate |
| **Replay Coach** | Post-stage | Replay and shot log | `list_shots`, `get_trajectory`, `get_wind_along_path`, `compare_to_ideal_dope` | Error attribution and a practice plan; each statement cites shot IDs |
| **Range Master** (stretch) | Training | Curriculum state | `list_lessons`, `grade_drill` | Adaptive drill selection |

## 2. Models
- Vision-language model for the spotter (local open-weight model by default; hosted optional) `[VERIFY model availability and VRAM fit]`.
- A comparison against the dedicated learned wind reader ([ML.md](./ML.md) §1): does a general VLM beat, match or fall short of a specialized model, and at what latency and cost?

## 3. Evaluation (a small benchmark: HorizonLine-Bench)
| Metric | Definition |
|---|---|
| Wind-call error | MAE of crosswind vs truth along the trajectory |
| Calibration | Coverage of stated intervals |
| Decision utility | Mean miss distance when a bot shooter uses the call |
| Faithfulness (coach) | Fraction of coach claims whose cited shot IDs support them (deterministic checker) |
| Cost/latency | Tokens, seconds per call |
Baselines: parametric error model, optical-flow regressor, specialized learned reader. Fixed held-out seeds, bootstrap CIs, ablations (view type, clip length, prompt variants).

## 4. Security
Spotter frames are images produced by the sim, but scenario text (target labels, stage names) is untrusted. Same layered defenses and reporting as Dark Fleet; injection tests limited to text fields.

## 5. Risks
VLM wind reading may be poor; report the result either way. Replays are the grounding source; the coach must not invent shots.
