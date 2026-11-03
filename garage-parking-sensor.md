# DIY garage parking assistant: ultrasonic sensor + 16×16 LED panel (ESPHome)

*Keywords: ESPHome parking sensor, JSN-SR04T AJ-SR04M, WS2812 16x16 matrix, garage stop light,
Hörmann SupraMatic E4 HCP bridge, aide au stationnement garage, Einparkhilfe Garage.*

A green arrow appears when the garage door opens, turns orange then red as the car gets closer, and a
blinking **STOP** shows at the right spot. The panel switches itself off once the car stands still.
Total hardware ≈ 40 € plus the door bridge. Full config: [esphome/parking-sensor.yaml](esphome/parking-sensor.yaml).

## Hardware

| Part | Notes |
|---|---|
| ESP32 DevKit (WROOM-32) + "ESP32-IO" sensor shield | every GPIO as S/V/G trio, jumper selects 3.3 V/5 V on the V rail |
| AJ-SR04M ultrasonic (JSN-SR04T family, waterproof probe) | default **mode 0** (pad R19 empty) = Trig/Echo, works with ESPHome `ultrasonic`, nothing to solder. Minimum range ~20–25 cm |
| WS2812 16×16 panel (256 LEDs) | |
| 5 V 6 A power supply | **LED panel only** |
| USB 5 V charger | ESP32 + shield + sensor |

## Wiring — two supplies, one common ground

```
USB 5V ──► ESP32 (USB-C) ── shield, jumper on 5V ──► sensor 5V / GND
5V 6A PSU ──► panel +5V / GND            (never through the shield: ~4 A)
PSU GND ─────────── shield G             (COMMON GROUND, mandatory for DIN)
                                         (never join the two +5V rails)
GPIO5  ──► sensor Trig
GPIO18 ◄── sensor Echo via divider 1 kΩ (Echo→mid) / 2 kΩ (mid→GND)  → 3.3 V
GPIO13 ──► 330 Ω (at the panel end) ──► panel DIN
```

Check with a multimeter before powering: no short +5 V/GND, a V pin reads ~5 V.

⚠️ **Power budget**: 256 LEDs full white ≈ 15 A, far beyond 6 A. The firmware only draws single-colour
symbols and caps brightness at 55 % (red full panel would be ~4 A).

## Mounting trick: ceiling sensor, inverted logic

The sensor is on the ceiling, tilted toward the incoming car, so the **distance decreases as the car
approaches** (opposite of a wall-mounted sensor). Calibrate with real measurements:
- empty garage: 2.50 m stable → `OFF = 2.45`
- car at the right position: `TARGET = 1.806`
- bands: green > TARGET+0.19, orange, red ≤ TARGET+0.12, STOP ≤ TARGET+0.05

## Firmware lessons

1. **Out of range must mean "no car"**. The ultrasonic timeout returns `nan`; `filter_out: nan` kept the
   last distance, so the panel stayed green forever. Map it instead:
   `lambda: return isnan(x) ? 9.0 : x;` then a short median.
2. **Don't drive the panel from a Home Assistant automation.** A brief `unavailable` blip on the door
   entity made the "door opening" trigger miss and left the panel dark. The firmware now reads the door
   state itself (`text_sensor: platform: homeassistant`) and only draws while the door is
   open/opening/closing — robust to HA restarts and Wi-Fi blips.
3. **Switch off when parked**: track movement (> 4 cm) and go dark after 30 s without movement,
   otherwise STOP blinks all evening.
4. A rotated panel is handled in software: rotate the (u, v) coordinates (`ROT = 90`), plus
   `SERPENTINE` / `FLIP_Y` constants for the panel's wiring order — no need to remount.
5. `binary_sensor` "garage occupied" = distance < 1.90 m, useful for presence logic.

## The door itself

Hörmann SupraMatic E **series 4** opener, controlled locally through an ESP32-S3 board that emulates the
UAP1 accessory on the motor's HCP bus (tynet "HCP Bridge", ESPHome firmware, flashed from esp.tynet.eu).
Gives a native `cover` with live position. Two traps: flash the **right series variant** (S3 vs S4
firmware differ), and after connecting any bus accessory run the motor's **bus scan** (menu 37, option 01,
long PROG until "bS-Scan") or the motor ignores the board.
