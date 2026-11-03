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
