# DATA: coverage, quality findings and cleaning

## 1. Dataset

**REFIT Electrical Load Measurements** (University of Strathclyde; Murray et al. 2017; CC BY 4.0).
20 houses in the Loughborough area (UK), House 1–21 without House 14, monitored 2013–2015. Each
house has a whole-house current clamp (`Aggregate`) and 9 individual appliance monitors
(`Appliance1`–`Appliance9`). Values are **active power in watts**, polled about every **6–8 s**;
the sensors are not synchronised and only report on change.

| Files used | Purpose |
|---|---|
| `REFIT_RAW_081116/RAW_House1_Part1.csv`, `RAW_House1_Part2.csv` | Part 1: raw inspection and cleaning |
| `CLEAN_REFIT_081116/CLEAN_House{h}.csv` for the 18 WM houses | Part 2 EDA (houses 1, 2, 7, 9, 10, 15) and Part 3 model (2, 7, 10, 15, 9, 8, 17) |
| `CLEAN_READ_ME_081116.txt` | appliance list per house, column meaning, cleaning applied |

The other raw files were not used, because the assignment asks for House 1 only.

## 2. Raw House 1: what the data looks like

Source: `outputs/tables/raw_h1_inspection.csv`, `raw_h1_extreme_values.csv`.

| | Part1 | Part2 |
|---|---|---|
| Rows | 4,377,877 | 4,140,886 |
| Span (UK local time) | 2013-10-09 14:06 → 2014-10-01 01:27 | 2014-06-27 21:57 → 2015-07-10 11:56 |
| `Time` parse failures | 0 | 0 |
| \|`Time` − `Unix`\| | ≤ 1 s | ≤ 1 s |
| Backward time steps | 1 (2013-10-27 01:59 → 01:00) | 0 |
| Duplicate `Unix` rows | 1,004,404 (595,935 fully identical rows) | 14,755 |
| Rows with some channels missing | 0 | 2,425,207 (5,284,083 empty cells) |
| Median sampling interval | 7 s | 7 s |
| Intervals between 5 and 10 s | 11.9 % | 98.8 % |
| Gaps > 1 min / > 5 min / > 1 h | 781 / 380 / 14 | 3,604 / 72 / 19 |
| Longest gap | 998.4 h (41.6 days) | 207.6 h |
| Aggregate > 23 kW / ≤ 0 W | 26 / 12 | 51 / 0 |
| Appliance readings > 4 kW | 3,038 | 911 |
| Negative values | 0 | 0 |

Findings:

- **Timestamps are local wall-clock time, not UTC.** Both `Time` and `Unix` jump back an hour at
  the autumn clock change (Part1, 27 Oct 2013) and leave a 1-hour hole each spring
  (2014-03-30, 2015-03-29). In Part2 the repeated autumn hour (26 Oct 2014) is sorted into
  the surrounding data, so the two copies of that hour cannot be separated.
- **The two files overlap for three months** (27 Jun – 1 Oct 2014; 1,257,865 Part1 rows). They
  line up with no time shift (1-minute aggregate correlation 0.793, MAE 57.8 W; shifting Part2
  by ±1 h drops it to 0.03–0.04; `raw_h1_overlap_alignment.csv`), so it is the same meter
  logged twice, but the values are not identical.
- **Many duplicate timestamps in Part1**, often with different values: 118,191 of 829,193
  duplicate groups disagree on at least one channel (Part2: 155 of 14,755). Part1's intervals
  alternate between roughly 2–3 s and 12–15 s, which looks like two logging streams interleaved a
  few seconds apart
  ([sampling-interval histogram](outputs/figures/raw_h1_sampling_intervals.png)).
- **Part2 splits each reading across rows**: the aggregate and some appliances arrive on separate
  rows a few seconds apart, with the other columns empty.
- **Impossible appliance values are bit errors:** the most common values above 4 kW are 4096,
  8192, 16384, 32767, 65535 and 98301 W, up to 98 kW on a fridge or TV channel. 62 of the 77
  aggregate readings above 23 kW are exactly 24,000 W, which looks like a sensor ceiling.

## 3. Cleaning pipeline (`src/clean_raw.py`)

