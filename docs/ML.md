# Horizon Line - Machine Learning Specification

**Status:** Draft v1 - design only
**Standard:** Per the platform ML standard: problem formulation, data, baseline, metrics, ablations, reproducible training, export, model card, CI evaluation. Thresholds are targets to justify, not results.

## 0. Common infrastructure
Headless `hr-core`/`hl-core` simulation with Python bindings; synthetic data generation with ground-truth wind; train/val/test split by scenario seed with a locked test set; PyTorch training with config-driven, seeded runs; experiment tracking with config/data/commit hashes; ONNX export with parity tests; bootstrap confidence intervals; model cards.

## 1. Learned wind reader (flagship)
- **Problem:** estimate the effective crosswind component along the shot line (and its uncertainty) from the same visual cues a human uses: moving grass/vegetation, flag angle, dust/vapor, mirage flow, plus optional wind-meter reading.
- **Data:** the engine renders short clips (scoped and unscoped views, multiple ranges/times of day/weather) paired with ground-truth wind along the trajectory from the sim. Domain randomization over vegetation, lighting, haze, camera shake.
- **Models:**
  1. *Classical baseline*: optical flow (e.g. dense flow) plus a regression head on aggregated motion features.
  2. *Learned*: a small video model (3D CNN or a compact video transformer) with a probabilistic head (mean + variance) trained with negative log-likelihood.
  3. *Cue-ablated variants* to measure the value of each cue.
- **Metrics:** RMSE/MAE of crosswind (m/s) overall and by condition; negative log-likelihood; calibration (ECE of predicted intervals); impact on miss distance when used by the scripted shooter (decision-level metric); inference latency.
- **Ablations:** each cue on/off; mirage on/off; view type; clip length; domain randomization strength.
- **Honesty requirement:** report sim-to-sim generalization (held-out ranges and rendering settings) and explicitly state that performance is on synthetic imagery only.
- **Why it matters:** it is a vision problem with an interpretable ground truth and a meaningful downstream metric (shot error), and it directly supports the game's skill-gradient design.

## 2. Spotter/shooter bot tiers (classical baselines)
- **Problem:** produce bot opponents spanning novice to elite for ghost and ladder play.
- **Approach:** a shooter bot with an explicit wind-estimate error model (bias, noise, update latency), range-estimate error and position dispersion; tier = parameter set.
- **Metrics:** monotonic score ordering across tiers with separation beyond confidence intervals (S7); calibration of tier ratings (Elo/Glicko).
- **Use of the learned wind reader:** a bot tier that uses the learned estimator instead of an error model, to compare "learned" versus "parametric" difficulty.

## 3. Skill estimation and rating calibration
- **Problem:** estimate player skill by division from stage scores with limited samples.
- **Approach:** hierarchical Bayesian or Glicko-2/TrueSkill-style rating; compare with Elo.
- **Metrics:** predictive log-likelihood of match/stage outcomes on held-out bot tournaments; rating convergence rate; ranking stability.

## 4. Anomaly detection (integrity)
- **Problem:** flag improbable wind-call accuracy or perfect-correction patterns that suggest tooling or tampering.
- **Approach:** features from input logs (turret input timing, correction-to-error relationships, dispersion vs position stability) with isolation-forest/GBM baselines; calibrated to a low false-positive budget.
- **Data:** synthetic "cheater" policies (e.g. oracle-wind bots) injected among legitimate bots and recorded human runs.
- **Metrics:** recall at fixed FPR; time-to-detection; per-attack-type results.

## 5. Optional: dispersion/zero model fitting (not ML-heavy)
Fit muzzle-velocity SD and group size from synthetic shot groups; Bayesian estimation exercise for the practice-range "true zero" feature. Low priority.

## 6. Release checklist (per model)
Config/seed/commit/data hash recorded; one-command reproduction; held-out evaluation with CI; ONNX parity; model card (intended use, data lineage, metrics vs baselines, limitations, license); negative results retained.

## 7. Evaluation matrix
| Component | Primary metric | Baseline | Split |
|---|---|---|---|
| Wind reader | Crosswind RMSE, NLL, decision-level miss distance | Optical-flow regression | Held-out seeds/ranges |
| Bot tiers | Rank ordering with CI | Parametric tiers | Seeded stages |
| Rating | Predictive log-likelihood | Elo | Bot tournaments |
| Anomaly detector | Recall @ fixed FPR | Rule thresholds | Held-out bots/replays |
