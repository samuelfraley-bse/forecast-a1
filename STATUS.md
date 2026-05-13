# Status

## Current Focus

We are defining and tuning binary spike/onset target variables for sovereign 10-year bond yields in the panel dataset `df`.

Relevant columns in the working notebook/dataframe:

- `isocode`
- `date`
- `yield_10y`

The immediate goal is to create a defensible event target for later modeling, using only backward-looking information and encoding **onsets** rather than persistent incidence.

## What Has Been Implemented In The Notebook

Notebook cells were drafted to:

1. Define a `THRESHOLDS` dictionary for easy retuning.
2. Compute candidate intermediate signals by country:
   - `delta_raw`
   - `delta_pct`
   - `rolling_var_12`
   - `dev_from_mean_12m`
   - `dev_from_mean_6m`
   - `dev_from_mean_3m`
3. Convert those signals into spike flags using thresholds.
4. Convert spike flags into **onset** variables using the per-country rule:
   - `onset = spike & ~spike.shift(1).fillna(False)`
5. Produce a comparison summary table and bar chart across candidate targets.
6. Draft a tuning cell focused on two simpler candidate rules:
   - deviation from trailing 12-month mean
   - month-over-month raw change

## Key Method Decisions

### 1. Use Onsets, Not Incidence

The target should mark the **first month of a spike episode**, not every month during the episode.

Implication:

- If a spike lasts 3 consecutive months, only month 1 receives `1`
- Months 2 and 3 receive `0`

This avoids overweighting long episodes and is better aligned with event modeling.

### 2. No Lookahead Leakage

All transformations are grouped by `isocode`.

All rolling means used for deviation targets are shifted by one period so the current month is excluded from the baseline:

- trailing 12m mean uses `shift(1).rolling(...)`
- trailing 6m mean uses `shift(1).rolling(...)`
- trailing 3m mean uses `shift(1).rolling(...)`

This preserves a proper forecasting/event-detection setup.

### 3. Interpret `yield_10y` In Percentage Units

`yield_10y` is already stored in percent units, e.g. `4.7` means `4.7%`.

Because of that, the deviation thresholds are interpreted in **percentage points**, not percent growth.

Example:

- trailing 12m mean = `4.20`
- current yield = `5.05`
- deviation = `0.85`

If `dev_12m` threshold is `0.75`, then this counts as a spike because the current yield is `0.75` percentage points above its trailing 12-month average.

## Candidate Definitions Considered

The broader candidate set currently includes:

- raw month-over-month change: `delta_raw`
- percent month-over-month change: `delta_pct`
- trailing variance: `rolling_var_12`
- deviation from trailing means: 12m, 6m, 3m

## Current Preference / Direction

The current discussion suggests narrowing attention to two simpler and more interpretable definitions:

### Option A: Level Relative To Recent History

Flag a spike when:

- `yield_10y - trailing_12m_mean > threshold`

Interpretation:

- the current sovereign yield is materially elevated relative to its recent one-year baseline

Why this is attractive:

- easy to explain
- uses level information
- robust to low-base distortions that can affect percent changes

### Option B: Sudden Monthly Jump

Flag a spike when:

- `yield_10y(t) - yield_10y(t-1) > threshold`

Interpretation:

- the yield jumped sharply this month

Why this is attractive:

- very simple
- directly captures abrupt moves

## Decision Leaning

At this stage, the most promising simple rule appears to be:

- deviation from trailing 12-month mean in **percentage points**

The raw month-over-month change rule is also being kept as a comparison benchmark and tuning candidate.

The percent-change rule is currently viewed as less appealing for bond yields because percentage changes can look artificially large when yields are starting from low levels.

The rolling variance rule is viewed as more of a volatility-regime indicator than a clean spike-event definition.

## Thresholds Discussed So Far

Initial defaults used in the drafted notebook logic:

```python
THRESHOLDS = {
    "raw": 0.5,
    "pct": 10.0,
    "var_q": 0.90,
    "dev_12m": 0.75,
    "dev_6m": 0.75,
    "dev_3m": 0.75,
}
```

Interpretation of the current leading candidate:

- `dev_12m = 0.75` means the current yield must be more than `0.75` percentage points above the trailing 12-month mean

## Tuning Work In Progress

A tuning notebook cell was proposed to sweep thresholds for:

1. `dev_from_mean_12m`
2. `delta_raw`

Example threshold grids under consideration:

- deviation from 12m mean: `0.25, 0.50, 0.75, 1.00, 1.25, 1.50`
- raw MoM change: `0.10, 0.25, 0.50, 0.75, 1.00`

For each threshold, the summary would compare:

- total onset events
- onset rate
- countries with at least one onset
- median episodes per country

This is intended to help choose a threshold that is rare enough to represent meaningful stress, but common enough to support later empirical analysis.

## Open Questions

1. What threshold for `dev_from_mean_12m` gives the right balance between interpretability and event prevalence?
2. What threshold for `delta_raw` gives a useful benchmark for abrupt jumps?
3. Should the final target use:
   - just `dev_from_mean_12m`
   - just `delta_raw`
   - or a hybrid rule requiring both elevation and jump?

## Recommended Next Step

Run the tuning cell for the two preferred rules and inspect event prevalence before locking the final target definition.

Most likely decision path:

1. tune `dev_from_mean_12m`
2. tune `delta_raw`
3. compare prevalence and country coverage
4. choose one final onset target, or define a hybrid if needed

## Current Repo Note

This status file documents notebook-level analytical progress and design choices. The notebook cells have been drafted in chat for manual paste/implementation.
