# Recommendation: estimating washing-machine use from smart-meter data

*Audience: distribution utility. Evidence: 20 UK REFIT homes, 2013–2015; full details in
[RESULTS.md](RESULTS.md) and [DATA.md](DATA.md).*

## What we built and how well it works

A model that estimates washing-machine (WM) power from **1-minute whole-house readings**, trained on
4 homes and tested on 2 homes it had never seen.

| Question | Answer on the unseen homes |
|---|---|
| Does it notice when a home washes? | **Yes, almost always.** It finds **9 in 10 real washes** (cycle recall 0.90). |
| Can it count washes? | **No.** Only **37 % of the washes it reports are real**: 1,617 reported vs 657 real, about 2.5 per real wash. Other heating loads (storage or immersion heaters, cooking) are often read as washes. Minute-level on/off F1 is 0.59. |
| How much energy did the WM use over the period? | **Roughly, for groups, not per home:** −18.9 % pooled (−20.1 % and −14.7 % per home), but **+32.4 %** on the validation home. The total is close partly because two errors cancel: real washes are predicted at about half their power, and false washes add energy that was never used. |
| WM energy on a given day? | Typical error **273 Wh/day**, against a typical wash of about 0.3–0.7 kWh, so single days are unreliable |
| Minute-by-minute power? | **No better than predicting "off"** on new homes (MAE 20.89 vs 20.32 W) |

The model is a **high-recall screen**: it rarely misses a home's washing, but it raises many false
alarms, so its output needs confirmation before it is acted on.

**Suitable uses**
- **First-pass screening** for households that may wash at the evening peak, to build a list that is
  then **confirmed** by survey, app question or a short sub-metering trial. Because few real washes
  are missed, few eligible households are left off the list.
- **Rough WM energy for large groups of homes**, reported with a wide error band (about ±30 %),
  and only alongside the uncertainty.

**Not suitable**
- Counting washes, or the share of washes at a given time: false washes (many at night and in the
  early morning) would distort these numbers.
- WM energy totals per home, per-day statements or minute-level feedback to customers.
- Billing, or anything with financial consequences for the customer.
- Homes with storage or immersion heaters, electric cooking at odd hours, washer-dryers, or washing
  that usually overlaps an electric shower.
- Use outside the UK without retraining on local sub-metered data.

---

## Section 4: practical implications

### 4.1 Opportunity: move evening-peak washes off the network peak

**Evidence.** From sub-metered data in six homes (not from the model), 16.2 % of washes start on a
**weekday between 16:00 and 19:59**, the evening network peak, ranging from 2.2 % to **29.0 %** of a
home's washes (House 10 starts 23 % of its washes at 19:00 alone). A wash uses a median 0.59 kWh,
most of it in a ~15-minute **~2 kW heating burst**, and three to eleven washes a week are typical.
Homes 1 and 15 already use night-time delay timers (many washes start 04:00–05:00), so shifting is
clearly acceptable to some households. Two further findings sharpen the offer:
- **Winter washes cost more at the peak.** Households wash about as often in every month, but the
  median wash uses about 60 % more energy in winter (680 Wh in January vs 423 Wh in August), when
  the network peak is also highest.
- **Long and hot programmes carry most of the energy.** Long washes (~2 h) are 41 % of washes,
  48 % of wash energy, and the most likely to start at the evening peak (22 %). Hot washes (31
  heating minutes, ~1.3 kWh) are 7 % of washes but 15 % of the energy.

**Offer.** Invite households to a **"delay-start" programme**: a message showing the cheaper
off-peak window, ideally combined with a time-of-use tariff or a small reward per shifted wash,
and **run it in winter**, when each shifted wash removes the most peak energy. Use the model only
to choose whom to invite first (homes with frequent detected evening activity), and confirm with
the household that they wash at that time. This saves the customer money and reduces peak demand;
it moves energy rather than reducing it. A second message, sent to all participants (this model cannot
tell programmes apart reliably, because it predicts the heater at about half its real power), can
promote 30–40 °C programmes, which cut the heater energy itself: moving
a hot wash to a standard one saves about 0.8 kWh.

**How to measure whether it worked.**

1. **Randomised trial:** randomly split eligible households into treatment (offer) and control
   groups, with at least 8 weeks of data before and after.
2. **Primary outcome: household energy in the weekday 16:00–20:00 window**, straight from the meter.
   It needs no model, so it is not affected by the model's false washes. Compare the change in
   treatment against the change in control (difference-in-differences).
3. **Secondary outcome:** the share of detected washes starting in that window. Because the
   detector reports about 2.5 washes per real one, look only at the **difference between groups**
   (the false-alarm rate should be similar in both), never at the level.
4. **Check the measurement:** validate on a small sub-metered or self-reported sample. Treat a
   change in detected washes as supporting evidence only; the meter-based outcome decides.

### 4.2 Limitations

- **Overlapping appliance signatures.** The model mainly looks for "about 2 kW above the base load".
  Storage and immersion heaters, ovens and hobs do the same: in testing, a ~3 kW heating load that
  stayed flat for an hour was read as a 0.7 kWh wash. Washes that run under big loads are missed:
  a 1.1 kWh wash during a morning shower was missed completely. Detected washes are predicted at
  only about half their real heater power.
- **Missing and faulty data.** 6–25 % of minutes are missing per home, including a 41-day
  outage. The published "cleaned" data forward-fills sensor gaps, so a dead meter looks like a
  flat line: up to 12 % of minutes in some homes. We treat flat runs of an hour or more as
  outages. Estimates must be flagged when data are incomplete, and outages should not be counted
  as "no washing".
- **Household differences.** Peak power is similar across homes (2.05–2.33 kW), but wash
  length (median 33–120 min), energy per wash (317–711 Wh) and timing vary several-fold. Four
  training homes cannot represent all households; the energy error on a new home ranged from
  −20 % to +32 %, so even its sign is not predictable.
- **From UK homes to India.**
  - **Different machines:** many Indian machines are cold-fill semi-automatic or top-loaders
    **without an internal heater**, which removes the signature this model relies on.
  - **Different appliance mix:** water heaters (geysers, 2–3 kW), air-conditioners and pumps
    overlap with the WM in power; geysers in particular would be read as washes, as storage
    heaters were here.
  - **Different supply and metering:** power cuts, inverter backups, voltage fluctuation, shared
    or multi-dwelling meters, and many meters reporting only every 15–30 minutes.
  - **What this means:** the UK model should not be transferred directly. It needs retraining and
    validation on sub-metered Indian homes, and the cycle rule needs re-tuning for low-power
    (~200–500 W) machines.

### 4.3 What gets harder with 15- or 30-minute smart-meter data

At 15 or 30 minutes, a whole wash spans only two to eight readings, and its defining feature, a
~15-minute 2 kW heating burst, becomes a single interval that is about 0.5 kWh higher than
usual. That bump looks the same as a kettle plus an oven, or a short shower. Every input this
model uses disappears: sharp on/off steps, the minute-level tumbling pattern, the timing of
heating within the wash. Start times can only be located to within the interval, so "did they wash
at 19:00 or 19:25" becomes unknowable. Short washes (House 1's are ~33 minutes) may fall entirely
inside one interval. Because this model already confuses washes with other heating loads at
1-minute resolution, the false-alarm rate would rise further. What remains possible is
**statistical**: estimating, over weeks and across many homes, how much energy and how many likely
washes there are from repeated ~0.3–0.7 kWh bumps, ideally with labelled training data at that
resolution. Results should be reported for groups of customers rather than individual days.
Measuring the loss by retraining this pipeline on data resampled to 15/30 minutes is the obvious
next experiment.
