# Wolf CGB-2: short cycling that ignores the anti-cycling timer (HG09) — diagnosed over eBUS

*Keywords: Wolf CGB-2-14, CGB-2-20, CGB-2-24, BM-2, MM mixer module, ebusd, Taktsperre, Brennertaktsperre
wird ignoriert, taktet, Softstart, Gradientenüberwachung, HG09, HG16, HG39, short cycling, court-cycle,
chaudière Wolf démarre toutes les 7 minutes.*

Status (25/11): **mechanism proven** (too little water flow through the boiler → safety stop after ~15 s,
no anti-cycling lock). Root cause of the low flow: **heating side** (mixer valve nearly closed, no bypass) —
a hot-water charge on the same boiler ran a single 13.5 min flame, so the exchanger itself is fine.
**Working fix since 22/11: block the burner through input E1 on a 45/30 min cadence** → starts per day
roughly halved (≈ 90 → 40–50) and flames ~2–10× longer, with no measurable comfort loss
(see "Step 5"). **Since 28/11 E1 is set to "burner block" (BOB)**, so the floor pumps keep running
during blocks (see "Step 6"). Updated as data comes in.

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
  15 s safety stops. But over the next 1.5 h, **7 of 9 cycles were still 15 s stops** → pump speed alone is not
  enough; something restricts the flow (see "Where is the flow blocked?").

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
| Burner controller firmware | 410 | `9a01` | signed /100 | 2.20 |

Lines added to `08..hc.csv` (reads only; the leading `00` byte is fine for reads):

