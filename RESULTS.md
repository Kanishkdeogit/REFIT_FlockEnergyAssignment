# RESULTS.md

Washing-machine EDA (Part A) and disaggregation model (Part B) on REFIT. All numbers come from the
notebook outputs saved in `outputs/section2/` and `outputs/section3/`. Statements worded as "likely",
"appears" or "hypothesis" were not verified against extra data.

---

## Part A: Washing-machine EDA (official cleaned REFIT, 1-minute resolution)

### Houses and periods used
Seven houses with a washing-machine channel: Houses 1, 2, 3, 5, 7, 9, 10. Selection rule: at least
30 complete cycles and at least 40% valid minutes (all seven pass). Houses were loaded for
loading-time reasons, not chosen at random: 7 of the 19 houses with a washer channel were analysed.
Not analysed: Houses 6, 8, 11, 15 to 21 (washer channel available, not loaded), House 4 (two washing
machines, needs separate handling), House 13 (washer channel changed on 25 Mar 2015), House 12 (no
washing machine), House 14 (not in the dataset).
Washer channels (REFIT README): H1 Appliance5, H2 Appliance2, H3 Appliance6, H5 Appliance3,
H7 Appliance5, H9 Appliance3, H10 Appliance5.

### Data quality and availability

| House | Period | Valid % | No-data % | Stuck % | Off time % | Cycles | Complete cycles | Cycles/week |
|---|---|---|---|---|---|---|---|---|
| 1 | 2013-10-09 to 2015-07-10 | 87.1 | 12.6 | 0.32 | 98.0 | 390 | 372 | 4.7 |
| 2 | 2013-09-17 to 2015-05-28 | 75.0 | 24.6 | 0.46 | 95.0 | 327 | 298 | 4.5 |
| 3 | 2013-09-25 to 2015-06-02 | 76.7 | 11.4 | 12.05 | 94.1 | 598 | 539 | 8.0 |
| 5 | 2013-09-26 to 2015-07-06 | 89.3 | 9.3 | 1.49 | 93.0 | 761 | 712 | 8.6 |
| 7 | 2013-11-01 to 2015-07-08 | 86.9 | 12.6 | 0.57 | 91.5 | 836 | 820 | 10.8 |
| 9 | 2013-12-17 to 2015-07-08 | 80.0 | 15.4 | 4.73 | 97.1 | 244 | 238 | 3.7 |
| 10 | 2013-11-20 to 2015-06-30 | 81.2 | 11.2 | 7.99 | 90.2 | 610 | 581 | 8.5 |

- Valid = aggregate and washer both present after short-gap interpolation (<= 5 min) and stuck-value
  removal. No-data = minutes with no row in the file. Stuck = >= 60 min of an identical value above 100 W.
  The cleaned files forward-fill missing values, so outages show up as flat plateaus; this rule only catches
  plateaus above 100 W, so outages at very low power are not detected.
- Off time = share of valid minutes with washer power <= 10 W (90 to 98%).
- Cycles/week is normalised by valid time, not calendar time.
- "Complete" excludes cycles whose neighbouring minute before the start or after the end is not valid
  (possibly cut off).
- Weakest data: House 2 (24.6% no data), House 3 (12% stuck), House 10 (8% stuck).
- Timezone: `Unix` is treated as UTC and converted to Europe/London. In the cleaned files the `Time` column
  equals the UTC conversion of `Unix`; the same raw sample is one hour later in the raw file, so raw
  timestamps appear to be UK local time labelled as UTC. Sanity check: mean aggregate load by local hour
  shows a night trough at about 3 to 5 h and an evening peak at about 17 to 19 h in most houses
  (House 1 has a high night baseline of about 600 W and a different shape).

### The REFIT `Issues` flag
`Issues = 1` (sum of sub-meters > aggregate) occurs 2 to 7.5% of the time overall but 12 to 30% of the
time while the washer is on (e.g. House 1: 2.1% overall vs 30.4% while on). A likely cause is that
unsynchronised sensors update at slightly different times when the 2 kW heater switches; this was not
verified. 87 to 98% of cycles contain at least one flagged minute. Discarding flagged minutes would
therefore remove active washer minutes and split cycles (first attempt: 8 to 89 complete cycles per
house instead of 238 to 820). We keep these minutes and report the share of affected cycles; Part B repeats
the test evaluation without them as a sensitivity check.

