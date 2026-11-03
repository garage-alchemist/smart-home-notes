# Home Assistant OS mini PC rebooting 3–4 times a day: NVMe APST

*Keywords: HAOS random reboot, HP EliteDesk 800 G4 mini, Samsung PM961 MZVLW256HEHP, APST,
nvme_core.default_ps_max_latency_us, iTCO_wdt watchdog, unsafe shutdowns.*

## Symptom

91 reboots in 25 days (3.6/day), max uptime 2.3 days. All containers restarted with `RestartCount: 0`
→ the host, not Home Assistant.

## Evidence chain

- Hard death: `systemd-fsck: Dirty bit is set`, `journald: ... uncleanly shut down`, no shutdown sequence.
- Not a kernel panic: no `panic=` on the command line — a panic would hang, not reboot.
- Hardware watchdog armed: `Using hardware watchdog 'iTCO_wdt' ... timeout of 5min`. A kernel freeze
  > 5 min → hardware reset. Matches a ~15 min gap in the journal.
- Environment clean: 37 °C, load 0.08, memory 12 %, no MCE/AER/ECC/NVMe/OOM messages.
- SMART: 418 unsafe shutdowns / 2117 power cycles, but 0 media errors and 6 % wear → the disk *suffers*
  the resets, it is not worn out.
- Red herring: a Bluetooth `hci0 ... -112` message before each death. Bluetooth disabled → it still rebooted.

## Cause

SSD **Samsung PM961 256 GB (MZVLW256HEHP, fw CXB73H1Q)** — known in Ubuntu bug #1737934 to drop off the bus
when APST puts it in its deepest power state. Its APST table switched to state 4 (5 mW) after 100 ms idle.
The drive sleeps, does not wake, I/O dies, nothing can reach the journal, the watchdog resets 5 min later.
Consistent with failures during idle periods (nights, absences).

## Fix

1. Immediately, live:
   `echo 0 > /sys/class/nvme/nvme0/device/power/pm_qos_latency_tolerance_us` → APST off.
   (The drive cannot save this feature permanently.)
2. Permanently, HAOS boot partition: `/mnt/boot/cmdline.txt` →
   `console=tty0 nvme_core.default_ps_max_latency_us=0`. Keep a `.bak` next to it.

Result: **44+ days of uptime** (vs 2.3 days max), confirmed to survive HAOS updates
(`/proc/cmdline` still contains the parameter after an OS upgrade).
