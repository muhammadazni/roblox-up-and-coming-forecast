# Roblox Up-and-Coming top 10 — Markov rank-transition forecast

Forecasts Roblox's Up-and-Coming chart place by place for the 72 hours after
entries close, from the chart's history in `/input/history.ndjson`.

## Model

Each game's rank follows a Markov chain over rank bins (places 1–10 each alone,
then 11–13, 14–17, 18–22, 23–28, 29–36, 37–48, 49–65, 66–97, and OUT), fitted
on every consecutive pair of reads in the history. The forecast is the mixture
of the chain's distributions over the 72 hourly horizons of the scored window
(horizons read from `task.json`, never hard-coded), sharpened (gamma 1.5,
renormalised per place) with a 0.2 unnamed residual left at every place for
games the history never saw. Where `details.created` is present (the final's
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

- Practice window: **log score -33.199268** (typical p 0.036 per place),
  top-10 hits 7.3 and agreement 55.6 per read — vs house baseline
  -43.512039 (6.4 hits / 53.5) and house model -37.2.
- Rolling 72h splits (fit on reads[0..upto], score the next ~10 reads):
  mean -37.76.
- Deterministic: two runs are byte-identical. Runs in ~0.5s on a
  final-like 82-read hourly history, far inside the 300s limit.

## Run

```sh
mkdir -p out && MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out node main.ts
```

Node 22, standard library only, no network, no randomness.
