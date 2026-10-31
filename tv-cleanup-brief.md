# Sony BRAVIA VH21 cleanup — Claude Code brief

Put this file in the root of the cloned `cleanyourtv` repo. Start Claude Code there and paste:

> Read `PROMPT.md`, `examples/sony-bravia-4k-vh21/debloat.txt`, `examples/sony-bravia-4k-vh21-eu/debloat.txt` and `tv-cleanup-brief.md`. Follow the brief: it narrows PROMPT.md to my TV and splits the job into phases. Start with Phase 0 and stop where the brief says to stop.

---

## My setup (fill in before starting)

- **TV:** Sony BRAVIA 4K VH21, Google TV 12 (Android 12). Model number: `KD-____` (Settings → System → About)
- **TV network:** Wi-Fi / Ethernet ← pick one
- **TV IP:** `192.168.__.__`
- **PC:** Windows 11. Use the PowerShell versions of the scripts (`tools\*.ps1`, `launcher\build-and-install.ps1`).

## Goals, in priority order

1. Free up storage. `/data` is full right now.
2. Turn off telemetry, ACR, ads, promos and preloaded bloat.
3. Make the TV more responsive, and measure it before and after.
4. *(Phase 3, separate session, optional)* Replace the Google TV launcher.

## Ground rules

- Use the repo's own tools (`tools/tv.py`, `disable.ps1`, `restore.ps1`, `check.py`, `bench.py`, `adb-closed.py`). Don't write replacements for them.
- Only use `pm disable-user --user 0`. Never `pm uninstall` a system package. Never `pm clear` anything without asking me.
- Stay read-only until I approve a written plan for each phase. Show me every batch before you apply it.
- Run one `pm` command per shell call, then confirm with `pm list packages -d`, not the exit code.
- Use batches of 8 packages or fewer. After each batch, take a screenshot and check Home, Settings, one HDMI input, playback in one streaming app, and the remote's Input button.
- Never disable anything on the keep list in the VH21 list header: `com.sony.dtv.tvx`, `com.sony.dtv.tvinput.*`, `com.sony.dtv.braviasync*`, `tvapi`, `sonysystemservice`, `imeproxy`, `settingsprovider`, `systemupdate`/`update`, the Sony settings apps, all `com.mediatek.*`, and Play Store / Play Services. The core Android packages listed in PROMPT.md are also off limits.
- Don't launch any package whose name contains factory, fac or test to find out what it does. Use `dumpsys package` instead.
- If anything breaks, stop and tell me before you try to fix it.

---

## Phase 0 — Connect, back up, baseline (read-only)

1. Connect over ADB on port 5555 and tell me when to accept the prompt on the TV.
2. Run a probe and report: `ro.product.device`, `ro.build.fingerprint`, `ro.build.version.sdk`. Compare my build with the VH21 list headers (`BRAVIA_VH21_M_PA … 682241` for the AU/PA list and `BRAVIA_VH21_EU … 682241` for the EU list). Mine is probably a US/NA build, so tell me how it differs.
3. Back up everything into `backup\` before any change: the full package list, the disabled-package list, `getprop`, `settings list global`, `settings list secure`, `settings list system`, and `wm size`, `wm density` and the animation scales.
4. Storage: `df -h /data`, plus the top 15 apps by size (app, data and cache) from `dumpsys diskstats`.
5. Memory: the summary from `dumpsys meminfo` (total, free, available, swap).
6. Responsiveness: run `tools/bench.py` against the stock launcher and save the JSON as `baseline.json`.
7. Diff my installed packages against the VH21 list. Report three groups:
   - packages in the list that aren't on my TV
   - Sony or partner packages on my TV that aren't in the list
   - anything already disabled at the factory

**Stop here and report.**

## Phase 1 — Free up storage

1. Run `pm trim-caches 2G` and re-check `df`.
2. Find every SmartTube install: `pm list packages -f | grep -i -E "smarttube|liskovsoft|teamsmart"`. Show me each package's version (`dumpsys package <pkg> | grep versionName`) and size. I'll tell you which one to remove. One of them is an old build from before the signing-key change.
3. List the bloat candidates from Phase 2 that have Play Store updates installed in `/data` (where the `codePath` is under `/data/app`), with the size of each. Propose reverting them to the factory version and wait for my yes.
4. Re-measure `df` and report how much space was freed.

## Phase 2 — Debloat (no launcher changes)

1. Before you start, remind me to switch off **Usage & diagnostics** and reset the **advertising ID** in the TV's own Settings menu.
2. Start from `examples/sony-bravia-4k-vh21/debloat.txt`. Save my working copy as `my-tv\debloat.txt`.
3. Leave these **commented out by default** and ask me about them with one checklist question:
   - Google Assistant / mic button (`katniss`, `sony.dtv.voicesearch`, `assistantadapter`)
   - Chromecast built-in (`mediashell`)
   - AirPlay / HomeKit (`airplayapp`, `homekit`)
   - DLNA
   - Sony IP control (`webapi.core`). Home Assistant's Bravia integration needs this one.
   - UEI QuickSet (IR control of a soundbar or cable box; HDMI-CEC isn't affected)
   - Google TTS (TalkBack and read-aloud depend on it)
   - Sony IoT / BRAVIA Connect phone link (`seeds.iot`)
   - Google TV movies app (`com.google.android.videos`). Disabling it removes access to purchases made in it.
   - Each preloaded streaming app
4. Do **not** disable `com.google.android.tungsten.setupwraith` in this phase. It's only needed for the launcher swap.
5. Set the window, transition and animator scales to 0.5. Record the old values first.
6. Reboot. Check that everything stayed disabled, then re-run `bench.py` and `meminfo`. Report the before and after numbers next to each other.
7. Write `my-tv\README.md` with what was disabled, what I chose to keep, and the exact restore command:
   `tools\restore.ps1 -TvHost <ip> my-tv\debloat.txt`

**Stop here.** I'll live with it for a few days before Phase 3.

## Phase 3 — Launcher (separate session, optional)

Only start this after I confirm Phase 2 has been stable.

1. Build environment: check for a JDK between 17 and 22 and for Android SDK platform 34 with build-tools 34. If they're missing and Docker is installed, use `tools/docker-build.sh`. Otherwise tell me exactly what to install. Don't install anything yourself.
2. Before you touch setupwraith, run `dumpsys wifi | grep -o 'cname=[^ ]*'`. If setupwraith owns my saved Wi-Fi network, **stop**. I'll forget the network and re-join it in Settings first, so the TV doesn't drop off Wi-Fi and ADB.
3. Follow the launcher section of PROMPT.md. In the config, set `live_tv_package` to `com.sony.dtv.tvx`. Ask me about the dock apps and their order.
4. After the handover, check that Home is the new launcher with `dumpsys activity activities | grep mFocusedApp`.
5. Ask me to press every remote button that matters: Input, Settings/gear, YouTube, Netflix, Prime Video, Disney+, volume hold. On VH21 sets the YouTube, Prime and Disney+ buttons share keycode 196, so map them by scancode if they stop working.
6. Run `bench.py` again and compare it with `baseline.json`.

## Finish (every session)

1. Switch off USB debugging and Network debugging on the TV, revoke the USB debugging authorisations, and reboot.
2. Run `tools/adb-closed.py <tv ip>` and confirm the port is closed.
3. After any future firmware update, run `tools/check.py my-tv\debloat.txt` to see what the update re-enabled.
