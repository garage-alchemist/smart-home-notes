# House overview

A short description of the system the other pages refer to. Deliberately generic.

## Building

- Detached house, energy class A, built ~2020, ~150 m² heated, three levels.
- Underfloor heating in a heavy concrete slab, 7–8 zones, one per room.
- Very high thermal inertia: see [thermal-and-sensors.md](thermal-and-sensors.md)
  (diurnal envelope time constant ≈ 55 h; free-running decay ≫ 1 week).

## Heating and hot water

| Part | Model / role |
|---|---|
| Boiler | Wolf **CGB-2-14** gas condensing boiler (14 kW, ~1.9 kW nominal minimum), internal modulating pump |
| Controller | Wolf **BM-2** (outdoor-reset curve), eBUS address 0x35 |
| Mixer module | Wolf **MM** — mixing valve + floor-circuit pump for the slab |
| Tank | 300 L dual-coil tank with solar thermal |
| Room control | KNX room push-buttons with sensors + MDT **AKH** heating actuators (valve %, setpoints) |

The boiler only sees the mixer module. Room valves (KNX) and the mixer valve decide how much heat is
actually taken — which matters a lot for short cycling: see [wolf-cgb2-ebus.md](wolf-cgb2-ebus.md).

## Home automation

- **Home Assistant OS** on a small office mini PC (Intel, NVMe SSD).
- **ebusd** (Home Assistant App) + an Ethernet eBUS adapter → MQTT discovery (~90 entities).
- **KNX** via an IP interface (tunnelling); ETS project imported into Home Assistant.
  Actuators: MDT heating (AKH), shutters (JAL), dimmers (AKD). A Gira G1 touch panel on another line.
- Ventilation: Komfovent unit over Modbus.
- Energy: Smappee EV Wall charger with whole-house current clamps; smart meter P1 port.
- Data: InfluxDB 2 + Grafana (local App).
- An LLM-based agent on a separate server (reachable over a private VPN) reads ~70 exposed entities and
  adjusts setpoints, shutters and EV charge current, with dry-run mode and hard safety limits.

## Working principles that paid off

1. **Read before you write.** Every bus write is preceded by a read of the original value and followed by
   a read-back. Most "it does not work" moments were measurement mistakes, not device faults.
2. **One change at a time**, with a dated before/after measurement.
3. **Poll fast when diagnosing.** A Home Assistant sensor updated every few minutes hides anything shorter
   (we lost days believing a 7.9 min "cycle" that was just the polling interval).
4. **Distrust a constant.** A sensor that never moves (100.0 % for 14 weeks, 32768 forever) is a
   missing value, not a measurement.
