# Washing-machine disaggregation on REFIT (Flock Energy assignment)

Estimate a household's washing-machine power (W, 1-minute resolution) from the whole-house
aggregate signal alone, using the REFIT dataset (20 UK households, Loughborough area, 2013 to 2015).

## What was built

| Part | What | Where |
|---|---|---|
| Section 1 | Inspection and cleaning of RAW House 1 (1-min resample, duplicates, gaps, outliers, stuck values) | `refit-datacleaning.ipynb`, findings in `DATA.md` |
| Section 2 | Washing-machine EDA on the official cleaned data, 7 houses: cycle detection, usage patterns, 3 required plots | `refit-eda-and-model.ipynb`, results in `RESULTS.md` Part A |
| Section 3 | Disaggregation: 2 baselines, LightGBM regression, LightGBM two-stage, household-wise validation, failure analysis | `refit-eda-and-model.ipynb`, results in `RESULTS.md` Part B |
| Section 4 | Utility-facing summary, opportunity, limitations | `Recommendation.md` |

Headline result (Houses 7 and 10, never used for any choice): two-stage model F1 0.55 pooled
(always-off 0.00, hour-of-day profile 0.25, regression 0.48), pooled energy error +7% but per-house
-6% to +17%. Cycle-level precision is about 0.5 and the model predicts about 1.8 times too many
cycles. It beats the baselines but is not accurate enough for per-cycle decisions; see `RESULTS.md`
and `Recommendation.md`.

## Repository layout

```text
├── README.md                          # Project overview, setup, methodology, and findings summary
├── DATA.md                            # Section 1: Data coverage, quality findings, and cleaning decisions
├── RESULTS.md                         # Section 2 (EDA) and Section 3 (Disaggregation & failure analysis) write-up
├── Recommendation.md                  # Section 4: Utility-facing summary, opportunity, and limitations
├── refit-datacleaning.ipynb           # Section 1 notebook: RAW House 1 inspection and cleaning pipeline
├── refit-eda-and-model.ipynb          # Sections 2 & 3 notebook: Multi-house EDA, cycle detection, and LightGBM models
└── outputs/
    ├── section1/                      # Cleaned House 1 outputs and diagnostic plots
    │   ├── before_after_2day.png
    │   ├── full_range_aggregate.png
    │   ├── raw_interval_hist.png
    │   ├── section1_findings.json
    │   └── house1_clean_1min.csv/
    │       └── house1_clean_1min.csv
    ├── section2/                      # Washing-machine EDA summary and plots across 7 houses
    │   ├── cross_house_comparison.png
    │   ├── example_cycles.png
    │   ├── trace_24h.png
    │   ├── usage_hour_dow.png
    │   └── washing_machine_summary.csv
    └── section3/                      # Disaggregation model artifacts, test results, and failure plots
        ├── actual_vs_pred_test.png
        ├── failure_false_alarm.png
        ├── failure_missed_cycle.png
        ├── test_cycle_metrics.csv
        ├── test_results.csv
        ├── window_ablation_val.csv
        └── models/
            ├── lgbm_regressor.joblib
            ├── lgbm_regressor.txt
            └── model_meta.json
```



## Data

- REFIT, University of Strathclyde, CC BY 4.0. Murray et al. (2015), "A data management platform for
  personalised real-time energy feedback", 8th Int. Conf. on Energy Efficiency in Domestic Appliances
  and Lighting.
- Raw: `RAW_House1_Part1.csv`, `RAW_House1_Part2.csv`. Cleaned: `CLEAN_House{N}.csv`.
  Active power in Watts only, nominal 8 s sampling (irregular).
- Files used in this work were read from a Kaggle mirror of the REFIT files
  (`/kaggle/input/datasets/valorantphoenix/...`), not downloaded from the official archive.
  The official archive is the reference source. `[FILL: confirm or change this line]`
- Coverage, quality findings and cleaning decisions are in `DATA.md`.

## Setup and how to run

Developed on Kaggle notebooks (Python 3, pandas, numpy, matplotlib, lightgbm; statsmodels for the
optional correlation plot). `[FILL: add exact versions if you want, e.g. pip freeze output]`



1. Put the raw House 1 files and the cleaned REFIT files where the notebooks expect them
   (paths are set in the first config cell of each notebook).
2. Run the Section 1 notebook top to bottom. The overlap-removal and stuck-value cells must run
   before the plots and the save cell. Output goes to `outputs/section1/`.
3. Run the Section 2/3 notebook top to bottom:
   - Section 2 loads each house once and caches a 1-minute pickle per house. The first pass is slow
     (each cleaned file is 250 to 425 MB); later runs read the cache.
   - Section 3 then trains and evaluates. The test cell must be run once, after the model choice is fixed.
4. Runtime on Kaggle: `[FILL]`.

Reproducibility notes: random seed 42 for subsampling and LightGBM. Results can vary slightly with
library versions and thread counts. `[FILL: run the notebook once end to end from a fresh kernel and
confirm the numbers match RESULTS.md before submitting.]`

## Preprocessing decisions (details in DATA.md and RESULTS.md)

