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
