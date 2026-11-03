# KNX (MDT actuators) with Home Assistant — traps and fixes

*Keywords: MDT AKH-0800.03 AKH-0400.03 heating actuator, setpoint Home Assistant, Basis-Komfort,
Standby verschiebt Komfortsollwert, Linienkoppler Filtertabelle, Gira G1, knx telegram trigger,
position_state_address, unique_id _2.*

## 1. Make room setpoints writable (MDT AKH)

Symptom: changing a KNX `climate` setpoint in Home Assistant does nothing on the bus.

- A `climate` with only `target_temperature_state_address` is read-only. You need
  `target_temperature_address`.
- MDT AKH channels offer two setpoint mechanisms: **absolute** (DPT 9.001 "preset setpoint") and
  **shift** (DPT 1.007, ±0.5 K pulses, actuator is master). Many projects only expose the shift object,
  which does not map to a native HA climate.
- Fix per room in ETS: enable the absolute setpoint object on the channel (the shift object can stay; wall
  push-buttons keep working), give it a group address, partial download, then add
  `target_temperature_address` in HA. Add `temperature_step: 0.5` to match the actuator.

Validated by watching the bus: HA `Outgoing 3/3/x = 22.0` → actuator echoes the new value on its status GA.

## 2. ⚠️ Never write a setpoint while the actuator is in Standby/Economy

With MDT AKH set to *"setpoints dependent on comfort base"* (Standby = Comfort − 2 K) and *"reset comfort
after mode change"* **inactive**, writing an absolute setpoint during Standby **shifts the comfort base**.
Lived example: 21.5 °C written in Standby → at the next Comfort switch the room targeted **23.5 °C**.

Rule implemented in our agent: no setpoint write if the zone preset is standby / economy /
building protection.

Related: when a room drops to Standby for a short absence, the MDT PI controller loses its integral (valve
100 % → 0 % → restarts at ~20 %) and takes hours to recover in a slab. We only switch to Standby after
8 h of absence.

## 3. HA writes that never reach a device on another line (coupler filter)

Topology: HA on line 1.1 (IP tunnel), a Gira G1 panel on line 1.0, a KNX IP router `1.1.0` as coupler.
Values sent by HA on group 5 (CO₂, wind) never reached the G1; group 3 did.

- Cause: the coupler's filter table only routes GAs that ETS sees used on both sides. A GA used only by the
  G1 (line 1.0) is blocked when it comes from line 1.1.
- Fix: add a **dummy device** on line 1.1 in ETS (e.g. `1.1.250`), link the GAs to it, **download only the
  coupler's filter table**. Quick alternative: set group routing of the coupler to "route all".
- Second trap: the G1 decoded the CO₂ GA as **DPT 9.008** (2-byte float), not 7.001 as the ETS project
  claimed (740 ppm displayed as "7"). Send `2byte_float` from HA.

## 4. Triggering an automation from a raw group address

What did not work (HA 2026.5): `platform: knx.telegram` on a destination never fired; adding a wrongly
nested `event:` block to `knx:` **broke the whole KNX integration** (`Action knx.send not found`).

What works: declare a `knx.sensor` on the GA and trigger on its state.

```yaml
knx:
  sensor:
    - name: "Shutter scene"
      state_address: "2/5/0"
      type: "scene_number"   # DPT 17.001: byte 4 is decoded as scene "5"
```

Needs a restart (not just a reload). Same value twice = no state change = no trigger.

## 5. Shutter positions frozen in HA

Two conditions: `position_state_address` (DPT 5.001) in HA **and**, in ETS, the actuator's
"current position" object set to **transmit after movement has stopped** (flags C/T/L), then download.
Without the ETS part HA reads the position once at boot and then stays frozen.
KNX 0 % = open ↔ HA 100 = open — HA inverts natively.

## 6. The `_2` entity trap (unique_id changes)

For KNX `climate` and `cover`, the unique_id is built from the addresses (for covers:
`{move_long}_{position_address}`). Adding an address therefore creates a **new entity `xxx_2`** and leaves
the old one "unavailable" — dashboards and automations still point to the old one.

Fix: stop HA core, edit the unique_id in `.storage/core.entity_registry` to the future value, start again
(HA reuses the entity), or afterwards delete the orphan and rename `_2` via the WebSocket
`config/entity_registry/update`.

## 7. The ETS project is readable from HA — use it

`/config/.storage/knx/knx_project.json` contains group addresses, devices and communication objects with
flags (`write` = listens, `transmit` = sends). Before guessing what a group address drives, look it up:

```python
d = json.load(open('/config/.storage/knx/knx_project.json'))['data']
d['communication_objects']['1.1.5/O-xx_R-yy']  # name, flags, group_address_links
```

Diagnostics traps:
- `.storage/knx/telegrams_history.json` is flushed periodically — **not real time**. For live telegrams
  subscribe via WebSocket `knx/subscribe_telegrams`.
- "KNX bus did not respond in time" at startup is normal for command GAs without the read flag.
- To know whether something moved, use `last_changed`, never `last_updated` (attribute refreshes and KNX
  resyncs update the latter).
