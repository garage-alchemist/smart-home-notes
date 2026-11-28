# EV charging and energy

## Smappee EV Wall: charging "stops" instead of slowing down

*Keywords: Smappee EV Wall suspended_evse, capacity maximum power, load balancing, charge stops.*

Symptom: overnight charging pauses repeatedly; the charger goes to `error`, then `suspended_evse`
(IEC state B1), and resumes on its own 5–35 min later.

Cause: the site cap (`capacity_maximum_power`, here 7 kW) minus house load leaves less than the
**AC minimum of 6 A per phase** (≈ 4.1 kW three-phase). The charger cannot go below 6 A, so it pauses.
Typical trigger: oven or other big loads during the night window.

Options: accept the pauses (they self-heal), raise the cap if the grid connection allows it, or shift big
house loads. A useful extra alert: "car plugged in during the cheap window, not full, session never started".

Other notes:
- The cloud integration's charger power sensor often freezes at 0 W during charging. Decide
  "is the car charging?" from the charging state + EVCC state C + per-phase currents (MQTT), not from power.
- A HACS integration update renamed every entity (new serial-based prefix) — automations using the old IDs
  silently died. After such updates, search automations for `unavailable` entities.

## Grid reference-power tariffs and three-phase charging

Where the grid fee depends on a reference power measured per 15 min, three-phase AC charging at the 6 A
floor (≈ 4.1 kW + house) exceeds a 3 kW reference by design. Compare the yearly surcharge with the cost of the
next reference level for *your* charging frequency. Here, regular 10 kW ramps pushed an automatic
reclassification to 7 kW, which turned out cheaper overall from ~3 charges a month.

## Car climate API: "fan speed" is really a mode byte (MG / SAIC integration)

In the `mg_saic` integration (profile MZS3E), `start_climate` → `fan_speed` selects the **mode**:
1 vent, 2 cool, 3 max cool, **4 heat**, 5 defrost. Sending 3 cooled the cabin in winter.
Also: front defrost on this car is compressor-based and switches climate to "cool" — don't send it right after
a heat request in near-freezing weather. And the car's GPS tracker can freeze while it sits in a garage; use
"charging gun connected" as a second "car is home" signal.

## MG S5 EV: 12 V battery went flat after 2 days parked — and the integration's polling helped (Nov 2026)

**Proven (from the car's own reported 12 V voltage, `ancillary_battery_voltage` in `mg_saic` 1.3.0):**
- Car parked in a garage, not driven and not charging, traction battery at ~68 % (≈ 42 of 62 kWh).
- Reported 12 V voltage over ~54 h: 11.9 → 11.4 → 11.1 → 10.6 → **9.3 V**, a steady ~0.1 V/h decline
  on the last day. The car then stopped answering the API and needed a jump start.
- **No top-up from the traction battery at any point**: a DC-DC top-up would show as a jump to ~13.5–14 V
  (what the car reports when powered on). None appeared in 2 days.
- At rest the battery already read only 11.5–12.0 V on the previous days, even after driving.
- The integration was polling the parked car **every 30 min (48×/day)** — the integration's default
  `update_interval`. Its own source notes that each idle poll is a live "ask the car" request that wakes
  the car (another owner measured a 0.3–0.5 V dip per poll with an external monitor).

**Probable:** the drain is a mix of a weak 12 V battery, the car's own parked activity, and frequent API
wake-ups. The reported voltage is likely ~0.4 V lower than the real one (the integration documents a
vehicle-reported vs. calibrated-meter gap), so the real rest voltage was ~12.0–12.3 V: low, not empty.

**Open:** whether the S5 has a parked 12 V top-up at all, and at what threshold. MG publishes nothing; forum
replies disagree. No recall or software campaign for the S5's 12 V was found (the 2026 S5 recall concerns the
cooling-fan controller).

**What we changed:**
- Idle polling `update_interval` 30 → **1440 min** (once a day). Charging (5 min), powered-on (15 min) and
  after-shutdown (2 min, then 10 min) intervals are separate options and stay fast, so charge tracking and
  remote pre-heating still work. Parked data (SoC, lock state) was of no use to us between drives.
- The integration also has a **holiday mode** switch (12 h interval) for long absences.

