# Wolf CGB-2: short cycling that ignores the anti-cycling timer (HG09) — diagnosed over eBUS

*Keywords: Wolf CGB-2-14, CGB-2-20, CGB-2-24, BM-2, MM mixer module, ebusd, Taktsperre, Brennertaktsperre
wird ignoriert, taktet, Softstart, Gradientenüberwachung, HG09, HG16, HG39, short cycling, court-cycle,
chaudière Wolf démarre toutes les 7 minutes.*

Status: **diagnosis proven (03/11), fix (HG16) under evaluation.** Updated as data comes in.

## TL;DR

- Symptom: burner restarts every **7–8 min**, ~120–160 starts/day, although the anti-cycling timer
  **HG09 = 15 min** (verified on the BM-2 display).
- Polling the boiler's **burner status** (telegram 10068) every 3–4 s showed the real sequence:
  pre-purge → ignition → **soft-start for only ~15 s** → post-purge → **"Off" with no anti-cycling lock**.
  During those 15 s the flow sensor did not move at all, yet ~2 min *after* the flame went out the flow
  temperature jumped by **+21 K** (27.7 → 49 °C).
- Meaning: almost no water was flowing through the heat exchanger. The ~105 kJ released in 15 s
  (≈ 7 kW soft-start × 15 s) stayed in ~1.3 L of water (+19 K calculated, +21 K measured). An internal
  safety (most likely the temperature-gradient monitor) cut the flame. That kind of stop is **not** a
  normal control shutdown, so **HG09 never applies**. This is why raising HG09 changed nothing.
- Fix under test: **HG16 (minimum boiler pump speed) 50 % → 80 %**. First cycles afterwards: flames of
  40 s and 3 min 50 s ending with a normal control stop **followed by a real 15 min lock**, instead of
  15 s safety stops. In the first 45 min, 2 of 4 cycles were still 15 s stops → evaluation continues.

## The system

Wolf CGB-2-14 + BM-2 + MM mixer module feeding a heavy concrete-slab underfloor circuit; room valves on
KNX. In mild weather the mixer valve is nearly closed most of the time (the slab is at setpoint), so the
boiler loop has very little flow path. See [house-overview.md](house-overview.md).

## Step 1 — the parameters that did NOT solve it

| Change | Result |
|---|---|
| HG09 7 → 15 min | No effect: the interval stayed at 7–8 min |
| HG01 (burner hysteresis) 12 → 20 K | Starts/day dropped from ~150 to ~75–120, still cycling every 7–8 min most of the time |

Lesson: if the starts are spaced *less* than HG09 apart, the stops are not the ones HG09 is about.
The manual says it plainly: *"Nach jeder Brennerabschaltung **im Heizbetrieb** ist der Brenner für die
Zeit der Brennertaktsperre gesperrt"* — a safety stop is a different path.

⚠️ Also: the Home Assistant entity for the burner start counter (Hg91) was only refreshed by ebusd every
~7.9 min, which looked exactly like a 7.9 min cycle. Measure with direct `ebusctl read -f` polling.

## Step 2 — read the burner status (telegram 10068)