- Time: Unix column treated as UTC and converted to Europe/London for hour-of-day features. In the raw
  files the `Time` column appears to be UK local time labelled as UTC (the same sample is one hour earlier
  in the cleaned file); the cleaned `Unix` was verified by the night-trough / evening-peak shape of mean
  aggregate load by local hour.
- Raw House 1 (Section 1): sorted; Part1 rows overlapping Part2 (from 2014-06-27) dropped because Part1
  stores unavailable sensors as 0 and Part2 as NaN; remaining duplicate timestamps averaged; values
  below 0 W, above 15 kW (aggregate) or above 4 kW (appliance) set to NaN, not clipped; 1-minute mean;
  gaps up to 5 minutes interpolated, longer gaps left empty and flagged; runs of 60 minutes or more of an
  identical value above 100 W set to NaN (a logger freeze found on a plot).
- Cleaned data (Section 2/3): same range limits and stuck-value rule applied again because the official
  files forward-fill missing values.
- `Issues` flag: not used to remove minutes. It fires in 12 to 30% of washer-on minutes versus 2 to 7.5%
  overall, so masking it would cut cycles apart (first attempt: 8 to 89 complete cycles per house instead
  of 238 to 820). Reported as sensitivity instead.
- Cycle rule: washer above 10 W is active; pauses up to 15 minutes merged; a cycle needs at least 10 minutes
  and 20 Wh; cycles touching invalid data are censored for duration and energy statistics.
- Houses: 7 of the 19 houses with a washer channel were analysed (1, 2, 3, 5, 7, 9, 10), chosen for
  loading-time reasons, not at random. Houses 4, 13 and 12 need special handling or have no washer.

## Validation design

- Household-wise split. Train: Houses 1, 2, 3, 9. Validation: House 5. Test: Houses 7, 10.
- Choices (window of +/-15 minutes, 150 trees, 25 W cut-off, two-stage probability threshold 0.30,
  OFF-minute weighting) were made with leave-one-house-out predictions on the five non-test houses.
  The threshold rule (highest mean F1 with mean SAE <= 0.5) and the choice of the two-stage model as primary
  were fixed before the test run. Final models were trained on all five non-test houses.
- Why not a random split: lag, lead and rolling features overlap in time, so neighbouring minutes in
  train and test would leak.
- Features come from the aggregate only; no appliance channel is an input. The model is offline (uses
  future minutes too), so it is not a real-time method.
- Residual leak risks: all houses share the same 2013 to 2015 period; only 2 test houses.
- Sanity checks (non-test houses): a deliberate leak feature gives F1 0.82 (pipeline works); a hand-written
  rule for the 2 kW burst has F1 0.18 at 500 W (the learned model reaches 0.28).

## Trade-offs and what did not work

- LightGBM (window features) over a neural network (Seq2Point): faster, easier to explain, easier to debug
  in 4 to 6 hours. A Seq2Point model was not tried.
- Minute-level F1 stayed near 0.3 across window sizes, OFF-minute weighting and early-stopping metric on
  leave-one-house-out data; the two-stage model improved it to about 0.34 there and 0.55 on the test houses.
  Test F1 is higher partly because the washer is on more often in the test houses (8.5% and 9.8% of the
  time versus about 3.9% in the training houses); do not read it as better generalisation.
- Two-stage versus regression: two-stage has better precision, F1, SAE and cycle precision; regression has
  better recall and better F1 at 500 W (heating phase). Both are reported.
- A hypothesis that low F1 came from OFF-minute subsampling was tested and rejected (weighting changed the
  precision/recall balance, not F1). A hypothesis that the model catches heating bursts but misses the tail
  was also rejected (F1 at 500 W was lower, not higher).
- The within-house time-split check (to separate "new house" error from "hard signal") was not run, so
  the two are not separated.

## Limitations

See `RESULTS.md` (failure analysis) and `Recommendation.md`: overlapping appliance signatures, missing data,
household differences, small number of houses, and UK-to-India transfer.

## AI tools used

Claude (Anthropic) was used throughout: explaining the assignment, drafting and debugging the code for
all three sections, suggesting diagnostics (duplicate/overlap check, stuck-value rule, Issues-flag analysis,
leave-one-house-out, leak and rule-based sanity checks), and drafting `DATA.md`, `RESULTS.md`,
`Recommendation.md` and this README.

What I did myself: Most of the code was written by me, ran every notebook cell on Kaggle with the real data;
inspected all plots and compared claims in the write-up to the printed outputs; decided the split,
thresholds and which model to report; edited the text.


## Possible improvements

- More houses for training and testing (6, 8, 11, 15 to 21), and cross-validation across more test houses.
- Run the within-house time-split check to separate household-generalisation error from signal difficulty.
- Features aimed at the washer's heating burst (plateau of about 1.7 to 2.5 kW lasting 8 to 15 minutes, step sizes)
  and post-processing to drop very short isolated predicted cycles.
- Sequence models (Seq2Point) for comparison, with the same household-wise protocol.
- Resample to 15 and 30 minutes and re-run to measure what is lost at coarse smart-meter resolution.
- Probability calibration for the classifier (the best threshold was 0.30, below 0.5, partly because of
  OFF-minute weighting; not checked).
- Per-house calibration using a few weeks of labelled data for a customer.