### Cycle detection rule (plain language)
A minute is "active" when washer power is above 10 W. Pauses of up to 15 minutes between active minutes are
treated as part of the same cycle (soak and rinse phases draw almost nothing). A cycle must last at least
10 minutes and use at least 20 Wh, which removes standby blips. Cycles that touch invalid data are marked
censored and excluded from duration and energy statistics.

Sensitivity of the 15-minute merge gap (gap tested from 5 to 60 min): median duration and energy barely change in
Houses 1, 7 and 10 (e.g. House 10: 120 to 121 min), so cycles are not being split by pauses. House 5 is sensitive
(median duration 42 to 78 min), probably because of long pauses inside programs or back-to-back loads, so
its duration and energy figures are approximate. 15 min was kept as the conservative choice. House 1 has a tight
mode at about 34 min (about 70% of its cycles last 32 to 36 min), which looks like a fixed short program, not a
fragmentation artefact; the program name is not in the data.

### Usage patterns (usage_hour_dow.png)
- Start times differ by house: House 7 has a large share of starts at about 7 to 8 h, House 10 mostly at about
  19 h, Houses 2, 3 and 5 mostly at about 8 to 10 h, House 9 is more spread out. House 1 also has a cluster at
  3 to 5 h that no other house shows; we did not investigate it.
- Day of week: roughly 30 to 37% of the cycles in Houses 5 and 9 start on a Saturday; House 3 is busiest on
  Monday and Tuesday and lowest at weekends; Houses 2 and 10 are roughly flat.
- Caveat: shares are not normalised for uneven valid coverage per weekday or per hour.

### Representative 24-hour trace (trace_24h.png)
House 7, 2015-04-18, the day with the most washer energy among days with more than 95% valid data
(a busy day, not a typical day). Four cycles are detected. The washer's 2 kW bursts are visible in the
aggregate, but other loads of similar size are present (about 2.5 kW flat pulses, a rapidly cycling 0.5 to
3 kW load in the evening, and a coincident load of about 2.3 kW around 08:10), so the washer cannot be read
off the aggregate by thresholding alone. This is the main difficulty for Part B.

### Detected cycles (example_cycles.png)
The median-energy cycle per house shows one or two heating bursts of about 2 kW lasting 8 to 15 min followed
by a long low-power tail (about 50 to 300 W: motor and pump). Most of the energy is in the heating phase; the
tail is small relative to the noise in the aggregate and is hard to recover from it.

### Cross-house comparison (cross_house_comparison.png)
Box plots (outliers hidden) of duration, energy and peak power per complete cycle. Medians by house:

| House | Duration (min) | Energy (Wh) | Peak (W) |
|---|---|---|---|
| 1 | 34 | 319 | 2325 |
| 2 | 101 | 639 | 2091 |
| 3 | 67 | 555 | 1922 |
| 5 | 47 | 288 | 2068 |
| 7 | 61 | 551 | 2083 |
| 9 | 74 | 667 | 2210 |
| 10 | 121 | 715 | 2053 |

House 10 is very consistent; Houses 5 and 7 are variable. Programs and machines differ a lot between
homes, so generalising to an unseen house is the hard case.

### How much context the model needs
For House 7 only, the cross-correlation between the detrended aggregate and washer power, and the
washer's own autocorrelation, both fall to near zero after about 15 to 20 minutes, consistent with
the 10-minute heating bursts. We therefore use dense lags and leads of +/-15 min and rolling statistics up to
60 min. This is a linear measure on one house and is only a rough guide; it did not drive model selection.

---

## Part B: Disaggregation model

### Task
Estimate washing-machine power (W, 1-minute resolution) from the household aggregate alone. The estimation is
offline (non-causal): features use both past and future aggregate minutes.

### Split and leakage control

| Set | Houses | Use |
|---|---|---|
| Train | 1, 2, 3, 9 | model fitting during development |
| Validation | 5 | development checks |
| Test | 7, 10 | evaluated once, after every choice was fixed |

- Splitting is by household, so no house appears in two sets and no overlapping windows cross the split.
  A random row split was rejected: neighbouring minutes share lag, lead and rolling features, so they would leak.
- Features are built from the aggregate only; no appliance channel is an input.
- Hyperparameters (window, trees, cut-off, probability threshold) were chosen with leave-one-house-out (LOHO)
  predictions on the five non-test houses (1, 2, 3, 5, 9). The final models were trained on all five.
  The test houses were not used for any choice.
