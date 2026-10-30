# Horizon Line - Verification Spike Ladder

**Created:** 2026-10-29 | **Status:** Not started

Each spike has a question, build, pass criteria, fail response and recorded evidence. A spike result is a measured fact; do not mark a spike passed without the evidence artifact.

| ID | Question | Build | Pass criteria | If it fails |
|---|---|---|---|---|
| **S1** | Can an MPM integrator reproduce published trajectories? | Headless C++ library + test harness; 5 cartridges at 100-1,500 m using public G1/G7 BCs and MVs | Drop and drift within ±0.1 mil of the reference at 500, 1,000 and 1,500 m; time of flight within ±1% | Revisit atmosphere, drag-curve interpolation, step size |
| **S2** | Is the sim deterministic? | Run 10,000 randomized scenarios twice, with 1 and N threads, `/fp:strict` build | Zero state-hash mismatches; replay re-sim identical | Pin math library; remove non-deterministic paths; restrict verification to identical builds |
| **S3** | Does visible wind equal sim wind? | UE scene with grass, flag, dust driven by sampled vector; automated capture and optical-flow comparison | Correlation ≥ 0.9 between sim crosswind and measured visual displacement across 50 captured conditions | Re-route vertex animation to sampled vector; simplify cues |
| **S4** | Can distance and zoom render within budget? | Benchmark range with targets at 300-3,000 m, scoped view | Reference tier ≥ 90 FPS at 1080p; Dev laptop ≥ 60 FPS floor; 1% low reported; no flicker of targets > 1 px under TAA | Reduce max range, detail corridor, tune AA, lower foliage density |
| **S5** | Is a believable, honest mirage feasible? | Post-process refraction field driven by temperature-gradient + near-ground wind | 8 blinded owner trials: reading mirage direction correct ≥ 75% when wind ≥ 3 m/s; absent cue ≈ chance | Use stylized but sim-linked shimmer; document as simplified |
| **S6** | Does 6-DOF improve ELR fidelity enough to justify cost? | Add 6-DOF integrator with aerodynamic coefficients where public data exists | Differences vs MPM at 2-3 km quantified; decision recorded | Stay MPM; hide transonic edge cases |
| **S7** | Do wind-reading bots produce a skill gradient? | Bots with error σ levels, 200 matches each on seeded stages | Monotonic score ordering; spread between best and worst ≥ stated threshold | Increase cue strength / variance; retune scoring |
| **S8** | Does server re-simulation reject cheating? | Craft tampered logs (impossible timing, modified turret values) | 100% of tampered logs rejected; 0% false rejects on 1,000 legitimate runs | Tighten bounds; improve logging |
| **S9** | Is hit adjudication exact? | Compare trajectory/geometry intersection with analytic cases | Zero disagreements on 10,000 cases within geometry tolerance | Fix swept-collision handling |
| **S10** | Does the optic model read correctly? | Reticle at known angular sizes | Ranging by reticle error ≤ 2% on test targets | Correct FOV/subtension math |
| **S11** | Does the learned wind reader beat the optical-flow baseline? | Render clips with ground-truth wind; train per ML.md section 1 | Lower crosswind RMSE than the baseline on held-out ranges (paired bootstrap p<0.05); interval coverage within 5 points of nominal; decision-level miss distance reduced | Improve data/randomization; report negative result |
| **S12** | Can a general VLM spotter call wind usefully? | Local VLM on spotting-scope frames vs truth | MAE, calibration and cost reported against baselines; decision recorded (include/exclude) | Drop spotter; keep coach only |
| **S13** | Does the coach ground its claims? | Deterministic checker on 200 coach outputs | >= 95% of cited shot IDs exist and support claims; unsupported claims flagged | Tighten tool outputs |

Pass thresholds are initial targets and must be justified before a pass is claimed.

## Order and dependencies
S1 → S2 → S3 → S4 → S5; S6 after S1; S7 after S1-S3; S8 after S2; S9 and S10 any time after S1.

## Evidence format
For each spike store: commit hash, build ID, input data hashes, output tables/plots, pass/fail verdict and date in `docs/spike-results/Sx.md`. Failed attempts are retained.
