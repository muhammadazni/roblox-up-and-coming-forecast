# Roblox Up-and-Coming top 10 — Markov rank-transition forecast

Forecasts Roblox's Up-and-Coming chart place by place for the 72 hours after
entries close, from the chart's history in `/input/history.ndjson`.

## Model

Each game's rank follows a Markov chain over rank bins (places 1–10 each alone,
then 11–13, 14–17, 18–22, 23–28, 29–36, 37–48, 49–65, 66–97, and OUT), fitted
on every consecutive pair of reads in the history. The forecast is the mixture
of the chain's distributions over the 72 hourly horizons of the scored window
(horizons read from `task.json`, never hard-coded), sharpened (gamma 1.5,
renormalised per place) with a per-place unnamed residual left for games the
history never saw: 0.25 at places 1–2, 0.2 at places 3–5, 0.15 at places 6–10.
Places 1–2 get more residual because the model is overconfident on the last
chart's #1/#2 and brand-new climbers land there unnamed; places 6–10 are
already covered by 48 named games each. Where `details.created` is present (the final's
house reads), a game stops contributing past created + 28 days, the chart's
age limit. The top 10 orders games by the place where they hold their highest
probability (modal place).

Why this model: on the practice history only rank and playing are measurable
(the archive reads carry no votes, genres or details), and rank is by far the
strongest predictor. Playing growth has no predictive power, and per-game
momentum (rank velocity, 24h growth) helps the next read but hurts a 72h
occupancy forecast (mean reversion), so the model uses rank transitions only.
The transition matrix is smoothed toward a Gaussian kernel over bin centres
and powered by the history's median read gap, so it adapts to hourly final
reads as well as the archive's ~6h gaps.

## Validation (local scorer, RULES.txt)

- Practice window: **log score -32.993** (typical p 0.037 per place),
  top-10 hits 7.3 and agreement 55.6 per read — vs house baseline
  -43.512039 (6.4 hits / 53.5) and house model -37.2.
- Second window (archive snapshots after the practice window: fit on 44 reads,
  score 6 reads Nov 10–12): **-35.64**.
- Rolling 72h splits (fit on reads[0..upto], score the next ~10 reads):
  mean -37.82.
- Walk-forward over the full 50-read history (12 splits, fit on reads[0..upto],
  score the next 6 reads): mean **-34.80**, winning 8/12 splits.
- v1 (uniform residual 0.2) scored -33.20 / -35.58 / -37.76 / -34.86 on the
  same four checks. The per-place residual is a small but consistent win;
  larger changes tried and rejected on these same checks: gamma 1.75, a
  trending-top10 boost, recency-weighted counts, per-place gamma, horizon
  reweighting, an occupancy-table blend, a uniform tail floor, and a
  direction-state (2nd-order) chain — all improved the practice window alone
  or nothing, and lost on the second window / walk-forward.
- Deterministic: two runs are byte-identical. Runs in ~0.4s on a
  final-like 82-read hourly history, far inside the 300s limit.

## Run

```sh
mkdir -p out && MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out node main.ts
```

Node 22, standard library only, no network, no randomness.
