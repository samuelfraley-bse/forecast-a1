# Professor's Forecasting Pipeline — Session 3 Reference

This document summarizes the forecasting pipeline taught in session_3. Use it as a reference when building the project pipeline to avoid methodological drift.

---

## 1. Data Setup

- Panel data indexed by `[isocode, period]` where `period` is integer YYYYMM.
- Index must be sorted monotonically increasing — the `FeatureEngineer` class asserts this and will fail otherwise.
- Group-level operations always group by `isocode`.

---

## 2. Feature Engineering (`FeatureEngineer`)

Initialized with `groupby_cols` (typically `'isocode'`). All methods take the full panel DataFrame, a target column name, and a list of parameter values. They return the original DataFrame with new columns appended.

### Continuous Features

**Lags** — `fe.lag(df, y_col, lags=[1, 3, 6])`
- Creates columns `{y_col}_basic_lag{n}` for each lag.
- NAs appear at the start of each entity's history (expected).

**Rolling sum** — `fe.rolling_sum(df, y_col, windows=[1,3,6], closed=None, return_logs=False)`
- Creates columns `{y_col}_rolling_sum{w}`.
- `min_periods=1` always — only 1 observation required to compute.
- `closed='left'` excludes the current observation; use this when the current period's value is the target or would cause leakage.
- `return_logs=True` stores `ln_{col}` and drops the raw version.

**Rolling mean** — same signature as rolling sum, produces `{y_col}_rolling_mean{w}`.

**Rolling min / max / std** — same pattern.

**Weighted rolling sum/mean** — `fe.weighted_rolling_sum(df, y_col, windows, alpha=0.8)`
- Applies exponential decay weights within each window.
- Higher `alpha` discounts older observations more aggressively.
- Weights are normalized so they sum to 1 within each window.

### Discrete / Threshold Features

**Since** — `fe.since(df, y_col, thresholds=[0, 10, 100], shift_knowledge=None)`
- For each threshold, counts the number of periods since `y_col` last exceeded it.
- Resets to 0 when the threshold is exceeded again.
- `shift_knowledge=1` shifts the result forward by one period (use when you don't observe the current period's value at prediction time).
- Returns only `y_col` and `{y_col}_since_{th}` columns.

**Ongoing** — `fe.ongoing(df, y_col, thresholds, shift_knowledge=None)`
- Sequential count of consecutive periods for which `y_col` has exceeded the threshold.
- Resets to 0 when the threshold is no longer exceeded.
- Same `shift_knowledge` parameter as `since`.
- Returns only `y_col` and `{y_col}_ongoing_{th}` columns.

> **Note on NAs in since/ongoing:** The threshold condition uses `>`, so NAs are treated as 0/False. Be deliberate about this when your data has missing values.

---

## 3. Text Features (LDA Stock Topics)

- Raw topic shares from LDA are converted to **stock topics** using a stocks-and-flows formula (geometric decay accumulation over history).
- Stock topics smooth out month-to-month noise and capture the accumulated informational weight of each topic up to each period.
- The pre-built `stock_topic_*` columns in `topics.csv` already have this transformation applied.
- These are merged into the feature set alongside the violence/conflict history features.

---

## 4. Target Variable

- **Incidence:** binary indicator — did the event occur this period?
- **Onset:** binary indicator — did the event begin this period (incidence=1 and previous period incidence=0)?
- Both are classification targets.
- The class task uses a **3-month forecasting horizon** as the baseline, with `gap = horizon - 1` in PanelSplit.

---

## 5. Cross-Validation with PanelSplit

```python
from panelsplit.cross_validation import PanelSplit
from panelsplit.application import cross_val_fit_predict

ps = PanelSplit(periods, n_splits, test_size=1, gap=gap)

preds, fitted_estimators = cross_val_fit_predict(
    estimator=model,
    X=X,
    y=y,
    cv=ps,
    method='predict_proba',
    drop_na_in_y=True
)
```

- `test_size=1` → rolling one-step-ahead pseudo-out-of-sample predictions.
- `gap` encodes the forecasting horizon: `gap = h - 1` (e.g. 3-month horizon → `gap=2`).
- `method='predict_proba'` required for classification; predictions are the probability of the positive class (`preds[:, 1]`).
- `drop_na_in_y=True` drops rows where the target is NaN before fitting — check the source to understand what is dropped.
- Use `ps.gen_test_labels(df)` to build a results DataFrame aligned to the test periods across all splits.

---

## 6. Mandatory Feature Set (Class Task Baseline)

The professor specifies this minimum feature set for the session task:

1. Rolling mean of the target for windows `[1, 3, 12, 36, 60]` (or weighted rolling mean).
2. `since` — months since the variable last exceeded 0.
3. `ongoing` — consecutive months the variable has exceeded 0.
4. LDA stock topics from `topics.csv`.

---

## 7. Model

Default model specified by the professor:

```python
from sklearn.ensemble import RandomForestClassifier

rf = RandomForestClassifier(
    max_depth=4,
    max_features=0.2,
    min_samples_leaf=100,
    random_state=42
)
```

- No hyperparameter tuning expected.
- Separate runs for: all features, history-only features, text-only features.
- Bonus: repeat for a 12-month horizon.
