# The Home Assistant installation

Migrated from Jeedom (on a Raspberry Pi 4) to Home Assistant OS in 2026. Everything local where possible,
open source preferred over cloud.

## Host

- Office mini PC (Intel, 8th gen, NVMe SSD) running **Home Assistant OS**. HA 2026.10.
- One kernel parameter was needed for stability: see [haos-nvme-apst-reboots.md](haos-nvme-apst-reboots.md).
- Backups to a NAS over SMB (the NAS file API failed for archives > 2 GB).

## Apps (formerly "add-ons")

| App | Role |
|---|---|
| ebusd | Wolf boiler on eBUS via an Ethernet eBUS adapter, MQTT discovery |
| Mosquitto | MQTT broker |
| ESPHome | DIY sensors (parking assistant, CO₂) |
| InfluxDB 2 (local build) + Grafana | long-term history (2.5 years imported from Jeedom) |
| AppDaemon | available; the AI agent ended up running outside HA instead |
| VPN client | lets an external server reach the HA API privately |
| SSH, VS Code | maintenance |

Note: the community InfluxDB 1.x App was archived in 2026. We built a small **local App** around the
official InfluxDB 2.7 image and migrated with `influxd upgrade --v1-dir` (keeps DBRP mappings, so Grafana
InfluxQL dashboards and HA's `influxdb:` writer kept working).

## Integrations (loaded)

KNX · MQTT (ebusd) · ESPHome · Komfovent (ventilation, Modbus) · Smappee + Smappee EV (HACS) · Netatmo ·
Shelly (garage/cellar temperature & flood sensors) · UniFi Network / Protect · Sonos · sleep-tracking mat
(→ wake-up routine) · Home Connect · Ecovacs · MG SAIC (car) · Synology DSM · alarm system ·
met.no · MCP Server · Anthropic (conversation) · HACS.

## Dashboard

Mushroom cards + a glassmorphism theme (Frosted Glass, needs card-mod). One view per floor, plus heating
valves ("real heat demand") on the home view — far more useful than setpoints in a slab house.

## Data model choices that helped

- One trusted outdoor temperature template choosing the best source (see
  [thermal-and-sensors.md](thermal-and-sensors.md)).
- Entities renamed "Subsystem — Measurement" through the entity registry (WebSocket), not customize.yaml.
- Fast diagnostic polling done outside HA (`ebusctl read -f`, KNX WebSocket telegram subscription), because
  HA sensors are only as fast as their polling.

## The AI layer

See [ai-agent.md](ai-agent.md).
