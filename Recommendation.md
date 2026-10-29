# Recommendation.md

## Summary for the utility

We built an offline model that estimates a household's washing-machine power from the whole-house aggregate at
1-minute resolution. It was trained on 5 REFIT houses and tested once on 2 unseen houses (7 and 10).

**Where it is useful.** Household-level patterns over weeks, for example when a home tends to run its washer.
On the test houses the two-stage model recovered pooled washer energy to within 7% of the truth, but per house
the error was +17% (House 7) and -6% (House 10).

**Where it is not reliable.**
- Per-cycle decisions: about 4 in 10 predicted ON minutes are false (precision 0.59) and about half of the true
  ON minutes are missed (recall 0.52).
- It predicts about 1.8 times as many cycles as exist; median per-cycle energy error is 33% (House 7) and 54%
  (House 10); cycles of 45 minutes or less in House 10 are detected only 43% of the time.
- The worst false alarm in the test (House 7) predicted 950 Wh of washing in a 45-minute window when the washer
  was off, probably caused by another large load.

| Use | Verdict |
|---|---|
| Weekly or monthly "when does this home wash" insight | Reasonable, if first validated on a sub-metered sample |
| Targeting households for a time-shift campaign | Reasonable, with the measurement plan below |
| Real-time "your washer just started" nudge | Not supported by these results |
| Per-cycle energy reporting or billing | Not supported |
| Use in India without retraining on local data | Not supported (see limitations) |

## Opportunity: shift washing out of the evening peak

**What the analysis suggests.** The time of day at which a home washes differs a lot between houses: in our data
House 7 starts many cycles around 7 to 8 h, House 10 mostly around 19 h, which is close to the evening aggregate
peak at about 17 to 19 h; roughly 30 to 37% of the cycles in Houses 5 and 9 start on a Saturday. A cycle uses a
median of about 290 to 715 Wh depending on the house, and homes run about 4 to 11 cycles per week. A washing
machine is a load many households can delay by a few hours (most machines have a delay-start function), so the
opportunity is peak shifting, not a large kWh saving. As an illustration only: a home like House 10 (about
8.5 cycles per week at about 715 Wh) uses roughly 6 kWh per week for washing, so moving a quarter of cycles
out of the peak would shift about 1.5 kWh per week. The real size depends on the tariff and the peak definition,
which we do not have.

**Proposed action.** Use the model to estimate each household's share of weekly washer energy in the peak window
(for example 17:00 to 20:00 local time, or the tariff's actual peak). Send a weekly message to households with a
high share, showing their own pattern and suggesting a later start.

**How to measure whether it worked.**
1. Randomise eligible households into a nudge group and a control group (no message, or a generic one), so that
   weather, holidays and tariff changes affect both groups equally.
2. Primary outcome: share of estimated washer energy in the peak window per household per week, before versus
   after and between groups (difference-in-differences), over at least 8 to 12 weeks after a pre-period of the
   same length.
3. Secondary outcome that does not depend on our model: peak-window aggregate demand of the nudged group
   versus control. Also track opt-outs and, if available, delay-start usage.
4. Validate the measurement itself: put sub-meters on a small sample of homes in both groups. If the model's
   peak-share estimate tracks the sub-metered truth, it is usable; given per-house energy errors of -30% to +17%,
   compare groups on relative changes rather than absolute kWh.
5. Randomisation cancels model errors that are the same in both groups, but not errors that change after the
   nudge (for example shifted night-time cycles may look different in the aggregate), which is why step 4 is needed.
6. Fix the success criterion in advance, for example a reduction of at least X percentage points in peak share
   versus control with a confidence interval excluding zero (X to be agreed with the utility).

## Limitations

- **Overlapping appliance signatures.** The washer's heating burst is about 2 kW for 8 to 15 minutes, similar to
  dryers, dishwashers, kettles and heaters. A hand-written rule for such a step had precision of only 0.14, and the
  worst test false alarm was a large oscillating load predicted as washing. The low-power tail (50 to 300 W) of a
  cycle is mostly lost in the aggregate.
- **Missing data.** Valid time is 75 to 89% in the houses used. Long outages are flagged, not filled, but the
  cleaned REFIT files forward-fill gaps, which we detect only above 100 W, so low-power outages can pass as
  "washer off". Early raw data stores unavailable sensors as 0.
- **Household differences.** Median cycle duration ranges from about 34 to 121 minutes and energy from about 290
  to 715 Wh across houses; only 5 houses were used for training and 2 for testing, chosen for convenience (7 of
  19 houses with a washer channel). We did not separate new-house error from the inherent difficulty of the signal.
- **Label consistency.** The REFIT `Issues` flag (sub-meter sum above aggregate) fires in 12 to 30% of washer-on
  minutes, so labels and aggregate are not perfectly consistent, which caps achievable accuracy.
- **UK to India** (reasoned from general knowledge, not tested on any Indian data):
  - REFIT is mostly UK detached homes with front-loading machines. Many Indian homes use top-loaders or
    semi-automatic twin-tubs, often with cold water and no heating element; without a 2 kW heating burst the
    signature our model relies on does not exist.
  - Washing frequency, load size and timing (including help from domestic staff) may differ.
  - Voltage fluctuation, power cuts and inverters or stabilisers change the aggregate and the missing-data pattern.
  - Other large loads (air conditioners, water heaters, pumps) overlap in time and size with the washer, and
    climate changes when they run.
  - Tariff structures and peak windows differ, so the peak definition above would need to be re-derived.
  - Deployment would need local sub-metered data for retraining and validation.

## What gets harder with 15-minute or 30-minute smart-meter data

The washer's key signature is a 2 kW burst of 8 to 15 minutes, and a whole cycle often lasts 30 to 120 minutes
(about 34 minutes for House 1's common program). In a 15- or 30-minute reading that burst is averaged with
everything else the house did in the interval, so a 10-minute 2 kW burst adds only about 0.7 kW to a 30-minute
average and is easily hidden among fridge cycles and other loads. A cycle would span only one to four readings,
so the rise-then-tail shape used by our cycle rule cannot be seen, and our correlation analysis (House 7) found
that useful correlation fades after about 15 to 20 minutes, so a 30-minute reading carries almost no
within-cycle information. Cycle detection and minute-level disaggregation would become much harder or impractical;
what would remain is statistical inference, such as the probability that a household washes in a given time slot
or on a given weekday. We did not test this by down-sampling our data; a simple extension would be to resample
to 15 and 30 minutes and re-run the pipeline to measure the loss.
