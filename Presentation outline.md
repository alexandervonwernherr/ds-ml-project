I would say we start with the presentation:
1. This is the data and the case. This is the overview of the data that we did: the EDA. We see that it's an imbalanced dataset and that these are the features that we had.
2. With these, we did the baseline model, this LGBM, as was suggested in the challenge description, and we got this result. Our goal was to improve this baseline. I think that's the second part.
3. We're saying, how did we do it? First of all, we did the grid, like what Alex did. We did some hyperparameter tuning, and we didn't see much improvement, so we went into feature engineering. We kind of discussed what was there, what the potential fraud cases are, and talked to Claude. We came up with this: it's not the actual features that predicted, but the changes between the features. We came up with these different features, and this is how they improved the model. What we can do is, I think it's maybe too much, but we can say, "Okay, only one of these features is the improvement," if both features were the improvement, just to show the KPIs. In the end, we go to this result, and then you kind of tell a consistent story, for example. I would do something like that.

## Main story: "The model got better when we looked at changes, not levels."

1. **The case.** Utility company, meter fraud, and what a missed fraud costs. Say who the audience is and what decision this supports. The expected output is fraud probability predicted per client.
2. **The data.** Two tables (clients and invoices) and what each contains and how they link together. + First thoughts on what datapoints can be indicative of fraud (region, consumption)
3. **EDA**: fraud is rare. Use the pie chart from your notebook (pie chart per client). Because so few clients are fraud, "accuracy" would be misleading (give the accuracy of predicting 100% no fraud), so we judge by AUC (how well the model ranks risky clients above safe ones). Explain how AUC works.
4. **Baseline**. The LGBM from the challenge description based on just the existing data without feature engineering, plus its score. This is the number to beat.
5. **Attempt 1: tuning the settings**. A grid of 36 combinations, and little improvement. This justifies moving on.
6. **Attempt 2: feature engineering to maximise fraud detection.** 
	1. What we found out - 
		1. remarque column is indicative of fraud (value assigned during the meter reading) -> @Peter to add the section with the table to the notebook
		2. Fraudsters tamper with meters, so what matters is **change in behaviour** not static values.
			1. a switch of tariff (and how many times it happened, showing the consumption pattern)
			2. index going backwards (when new index measurement is lower than the old one)
			3.  consumption drop between earlier and later invoices
	2. Correlation matrix across all features + decision to drop unnecessary features
	3. Slide structure - feature, explanation + code snippet + visualisation (table with quick heuristic of fraud rate by feature group e.g. share fraudsters by number of tariff switches)
7. **Results:** the KPI ladder. One bar chart: baseline → tuned → plus each feature group → final. This is your "which feature actually helped" slide, and it keeps the story consistent. I'd show each group added separately and then combined. We need to show both training and the test results to see if we overfitted or under-fitted. Explain the AUC metric improvement from the baseline and what it means.
8. **Recommendations**, data product, and next steps. Use the confusion matrix to explain the trade-off: inspections caught versus customers wrongly flagged. The data product is a ranked weekly inspection list. Future work: more data, other algorithms, threshold tuning.

Backup slide: the feature list and tuning grid, in case someone asks.

## Final features and numbers (full run of the notebook on main, after PR #9)

**Features used (25 in total)**
- Client table (3): `client_catg`, `region`, `creation_date`
- Raw invoice columns, averaged per client (11): `counter_statue`, `counter_code`, `reading_remarque`, `counter_coefficient`, `consommation_level_1` to `_4`, `old_index`, `months_number`, `counter_type`
- Engineered features (11): `mean_diff_index`, `n_tarif_types`, `n_counters`, `n_invoices`, `cons_mean`, `cons_std`, `cons_min`, `cons_zero_share`, `n_negative_diff`, `share_negative_diff`, `cons_change`
- Left out: `tarif_type` (99.6% correlated with `counter_type`), `new_index` (99.3% with `old_index`), `disrict` (coarser than `region`). We also tried dropping `counter_code` and `counter_statue`, but about 800 more honest clients were wrongly flagged, so they stay.

**Data (slides 2 and 3):** 135,493 clients, 4,476,749 invoices, median 30 invoices per client. Fraud rate 5.58%. Always predicting "no fraud" is 94.4% accurate. Test set: 1,892 fraud and 31,982 honest clients.

**Score ladder (slides 4, 5, 7):** AUC on the same client split.

| Model | Train AUC | Test AUC |
|---|---|---|
| 1. Baseline (raw data, default settings) | 0.8716 | 0.7962 |
| 2. Baseline + tuned settings (36 combinations) | 0.8708 | 0.7966 |
| 3. Engineered features, default settings | 0.9160 | 0.8726 |
| 4. Engineered features + tuned settings | 0.9174 | 0.8718 |

Tuning moves the score by 0.0004. Feature engineering adds about 0.076 test AUC. Train is higher than test by about 0.05 for the engineered models and 0.075 for the baseline, so there is some overfitting but no collapse.

**Inspections (slide 8):** same final model, three ways to flag clients, test set.

| Setup | Fraud caught | Recall | Precision | Honest wrongly flagged |
|---|---|---|---|---|
| Model 4, cut-off 0.5 | 179 | 9.5% | 58.9% | 125 |
| + fraud weighting, cut-off 0.5 | 1,441 | 76.2% | 19.3% | 6,030 |
| + weighting, cut-off 0.547 (chosen, target 70% recall) | 1,369 | 72.4% | 21.2% | 5,080 |

About 1 in 5 inspections finds fraud, against 1 in 18 at random (5.58%).

Caveats: each number is from a single run, so differences of a few dozen fraud clients caught are within run-to-run noise. The per-feature-group ablation is still not done, so the "which feature helped" split is not available yet. Earlier numbers from before the column drops (1,381 caught, 5,158 wrongly flagged) were close to these.

Gaps to fill before building slides
- [LATER]The beautified Marina notebook has no saved outputs, so there are no baseline or final numbers anywhere yet. Slides 4, 5 and 8 need a full run, and FAST_MODE is currently True, which only gives a quick sanity check. I'd run it with FAST_MODE = False.
- [NOW] Slide 8 needs the feature groups scored separately. The notebook doesn't do that yet, so it needs an ablation loop (add each group alone, then all together), about 15 minutes of work.
- [NOW] Your outline calls the baseline "LGBM as in the challenge". The notebook has no untuned baseline cell, so that needs adding as well.