- Residual risks: all houses cover the same 2013 to 2015 period, so shared seasonal effects are not isolated;
  only two test houses.

### Features (final: window +/-15 min, 53 features)
Aggregate at t; lags and leads of 1 to 15 min; centred rolling mean, std, min and max over 5, 15, 30 and
60 min; first differences; aggregate minus a 120-min rolling median baseline; local hour and day of week.

### Models and baselines
- **Always-off:** predicts 0 everywhere.
- **Hour-of-day profile:** mean washer power per local hour, learned on the training houses.
- **LightGBM regression:** direct power regression; predictions below 25 W are set to 0.
- **LightGBM two-stage (primary):** a classifier estimates P(washer ON, > 10 W); a regressor trained on ON minutes
  only estimates power; output = regressor power where P >= 0.30, otherwise 0.
- Training: 30% of OFF minutes kept for speed, compensated with sample weights (1/0.3 on kept OFF minutes);
  150 trees, learning rate 0.05, 63 leaves, min 50 samples per leaf, subsample 0.8, column sample 0.7, seed 42.
  No early stopping on held-out data in the final models.

### Metrics and why
The washer is ON only 4 to 10% of the time, so MAE alone is misleading: always-off has the best MAE in every table
below while detecting nothing. We report:
- precision, recall and F1 for ON/OFF at 10 W, plus F1 at 500 W (heating phase only);
- SAE = |predicted energy - true energy| / true energy, the energy a utility cares about;
- cycle-level recall, cycle precision and median absolute per-cycle energy error, applying the same cycle rule
  to truth and prediction.
SAE is unsigned, so pooled values can hide over- and under-prediction cancelling out; per-house energy totals
are given separately.

### How the final configuration was chosen (including what did not work)
1. First attempt: regression with early stopping on L1 and a window comparison on House 5. F1 was 0.268 to 0.277 for
   every window (0, 5, 15, 30 min), so the comparison on one house was not informative. ("0" still includes the
   rolling statistics.) Window +/-15 min was fixed from the correlation analysis, not selected by this ablation.
2. Re-weighting the down-sampled OFF minutes and switching the early-stopping metric to L2 changed the balance
   between precision and recall but not F1 (0.278 to 0.283 across windows); sampling was not the main cause of
   the low F1. The hour-of-day profile was close to "always on" (recall 0.79, precision 0.09).
3. At 500 W (heating phase), regression F1 on validation was 0.17, lower than at 10 W (0.28), so the model does
   not recover the 2 kW bursts well; our initial guess that only the low-power tail was missed was wrong.
4. LOHO on the five non-test houses (regression, window 15, 150 trees):

| Held-out house | Regression F1 | Hour profile F1 | Always-off F1 |
|---|---|---|---|
| 1 | 0.17 | 0.05 | 0.00 |
| 2 | 0.29 | 0.14 | 0.00 |
| 3 | 0.31 | 0.16 | 0.00 |
| 5 | 0.31 | 0.16 | 0.00 |
| 9 | 0.23 | 0.08 | 0.00 |
| mean | 0.26 | 0.12 | 0.00 |

   The 25 W cut-off was best on pooled out-of-fold predictions (F1 0.30). The cycle-level LOHO result for
   regression was: recall 0.93, precision 0.28, about 3 times as many predicted cycles as true ones, median
   per-cycle energy error 75%.
5. Two-stage model, probability threshold sweep on the five non-test houses (mean over held-out houses):

| Threshold | Mean F1 | Mean SAE | Mean signed bias | Cycle recall | Cycle precision | Per-cycle energy error |
|---|---|---|---|---|---|---|
| 0.10 | 0.32 | 1.65 | +1.65 | 0.86 | 0.26 | 45% |
| 0.15 | 0.34 | 1.13 | +1.10 | 0.80 | 0.30 | 54% |
| 0.20 | 0.35 | 0.79 | +0.66 | 0.75 | 0.33 | 60% |
| 0.25 | 0.35 | 0.59 | +0.32 | 0.70 | 0.36 | 65% |
| **0.30** | **0.34** | **0.44** | **+0.05** | 0.65 | 0.39 | 70% |
| 0.35 | 0.32 | 0.39 | -0.17 | 0.60 | 0.42 | 75% |
| 0.40 | 0.29 | 0.40 | -0.35 | 0.54 | 0.45 | 82% |

   Rule fixed before the test run: highest mean F1 subject to mean SAE <= 0.5, which selects 0.30. The two-stage
   model was declared the primary model before the test run; regression is reported as a comparison.

