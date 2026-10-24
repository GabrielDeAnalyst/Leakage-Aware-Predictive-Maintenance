# Leakage-Aware Predictive Maintenance

Code and results for early failure detection on multi-machine industrial sensor data. A model predicts whether a machine will fail within the next 24 hours, using leakage-safe temporal features and validation that respects time and machine identity.

This repository accompanies the paper:

> Falade, Ogundapo, Olaogun, Ayoola & Omoniyi (2026). *Leakage-Aware Temporal Feature Engineering for Early Industrial Failure Detection: Chronological and Unseen-Machine Validation on Multi-Machine Sensor Data.* International Journal of Scientific Research and Management, 14(10), 3169-3181. https://doi.org/10.18535/ijsrm/v14i10.ec05

## Overview

Predictive maintenance models can look far better than they are when target-derived columns leak into the features or when validation ignores time. This project:

- audits the data for leakage, including remaining useful life and other target-derived columns
- builds lag, rolling and change features per machine from past and current observations
- validates chronologically with a rolling-origin design, a final held-out period, and a test on machines never seen in training
- measures which feature families actually help with an ablation study

## Dataset

File used: `predictive_maintenance_v3.csv`

- 24,042 observations and 15 columns
- 20 machines of four types: Pump, Robotic Arm, Compressor and CNC
- 1 to 14 January 2024
- Target: `failure_within_24h` (about 14.8% positive), covering 38 independent failure episodes
- Columns: `timestamp`, `machine_id`, `machine_type`, `vibration_rms`, `temperature_motor`, `current_phase_avg`, `pressure_level`, `rpm`, `operating_mode`, `hours_since_maintenance`, `ambient_temp`, `rul_hours`, `failure_within_24h`, `failure_type`, `estimated_repair_cost`

**Data availability:** `[state the source of the dataset and whether it is included in this repository]`

## Method

1. **Data audit:** missing values, duplicates, sampling intervals and failure episodes.
2. **Leakage audit:** `rul_hours`, `failure_type`, `estimated_repair_cost` and other target-derived columns are excluded from the predictors, and the lag and rolling features are checked for temporal validity.
3. **Feature engineering (per machine):** lag-1 and lag-6 values, timestamp-aware 1-hour rolling mean, standard deviation and maximum, and change and percent change for five sensors (vibration, motor temperature, phase current, pressure, rpm).
4. **Chronological split:** train on 1 to 10 January, validate on 11 to 12 January, and hold out 13 to 14 January.
5. **Rolling-origin validation:** four expanding-window folds over 5 to 12 January.
6. **Feature-family ablation:** raw context, then lag, then rolling, then change, then missingness indicators.
7. **Final model:** Random Forest on 36 features (raw, lag and rolling), decision threshold 0.333, with conformal prediction (alpha 0.10, 90% target coverage) that lets the model abstain when uncertain.
8. **Unseen-machine validation:** 15 machines for training, 5 completely unseen machines for testing.

## Results

**Feature-family ablation** (mean across four rolling-origin folds):

| Feature set | Features | ROC-AUC | PR-AUC |
|---|---|---|---|
| Raw context | 11 | 0.9460 | 0.8214 |
| + Lag | 21 | 0.9673 | 0.8916 |
| + Rolling | 36 | 0.9779 | 0.9227 |
| + Change | 46 | 0.9779 | 0.9229 |
| + Missingness indicators | 51 | 0.9778 | 0.9226 |

Lag and rolling features give the main gain. Change and missingness features add nothing, so the final model uses the 36-feature set.

**Unseen-machine validation** (5 held-out machines, 6,053 observations, 9.25% failure rate):

| Metric | Value |
|---|---|
| ROC-AUC | 0.9944 |
| PR-AUC | 0.9510 |
| Brier score | 0.0261 |
| Precision | 0.7712 |
| Recall | 0.9571 |
| Specificity | 0.9711 |
| F1 | 0.8542 |

Confusion matrix: TN 5,334, FP 159, FN 24, TP 536.

**Held-out 13 to 14 January** (3,515 observations, 80 positive rows): ROC-AUC, PR-AUC, precision and recall were all 1.000, with no false positives or false negatives.

## Repository structure

```
├── predictive_maintenance_leakage_aware.ipynb   # full analysis, code and outputs
├── requirements.txt
└── README.md
```

## How to run

**Google Colab (as written):** open the notebook in Colab, run the cells in order, and upload `predictive_maintenance_v3.csv` when the upload cell asks for it.

**Locally:**
1. `pip install -r requirements.txt`
2. In the notebook, replace the Colab upload cells (`files.upload()`) with `df = pd.read_csv("predictive_maintenance_v3.csv")`.
3. Run all cells.

## Limitations

- The data covers only 14 days, 20 machines and 38 independent failure episodes, so results may not carry over to longer horizons or other equipment.
- Consecutive rows inside a failure episode are strongly correlated, so row-level metrics overstate the amount of independent evidence.
- The 13 to 14 January test contains only 80 positive rows, and its perfect scores should not be read as expected real-world performance.
- The unseen-machine result comes from a single random split of machines (`random_state=42`).
- Repeating the unseen-machine test over several random machine splits would give a more stable estimate.

## Author

**Olaogun Gabriel Olatunji**
CS graduate and Research Assistant, LAUTECH
[GitHub](https://github.com/GabrielDeAnalyst)