| Step | Rule and reason | Rows / count after |
|---|---|---|
| Load | both parts concatenated | 8,518,763 rows |
| Overlap | Part2 is the **primary** source for 27 Jun – 1 Oct 2014; Part1's rows for that window are set aside. Averaging two slightly different logs of the same meter would blur the signal, and Part2 has the steadier sampling. | 7,260,898 |
| Autumn clock change | drop 01:00–02:00 local on the last Sunday of October: that hour happens twice and cannot be separated in Part2 (about 1 h lost per year, left as an outage) | 7,258,713 |
| Sort + duplicates | stable sort by `Unix`; for duplicate timestamps keep the **first non-missing value per channel**. For complete rows this is the same as "keep first"; for Part2's split rows it stops appliance readings being lost. | 6,662,139 rows |
| Impossible values | aggregate ≤ 0 or > 23 kW, and each appliance above **its own plausibility limit** (table below) → missing, **for that channel only** (one bad sensor does not remove the row) | 4,792 cells (2,657 above the general 4 kW / aggregate limits, 2,135 more caught only by a per-appliance limit) |
| Resample | 1-minute mean on a complete minute grid (missing minutes stay missing) | 920,031 minutes |
| Fill from Part1 | in the overlap window, a minute with **no Part2 reading** takes Part1's 1-minute mean for that channel (Part1 cleaned with the same steps). Both files log the same meter with no time shift (§2), so Part1 is a valid reading where Part2 has none; where both exist, Part2 is kept. | 4,549 aggregate minutes (6,146 WM minutes) |
| Short gaps | linear interpolation for gaps **≤ 5 min**: short enough that interpolation cannot invent a whole appliance event | 854 aggregate minutes |
| Long gaps | left missing and flagged `agg_outage` (not filled: filling hours or days would be invented data) | 114,398 aggregate minutes, 377 outages |

**Per-appliance limits** (`APPLIANCE_LIMIT_W` in `src/config.py`; appliances from the REFIT readme;
evidence in `raw_h1_extreme_values.csv`). A single 4 kW limit is far above what most of House 1's
appliances can draw, so it let impossible readings through: 1,580 fridge readings between 1 and
4 kW. Readings between each appliance's limit and 4 kW are almost all **bit errors**: exact powers
of two, or a power of two added to a normal reading.

| Channel | Appliance | Limit | Readings removed (limit → 4 kW) | Most common values there |
|---|---|---|---|---|
| A1 | fridge | 1,000 W | 1,580 | 2048, 1024, 2056 (= 2048 + 8), 3072 |
| A2 | freezer (1) | 1,000 W | 86 | 1073 (= 1024 + 49), 2094 (= 2048 + 46) |
| A3 | freezer (2) | 1,000 W | 307 | 1024, and a cluster at 1,001–1,019 W |
| A4 | washer-dryer | 3,000 W | 1 | 3584 |
| A5 | washing machine | 3,000 W | 2 | 3072, 3040 |
| A6 | dishwasher | 3,000 W | 0 | – |
| A7 | computer site | 500 W | 306 | 2048, 1061, 512, 1024 |
| A8 | television site | 500 W | 59 | 512, 1024, 2048 |
| A9 | electric heater (2 kW) | 2,500 W | 0 | – |

The limits sit well above each appliance's normal range (fridge and freezers: 99.9 % of readings
≤ 200 W; washing machine heater ~2.3 kW) and well below the bit-error values.

**Result** (`raw_h1_coverage.csv`): for the aggregate, 87.47 % of minutes are observed (0.49 %
of them filled from Part1), 0.09 % interpolated and 12.43 % outage (12.92 % without the Part1
fill). The appliance channels look similar, except **Appliance9 (electric heater)**: Part2 has no
heater readings for about June–November 2014, and Part1 (which runs to 1 October 2014) covers
much of that gap, so its outage
falls from 36.68 % to **23.01 %** (13.64 % of its minutes come from Part1). Part2's 108-hour
outage (2014-08-28 → 09-01) shrinks to 53 hours. The longest remaining outages are
2014-01-24 → 2014-03-06 (998 h; the "February 2014" outage in the readme), 2015-05-10 → 05-19
(208 h) and 2015-06-10 → 06-18 (180 h) (`raw_h1_outages.csv`).

Figures:

- [Before/after overview](outputs/figures/raw_h1_before_after_overview.png): hourly maxima of
  the aggregate, fridge (A1) and WM (A5). The 30–98 kW spikes disappear; outages become visible
  gaps instead of being bridged.
- [Before/after for one day (25 May 2014)](outputs/figures/raw_h1_before_after_day.png): the
  cleaned 1-minute series follows the raw readings; two ~2.3 kW WM cycles are kept; one bad
  WM reading above 4 kW is removed.
- [Coverage](outputs/figures/raw_h1_coverage.png): daily share of minutes with valid readings
  per channel, and the aggregate's daily outage and interpolation share.

## 4. Official cleaned data (`src/prepare_clean.py`)

The readme says the official cleaning corrected the DST shift (so `Unix` is UTC), moved appliance
monitor columns when householders swapped plugs, **forward-filled missing values**, and
**replaced appliance spikes > 4 kW with 0**. It also added `Issues = 1` where the appliance
channels add up to more than the aggregate.

