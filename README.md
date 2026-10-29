# Computing–Electricity Coordinated Scheduling — Huashu Cup 2026, National Second Prize

**A complete decision loop from GPU demand forecasting to carbon-aware, storage-coupled
multi-objective scheduling: a TPE-tuned GBR fusion forecaster (38-dim multi-timescale
features, direct multi-step prediction), a composite carbon-cost matrix, an
Epigraph-linearized storage MILP, and an offline + MPC two-stage solver that compresses
solve time from 1420 s to 10.8 s at 1.16% optimality cost.**

7th Huashu Cup National Mathematical Contest in Modeling (华数杯), 2026 · **Problem C ·
National Second Prize** · Team participation code CM2601062.

> Team of three. I was responsible for **machine-learning model construction, all
> competition code, and model-effect validation** — the Problem 1 forecasting stack
> (feature engineering, GBR fusion, Optuna tuning, honest diagnostics) and the
> validation methodology used throughout.

The full competition paper (Chinese, 34 pages) is in [`paper/CCM2601062.pdf`](paper/CCM2601062.pdf).

## Problem

A high-density data-center operator must schedule heterogeneous GPU tasks (real-time
inference / batch inference / AI training) across regions while jointly managing
electricity cost, carbon emissions, and renewable intermittency — 2,400 hours of
panel data. Four chained sub-problems: demand forecasting → carbon-aware regional
scheduling → storage-coupled MILP → real-time re-scheduling under wind volatility.

## Act 1 — Demand forecasting: TPE-tuned GBR fusion (my core work)

**Feature engineering — 9 classes, 38 dimensions**, built on 42,336 hourly sequence
samples:

| Class | Examples | Dim |
|---|---|---|
| Time cycle | intra-day / intra-week, sin-cos encoding | 5 |
| Lags | t−1, t−2, t−24, t−48, t−168 | 8 |
| Rolling stats | 6/12/24/48 h mean & std | 8 |
| EW decay | multiple half-lives | 4 |
| Differences | 1st / 2nd order / weekly | 3 |
| Categorical | workday / holiday, peak-valley tariff window | 3 |
| External covariates | regional real-time price, marginal carbon factor | 2 |
| Cross-sequence | cross-DC load mean / variance / correlation | 3 |
| Inference-specific | batch/real-time ratio, SLA limits | 2 |

**Model**: XGBoost / LightGBM / CatBoost each win on different feature blocks and
horizons, so they are fused into one **GBR ensemble**; all hyper-parameters tuned by
Optuna TPE Bayesian optimization (60 trials, objective = validation RMSE, converged
at ~trial 40, RMSE 44.91). Direct multi-step prediction over a 24 h horizon — no
recursive error accumulation (per-horizon MAE stays 10.2–21.8 across h = 1…24).

**Honest test-set numbers** (last 24 h held out, never seen in tuning):

| Metric | Value | Comment |
|---|---|---|
| MAE | 16.35 | |
| RMSE | 25.75 | |
| R² | 0.8912 | |
| MAPE | 57.28% | honest — near-zero-demand hours dominate APE |
| 95% CI coverage | 96.8% | residual ≈ N(μ=−1.51, σ=25.70), near-unbiased |

**Why one task type resisted prediction — a statistical root-cause audit.** Batch-inference
demand forecast poorly. Instead of hiding it, the paper diagnoses it across six
dimensions: variance share in the loss (2.35% vs 97.30% for AI training), SNR
(−10.87 dB), ACF(1) = 0.038, flat spectrum (top-3 peak energy 1.45%), and
approximate entropy 1.865 — i.e., the series is statistically close to
unpredictable noise, which is a property of the demand, not a modeling failure.
This diagnosis is what lets the scheduler treat batch-inference forecasts as
low-confidence and carry safety margins instead of trusting point predictions.

Top features: Lag 48 and Lag 24 (strong daily periodicity) + task-type aggregate
load — confirming GPU demand is dominated by macro-structural periodicity.

## Act 2 — Carbon-aware multi-region scheduling

Composite carbon-cost matrix **C(r, h)** encodes regional electricity price, carbon
intensity, and renewable output into one decision index; a local search over the
Act-1 baseline schedule mines the **Pareto front under four weight profiles**.
Renewable-priority corner of the front: operating cost **−93.1%** (¥78.7k → ¥5.4k),
carbon **−93.2%** (112.38 → 7.66 tCO₂), peak grid purchase **−84.8%** (39.37 → 5.99 MW).

## Act 3 — Storage-coupled MILP

Computing–storage–electricity co-optimization: charge/discharge, buy/sell, and
wind/solar curtailment as decision variables; peak and fluctuation objectives made
tractable via **Epigraph transformation + L1-norm linearization**. Within the data and
model boundary: carbon 8.85 → 0.00 tCO₂, peak purchase 31.6 → 0 MW, load fluctuation
−99.8%, net revenue +9.4%. The paper states explicitly this is a boundary-condition
result, not a claim of lifecycle zero-carbon storage.

## Act 4 — Real-time re-scheduling under wind volatility

Two-stage structure: an **offline backbone schedule** + **MPC rolling-horizon**
fine-tuning, weighted by AHP–entropy combined assignment and solved with a directed
neighborhood search. Sensitivity analysis shows a **24 h prediction window is the
engineering-optimal compromise**: vs. the global exact MILP solution the two-stage
scheme costs only **+1.16%** but cuts solve time **1420 s → 10.8 s** (131×), making
real-time response feasible.

## Why this repo exists

The paper is published as-is for transparency; the competition code is not included
because it was written under contest conditions on private data. The methodology —
especially the honest-MAPE forecasting table, the six-dimension unpredictability
audit, and the 1.16%-optimality-cost / 131×-speedup MPC trade-off — is fully
reproducible from the paper's specifications.

## Repo structure

```
paper/CCM2601062.pdf   # full competition paper (Chinese, 34 pages)
README.md
LICENSE                # Apache-2.0
```

## Credits

Team of three (participation code CM2601062). Organized by the Huashu Cup
organizing committee (华数杯全国大学生数学建模竞赛). Related forecasting work on
honest validation design: [Beyond R²](https://github.com/ZhenruiWeng/beyond-r2-bmw-price-pipeline).

## License

Apache-2.0. The paper is shared for reference; competition problem data belongs to
the organizers.
