# REFIT: raw-data cleaning, washing-machine EDA and disaggregation

Take-home assignment for Flock Energy. A small, reproducible Python pipeline that

1. inspects and cleans the **raw** REFIT data for House 1 and resamples it to 1 minute,
2. explores **washing-machine (WM)** use in 6 houses of the **official cleaned** REFIT data,
   including seasonality and the wash programmes households use,
3. trains and evaluates a **LightGBM regressor** that estimates WM power from the household
   aggregate at 1-minute resolution, with cycle post-processing, against two simple baselines,
4. turns the results into a utility-facing recommendation.

| Document | Content |
|---|---|
| [DATA.md](DATA.md) | Raw-data coverage and quality findings, cleaning steps, gap/outlier decisions, remaining concerns |
| [RESULTS.md](RESULTS.md) | WM EDA, model results, validation design, failure analysis |
| [Recommendation.md](Recommendation.md) | One-page utility summary and answers to the Section 4 questions |

## Repository layout

```
flock_energy_end_to_end.ipynb  the whole pipeline in one notebook (same code as src/, with outputs)
run_all.py               one command: runs every stage below in order
src/config.py            paths, WM channel per house, every threshold, house split
src/clean_raw.py         Stage 1  raw House 1 -> inspection, cleaning, 1-min, plots
src/prepare_clean.py     Stage 2  official cleaned CSVs -> 1-min parquet cache (18 WM houses)
src/eda_washing.py       Stage 3  cycle detection, availability, patterns, seasonality,
                                  programme types, sensitivity, plots
src/model.py             Stage 4  features, baselines, LightGBM, metrics, plots, failure cases
outputs/figures/         every figure referenced in the docs
outputs/tables/          every number quoted in the docs (CSV)
data/                    REFIT data (not in git, see below); data/cache/ is generated
```

## Setup and run

Tested on Windows 11 with Python 3.11.9.

```bash
pip install -r requirements.txt
```

The data is not in this repository (several GB). Download it from the University of Strathclyde:

- Raw data: <https://pureportal.strath.ac.uk/files/62090177/REFIT_RAW_081116.7z>
- Cleaned data: <https://pureportal.strath.ac.uk/files/62090184/CLEAN_REFIT_081116.7z>
- Cleaned-data readme: <https://pure.strath.ac.uk/ws/portalfiles/portal/62090183/CLEAN_READ_ME_081116.txt>
- Dataset page: <https://pureportal.strath.ac.uk/en/datasets/refit-electrical-load-measurements/>

Extract the two `.7z` archives (for example with 7-Zip) into `data/` so that the folder looks like
this:

```
data/REFIT_RAW_081116/RAW_House1_Part1.csv ... RAW_House21_Part2.csv
data/CLEAN_REFIT_081116/CLEAN_House1.csv ... CLEAN_House21.csv
data/CLEAN_READ_ME_081116.txt
```

Then run everything end to end, either as scripts:

```bash
python run_all.py
```

or as a notebook: open [flock_energy_end_to_end.ipynb](flock_energy_end_to_end.ipynb) from the
repository root (needs `pip install notebook`) and choose *Run All*. The notebook contains the same
code as `src/`, cell by cell, with the reasoning between steps and every table and figure shown
inline (cleaning → EDA → model predictions → evaluation). If the data lives elsewhere, set the
environment variable `REFIT_DATA_DIR` to that folder before starting the notebook.

A full run takes about 12 minutes on a laptop and regenerates `data/cache/`, `outputs/figures/`
(18 PNGs) and `outputs/tables/` (24 CSVs). Each stage can also be run on its own from `src/`
(`python clean_raw.py`, etc.); stages 3 and 4 read the cache written by stage 2. All randomness is
seeded (`RANDOM_SEED = 42`, LightGBM `deterministic=True`), so reruns reproduce the tables exactly.

## Assumptions

- **Timestamps.** `Unix` is the authoritative time. In the *raw* files it is UK local wall-clock
  time (it jumps ±1 h at daylight-saving changes); in the *cleaned* files it is true UTC (the
  readme says so, and it matches the raw data at a 0 h shift in winter and +1 h in summer,
  correlation 0.999). Hour-of-day and day-of-week analysis uses Europe/London local time.
