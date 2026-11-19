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