### Sanity checks (non-test houses, not part of the result)
- Leak check: adding the washer's own previous-minute power as a feature (a deliberate cheat, never reported as
  a result) gave F1 0.82 on the validation house, so alignment and metrics are working.
- Rule-based detector (aggregate 1.5 to 2.6 kW above baseline for most of 9 min): mean F1 at 500 W of 0.18,
  precision 0.14, below the learned regression model (0.28). Most such steps in the aggregate are not the washer.
- The within-house time-split check (to separate "new house" error from "hard signal") was not run, so these two
  sources of error are not separated.

### Test results (Houses 7 and 10, evaluated once)
Minute level, all valid minutes:

| House | Model | MAE (W) | SAE | Precision | Recall | F1 | F1 at 500 W |
|---|---|---|---|---|---|---|---|
| 7 | always-off | 38.0 | 1.00 | n/a | 0.00 | 0.00 | 0.00 |
| 7 | hour-of-day profile | 52.4 | 0.49 | 0.13 | 0.98 | 0.23 | 0.00 |
| 7 | LightGBM regression | 39.9 | 0.17 | 0.41 | 0.76 | 0.53 | 0.57 |
| 7 | LightGBM two-stage | 42.6 | 0.17 | 0.61 | 0.61 | 0.61 | 0.47 |
| 10 | always-off | 35.6 | 1.00 | n/a | 0.00 | 0.00 | 0.00 |
| 10 | hour-of-day profile | 50.2 | 0.45 | 0.15 | 0.95 | 0.26 | 0.00 |
| 10 | LightGBM regression | 40.3 | 0.30 | 0.35 | 0.56 | 0.43 | 0.48 |
| 10 | LightGBM two-stage | 41.2 | 0.06 | 0.57 | 0.43 | 0.49 | 0.43 |
| pooled | always-off | 36.8 | 1.00 | n/a | 0.00 | 0.00 | 0.00 |
| pooled | hour-of-day profile | 51.4 | 0.47 | 0.14 | 0.96 | 0.25 | 0.00 |
| pooled | LightGBM regression | 40.1 | 0.23 | 0.38 | 0.66 | 0.48 | 0.53 |
| pooled | LightGBM two-stage | 41.9 | 0.07 | 0.59 | 0.52 | 0.55 | 0.45 |

Washer ON share (power > 10 W) in the test houses: 8.5% (House 7) and 9.8% (House 10).

Energy over the valid test minutes (kWh, predicted vs true):

| | True | Regression | Two-stage |
|---|---|---|---|
| House 7 | 486 | 404 (-17%) | 569 (+17%) |
| House 10 | 407 | 285 (-30%) | 383 (-6%) |
| Pooled | 893 | 689 (-23%) | 951 (+7%) |

The two-stage pooled SAE of 0.07 is partly cancellation of a 17% over-prediction and a 6% under-prediction.

Cycle level (same cycle rule on truth and prediction):

| Model | House | True cycles | Predicted cycles | Recall | Precision | Median abs. energy error |
|---|---|---|---|---|---|---|
| regression | 7 | 836 | 1896 | 1.00 | 0.43 | 56% |
| regression | 10 | 610 | 1605 | 0.95 | 0.37 | 70% |
| two-stage | 7 | 836 | 1476 | 0.90 | 0.52 | 33% |
| two-stage | 10 | 610 | 1089 | 0.80 | 0.51 | 54% |

Cycle recall by true duration (two-stage): House 7: <= 45 min 0.89, 45 to 90 min 0.87, > 90 min 0.95.
House 10: <= 45 min 0.43, 45 to 90 min 0.80, > 90 min 0.84.
Recall alone is inflated when the model predicts far too many cycles (any overlap counts as a hit), so it is
always read together with precision.

### Sensitivity to the REFIT `Issues` flag
Excluding Issues = 1 minutes (pooled test): two-stage F1 0.52 (SAE 0.04), regression F1 0.46 (SAE 0.22),
hour profile F1 0.22, always-off 0.00. The ranking does not change. Because the flag fires disproportionately
while the washer is on, removing those minutes also removes true ON minutes, so this is a sensitivity check
and not a cleaner estimate.