```csv
r,,,Hg39,HG 39 - Zeit Softstart (0-10) TN 339,,,5022,005301,value,,UIN,,min,minutes
r,,,Hg33,HG 33 - Laufzeit Brennerhysterese (1-30) TN 33,,,5022,002100,value,,UIN,,min,minutes
r,,,Hg60,HG 60 - Minimale Schalthysterese Brenner (2-30) TN 10808,,,5022,00382a,value,,SIN,10,°C,temperature
r,,,Brennerstatus,Brennerstatus TN 10068,,,5022,005427,value,,UIN,0=Aus;1=Ruhekontakt;2=Vorspuelen;3=Zuenden;4=Stabilisierung;5=Ein;6=Ventilpruefung;7=Nachspuelen;8=Softstart;9=Taktsperre;10=Betrieb_ohne_Brenner;11=Abgasklappe;12=Stoerung;13=Gradienten_Ueberwachung;14=Gasdruck;15=Spreizung_hoch;16=Spreizung_KF,,
r,,,Betriebsart,Betriebsart der Therme TN 10067,,,5022,005327,value,,UIN,0=Test;1=Start;2=Frost_Heizkreis;3=Frost_Warmwasser;4=Schornsteinfeger;5=Kombibetrieb;6=Parallelbetrieb;7=Warmwasser;8=Warmwassernachlauf;9=Mindest_Kombizeit;10=Heizbetrieb;11=Nachlauf_Heizkreispumpe;12=Frostschutz;13=Standby;14=Kaskadenbetrieb;15=GLT_Betrieb;16=Kalibration;17=Kalibration_Heizbetrieb;18=Kalibration_Warmwasserbetrieb;19=Kalibration_Kombibetrieb;20=Warmwasser_Schnellstart;21=Externe_Deaktivierung;22=Durchfluss_gering,,
r,,,Pumpspeed,Drehzahl Heizkreispumpe TN 10071,,,5022,005727,value,,UIN,,%,percent
r,,,Firmware,Firmware Feuerungsautomat TN 410,,,5022,009a01,value,,SIN,100,,
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
| 10:38, 10:46, 10:53, 11:01, 11:08 | 13–16 s each | safety-type stop | Aus (no lock), restart ~7.3 min later |

What separates good from bad cycles (same 80 % pump command): in the good ones the flow sensor starts
rising **7–11 s after ignition**; in the bad ones it does not move during the flame and the heat shows up
15–30 s *after* it. So the **actual flow varies from cycle to cycle** while the pump command does not.
During all short cycles the mixer motor was idle and the mixed flow was at its setpoint (slab satisfied).

During the good 10:12 cycle: flow 41.5 °C, return 25 °C (ΔT ≈ 16.5 K) at roughly 3.5 kW →
**≈ 180 L/h at 80 % pump**, far below what the internal pump delivers into an open circuit.

The same fix is reported in a [haustechnikdialog thread](https://www.haustechnikdialog.de/Forum/t/260260/Wolf-CGB-2-14-taktet-58-800-x-in-26-000-Std-)
(CGB-2-14, underfloor heating, 58 800 starts in 3 years): HG16 55 % still dropped into the lock after
soft-start; HG16 80 % / HG17 100 % made it run properly.

Why it works: more flow through the exchanger carries the soft-start heat away instead of letting it
build up in 1.3 L of water. Why it may not be enough: if the mixer valve is nearly closed and the
hydraulics have no bypass/low-loss header, extra pump head only buys a little extra flow.

### Where is the flow blocked?

| Hypothesis | Where | Remedy |
|---|---|---|
| A. Mixer valve nearly closed and no bypass / low-loss header | heating side | bypass or differential-pressure valve; or block the burner via E1 when room valves take little water |
| B. Fouled heat exchanger / sludge (untreated fill water) | inside the boiler | exchanger cleaning, flushing, treated water, magnetic dirt separator |

Hypothesis B comes from a [haustechnikdialog thread](https://www.haustechnikdialog.de/Forum/t/228535/Wolf-CGB-2-14-von-Softstart-in-Taktsperre)
(CGB-2-14, underfloor, 2016 house): raising the pump did nothing; the exchanger was found almost fully
clogged ("coffee-coloured" water) and the ionisation electrode slightly misaligned; cleaning + flushing +
treated water fixed it. Note their symptom differed (stuck at 64 % for minutes, typical of an ionisation
problem: at 27 % output the ionisation current should be > 50).

**Discriminating test, no change needed:** watch a hot-water (tank) charge. The diverter valve then sends
the water through a short open circuit that does not depend on the mixer. Long flames with a fast-reacting
flow sensor → exchanger fine → hypothesis A. Short 15 s flames during tank charging too → hypothesis B.

**Result of that test (03/11, 11:27–11:41):** one single burner start, **one continuous flame of 13.5 min**
(soft-start → *Ein* in ~12 s), flow sensor reacting within seconds, flow−return ΔT only **3–6 K** at the
same 80 % pump command (vs ~16 K on the heating side) → several times more flow. **The exchanger is fine;
the restriction is on the heating side (hypothesis A).** Next: bypass / differential-pressure valve with the
installer, or block the burner via E1 while the slab takes little water.

Firmware: the burner controller (TN 410) reads **2.20** here. No Wolf release note found that changes
soft-start or gradient monitoring; a firmware update cannot create flow, so it is not the primary fix.

### Next levers, in order

1. HG09 longer (20–30 min) once the stops are normal control stops.
2. HG39 soft-start 3 → 1 min (some installers report it helps; the effect is installation-dependent).
3. HG02 back to the factory value for the 14 kW unit (26 %) if it was raised — **installer setting**,
   do not go below factory without a combustion (CO) check.
4. Use input E1 (HG13) to block the burner when the sum of room-valve openings is low —
   reported to cut starts from 2.4/h to 0.4–0.8/h in the same thread.

## Step 5 — blocking the burner through input E1 (results, Nov 2026)

*Keywords: Wolf E1 Raumthermostat Eingang, HG13 = 1, Brenner sperren, Taktsperre extern, burner block
contact, court-cycle plancher chauffant, Mischer zu.*

Idea: since the boiler cannot run long flames while the slab takes little water, **don't let it try**.
Open input E1 for 45 min, close it for 30 min, and let the slab's thermal mass (time constant ≈ 28 h)
ride through the pause. The boiler then starts once per free phase, with the mixer opened wider, and
burns long instead of 15 s safety stops.

Hardware / settings:
- **HG13 (E1 function) = 1** ("room thermostat": E1 open = heating blocked, hot water still allowed,
  frost protection never blocked). Written over eBUS (TN 2651, see the CRC section). Verified: with E1
  open the boiler goes to standby within ~30 s (setpoint 5 °C, boiler pump 0 %, mixer pump off).
- E1 is a **low-voltage, potential-free input**: drive it with a dry contact only. Here: the **NC contact**
  of a small relay (relay unpowered = E1 closed = normal heating, so a dead controller fails safe) in
  series with a **surface thermostat on the floor flow pipe** that opens on over-temperature — the
  maximum-temperature protection for the slab that the MM manual requires, which this installation
  never had (only a bridge). Never put mains or any external voltage on E1.
- The relay is switched by a KNX actuator channel, commanded from Home Assistant.

Control logic (runs every minute):
- 75 min clock cycle: 45 min blocked, 30 min free.
- **Room guard:** no new block if any room is > 0.5 K under setpoint; an ongoing block is released at
  once if a room drops > 0.7 K under setpoint.
- No block below **−3 °C outdoor** (and none above 16 °C, where the boiler doesn't heat anyway).
- Hard cap 50 min blocked in a row; the command is re-sent every 10 min (the actuator's fail-safe falls
  back after 30 min without a telegram).

Results (same house, burner starts per day from the boiler's counter HG91):

| Day | Outdoor mean | E1 blocked | Starts/day |
|---|---|---|---|
| 19/11 (before) | 9.3 °C | 0 min | 97 |
| 20/11 (before) | 8.3 °C | 0 min | 92 |
| 21/11 (before) | 3.1 °C | 0 min | 81 |
| 22/11 (from 09:10) | 2.3 °C | 331 min | 67 |
| 23/11 | 0.2 °C | 571 min | 43 |
| 24/11 | −0.2 °C | 509 min | 52 |
| 25/11 (to 18 h) | 1.2 °C | 275 min | 37 |

- Burner time per start: **~12 min** on a blocked day vs **1.3–6.8 min** before.
- Comfort: room temperatures between 22:00 and 07:00 over several nights were the same with and without
  blocking. The guard, not the cadence, decides on cold days: the coldest room keeps E1 free.
- **Gas:** burner hours per degree-day (hot water removed) 0.50 before vs 0.48 with E1 — **no measurable
  saving after 3 days**, within the noise. Confirmed with the gas meter's daily export (22–27/11, 6 days,
  degree-days from the grid operator's temperature): 33.1 m³ measured vs 34.4 m³ predicted by a
  m³-vs-degree-day fit on the 15 days before (−4 %), while single days scatter by ±16 % (one day without
  blocking came out −17 %). So: no saving you can claim. The win is wear (ignitions, electrode, fan), not m³. This
  matches the literature on condensing boilers: longer cycles mostly reduce wear; fuel gains are a few % at most.

## Step 6 — E1 as "burner block" (BOB) instead of "room thermostat" (28/11)

*Keywords: Wolf HG13 BOB, Betrieb ohne Brenner, Brennersperre, « Bloc brû. », fonctionnement sans brûleur,
blocage du brûleur, Mischerkreispumpe läuft weiter, burner block keep pumps running.*

Why: with HG13 = "room thermostat" (Step 5), an open E1 puts the boiler in **summer mode** — and the MM
module then **stops the mixer-circuit pump**. Here that meant ~9 h a day of blocks with no water moving
in the slab, and it cancelled most of the BM-2 mixer correction (floor flow stayed ~28 °C).

HG13 has another option, missed at first because only the setting copied from another project was read:
**BOB, "operation without burner"**. Per the CGB-2 manual: *E1 closed = burner blocked; heating pump,
3-way valve and tank charging pump keep running normally; burner still allowed for chimney-sweep and
frost protection; E1 open = burner released.*

How it shows up — **don't go by the number**:
- On the BM-2 (French UI) the list reads: *Sond amb, ECS, Samb/ECS, Zirkomat, **Bloc brû.**, Clpt gaz,
  OWH, Externe, SEE sans coup*. "Bloc brû." is BOB.
- Read back over eBUS (TN 2651) it is **5**. A CGU-2 manual numbers BOB as **8** — writing 8 blindly would
  have selected another function. Set it on the display, then read it back.

Wiring change (logic is inverted, so both contacts change):
- Relay: **NO contact** instead of NC → relay unpowered = E1 open = burner free. A dead controller still fails safe.
- Floor safety thermostat: the contact that **closes on rising temperature**, now **in parallel** with the
  relay (closing either one blocks the burner). In series, an over-temperature would have *released* the burner.
- New weak point: a broken common wire to E1 silences both relay and thermostat, and nothing shows it.
  Covered by a software check (below) plus a monthly test (E1 wires off, thermostat at minimum → contact must close).

Verified on the bus (relay closed at 17:41, released at 18:00):

| Time | Burner status (TN 10068) | Mode | Mixer pump | Floor flow |
|---|---|---|---|---|
| 17:42:54 | **Betrieb_ohne_Brenner** | Heizbetrieb | on | 24.3 °C |
| 17:46:37 | Betrieb_ohne_Brenner | Heizbetrieb | on | 25.0 °C (residual heat, no gas) |
| 18:00:12 | Aus (relay released) | Heizbetrieb | on | 25.0 °C |
| 18:01:16 → 18:04:29 | pre-purge → soft-start → **On** | Heizbetrieb | on | 35 °C |

Side effects and pitfalls:
- In BOB the burner is blocked **for hot water too**. The controller releases E1 when the tank needs a
  charge (tank ≤ setpoint − hysteresis + 1 K, until setpoint − 0.5 K). In "room thermostat" mode that rule
  was useless (hot water still ran) and only shortened blocks — gate it on the HG13 value.
- **Between rewiring and changing HG13, the logic is inverted**: relay at rest = E1 open = summer mode =
  no heating. Change HG13 right after powering the boiler back on.
- Powering the boiler off for the wiring takes the eBUS down ("signal lost" in ebusd) until it is back on.
- The burner status was **never polled**: its CSV line was a plain `r`, so Home Assistant showed "Aus" for
  two days while the burner ran. Changing it to `r9` in `08..hc.csv` + `ebusctl reload` (rescan ~1 min)
  gives a read every 3–6 min. The controller now alerts if the relay is closed and the status shows
  Zuenden/Softstart/Ein for ≥ 3 min (once per block, BOB only).

Gain: what BOB adds over Step 5 is **circulation during blocks** — the slab keeps taking heat, including
what is left in the boiler after a hot-water charge, and the mixer correction is no longer cancelled.
Starts/day were already roughly halved by Step 5 (≈ 90–110 → 43–52, partly also colder weather). Gas: not
measured yet for BOB; Step 5 showed no measurable m³ saving. Results to follow after several days.

## BM-2 "mixer valve" correction is local only (not on eBUS)

On the BM-2 screen "Vanne mélangeuse / Mischer" (key 2, scale −4…+4) each step shifts the mixer-circuit
setpoint (`mischersolltemp` on the MM module): measured **+1.6 K** for 0 → +1 at 1.3 °C outdoor and
**+1.3 K** for +1 → +2 at 1.1 °C. An ebusd `grab` during the change showed **no write telegram**: the value
is internal to the BM-2, so it cannot be set from Home Assistant — do it by hand.
Use it when one room is under setpoint with its valve already at 100 % (here a north-west room with no sun);
raising that room's thermostat setpoint does nothing more.

Also normal, but confusing right after such a change: during a **hot-water tank charge** (boiler at
63–65 °C here, ~20 min) the mixer circuit gets nothing, so the floor flow stays at the old temperature
until the charge ends.

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
