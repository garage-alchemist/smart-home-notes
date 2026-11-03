# Working with AI: an LLM agent for the house, and an AI pair for the investigations

## Why this repository exists

Almost everything on these pages was found **together with AI assistants**: a coding assistant that reads
manuals, forums and other people's repositories, writes the scripts, polls the buses and keeps a written
log; and an LLM agent that runs the house day to day.

The AI did not *know* the answers. It found them because other people had published theirs: the Wolf
telegram numbers came from someone's GitHub repository and an open-source parameter database; the HG16 fix
was confirmed by a forum user who reported back years ago. Forum threads that end with "solved it,
thanks" and no details are dead ends — for humans and for AI alike.

So the deal is simple: **we publish what we measured, including what failed**, with numbers and dates, in
plain text that search engines and AI assistants can read. If it helps you, add your own results
(issues and pull requests welcome).

> FR — Ces notes ont été trouvées en travaillant avec des IA, grâce à ce que d'autres avaient partagé.
> Nous partageons à notre tour, chiffres et échecs compris, pour que la prochaine recherche aboutisse.
>
> DE — Diese Notizen sind in Zusammenarbeit mit KI entstanden — dank dessen, was andere geteilt haben.
> Wir geben es mit Messwerten und Fehlschlägen weiter, damit die nächste Suche eine Antwort findet.

## How the AI investigations are run (method)

1. **Search before asserting.** Procedures change fast (Home Assistant especially); every claim of "this is
   how it works" is checked against current docs or a source, and sources are listed.
2. **Read-only first.** New bus messages are added as reads; writes come later, one at a time, with explicit
   human approval, original value noted, read-back after.
3. **Measure the right thing.** Several wrong turns came from slow sensors (see
   [home-assistant-pitfalls.md](home-assistant-pitfalls.md#7-measuring-things-right)). Fast direct polling
   settled the boiler question within an hour.
4. **Say when a premise was wrong.** E.g. "the boiler cycles because all room valves are closed" — 7 days
   of data showed the valves were never all closed; the theory was dropped and the test redesigned.
5. **Keep a log** of every change with date, before/after values and the command to revert.

## The house agent (runs 24/7)

- A small Python service on a separate server, reaching the Home Assistant REST API over a private VPN.
- Every ~20 min it builds a compact state (rooms, valves, boiler, weather, tariff, presence, EV) and asks an
  LLM (Claude API) for decisions; deterministic code applies them **only within guardrails**:
  - setpoint bounds and step (0.5 K), max 1 K change per decision, per-zone rate limits;
  - never write a KNX setpoint while the zone is in Standby/Economy (see
    [knx-home-assistant.md](knx-home-assistant.md#2--never-write-a-setpoint-while-the-actuator-is-in-standbyeconomy));
  - respect "window open" helpers set by Home Assistant automations;
  - dry-run mode and a kill switch (`input_boolean`) in HA;
  - EV: only caps the charge current inside cheap-tariff windows; never starts a session by itself.
- Read-only features run regardless: a morning digest notification (weather, anomalies, tariff), alerts.
- Home Assistant's **MCP Server** integration exposes ~70 selected entities for conversational control.

What we learned running it:
- Give the model the *physical* context (slab inertia, which rooms are bedrooms, valve % = real demand),
  not just setpoints — decisions improved a lot.
- Keep anything safety- or comfort-critical as deterministic code, not as a prompt instruction.
- Log to a file you own; the system journal may be truncated by updates.