### Interpretation
- Both LightGBM models beat both baselines on F1 and SAE in each test house. The hour-of-day profile behaves almost
  like "always on" (recall 0.96, precision 0.14), so its low SAE is not evidence of skill. Always-off has the best
  MAE and detects nothing, which is why MAE is not used for model choice.
- The two-stage model has the better F1, precision, SAE and cycle precision. Regression has the better recall,
  F1 at 500 W and cycle recall, but over-fragments (it predicts 2.3 to 2.6 times as many cycles as exist; two-stage
  predicts about 1.8 times as many). The two-stage model was fixed as primary before the test run and was not
  changed afterwards.
- Test F1 (0.55) is higher than the LOHO F1 (0.34), but this is not evidence of better generalisation: the washer
  is on more often in the test houses (8.5% and 9.8%) than in the training houses (about 3.9%), which raises F1
  for every model (the hour profile also rose from 0.12 to 0.25), and the final models were trained on five houses
  instead of four. We did not separate these effects.
- Per-house energy error ranges from -30% to +17%, so energy totals are only reliable in aggregate over many
  households, not per household.

### Actual vs predicted (actual_vs_pred_test.png)
Two-stage predictions for each test house on three days: the day with the most washer energy, the day closest to
the median washer energy, and one randomly chosen day (fixed seed). Days are chosen only among days with more than
95% valid data on which the washer ran, so they are not a random sample of all days and not a statistical summary.
Heating bursts of about 2 kW are mostly located well, especially in House 7 on busy days, but predicted bursts are
often somewhat lower than the real 2 kW, and the long low-power tail (about 50 to 300 W) is only partly recovered.
Errors visible in the plots: isolated predicted bursts while the washer is off (House 7 on 2014-08-20 around 16:00,
House 10 on 2013-11-27 in the morning, and apparently House 7 on 2015-03-26 in the evening); a real cycle that is
almost completely missed (House 10 on 2013-11-27 in the evening); and apparently at least one real burst with no
matching prediction on House 10's random day (2014-10-31). These readings come from the plots, not from extra
computation.

### Failure analysis
1. **False alarm** (failure_false_alarm.png): House 7, 2013-12-06, 18:19 to 19:04 UTC, 950 Wh of washer energy
   predicted while the washer was off. During this window the aggregate is 4 to 7 kW and oscillating, and the model
   predicts about 1.3 to 1.7 kW of washer power for about 45 min. We cannot identify the appliance from the data
   (only the washer channel is used); the pattern resembles a large thermostat-cycling load, consistent with the
   high-power unmetered loads seen in House 7 in Part A. Large oscillating loads can look like heating bursts to a
   model that only sees the aggregate. This explanation is a hypothesis.
2. **Missed cycle** (failure_missed_cycle.png): House 7, 2013-11-26, starting 19:18 UTC, a 606 Wh cycle (about
   2 kW for about 15 min, then a low-power tail) predicted as 0 W. The aggregate in the surrounding hours already
   shows 3 to 4 kW blocks from other loads, and the washer's step is not obviously separable from them in the plot.
   We did not verify which other loads were on.
3. **Short cycles in House 10:** cycles of 45 minutes or less are detected only 43% of the time against 80 to 84% for
   longer ones. A plausible reason is that short programs have little heating energy; this was not checked.
4. **Low-power tail:** the 50 to 300 W tail of a cycle is the hardest part to recover from an aggregate that
   already contains fridge cycles and other loads of similar size.

### Limitations of these results
- Only two test houses; per-house results differ noticeably and the sample is too small for confidence intervals.
- Minute precision of 0.59 and recall of 0.52 mean about 4 in 10 predicted ON minutes are false and about half of
  true ON minutes are missed. The model still predicts about 1.8 times as many cycles as exist.
- The per-cycle energy error is a median over all true cycles; missed cycles count as large errors.
- The thresholds (10 W, 25 W, probability 0.30) were chosen on five houses and may not transfer.
- Houses were selected for convenience, so the results should not be taken as representative of all REFIT
  homes, let alone other countries.

### Files
`outputs/section2/`: washing_machine_summary.csv, usage_hour_dow.png, trace_24h.png, example_cycles.png,
cross_house_comparison.png.
`outputs/section3/`: test_results.csv, test_cycle_metrics.csv, actual_vs_pred_test.png, failure_false_alarm.png,
failure_missed_cycle.png, config.json.
