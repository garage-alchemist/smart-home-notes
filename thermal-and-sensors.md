# Thermal behaviour of a heavy-slab low-energy house, and sensors you can trust

## Measured time constants

From 56 days of summer data (room sensors + weather model) and 2.5 winters of InfluxDB history:

| Node | Time constant | What it means |
|---|---|---|
| Indoor air | τ ≈ 5–7 h (≈ 4 h phase lag behind outdoor) | what ventilation / night cooling acts on |
| Envelope, day–night | τ ≈ 55 h (IQR 43–81 h) | an 11 °C outdoor day/night swing becomes ~0.7 °C inside (93 % damped) |
| Structure | k ≈ 0.022 / day (τ ≈ 45 days) | passive cooling is negligible; a free-running house drifts only ~0.1 °C/day |
| Free-running, empty house (8 days, heating off) | ≥ 300 h | outdoor 12–31 °C, indoor 24.7–26.3 °C |

Method: amplitude ratio `f = 1/sqrt(1 + (2πτ/T)²)` with T = 24 h, cross-correlation for the lag, and a
first-order EMA fit for the free-running period. For model predictive control a 2R2C model
(air ≈ 6 h, structure ≈ 45 days) or a single node τ ≈ 55 h for 24 h planning works.

Practical consequences: full equilibrium takes ~3τ (days); one cool night does almost nothing to a hot slab;
heating "coasts" for a very long time — which is also why the boiler sees tiny loads in mild weather
(see [wolf-cgb2-ebus.md](wolf-cgb2-ebus.md)).

## Outdoor temperature: which sensor to believe

Clear morning comparison: boiler outdoor sensor 4.5 °C, car 6 °C, weather model 11.3 °C, ventilation unit
"outdoor" 14 °C.

- Weather models (gridded) miss local radiative cooling on clear nights by several degrees.
- The boiler sensor (on a south-west wall) is right at night and wrong only when the **sun is in the west**
  (azimuth ~240–300°): +1.7 °C on average, up to +9.7 °C.
- The ventilation unit's outdoor probe reads ~+9 °C at night (heated by the building) → unusable.

Template used: boiler sensor by default; with sun in the west, the lower of boiler and weather model;
weather model if the boiler sensor is stale > 2 h. The selected source is exposed as an attribute.

## Open-window detection without window contacts

Per room, keep 11 one-minute samples; state = max − current. A drop ≥ 0.7 °C in 10 min → setpoint 16 °C
for 30 min, extended while the drop continues, restored afterwards (max 2 h + a safety restore on restart).

Lessons from backtests:
- In mild weather (8–11 °C outside) an open window only costs 0.3–0.5 °C/h — fast thresholds miss it.
  Added a slow signature: −0.5 °C in 60 min with valve ≥ 20 %.
- "Closed again" = the room **rises** 0.3 °C from its minimum; a plateau proves nothing (warm slab).
- Some windows are physically undetectable this way (room stable at setpoint, valve open). Only contacts
  would see them.
