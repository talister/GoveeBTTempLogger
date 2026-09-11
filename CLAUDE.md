# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Linux daemon (`goveebttemplogger`) that listens for Bluetooth LE advertisements from Govee thermometers/hygrometers (and Ruuvi tags), logs readings to tab-separated text files, optionally connects to devices to download stored history, and renders SVG graphs. A companion tool (`gvh-organizelogs`) sorts/dedupes/merges those log files. Upstream is `wcbonner/GoveeBTTempLogger`; this is a fork.

## Build, test, package

Dependencies (Debian/Ubuntu): `build-essential cmake git libbluetooth-dev libdbus-1-dev libssl-dev`.

```sh
cmake -S . -B build
cmake --build build              # builds both executables
cd build && ctest --verbose      # "tests" are just `<binary> --help` smoke runs
cd build && cpack .              # produces goveebttemplogger_*.deb
```

Build out-of-source in `build/`. There are stray in-source CMake artifacts at the repo root (`CMakeCache.txt`, `Makefile`, `goveebttemplogger` binary, etc.) that are untracked — don't commit them and don't rely on them.

CI (`.github/workflows/ci.yml`) builds on Ubuntu 22.04 x86_64 and arm with both g++ and clang++, so keep the code warning-clean under both.

Version is set in `CMakeLists.txt` (`project(... VERSION 5.YYYYMMDD.0 ...)`) and injected via `goveebttemplogger-version.h.in`. Bump the date component when making a release-worthy change.

The CMake config has no lint/format target; `.github/workflows/flawfinder.yml` runs flawfinder on push.

## Runtime / install layout

- `postinst` creates system user `goveebttemplogger` (groups `www-data`, `netdev`, `bluetooth`, `rfkill`), creates `/var/log/goveebttemplogger`, `/var/cache/goveebttemplogger`, `/var/www/html/goveebttemplogger`, and sets `cap_net_raw,cap_net_admin` on the binary (needed only for the legacy HCI path; DBus needs no special privileges).
- `goveebttemplogger.service` is the shipped systemd unit; the default `ExecStart` flags there are the canonical "production" configuration. Note the local working-tree change to `Group=apache` (RHEL-style) versus upstream's `www-data` — this fork runs on EL9.
- Running as non-root requires the Bluetooth adapter not be soft-blocked; `main()` checks and tries to clear rfkill before starting.

## Source layout

Almost everything lives in one ~7500-line file, `goveebttemplogger.cpp`. Read it by section rather than whole:

| Lines (approx) | Section |
|---|---|
| 126–245 | `_BLUEZ_HCI_`-only shims for extended-scan HCI commands missing from older BlueZ headers |
| 248–330 | `ThermometerType` enum + string conversions — add new Govee models here |
| ~330–1075 | `Govee_Temp` (a single reading: time, up to 4 temps, humidity, battery, model) and `Govee_Device` (per-address state incl. download bookkeeping). `Govee_Temp::ReadMSG` parses raw advertisement manufacturer data per model; `ReadCache` parses the cache-file line format. Globals holding all state are declared at ~1399 and ~1828. |
| 1112–1400 | `Ruuvi_Tag` — parallel type for Ruuvi RAWv2 advertisements |
| 1406–1420 | Signal handlers (SIGINT/SIGHUP/SIGALRM) driving the main loops |
| 1422–1720 | Filesystem helpers, `gvh-thermometer-types.txt` persistence file (address → model, plus download state), log file writer |
| 1756–2840 | MRTG/cache/SVG pipeline: `ReadLoggedData` → `UpdateMRTGData` → `GenerateCacheFile` → `WriteSVG`/`WriteAllSVG`/`WriteSVGIndex` |
| 2845–4945 | `_BLUEZ_HCI_` path: raw HCI socket scanning (`bt_LEScan`), L2CAP/GATT download (`BlueZ_HCI_ConnectAndDownload`), `BlueZ_HCI_MainLoop`. Also the AES/RC4 `encrypt_packet`/`decrypt_packet` used by newer devices (protocol v5). |
| 4949–7090 | DBus path: `bluez_*` helpers wrapping org.bluez, `bluez_dbus_msg_iter` (advertisement parsing dispatch), `bluez_device_download`, `BlueZ_DBus_Mainloop` |
| 7095–7220 | rfkill helpers |
| 7223–end | `usage()`, getopt tables, `main()` |