- **Units.** All power values are active power in watts; energy is integrated from 1-minute means
  (W x 1/60 h).
- **WM channel** per house is taken from the official readme (`WM_CHANNEL` in `src/config.py`).
  House 4 (two washing machines) and House 12 (none) are excluded.
- **Impossible values.** Aggregate > 23 kW (about a 100 A single-phase supply) or ≤ 0 W is a
  sensor error. Each raw House 1 appliance has its own plausibility limit (fridge and freezers
  1 kW, computer and TV sites 0.5 kW, 2 kW heater 2.5 kW, washing machine, washer-dryer and
  dishwasher 3 kW; `APPLIANCE_LIMIT_W` in `src/config.py`), chosen from the appliance list in the
  readme and checked against the data: readings above them are almost all bit errors.

## Preprocessing decisions (details in [DATA.md](DATA.md))

- **Raw House 1:** where Part1 and Part2 overlap, Part2 is the primary source and minutes it is
  missing are filled from Part1 (same meter, no time shift); the ambiguous repeated hour at each
  autumn clock change is dropped; for duplicate timestamps the first non-missing value per channel
  is kept; impossible values (per-appliance limits) are set to missing per channel; data is
  resampled to 1-minute means; gaps ≤ 5 min are linearly interpolated and longer gaps are left
  missing and flagged as outages.
- **Cleaned data:** used as published (already de-duplicated, spike-filtered, DST-corrected). The
  same 1-minute and ≤ 5-minute-gap rule is applied. Minutes where WM > aggregate (impossible,
  < 0.1 % of minutes in the houses used) are excluded from model training and scoring. For the
  model, the aggregate is treated as missing wherever it is exactly flat for an hour or more:
  a stuck, forward-filled sensor (up to 8 % of a model house's minutes).

## Validation design

- **Split by house:** train on houses 2, 7, 10, 15; validate on house 9; test on houses 8 and 17,
  which are never used for EDA, feature choice or tuning.
- **Split by time within the training houses:** the model only sees the first 80 % of each training
  house's timeline. The last 20 % (spring–summer 2015), after a 1-day embargo that is longer
  than the ±60-minute input window, is a second test set ("same house, future months").
- **No random shuffling of minutes**, because neighbouring windows overlap almost completely and a
  random split would leak near-copies of test inputs into training.
- The validation house is used only to (a) pick the feature set (by RMSE), (b) pick the on/off
  threshold for the predicted power (by on/off F1) and (c) decide whether to apply cycle
  post-processing (by on/off F1).

## Trade-offs

- **LightGBM on hand-made window features rather than a neural network.** Quick to train, easy to
  inspect and easy to reproduce. The idea of predicting the midpoint of a window comes from
  Seq2Point, but the window is summarised with rolling statistics instead of a CNN.
- **A single regressor that predicts power directly.** It is the simplest model that answers the
  task, and one threshold turns it into an on/off detector. The cost: about 94 % of training minutes
  are "off", so the regressor learns to hedge. It predicts the ~2 kW heater at roughly half its
  height and leaves low, spread-out power on loads that only resemble a wash. Total energy partly
  balances out (false-positive energy offsets the under-predicted heater and missed washes), but
  per-minute accuracy and wash counts suffer: the model reports about 2.5 washes per real wash on
  the test houses, and its per-minute MAE is slightly worse than predicting "always off".
- **Shape-only features chosen on validation:** the selected feature set has no clock-time features
  (they encode the training houses' habits) and no absolute power levels (they differ between homes'
  base loads), only changes and shapes relative to the window. It had the lowest validation RMSE
  (149.8 W vs 150.6 W without time features and 151.2 W with all features).
- **Fixed 600 boosting rounds instead of early stopping:** the validation error curve is almost
  flat, so early stopping stopped after about 15 trees, an under-fitted model (see RESULTS.md).
- **Cycle post-processing** (keep only predicted washes ≥ 10 min and ≥ 50 Wh) raised validation
  on/off F1 from 0.40 to 0.47 and was kept. It removes short blips but cannot remove long false
  washes, so the number of predicted washes barely changes.
- **Per-appliance limits instead of one 4 kW limit.** A single limit is easy to explain but let
  1,580 impossible fridge readings (1–4 kW) through. Per-appliance limits need the appliance list
  and a check of each channel's values, but they removed 2,135 more bad readings, almost all
  recognisable bit errors.
- **Filling Part2's gaps from Part1** rather than discarding Part1 for the whole overlap window. It
  recovers 4,549 aggregate minutes and most of the heater channel's missing summer (its outage
  falls from 36.7 % to 23.0 %), at the cost of mixing two logs whose values differ slightly
  (MAE 57.8 W where both exist). Averaging both logs everywhere was rejected because it would blur
  the signal.
