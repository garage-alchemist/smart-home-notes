# Home Assistant pitfalls we actually hit

## 1. A `for:` state trigger never fires when the state is already there

```yaml
trigger:
  - platform: state
    entity_id: vacuum.x
    to: unavailable
    for: "02:00:00"   # never fires if the entity is ALREADY unavailable when the automation loads
```

A state trigger needs a transition. A monitoring alert stays silent exactly during the ongoing failure.
Pattern that works — poll, and let the *condition* measure the duration:

```yaml
trigger:
  - platform: time_pattern
    minutes: /30
condition:
  - condition: state
    entity_id: vacuum.x
    state: unavailable
    for: "02:00:00"          # here `for` evaluates the current duration
  - condition: template      # anti-repeat
    value_template: >-
      {{ this.attributes.last_triggered is none
         or (now() - this.attributes.last_triggered).total_seconds() > 43200 }}
```

## 2. YAML config silently ignored after HA imported it into a config entry (InfluxDB, 2026.9)

After upgrading, HA imported the `influxdb:` YAML block into a **config entry** (`source: import`).
From then on the YAML is **no longer read**: change `host:`, run `check_config`, restart — nothing changes,
no explicit error. The only hint, with `logger: homeassistant.components.influxdb: debug`:

```
Config entry 'homeassistant (...)' for influxdb integration not ready yet
```

Fix: stop core, edit `.storage/core.config_entries` (parse it with Python — `grep` on the JSON fails because
spacing varies), start core. Probably applies to other YAML-to-entry migrations.

## 3. `customize.yaml` does not rename MQTT-discovery entities

The entity registry wins. Rename via WebSocket `config/entity_registry/update` (`entity_id`, `name`).

## 4. ebusd App publishes values but no MQTT discovery

Without `--mqttint=/config/mqtt-hassio.cfg` ebusd publishes `ebusd/<circuit>/<msg>` but zero
`homeassistant/#` configs. Also enable `definition-retain = 1` and `def_global-retain = 1` in that file so
discovery survives HA restarts (here: 0 → 90+ entities).

## 5. HAOS update "installed" but still running the old version

Since HAOS 18.x the Supervisor writes the new slot, logs *"reboot required"* and **waits**. The update
entity already shows the new version. Check `/proc/cmdline` (`rauc.slot`) or `version_pending` in
`/os/info`, then reboot the host.

## 6. Energy dashboard picked the wrong sensors

The UI wizard chose the charger's own meter as grid import and an "always on" sensor as live power. Check
`energy/get_prefs`. In 2026.5 the `energy/save_prefs` grid schema is **flat** (`stat_energy_from`,
`stat_rate`, `power_config` on the source), not the older `flow_from[]` list.

## 7. Measuring things right

- Don't infer an event rate from a sensor that is only refreshed every N minutes (we "found" a 7.9 min
  boiler cycle that was the polling interval).
- A value that never moves for weeks is a sentinel, not a measurement.
- "Writes still work after the upgrade" proves nothing if the old and new config hold the same value.