Other files:
- `gvh-organizelogs.cpp` — standalone; duplicates a few helpers (`ValidateDirectory`, `ba2string`, `GenerateLogFileName`) rather than sharing a header. Keep the two copies in sync if you change log filename conventions.
- `wimiso8601.{h,cpp}` — ISO8601/Excel time formatting shared by both binaries. `#ifdef _MSC_VER` branches exist because this file is shared with the author's Windows projects.
- `att-types.h`, `uuid.{h,c}` — copied from BlueZ; used only by the HCI path.
- `GoveeBTTempLogger.sln`/`.vcxproj`, `CMakePresets.json` — Visual Studio / WSL cross-compile support; not used on Linux.
- `BTData/`, `BLEScanning/`, `ruuvi/`, `DeviceImages/` — packet captures, reference docs, images. `BTData/hci-*.txt` are per-model advertisement dumps useful when adding model support.

## Key architectural points

**Two Bluetooth backends, one data model.** `main()` first tries `BlueZ_DBus_Mainloop`; if it returns non-zero (e.g. bluetoothd not available) and `_BLUEZ_HCI_` is compiled in, it falls back to `BlueZ_HCI_MainLoop`. `--HCI` forces the legacy path. Both loops populate the same globals (`GoveeTemperatures`, `GoveeMRTGLogs`, `GoveeThermometers`, `GoveeDevices`, and Ruuvi equivalents) and call the same logging/SVG functions. Any change to parsing or logging must work from both loops. HCI code is gated by `#ifdef _BLUEZ_HCI_` (defined in `CMakeLists.txt`); removing the define should still compile.

**Data flow.** Advertisement → `Govee_Temp::ReadMSG(Manufacturer, Data)` parses per model → queued per `bdaddr_t` in `GoveeTemperatures` → flushed to `gvh-<ADDR>-YYYY-MM.txt` every `--time` seconds by `GenerateLogFile` → also fed into in-memory MRTG-style ring buffers (`UpdateMRTGData`, day/week/month/year) → periodically written as SVG (`WriteAllSVG`) and cached (`--cache` dir) so restarts don't re-read all history.

**Log file format is a public contract.** Tab-separated: `Date(UTC) Temp(C) Humidity Battery [Model Temp2 Temp3 Temp4]`. Older `gvh507x_*.txt` names are still read. Don't change column order.

**Persistence file** `gvh-thermometer-types.txt` (in the log dir) records address→model and last-download time; it's read at startup and rewritten at shutdown (`GeneratePersistenceFile`). Downloads are throttled to once per `--download N` days per device.

**Device download protocol.** Service UUID `494e5445-4c4c-495f-524f-434b535f4857` ("INTELLI_ROCKS_HW"). Newer devices (identified by extra service UUIDs `00010203-…-1910` / `02f00000-…-fe00`) require the AES+RC4 session-key handshake documented in README "2026-05-21 Encryption Details". The README is the protocol spec — consult it before touching `bluez_device_download` or `BlueZ_HCI_ConnectAndDownload`.

**Offline modes.** `--no-bluetooth` / `--monitor` skip scanning and just (re)process the log directory into SVGs; `--index` writes an HTML index and exits; `--mrtg ADDR` prints one reading for MRTG and exits. These are handy for testing the SVG/log pipeline without a Bluetooth adapter.

## Adding support for a new Govee model

1. Add an enumerator to `ThermometerType` and both string conversion functions.
2. Add the manufacturer-data parsing branch in `Govee_Temp` (match on model/name/UUID as seen in `BTData/hci-*.txt` captures).
3. If the model reports multiple temperatures or different battery/humidity encodings, check `GenerateLogFile`/`WriteSVG` handle it.
4. Update the README model list.

## Serena note

`.serena/project.yml` may be auto-generated with `language: typescript`; this is a C/C++ project — set it to `cpp` (or `cpp_ccls`) or Serena's symbol tools will fail to start.
