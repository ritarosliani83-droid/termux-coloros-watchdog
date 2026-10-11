# wireless debugging on, then inside Termux:
adb connect 127.0.0.1:<port>
./diagnose.sh                      # or: ./diagnose.sh com.termux some.other.app# termux-coloros-watchdog

Keep long-running Termux scripts alive on OPPO / OnePlus / realme phones (ColorOS, OxygenOS), and find out **why** Android killed them.

## The problem

You run a background script in Termux. You have done everything the guides say:

- battery optimization off for Termux, Termux:API and Termux:Boot
- "Allow background activity" and "Auto launch" on
- `termux-wake-lock`, Termux pinned in recent apps
- phantom process killer disabled via adb

The script still dies silently after a while, sometimes after minutes, sometimes overnight. The log shows no shutdown message, because the process got `SIGKILL`. It only comes back when you open Termux again.

## What is actually happening

On my device (OPPO PLG110, Android 16, ColorOS) Android records the kill reason, and you can read it with adb:

```
$ adb shell dumpsys activity exit-info com.termux
  timestamp=2026-10-03 06:26:23  process=com.termux
  reason=13 (OTHER KILLS BY SYSTEM) subreason=2040 (UNKNOWN)
  importance=125  description=o-kill(40)
```

- **`o-kill(N)`** is ColorOS's own killer. It is not a standard Android mechanism. I saw codes `35`, `40` and `72`. Their meaning is not documented.
- **`importance=125`** means Termux was running a **foreground service** with its notification at that moment, which is the highest priority a normal app can get without root.
- At the same time every standard exemption was in place and verified: battery whitelist, standby bucket 5 (exempted), `RUN_ANY_IN_BACKGROUND: allow`, phantom process killer off.

**Conclusion: the per-app settings work as far as Android is concerned. ColorOS kills anyway.** I don't know of a non-root setting that stops `o-kill`.

### Trap: Termux:Boot gets force-stopped

```
$ adb shell dumpsys activity exit-info com.termux.boot
  reason=10 (USER REQUESTED) subreason=21 (FORCE STOP)
  description=stop com.termux.boot due to from pid 4501 (com.oplus.battery)

$ adb shell dumpsys package com.termux.boot | grep -o 'stopped=[a-z]*'
stopped=true
```

ColorOS's battery manager force-stops Termux:Boot, probably because it has no UI and seems unused. A force-stopped app **does not receive the boot broadcast**, so your boot scripts silently stop running after the next reboot.

Fix: open the Termux:Boot app once, or run `adb shell am start -n com.termux.boot/.BootActivity`. Then check `stopped=` again.

## Diagnose your own device

[`diagnose.sh`](diagnose.sh) is read-only and changes nothing. It prints the kill history, force-stop state, standby bucket, whitelist and phantom process settings for the Termux packages, plus the scheduled Termux:API jobs.

```sh
# wireless debugging on, then inside Termux:
adb connect 127.0.0.1:<port>
./diagnose.sh                      # or: ./diagnose.sh com.termux some.other.app
```

If you see `o-kill(...)` or another `OTHER KILLS BY SYSTEM` entry while all exemptions say yes/allow, you have the same problem.

## The workaround: a watchdog in a different package

Since the kill cannot be prevented, the script is restarted automatically:

- `termux-job-scheduler` (Termux:API) runs a watchdog script every **15 minutes**, which is the minimum Android allows for periodic jobs.
- The job belongs to **Termux:API**, not to Termux. When Termux gets killed, the job survives and starts Termux again. A `o-kill` is not a force-stop, so scheduled jobs are kept.
- `--persisted true` keeps the job across reboots, so it also covers the case where Termux:Boot was force-stopped.
- The watchdog runs your command in the **foreground**. That makes it a Termux task, which keeps Termux's foreground service up as long as it runs.

### Setup

```sh
pkg install termux-api            # plus the Termux:API app (same source as Termux!)
cp termux-watchdog ~/.local/bin/my-daemon-watchdog
# edit NAME and the command at the bottom of the file
termux-job-scheduler --job-id 4711 \
  --script ~/.local/bin/my-daemon-watchdog \
  --period-ms 900000 --persisted true \
  --network none --battery-not-low false
```

`--network none --battery-not-low false` is important: by default the job would wait for a network connection and a non-low battery.

Check that the job is registered. `termux-job-scheduler -p` printed nothing for me, so use adb:

```sh
adb shell dumpsys jobscheduler | grep -A8 'JOB #.*/4711'
```

Test it by killing your daemon with `kill -9 <pid>`, then schedule a one-shot job with the same script (e.g. `--job-id 4712`, no `--period-ms`). The one-shot job runs within seconds.

### Real-world result

The first kill after setting this up, from `vpn-on-demand log` and `diagnose.sh`:

```
10:26:20  last heartbeat of the daemon
10:26:25  com.termux killed: o-kill(40), importance=125 (foreground service)
10:41:25  watchdog: daemon not running (last alive: 10:26:20), starting
```

The daemon was back after 15 minutes without anyone opening Termux.

### Limits

- **Up to ~15 minutes downtime** after a kill. Shorter periodic jobs are not possible. Staggering several jobs does not help, because each job may run anywhere in its 15-minute window and Android batches them.
- In deep **Doze** (phone lying unused, screen off), jobs are deferred to maintenance windows, even for whitelisted apps.
- If ColorOS ever **force-stops Termux:API**, its jobs are deleted and the watchdog is gone. I have only seen `o-kill`s for it so far, not force-stops.
- Findings are from **one device** over a few days. Reports from other devices are welcome, see below.

## Example: VPN on Demand for Tailscale

[`examples/vpn-on-demand`](examples/vpn-on-demand) is the script that started this: an iOS-style "VPN on Demand" for the Tailscale Android app. Tailscale is switched off on trusted home Wi-Fi and on everywhere else (mobile data, foreign Wi-Fi), via Tailscale's `CONNECT_VPN` / `DISCONNECT_VPN` broadcast intents.

It has its own `watchdog` subcommand: it writes a heartbeat every loop, so the log shows when the daemon was last alive before a kill. A `stop` disables the watchdog until the next `start`.

```sh
pkg install termux-api jq
cp examples/vpn-on-demand/vpn-on-demand examples/vpn-on-demand/vpn-on-demand-watchdog ~/.local/bin/
cp examples/vpn-on-demand/boot-vpn-on-demand ~/.termux/boot/   # needs Termux:Boot
vpn-on-demand edit        # set TRUSTED_SSIDS
termux-job-scheduler --job-id 4711 --script ~/.local/bin/vpn-on-demand-watchdog \
  --period-ms 900000 --persisted true --network none --battery-not-low false
vpn-on-demand status
vpn-on-demand log         # "watchdog: daemon not running (last alive: ...)" = a kill happened
```

Reading the Wi-Fi name needs location permission for Termux:API and location turned on.

## Contributing

If you have a different device or ColorOS version, please open an issue with the output of `diagnose.sh`. Especially wanted: other `o-kill` codes, and whether the watchdog keeps working over weeks.

## Related

- [termux/termux-app#5150](https://github.com/termux/termux-app/issues/5150): processes killed on Android 15 despite wake lock and exemptions
- [termux/termux-app#4219](https://github.com/termux/termux-app/issues/4219): Termux killed on OnePlus/ColorOS 15. That is a different mechanism (kills at startup / proot, fixed by OS updates), not the background `o-kill` described here
- [dontkillmyapp.com/oppo](https://dontkillmyapp.com/oppo)
- [Termux no longer works on OxygenOS/ColorOS 15](https://oxygenupdater.com/article/484/)

## License

MIT