- **Programme types from k-means on two features** (duration, heating minutes) rather than a
  richer shape-based clustering: easy to explain and stable, but it describes the electrical
  load, not the programme selected on the machine.
- **Simple cleaning rules** rather than reproducing the official REFIT corrections (e.g. moving
  appliance monitors that householders swapped). These are documented as remaining concerns.
- **Six EDA houses and seven model houses** out of 18 available, chosen on data quality.

## Research used

- **Murray, D., Stankovic, L. & Stankovic, V. (2017).** *An electrical load measurements dataset of
  United Kingdom households from a two-year longitudinal study.* Scientific Data 4, 160122.
  Used for the dataset description, sampling, known collection issues and the raw-vs-cleaned
  distinction. The dataset licence also asks for: Murray, D. et al. (2015), *A data management
  platform for personalised real-time energy feedback*, EEDAL'15.
- **Zhang, C., Zhong, M., Wang, Z., Goddard, N. & Sutton, C. (2018).** *Sequence-to-point learning
  with neural networks for non-intrusive load monitoring.* AAAI-18. Adopted: predict the appliance
  power at the **midpoint** of a window of aggregate readings (non-causal context on both sides),
  and test on houses not seen in training.
- **Kelly, J. & Knottenbelt, W. (2015).** *Neural NILM: deep neural networks applied to energy
  disaggregation.* BuildSys '15. Adopted: defining appliance activations with an on-power
  threshold plus minimum on/off durations (our cycle rule), testing on unseen houses, and
  reporting several complementary metrics (on/off precision/recall/F1, relative error in total
  energy, mean absolute error) instead of one number.

## AI tools used

*(Rewrite this paragraph in your own words before submitting.)* This project was built with
**Claude Code** (Anthropic) as a coding assistant. It drafted the scripts, ran the analysis and
drafted the documentation, working phase by phase from an agreed plan; each phase's outputs were
reviewed before moving on. All numbers in the docs come from `outputs/tables/`.

## Possible improvements (next steps, in priority order)

1. **Separate "is it on?" from "how much?"** (a classifier for on/off plus a power regressor
   trained only on "on" minutes). *Why:* the single regressor is pulled towards 0 by the ~94 %
   off minutes, which halves the predicted heater power and produces many weak false washes.
   Splitting the two questions is the standard fix for zero-heavy targets.
2. **Features that separate a WM heater from long, flat heating loads** (length of the current
   high-power stretch, share of the window that is high and perfectly flat). *Why:* the largest
   false positive is a ~3 kW load that stays flat for an hour; a WM heater burst lasts 10–25 min
   and is surrounded by drum flicker.
3. **Recover washes hidden under big loads** (e.g. mixing real WM cycles into other houses' busy
   periods, as in Kelly & Knottenbelt 2015). *Why:* washes that run during an electric shower are
   missed completely.
4. **Leave-one-house-out cross-validation.** *Why:* energy error changes sign between houses
   (+32 % on validation, −15 % and −20 % on the test houses); one validation house is a thin basis
   for choosing settings.
5. **A coarse-resolution experiment** (15/30 minutes). *Why:* most smart meters report at that
   resolution; it would measure how much accuracy is lost (see Recommendation.md).