If you poll an MG/SAIC car from Home Assistant: lengthen the idle interval, log the 12 V voltage, and have
the battery tested if it rests below ~12 V (reported). Owner threads on the same symptom:
[MGS5 12V battery dead](https://www.mgevs.com/threads/mgs5-12v-battery-dead.24319/),
[MGS5 12 Volt Battery Issues](https://www.mgevs.com/threads/mgs5-12-volt-battery-issues.24881/),
[12V battery drained in 48 hours whilst locked](https://www.mgevs.com/threads/12v-battery-drained-in-48-hours-whilst-locked.24616/),
[saic-python-client-ng #30](https://github.com/SAIC-iSmart-API/saic-python-client-ng/issues/30).

## "Which car is in the garage?" — don't trust parked-car telemetry (Nov 2026)

*Keywords: two EVs one garage, preconditioning automation, MG S5 exterior temperature stale, SAIC GPS frozen,
car cloud API mileage polling, pre-heat the car parked outside.*

Goal: pre-heat/defrost only the car parked **outside** on cold mornings (one garage space, two cars, a
distance sensor that only says "space occupied").

What did **not** work:
- **Car-reported exterior temperature (MG S5 / SAIC):** it freezes at the last value measured while driving.
  Parked in a 15 °C garage, the car kept reporting **0.0 °C** all night, even right after a forced data refresh
  in the morning. The logic "car colder than the garage → it's outside" therefore always says "outside".
- **Car GPS (MG S5):** frozen at the last fix before entering the garage — it placed the car **2.8 km** away.
- **Arrival order from odometer changes:** cloud integrations push the odometer only every ~15 min while
  driving. On 3 of 7 evenings both cars arrived within that window; once it would have picked the wrong car.
- **The other car's GPS when parked:** all positions within 2–8 m of the house centre — the same as GPS noise, so garage
  vs. driveway can't be separated.
- "Plugged in" is not "in the garage": the charging cable reaches the driveway.

What we do now: **pre-heat both cars** below 10 °C (defrost below 3 °C). The one in the garage heats for
nothing (a few tenths of a kWh); no car is ever left frozen, and the MG no longer needs a morning wake-up
poll (good for its 12 V). Presence check for the MG: GPS "home" **or** charging cable connected **or** last
known position < 10 km (because the GPS freezes). Reliable identification would need a plate-reading camera
or a per-car BLE tag.

## One charger, two cars: identifying the car on the cable

Each car only reports "a cable is connected" — also when it is charging at a public station. To pick the
right car's state of charge for the overnight ramp, our agent compares the power the car reports with the
power the charger measures, and drops a car whose numbers don't match (case seen: one car at 9.9 kW on a
public charger while the other drew 4.1 kW at home).

Pitfall: the car's reported power comes from its cloud and can be **over an hour old** (3.4 kW reported while
the charger delivered 5.9 kW). With a 2 kW tolerance, the only car plugged in was rejected every other cycle,
and the ramp fell back to the 6 A default → current see-sawing 6 ↔ 13 A every 20 min. Fix: 4 kW tolerance
when only one car reports a cable, 2 kW when both do.

Also worth doing the arithmetic before blaming the ramp: with a 7 kW site cap and the house drawing its
share, the car got at most **~5.9 kW**; 6 h of off-peak × 5.9 kW ≈ 35 kWh at the wall ≈ 32 kWh in the battery.
From 33 % on a 77 kWh battery that is ~74–75 % — the 80 % target was out of reach whatever the control did.

## One month of smart-meter data: what actually costs money (Nov 2026)

*Keywords: smart-meter data hub export, 15-minute load profile, Grundlast, talon de consommation, standby power,
heat pump circulation pump watts, VMC/MVHR power, time-of-use EV charging share.*

30 days (29/10–27/11) from the grid operator's 15-min export, house with a gas boiler and two EVs on one charger:

| Use | Energy | Off-peak | Normal | Peak |
|---|---|---|---|---|
| EV charging (15-min power > 3.3 kW, minus 0.4 kW house) | ≈ 543 kWh | **69 %** | 23 % | 8 % |
| House without EVs | ≈ 470 kWh (15.7 kWh/day) | 25 % | 40 % | 35 % |

- **Reference power:** the operator moved the house from 3 to 7 kW automatically (yearly review). With the
  charger capped near 7 kW the month's highest 15-min value was 6.43 kW → **zero excess-power charge**.
- **EV timing is nearly solved:** the 166 kWh charged outside off-peak cost ~7.6 € more than off-peak would have.
- **The night baseload is the real lever:** 02:00–05:00 the house never drops below **0.25–0.31 kW**
  (median 0.30). That is ~7 kWh/day — almost half the non-EV use — ≈ 2,600 kWh/year. Every 100 W removed
  around the clock saves ~875 kWh/year.
- **It is not the ventilation or the heating pumps.** The MVHR unit draws 34 W at night (its own power
  reading). The heating pumps were measured by accident: with the boiler input set to "room thermostat",
  each burner block stops both the boiler pump and the mixer-circuit pump. Comparing night power with
  blocks vs without (5 nights, EV charging excluded, ~400–500 min each): 10th percentile 277 vs 281 W,
  median 315 vs 319 W. **Both high-efficiency pumps together ≈ 5–10 W.** The remaining ~200 W is
  elsewhere (network/IT, fridge, standby loads) — next step is a breaker-by-breaker night test.
- Pitfall: the gas operator's **hourly** export labels the unit "kWh" but the values are **m³** (they sum to the daily m³ export).
