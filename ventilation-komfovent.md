# Komfovent MVHR: filter clogging stuck at 100 % and permanent W1 alarm

*Keywords: Komfovent C6 filter clogging 100%, W1 clogged air filters, clean filter calibration, Modbus 917.*

Observation: `filter_clogging` = exactly **100.0 %** (min = max = mean) for 14 consecutive weeks from the
first day of logging, and alarm **W1** ("Clogged air filters") permanently active.

Meanwhile the unit was clearly not restricted: power flat at 54–58 W, fans 46–59 % matching their
setpoints, **SFP 0.16–0.26 W/(m³/h)** (excellent for a residential unit).

Explanation: the C6 manual for W1 says *"Change the filters… After filter replacement start clean filter
calibration."* Without that calibration the controller has no reference, computes 100 % and raises W1 in a
loop. Sensor at 100 % and W1 are the same symptom.

What to do: inspect the filters, replace if dirty, **then** run "clean filters calibration". Only after
that does the clogging value mean anything. Until then, track SFP and power at constant mode instead.

**Confirmed (Nov 2026), filters replaced + calibration run:**
- On restart after the filter change: `filter_clogging` **100 → 0 %**, **W1 cleared**.
- Power unchanged at the same fan setting (**34–35 W** at 50 %) → the old filters were not restricting
  airflow; only the indicator was wrong.
- Calibration started from Home Assistant (`button.<name>_clean_filters_calibration` in the
  [hass-komfovent](https://github.com/lnagel/hass-komfovent) integration). It is visible in the data:
  both fans **50 → 40 %** and power **35 → 23 W** for about 2 minutes, then back to normal. No alarm.
- From now on the clogging value should creep up slowly over weeks; that is the real filter indicator.

Web interface of the C6 controller: `http://<unit-ip>` (plain http, LAN only). Login user `user`; factory
password is `user` in most manuals, `user2` in some Komfovent documents — try both before resetting.
