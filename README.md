# Catching Meter Fraud

**Predicting which electricity and gas clients are tampering with their meters, so inspectors visit the riskiest clients first.**

Team project from the neuefische Data Science bootcamp, based on the Zindi challenge [Fraud Detection in Electricity and Gas Consumption](https://zindi.africa/competitions/fraud-detection-in-electricity-and-gas-consumption-challenge).

> Main finding: the model got better when we looked at **changes** in a client's behaviour, not at static **levels**. Tuning the model added almost nothing (+0.0004 AUC); new features describing change added +0.076.

## The problem

A utility loses revenue when clients tamper with their meters: energy is used but never billed. Inspections cost staff and time, so the utility cannot visit everyone. Our output is a **fraud probability for every client**, which turns into a ranked inspection list.

## The data

| Table | Rows | One row per | Content |
|---|---|---|---|
| `client_train.csv` | 135,493 | client | region, category, creation date, fraud label (`target`) |
| `invoice_train.csv` | 4,476,749 | bill | meter readings, consumption levels 1 to 4, tariff, meter status, reader's remark |

The tables link on `client_id` (median 30 invoices per client). Only **5.58%** of clients are fraud, so a model that never flags anyone is already 94.4% accurate. That is why we judge models by **ROC AUC**: the chance that a random fraud client gets a higher risk score than a random honest one.

## Approach

1. **Baseline:** LightGBM with default settings on the raw data (client columns plus each invoice column averaged per client).
2. **Attempt 1, tuning:** grid search over 36 combinations (learning rate, number of trees, bin sample size) with 3-fold cross-validation.
3. **Attempt 2, feature engineering:** fraudsters tamper with meters, so we described *change* per client:

| Group | Features | Idea |
|---|---|---|
| Meter index | `mean_diff_index`, `n_negative_diff`, `share_negative_diff` | a meter index should never go backwards |
| Tariffs & meters | `n_tarif_types`, `n_counters`, `n_invoices` | more tariff switches, more fraud |
| Consumption | `cons_mean`, `cons_std`, `cons_min`, `cons_zero_share`, `cons_change` | zero bills and drops over time |

We dropped three near-duplicate columns from every model, baseline included: `tarif_type` (99.6% correlated with `counter_type`), `new_index` (99.3% with `old_index`) and `disrict` (a coarser `region`). Final model: 25 features.

## Results

All models are scored on the same stratified hold-out set of 33,874 clients (25%).

| Model | Train AUC | Test AUC |
|---|---|---|
| 1. Baseline (raw data, default settings) | 0.8716 | 0.7962 |
| 2. Baseline + tuned settings | 0.8708 | 0.7966 |
| 3. Engineered features, default settings | 0.9160 | 0.8726 |
| 4. Engineered features + tuned settings | 0.9174 | 0.8718 |

From scores to inspections (test set: 1,892 fraud, 31,982 honest clients). With fraud weighting and a cut-off of 0.547, chosen on training data to catch 70% of fraud:

| Fraud caught | Recall | Precision | Honest clients flagged |
|---|---|---|---|
| 1,369 | 72.4% | 21.2% | 5,080 |

About **1 in 5 inspections finds fraud**, against 1 in 18 when picking clients at random.

## Next steps

- Score each feature group on its own (ablation) to see which one carries the gain
- Add data such as past inspection results
- Try other algorithms such as XGBoost or CatBoost
- Calibrate the scores, since fraud weighting distorts probabilities

## Repository

| File | Description |
|---|---|
| [`fraud_detection_beautified_marina.ipynb`](fraud_detection_beautified_marina.ipynb) | **Final notebook**: data, baseline, feature engineering, tuning, results |
| [`Catching Meter Fraud Project.pptx`](Catching%20Meter%20Fraud%20Project.pptx) | Final presentation |
| [`Presentation outline.md`](Presentation%20outline.md) | Story of the presentation and the final numbers |
| [`Fraud_Detection_Starter.ipynb`](Fraud_Detection_Starter.ipynb) | Zindi starter notebook the baseline is based on |
| [`01_assignment.md`](01_assignment.md) | Bootcamp project brief |

## How to run

1. Install [uv](https://docs.astral.sh/uv/), then install the dependencies:

   ```bash
   uv sync
   ```

2. Download the data from the [Zindi challenge page](https://zindi.africa/competitions/fraud-detection-in-electricity-and-gas-consumption-challenge/data) and place the files like this (the data is not tracked in git):

   ```
   data/train/client_train.csv
   data/train/invoice_train.csv
   ```

3. Open `fraud_detection_beautified_marina.ipynb` in VS Code from the project root and select the `.venv` kernel. A full run takes several minutes because of the 36-combination grid search.

## Team

- [alexandervonwernherr](https://github.com/alexandervonwernherr)
- [curls-me](https://github.com/curls-me)
- [ParticlePeter](https://github.com/ParticlePeter)
- [rambabugupta](https://github.com/rambabugupta)
- [rouzbeh-enayatzadeh](https://github.com/rouzbeh-enayatzadeh)
