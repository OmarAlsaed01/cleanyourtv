# My TV: Sony BRAVIA XR-65X90K (VH21, US firmware)

Cleaned on 31 Oct 2026 with this repo's tools: Phase 0 (backup and baseline), Phase 1 (storage) and Phase 2 (debloat).
The launcher swap (Phase 3) has not been done; Google TV Home is still the launcher.

| | |
|---|---|
| Model | XR-65X90K, device `BRAVIA_VH21`, `ro.soc.model` MT5835, 3 GB RAM |
| Firmware | `BRAVIA_VH21_M_UC-user 12 STT2.230505.001.S136 682241` (Android 12, Google TV, SDK 31) |
| Network | Wi-Fi. setupwraith owns the saved network, so leave it enabled. |
| Backups | `backup\` (package lists, getprop, settings, display, sizes, bench). Not committed. |

## What changed

**Uninstalled** (Phase 1, user-installed, re-download from the Play Store any time):
Spotify, Sling, ESPN, Crave, YouTube TV, Twitch, YouTube Kids, Hulu, Xumo, Haystack.

**Disabled** (Phase 2, 40 packages, full list in `debloat.txt`):

| Group | Packages |
|---|---|
| Tracking and telemetry | Samba TV (ACR), Sony IDD agent, user context, Qterics agent, remote support, Sony bug reports, Google feedback |
| Sony ads and promos | promos, Sony Select, Sony tab, help, help guide, eco dashboard, customer support |
| Shop demo mode | demosupport, demosystemsupport, multiscreendemo |
| Streaming and Google apps | Netflix, Disney+, YouTube, YouTube Music, Apple TV, Sony Pictures Core, Google TV movies, Play Games |
| Features not used | DLNA (3), Sony IP control (webapi.core), UEI QuickSet, Google TTS |
| BRAVIA Connect and BRAVIA CAM | seeds.iot, seeds.auth, remote start, gestures, camera updater and settings, distance alert, Living Fit |

**Reverted to the factory version before disabling** (`pm uninstall-system-updates <pkg>`): promos, Sony tab,
Netflix, Disney+, YouTube, YouTube Music, Apple TV, Sony Pictures Core, Google TV movies, Play Games, TTS.
Their app data and logins were wiped with the update.

**Settings changed:**

| Setting | Before | After |
|---|---|---|
| `window_animation_scale` | 1.0 | 0.5 |
| `transition_animation_scale` | 1.0 | 0.5 |
| `animator_duration_scale` | unset | 0.5 |
| `screensaver_components` (secure) | `com.sony.dtv.livingfit/com.sony.dtv.livingfit.MainDreamService` | `com.google.android.apps.tv.dreamx/.service.Backdrop` |

On the TV itself: Usage & diagnostics off, advertising ID deleted (Settings > Privacy).

**Kept by choice:** Prime Video, Google Assistant / mic, Chromecast built-in, AirPlay / HomeKit, firmware updates,
Downloader, Steam Link, LocalSend, File Commander, and everything that runs HDMI, the tuner and the remote.

## Results

| | Before | After |
|---|---|---|
| `/data` free | 441 MB (90% used) | 2.1 GB (46% used) |
| RAM available, after a reboot | 1.0 GB | 1.2 GB |
| Swap used, after a reboot | 40 MB | 7 MB |
| Key response p50 / p95 | 33 / 147 ms | 31 / 136 ms |
| Focus movement p50 | 1110 ms | 648 ms (mostly the 0.5 animation scale) |
| Jank | 0.74% | 0.87% (slightly worse, within run-to-run noise) |
| App cold start p50 | 377 ms | 356 ms |
| Idle CPU, whole TV (launcher) | 16% (9%) | 12% (4%) |
| Launcher memory | 127 MB | 104 MB |

Bench: `tools/bench.py --keys 36 --series 3`, both runs right after a reboot. Raw numbers in
`backup\baseline.json` and `backup\bench-after-phase2\result.json`. The return-to-Home time was not measured
in either run (n=0), and the baseline lost 34 of 108 keys (the second run lost 16).

## Undo

Every command below needs ADB open again first: Settings > System > Developer options > USB debugging on,
then accept the prompt on the TV when the first command connects. Switch it off again afterwards.

Everything at once, from the repo root:

```powershell
tools\restore.ps1 -TvHost <tv ip> my-tv\debloat.txt -Stock com.google.android.apps.tv.launcherx
```

That re-enables the 40 packages and puts the animation scales back. It does not touch the screensaver,
which goes back with:

```powershell
$env:TV_HOST='<tv ip>'; tools\venv\Scripts\python tools\tv.py "settings put secure screensaver_components com.sony.dtv.livingfit/com.sony.dtv.livingfit.MainDreamService"
```

The undo record for `debloat.txt` was merged from the five per-batch records in `~\.cleanyourtv\`
(`recovery-<tv ip>-*.json`). Keep that folder. Restoring a single batch also works:
`tools\restore.ps1 -TvHost <tv ip> my-tv\batch3.txt -Stock com.google.android.apps.tv.launcherx`.

One app:

```powershell
$env:TV_HOST='<tv ip>'; tools\venv\Scripts\python tools\tv.py "pm enable com.netflix.ninja"
```

A reverted app comes back at its factory version; the Play Store updates it after that.

## After a firmware update

Updates can re-enable packages. Switch USB debugging on (as in Undo), then check, and fix only what the
check lists:

```powershell
$env:TV_HOST='<tv ip>'; tools\venv\Scripts\python tools\check.py my-tv\debloat.txt
```
