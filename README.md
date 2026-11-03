# smart-home-notes

Field notes from one DIY smart home: what was measured, what worked, what did not.
Written so that the next person — or the next AI assistant — searching for the same problem finds a
tested answer instead of a forum thread that ends without a solution.

Everything here was tried on real hardware. Each page says what is **proven**, what is **probable** and
what is still **open**. Dates are given so you can judge how current a finding is.

## Made with AI — which is why we share

These findings were made **working together with AI assistants**. They could only find answers because
other people had published theirs: a GitHub repo with Wolf telegram numbers, an open-source parameter
database, a forum user who came back to post what fixed his boiler. Shared knowledge is what makes an AI
search succeed; unshared knowledge is invisible to it. So we publish ours — numbers, dates and failures
included. More in [ai-agent.md](ai-agent.md).

*FR — Trouvé en travaillant avec l'IA, grâce à ce que d'autres avaient partagé : nous partageons à notre tour.*
*DE — Mit KI erarbeitet, dank dem, was andere geteilt haben — deshalb teilen wir weiter.*

## The house in one paragraph

Low-energy detached house (energy class A, built ~2020, ~150 m²) in Western Europe. Underfloor heating in
a heavy concrete slab, one zone per room, driven by KNX valve actuators. Heat from a small gas condensing
boiler (Wolf CGB-2-14) with a Wolf BM-2 controller and an MM mixer module; 300 L hot-water tank with solar
thermal. Balanced ventilation with heat recovery (Komfovent). Home Assistant OS on a mini PC is the hub;
a small AI agent on a separate server adjusts heating, shutters and EV charging. Details:
[house overview](house-overview.md).

## Pages

| Topic | Page | Headline finding |
|---|---|---|
| Gas boiler on eBUS | [wolf-cgb2-ebus.md](wolf-cgb2-ebus.md) | Wolf CGB-2 short cycling every 7–8 min that ignores the anti-cycling timer (HG09): the burner is stopped by an internal safety after ~15 s of soft-start because boiler water flow is too low. Raising minimum pump speed **HG16** fixed most cycles. Full telegram-number table, write CRC, live burner-status polling. |
| KNX (MDT) + Home Assistant | [knx-home-assistant.md](knx-home-assistant.md) | Writable setpoints on MDT AKH, the "standby shifts comfort base" trap, line-coupler filter tables, listening to the bus, shutter position feedback, the `_2` unique_id trap. |
| Home Assistant pitfalls | [home-assistant-pitfalls.md](home-assistant-pitfalls.md) | `for:` triggers that never fire, YAML silently ignored after import into a config entry, `last_updated` vs `last_changed`, OS updates that wait for a manual reboot. |
| Mini-PC host stability | [haos-nvme-apst-reboots.md](haos-nvme-apst-reboots.md) | Random reboots 3–4×/day = NVMe SSD (Samsung PM961) dropping out in deep power saving (APST) + hardware watchdog. One kernel parameter, 44+ days uptime since. |
| Thermal behaviour & sensors | [thermal-and-sensors.md](thermal-and-sensors.md) | Measured time constants of a heavy-slab house, outdoor-sensor sun bias, open-window detection without contacts. |
| EV charging & energy | [ev-and-energy.md](ev-and-energy.md) | Why a charger with a site power cap pauses ("suspended EVSE") instead of slowing down; 6 A/phase floor; a car-climate API whose "fan speed" is really a mode byte. |
| Ventilation (MVHR) | [ventilation-komfovent.md](ventilation-komfovent.md) | Filter clogging stuck at 100 % + permanent W1 alarm = filter calibration never run. |
| Garage parking assistant (DIY) | [garage-parking-sensor.md](garage-parking-sensor.md) | ESP32 + waterproof ultrasonic + 16×16 LED panel: arrow → STOP, two-supply wiring, firmware that does not depend on HA automations. |
| The Home Assistant installation | [home-assistant-setup.md](home-assistant-setup.md) | Host, Apps, integrations, dashboard, InfluxDB 1→2 migration with a local App. |
| AI agent & method | [ai-agent.md](ai-agent.md) | An LLM agent running heating/ventilation/EV with hard guardrails, and how the AI-assisted investigations are run. |

## Credits

This builds on public work by others — thank you:
- [Birger90/Wolf-cbg2-ebus-](https://github.com/Birger90/Wolf-cbg2-ebus-) — CGB-2 / BM-2 ebusd definitions and protocol notes
- [zivillian/ism7mqtt](https://github.com/zivillian/ism7mqtt) — Wolf SmartSet parameter database
- [dktzde/ebusd-configuration-wolf](https://github.com/dktzde/ebusd-configuration-wolf) and [john30/ebusd](https://github.com/john30/ebusd)
- Forum threads on haustechnikdialog.de, heizungsforum.de and community.home-assistant.io (linked on each page)

## Licence and disclaimer

Text: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Code snippets: MIT.
Changing boiler parameters can void warranties or cause faults; gas-related settings are meant for a
qualified installer. Everything here is shared as-is, without any guarantee. Write one parameter at a
time, note the original value, and measure before and after.