The community `08..hc.csv` lacks several useful messages. Wolf parameters are addressed by their
**SmartSet telegram number (TN)**, little-endian, in a `5022` read. The numbers below match the Wolf
SmartSet database extracted by [ism7mqtt](https://github.com/zivillian/ism7mqtt) (`parameter.xml` +
`converter.xml`, CGB-2 template) and [Birger90's definitions](https://github.com/Birger90/Wolf-cbg2-ebus-),
and were cross-checked on this boiler against parameters whose value is known from the display.

| Item | TN | ID in CSV (LE16) | Type | Read here |
|---|---|---|---|---|
| HG01 burner hysteresis | 34 | `2200` | /10, K | 20.0 |
| HG02 lower burner output | 346 | `5a01` | % | 30 |
| HG09 anti-cycling time | 323 | `4301` | min | 15 |
| HG13 input E1 function | 2651 | `5b0a` | enum | – |
| HG16 pump min | 341 | `5501` | % | 50 → 80 |
| HG17 pump max | 342 | `5601` | % | 100 |
| HG33 hysteresis run time | 33 | `2100` | min | 10 |
| HG39 soft-start time | 339 | `5301` | min | 3 |
| HG60 minimum hysteresis | 10808 | `382a` | signed /10, K | 7.0 |
| **Burner status** | **10068** | `5427` | enum | live |
| **Operating mode** | **10067** | `5327` | enum | live |
| Pump speed (actual) | 10071 | `5727` | % | live |

Lines added to `08..hc.csv` (reads only; the leading `00` byte is fine for reads):

```csv
r,,,Hg39,HG 39 - Zeit Softstart (0-10) TN 339,,,5022,005301,value,,UIN,,min,minutes
r,,,Hg33,HG 33 - Laufzeit Brennerhysterese (1-30) TN 33,,,5022,002100,value,,UIN,,min,minutes
r,,,Hg60,HG 60 - Minimale Schalthysterese Brenner (2-30) TN 10808,,,5022,00382a,value,,SIN,10,°C,temperature
r,,,Brennerstatus,Brennerstatus TN 10068,,,5022,005427,value,,UIN,0=Aus;1=Ruhekontakt;2=Vorspuelen;3=Zuenden;4=Stabilisierung;5=Ein;6=Ventilpruefung;7=Nachspuelen;8=Softstart;9=Taktsperre;10=Betrieb_ohne_Brenner;11=Abgasklappe;12=Stoerung;13=Gradienten_Ueberwachung;14=Gasdruck;15=Spreizung_hoch;16=Spreizung_KF,,
r,,,Betriebsart,Betriebsart der Therme TN 10067,,,5022,005327,value,,UIN,0=Test;1=Start;2=Frost_Heizkreis;3=Frost_Warmwasser;4=Schornsteinfeger;5=Kombibetrieb;6=Parallelbetrieb;7=Warmwasser;8=Warmwassernachlauf;9=Mindest_Kombizeit;10=Heizbetrieb;11=Nachlauf_Heizkreispumpe;12=Frostschutz;13=Standby;14=Kaskadenbetrieb;15=GLT_Betrieb;16=Kalibration;17=Kalibration_Heizbetrieb;18=Kalibration_Warmwasserbetrieb;19=Kalibration_Kombibetrieb;20=Warmwasser_Schnellstart;21=Externe_Deaktivierung;22=Durchfluss_gering,,
r,,,Pumpspeed,Drehzahl Heizkreispumpe TN 10071,,,5022,005727,value,,UIN,,%,percent
```

`ebusctl reload` is enough to load them — no App restart.

Polling loop used (run on the Home Assistant host; container name is install-specific):

```bash
while true; do l=$(date +%T)
  for p in Brennerstatus Betriebsart Pumpspeed Flowtemp Returntemp Flowtempdesired Hg91; do
    l="$l;$(docker exec <ebusd-container> ebusctl read -f -c hc $p | head -1 | cut -d' ' -f1)"
  done; echo "$l"; sleep 2
done
```

## Step 3 — what the status showed (HG16 = 50 %)

| Time | Burner status | Flow °C | Note |
|---|---|---|---|
| 09:23:57 | Vorspülen | 27.7 | flow just below setpoint 27.8 |
| 09:24:01 | Zünden | 27.7 | |
| 09:24:04 | **Softstart** | 27.7 | start counter +1 |
| 09:24:19 | Nachspülen | 27.7 | **flame lasted ~15 s**, flow unchanged |
| 09:24:33 | **Aus** | 27.7 | **no Taktsperre** |
| 09:26:16 | Aus | **49.0** | heat reaches the flow sensor ~2 min later |

Identical on three consecutive cycles (one of them with a valve test, *Ventilprüfung*, before post-purge;
in one the flow was even *below* setpoint when the flame stopped). Pump speed read 50–55 % = the HG16 floor.
No error code on the bus (E000). A 15 s flame is too short for the 3–4 s polling to catch a transient
"Gradienten Überwachung" state, so the exact internal trigger is inferred, not observed.

## Step 4 — the fix: HG16 50 → 80 %

| Cycle (HG16 = 80 %) | Flame | Stop | Then |
|---|---|---|---|
| 09:49 | ~40 s, flow sensor rising *during* the flame | control stop at ~50 °C (setpoint + HG01) | **Taktsperre exactly 15:01** |
| 10:05 | ~15 s | safety-type stop | Aus (no lock) |
| 10:12 | **~3 min 50 s**, reached *Ein* (modulating) | control stop | Taktsperre (15:02) |
| 10:31 | ~16 s | safety-type stop | Aus (no lock) |

The same fix is reported in a [haustechnikdialog thread](https://www.haustechnikdialog.de/Forum/t/260260/Wolf-CGB-2-14-taktet-58-800-x-in-26-000-Std-)
(CGB-2-14, underfloor heating, 58 800 starts in 3 years): HG16 55 % still dropped into the lock after
soft-start; HG16 80 % / HG17 100 % made it run properly.

Why it works: more flow through the exchanger carries the soft-start heat away instead of letting it
build up in 1.3 L of water. Why it may not be enough: if the mixer valve is nearly closed and the
hydraulics have no bypass/low-loss header, extra pump head only buys a little extra flow.

### Next levers, in order

1. HG09 longer (20–30 min) once the stops are normal control stops.
2. HG39 soft-start 3 → 1 min (some installers report it helps; the effect is installation-dependent).
3. HG02 back to the factory value for the 14 kW unit (26 %) if it was raised — **installer setting**,
   do not go below factory without a combustion (CO) check.
4. Use input E1 (HG13) to block the burner when the sum of room-valve openings is low —
   reported to cut starts from 2.4/h to 0.4–0.8/h in the same thread.

## Writing a parameter (5023 + CRC)

ebusd's CSV writes put `00` in the first ID byte and the boiler **silently ignores them**. That byte is a
CRC-8 (polynomial 0x5C, "shift-in", init 0) over the rest of the payload
(TN LE16 + value LE16 + suffix `9d010000`) — see ebusd issue #1696 and dktzde's notes.

```python
def crc_wolf(data, poly=0x5C):
    c = 0
    for b in data:
        for i in range(8):
            top = c & 0x80
            c = ((c << 1) | ((b >> (7 - i)) & 1)) & 0xFF
            if top:
                c ^= poly
    return c

def telegram(id_hex, value, suffix="9d010000"):
    d = bytes.fromhex(id_hex) + int(value).to_bytes(2, "little") + bytes.fromhex(suffix)
    return f"035023{len(d) + 1:02x}{crc_wolf(d):02x}{d.hex()}"

telegram("4301", 15)  # HG09 = 15  -> 035023097443010f009d010000 (verified on the display)
telegram("5501", 80)  # HG16 = 80  -> 03502309dc550150009d010000
```

Procedure used (one parameter at a time):
1. Add `--enablehex` to the ebusd App `commandline_options`, restart the App.
2. Read the parameter, write `ebusctl hex <telegram>`, read back.
3. Remove `--enablehex`, restart, read back again (the value survives).
4. Check it on the BM-2 display at least once — here HG09/HG02/HG16 on the display matched the bus.

## Things that turned out to be dead ends on this boiler

- `Hg73` (ionisation current), `Hg74` (fan speed), `Hg06`: always 32768 (= no value) on this firmware,
  48 h of polling across 16 burner starts. Don't build sensors on them.
- `binary_sensor` "burner on" derived from slow polls: misses most short flames. Use ΔT flow−return or the
  burner status above.

## Sources

- Wolf CGB-2 installation manual (parameter table HG01–HG61, HG04 power table)
- [Birger90/Wolf-cbg2-ebus-](https://github.com/Birger90/Wolf-cbg2-ebus-) — `hc.csv`, `docs/parameter_vollstaendig.md`, `docs/protokoll.md`
- [zivillian/ism7mqtt Resources](https://github.com/zivillian/ism7mqtt/tree/main/src/ism7mqtt/Resources)
- [dktzde/ebusd-configuration-wolf](https://github.com/dktzde/ebusd-configuration-wolf)
- [haustechnikdialog: Wolf CGB-2-14 taktet 58.800 x in 26.000 Std.](https://www.haustechnikdialog.de/Forum/t/260260/Wolf-CGB-2-14-taktet-58-800-x-in-26-000-Std-)
- [haustechnikdialog: Wolf CGB2-14 Taktsperre sofort in Softstartphase](https://www.haustechnikdialog.de/Forum/p/2567469)
- [heizungsforum: Gradientenüberwachung Wolf CGS-2](https://www.heizungsforum.de/threads/gradientenueberwachung-wolf-cgs-2-20-160l.14423/)