Checks on the 18 WM houses (`clean_wm_houses_availability.csv`): no backward steps, no
duplicate timestamps, no empty cells and no WM readings above 4 kW. UTC was confirmed against
my raw House 1 output: correlation 0.999 at a 0 h shift in winter and at +1 h in summer,
against 0.225 and 0.068 at the wrong shift (`clean_timezone_check.csv`). The same
1-minute / ≤ 5-minute rule was applied.

| House | Period (local) | Usable days | Outage % | `Issues` minutes % | WM > aggregate minutes % | Aggregate flat ≥ 60 min % | Use |
|---|---|---|---|---|---|---|---|
| 1 | 2013-10-09 → 2015-07-10 | 558.8 | 12.54 | 1.81 | 0.082 | 0.31 | EDA |
| 2 | 2013-09-17 → 2015-05-28 | 465.8 | 24.56 | 1.51 | 0.013 | 0.74 | EDA, train |
| 7 | 2013-11-01 → 2015-07-08 | 536.6 | 12.48 | 4.06 | 0.067 | 0.57 | EDA, train |
| 8 | 2013-11-01 → 2015-05-11 | 498.6 | 10.16 | 1.54 | 0.017 | 1.01 | test |
| 9 | 2013-12-17 → 2015-07-08 | 481.2 | 15.28 | 2.02 | 0.004 | 4.73 | EDA, validation |
| 10 | 2013-11-20 → 2015-06-30 | 523.3 | 10.85 | 1.65 | 0.026 | 7.99 | EDA, train |
| 15 | 2013-12-17 → 2015-07-08 | 490.8 | 13.50 | 1.40 | 0.044 | 0.81 | EDA, train |
| 17 | 2014-03-06 → 2015-06-19 | 441.5 | 6.06 | 4.01 | 0.058 | 2.16 | test |
| 3 | 2013-09-25 → 2015-06-02 | 545.4 | 11.26 | 6.85 | 0.228 | 12.06 | not used |
| 5 | 2013-09-26 → 2015-07-06 | 589.0 | 9.16 | 7.59 | 0.343 | 0.42 | not used |
| 11 | 2014-06-03 → 2015-06-30 | 349.6 | 10.85 | 1.73 | 0.132 | 1.11 | not used |
| 13 | 2014-01-17 → 2015-05-31 | 386.1 | 22.56 | 3.08 | 0.014 | 2.22 | not used |

(The other six WM houses, 6, 16, 18, 19, 20 and 21, are in the CSV.) Houses 3 and 5 were left out
because they have the most `Issues` and WM > aggregate minutes; House 13 for its 22.6 % outage;
House 11 for the shortest record and months with no washing.

**Decisions:** minutes where WM > aggregate are excluded from model training and scoring.
Other `Issues` minutes are kept (the WM reading itself is still plausible) and reported.

## 5. Remaining concerns

1. **Stuck aggregate sensor in the cleaned data.** Because missing values were forward-filled, a
   failed aggregate clamp shows up as a perfectly flat line. Up to 12.06 % of minutes (House 3) and
   7.99 % (House 10, a training house) sit in runs where the aggregate is exactly constant for at
   least an hour. In one House 8 example the aggregate stays at 880 W for hours while the WM
   draws 2.3 kW. **The model treats these runs as outages**: the aggregate is set to missing, so
   they are neither inputs nor scored (`model_excluded_minutes.csv`). Shorter stuck stretches
   (< 60 min) cannot be told apart from a genuinely steady load and remain. Outage percentages
   in the table above count only missing rows, so they are **lower bounds**.
2. **Zeros may be removed spikes.** The official cleaning replaced appliance spikes > 4 kW with
   0, so a 0 does not always mean "off".
3. **Appliance identity.** The readme warns that householders may have moved monitors. The WM
   channels all show the expected 2 kW heater signature, but House 19's rarely exceeds 1.1 kW.
4. **Limits of the raw cleaning.** Per-appliance limits catch bit errors that add 1 kW or more;
   smaller bit errors (e.g. + 256 or + 512 W on a 100 W fridge reading) stay. Freezer (2) has a
   cluster of readings at 1,001–1,019 W that are not exact powers of two; they are removed by the
   1 kW limit but could be a real defrost or fault. 1-minute aggregate means of 20–22 kW remain
   (below the limit, plausibly real). Minutes filled from Part1 come from a log whose values
   differ slightly from Part2's (1-minute MAE 57.8 W where both exist, §2), so they carry a little
   more measurement noise.
5. **Interleaved Part1 streams.** The two interleaved logging streams in Part1 are averaged
   together in each minute. That is fine at 1-minute resolution but would matter for
   sub-minute work.
