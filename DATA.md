# DATA.md

## Source and scope
- REFIT raw data, House 1 (RAW_House1_Part1.csv + RAW_House1_Part2.csv), University of Strathclyde,
  licensed CC BY 4.0. Cite: Murray et al. (2015), "A data management platform for personalised
  real-time energy feedback", 8th Int. Conf. on Energy Efficiency in Domestic Appliances and Lighting.
- Channels: Aggregate + 9 Individual Appliance Monitors (IAMs). Active power in Watts only.
  House 1: 1 Fridge, 2 Freezer(1), 3 Freezer(2), 4 Washer Dryer, 5 Washing Machine,
  6 Dishwasher, 7 Computer, 8 TV site, 9 Electric Heater.
- Timestamps: Unix seconds (UTC). Coverage: 2013-10-09 14:06:17 to 2015-07-10 11:56:32 UTC
  (~639 days = 920,031 one-minute bins).
- Nominal sampling 8 s; per the README, data was logged on change and sensors are unsynchronised.

## Inspection findings (raw)
- 8,518,763 rows; 0 unparseable timestamps; 0 negative values.
- Ordering: not monotonic; 2 out-of-order steps (consistent with Part1/Part2 concatenation).
- Duplicate timestamps: 1,211,048 rows (14.2%). Two causes:
  (a) 764,126 timestamps appear more than once within one file: timestamps have 1 s resolution
      and rows are sparse/staggered, so several rows share a second. Averaging combines their
      channels (NaN ignored).
  (b) 191,889 timestamps appear in both files: Part1 (2013-10-09 to 2014-10-01) and Part2
      (2014-06-27 to 2015-07-10) overlap for ~96 days.
- Sampling interval: median 7 s, mean 7.55 s, p01 1 s, p99 15 s; only 53% of intervals are
  6-10 s (irregular, see raw_interval_hist.png). Longest interval 3.59M s (~41.6 days).
- Gaps: 4,040 gaps > 1 min, 21 > 1 hour, 12 > 1 day.
- Rows are sparse: each row may hold only some channels (others NaN). Raw per-column NaN counts
  (e.g. Appliance9: 1.9M) are therefore not outages; real missingness is measured after
  1-min resampling.
- Raw values above range limits (aggregate > 15 kW, appliance > 4 kW): Aggregate 186,
  Appliance1 846, Appliance2 40, Appliance3 118, Appliance4 2,085, Appliance5 221,
  Appliance6 36, Appliance7 374, Appliance8 179, Appliance9 50.

## Cleaning steps
1. Sort by timestamp.
2. Part1/Part2 overlap: Part1 rows at or after the first Part2 timestamp (2014-06-27 21:57:49)
   were dropped (1,257,865 rows, 29% of Part1). Reason: Part1 stores unavailable sensors as 0,
   Part2 as NaN, so Part1 zeros could be mistaken for "appliance off".
   Trade-off: ~3,300 min of real aggregate data (28 to 30 Aug 2014) that exists only in Part1
   was lost, because Part2 has an outage there.
   Remaining duplicate timestamps (597,182 rows): mean per timestamp.
3. Impossible values set to NaN (not clipped): < 0 W, aggregate > 15 kW (~65 A x 230 V),
   appliance > 4 kW (same threshold as the official REFIT cleaning, which uses 0 instead of NaN).
   Counts: Aggregate 147, Appliance1 762, Appliance4 1,010, Appliance5 70
   (other channels in section1_findings.json).
4. Resample to 1-min mean on a full regular grid (920,031 bins).
5. Gaps <= 5 min: time interpolation (aggregate 899 min, 0.10%).
6. Gaps > 5 min: left NaN and flagged (`*_long_gap`; aggregate 118,793 min, 12.9%).
   A washing cycle lasts 1-2 h, so filling longer gaps would invent or erase cycles.
7. Stuck values: identical 1-min value for >= 60 min and > 100 W -> NaN and flagged (`*_stuck`).
   Found on plots (8 Feb 2015, ~13:00-16:00: aggregate ~12.5 kW and washer ~2.25 kW perfectly
   flat for ~3 h, not physical; the logger likely repeated its last value). These values were
   inside the range limits, so step 3 did not catch them.
   Flagged minutes: Aggregate 3,431 (12 runs), Appliance5 168, Appliance4 86, Appliance9 4,309.
   Longest aggregate runs: 1,251 min (2014-08-01 17:13 to 08-02 14:03) and 873 min
   (2014-09-06 22:52 to 09-07 13:24); most others are 60-323 min.

## Gaps and outages
- 382 long (> 5 min) aggregate outages, 118,793 min in total.
- Largest: 2014-01-24 to 2014-03-06 (59,906 min, ~41.6 days, 50% of flagged time), matching
  the README note on missing data in Feb 2014. Others: 2015-05-10 to 05-19 (8.6 d),
  2015-06-10 to 06-18 (7.5 d), 2014-08-28 to 09-01 (4.5 d), 2015-04-03 to 04-07 (4.4 d).
- Top 10 outages = 91% of flagged minutes; the other 372 average ~28 min.
- Many short outages in Oct 2013 to Jan 2014 (visible as thin red lines in
  full_range_aggregate.png): logging was less stable at the start of the study.
- Valid aggregate minutes after gap flagging and stuck removal: 797,807 (86.7%).
  Aggregate and washing machine both valid: 85.9% of minutes.

## Plots
- before_after_2day.png: Aggregate, Appliance1 (fridge) and Appliance5 (washing machine),
  raw (~8 s) vs cleaned 1-min, for the 2 days where the washer used the most energy among
  well-covered days (23-25 Feb 2015). The "before" panels show raw data after the Part1/Part2
  overlap removal. Cleaning preserves the cycle shapes; interpolated minutes are marked orange
  (none in this window).
- full_range_aggregate.png: cleaned aggregate over the full range, long gaps shaded red.
- raw_interval_hist.png: raw sampling-interval distribution (mode ~7 s, secondary peaks at ~2 s
  and ~13-15 s), showing irregular sampling and why resampling to 1 min is needed.

## Remaining concerns
- Heater (Appliance9) is missing 36.8% of minutes; not used for modelling.
- Washing machine (Appliance5) is missing ~6.9 more days than the aggregate (129,691 vs 119,692
  minutes); modelling uses only minutes where both are valid.
- Part1 uses 0 for "sensor unavailable"; before 2014-06-27 a 0 may mean "no data", not "off".
- Sensor spikes between 100 W and 4 kW are not removed (e.g. a ~770 W fridge spike in raw
  data remains ~300 W after 1-min averaging).
- Sensors are unsynchronised: appliance and aggregate timestamps can differ by a few seconds.
- The aggregate has frequent 10-11 kW peaks and 1-3.5 kW cycling bands that no monitored
  appliance explains (unmetered load). This will make disaggregation harder.
- Aggregate level shifts (e.g. ~1.2 kW cap in Mar 2015, ~300 W flat baseline in July 2015)
  suggest changes in household occupancy or loads; train/test splits should account for this.
- Thresholds (5 min fill, 15 kW, 4 kW, 60 min stuck) are judgement-based and not validated.
- Stuck-value detection only catches runs of >= 60 min; shorter freezes remain.
- Raw data was read from a Kaggle mirror of the official REFIT raw files (same file names
  and columns); the official archive is the reference source.