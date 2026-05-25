# Android integration guide

This document covers building, configuring, and running **thermal-daemon** (Intel thermald) on Android, especially **desktop-class x86/x86_64** targets.

For Linux desktop builds, see [README.txt](README.txt). For validation steps, see [TESTING.md](TESTING.md).

The older note in [distribution_integration/android_dbus_service_information](distribution_integration/android_dbus_service_information) is **out of date** — the Android binary does not include D-Bus support.

---

## How Android differs from desktop thermald

| Topic | Linux desktop | Android (`Android.mk`) |
|---|---|---|
| Entry point | `src/main.cpp` | `src/android_main.cpp` |
| D-Bus control API | Built with GLib/GDBus | **Not compiled in** |
| Logging | syslog / g_log | `logcat` tag `THERMALD` |
| Config install | autotools / package manager | `Android.mk` ETC prebuilts |
| Primary XML config path | `/etc/thermald/thermal-conf.xml` | `/data/vendor/thermal-daemon/thermal-conf.xml` *or* `--config-file` |
| Vendor config path | N/A | `/vendor/etc/thermal-daemon/` (`TDCONFDIR`) |
| Runtime state | `/var/run/thermald/` | `/data/vendor/thermal-daemon/` (`TDRUNDIR`) |
| Critical shutdown | `reboot()` | `property_set("sys.powerctl", "shutdown,thermal")` |
| ThermalService / HAL | N/A | **Separate** — thermald does not register with Android framework thermal APIs |

Thermald throttles by writing directly to kernel interfaces (`intel_pstate`, RAPL, powerclamp, cpufreq). It does **not** replace or drive `android.hardware.thermal`.

---

## Building

### AOSP / Soong-wrapped `Android.mk`

The module is defined in [Android.mk](Android.mk):

- **Binary:** `thermal-daemon` (vendor proprietary executable)
- **Libraries:** `liblog`, `libcutils`, `libutils`, static `libxml2`
- **Defines:** `-DANDROID`, `TDRUNDIR`, `TDCONFDIR`

Build targets:

```text
m thermal-daemon
```

Or include in your product package list as needed.

### What gets installed

`Android.mk` installs:

| On device | Source in repo |
|---|---|
| `/vendor/bin/thermal-daemon` | Built from `src/` |
| `/vendor/etc/thermal-daemon/thermal-conf.xml` | [data/thermal-conf.android.xml](data/thermal-conf.android.xml) |
| `/vendor/etc/thermal-daemon/thermald-features.xml` | [data/thermald-features.xml](data/thermald-features.xml) |
| `/vendor/etc/thermal-daemon/thermal-cpu-cdev-order.xml` | [data/thermal-cpu-cdev-order.xml](data/thermal-cpu-cdev-order.xml) |

On most devices `/system/vendor/etc` resolves to the same vendor partition.

**Do not duplicate** these files with `PRODUCT_COPY_FILES` in your device makefile unless you intentionally override them.

### Eng vs user builds

On **eng** builds, `LOG_DEBUG_INFO=1` enables `THERMALD` info/debug logcat lines. On **user** builds, only errors and warnings are typically visible.

---

## init.rc service

Recommended service definition for desktop-class x86 Android:

```rc
service thermal-daemon /vendor/bin/thermal-daemon --ignore-cpuid-check \
    --config-file /vendor/etc/thermal-daemon/thermal-conf.xml
    class main
    user root
    group root
    capabilities SYS_ADMIN
```

Notes:

- **`--ignore-cpuid-check`** — required on Android; the Intel CPUID whitelist is not compiled in for `#ifdef ANDROID`, and desktop CPUs are not in the mobile-oriented upstream tables.
- **`--config-file`** — required to use the vendor-installed XML. Without it, Android looks for config at `/data/vendor/thermal-daemon/thermal-conf.xml`, which is usually empty on first boot.
- **`user root`** — needed to write throttle sysfs nodes (`intel_pstate`, RAPL, thermal policy). Running as `system` often allows startup but prevents actual throttling.
- **Avoid `--adaptive`** on generic desktop x86 unless INT3400 + `data_vault` are verified present. Default mode + XML config is more predictable.

Start at boot (remove `disabled` or trigger explicitly):

```rc
on boot
    start thermal-daemon
```

---

## Configuration files

### `thermal-conf.android.xml`

Shipped as `/vendor/etc/thermal-daemon/thermal-conf.xml`. Designed for desktop x86 Android:

- `<ProductName>*</ProductName>` — matches without reading DMI (often blocked by SELinux)
- Augments the auto **cpu** zone discovered from **coretemp**
- Passive trip at **85°C** (`85000` millidegrees)
- Cooling order: `intel_pstate` → `rapl_controller` → `intel_powerclamp`

Tune `Temperature` for your hardware. If coretemp uses a different sensor name on your platform, adjust `SensorType` (common values: `temp1_input`, `x86_pkg_temp`).

### `thermald-features.xml`

Enables or disables parser, D-Bus (ignored on Android), data vault loading, and kobject uevents. If missing, the daemon defaults all features to **enabled** (see `thd_features_parse.cpp`).

### `thermal-cpu-cdev-order.xml`

Optional override for CPU cooling device order. Same defaults are hardcoded in `thd_zone_cpu.cpp` if this file is absent.

### Runtime preference

File: `/data/vendor/thermal-daemon/thd_preference.conf`

| Value | Mode | Effect |
|---|---|---|
| `1` | ENERGY_CONSERVE (default) | Passive + max trips apply |
| `0` | PERFORMANCE | Passive throttling skipped |
| `2` | DISABLE | All thermald throttling off |

---

## Engine modes and CLI flags

Android entry point: [src/android_main.cpp](src/android_main.cpp)

| Flag | Purpose |
|---|---|
| `-i` / `--ignore-cpuid-check` | Allow startup without CPUID / platform list match |
| `-a` / `--adaptive` | DPTF/GDDV adaptive engine (requires INT3400 `data_vault`) |
| `-n` / `--no-daemon` | Foreground mode — use for debugging |
| `-c` / `--config-file PATH` | Explicit thermal-conf.xml path |
| `-e` / `--exclusive_control` | Take over kernel thermal zone policy |
| `-p N` | Poll interval seconds (default 4; `0` disables polling) |
| `-d` / `--ignore-default-control` | Skip auto cpu zone creation |

There is **no** `--dbus-enable` on Android.

---

## Fork-specific behavior (desktop Android)

This tree includes Android-specific changes upstream may not have:

1. **ACPI PM profile** — on Android, all PM profiles are allowed (desktop `pm_profile=1` no longer causes exit). See `check_acpi_platform_profile()` in `src/thd_engine.cpp`.
2. **Feature parser defaults** — `feature_list` is initialized even when `thermald-features.xml` is missing, preventing a startup SIGSEGV in `parser_init()`.
3. **Android build sources** — `thd_features_parse.cpp` and `thd_util.cpp` must be in `LOCAL_SRC_FILES` (linked on desktop autotools, easy to miss on Android).

---

## Kernel requirements

Ensure the target kernel exposes:

```bash
# Temperature
/sys/class/hwmon/hwmon*/name  → coretemp
/sys/class/hwmon/hwmon*/temp*_input

# Throttling (at least one should work)
/sys/devices/system/cpu/intel_pstate/max_perf_pct
/sys/class/powercap/intel-rapl*/

# Optional
/sys/devices/virtual/thermal/cooling_device*   # powerclamp
```

Recommended kernel options: `CONFIG_X86_PKG_TEMP_THERMAL`, `CONFIG_X86_INTEL_PSTATE`, `CONFIG_INTEL_RAPL`, `CONFIG_INTEL_POWERCLAMP`.

---

## SELinux

Common denials on Android x86:

- **`vendor_sysfs_dmi_id`** — DMI reads for platform matching (use `<ProductName>*</ProductName>` to avoid dependency)
- **`intel_pstate` / `powercap` writes** — required for throttling
- **`sysfs_thermal_management`** — thermal zone policy changes if using exclusive control

Example checks:

```bash
adb logcat -d | grep -i 'avc.*thermal'
adb shell getenforce
```

---

## Relationship to Android Thermal HAL

`vendor.thermal-hal-*` and `ThermalService` are independent. If framework-visible thermal status or app throttling callbacks are required, implement or extend a **Thermal HAL** that reads the same sensors. thermald alone caps CPU performance via kernel sysfs.

---

## Troubleshooting quick reference

| Symptom | Likely cause |
|---|---|
| `restarting` / SIGSEGV at boot | Missing `thd_features_parse.cpp` / `thd_util.cpp` in build, or unfixed feature-list crash |
| Exits immediately, no crash | PM profile gate (if not using this fork), engine init fatal (no sensors/zones), PID lock failure |
| Runs but `max_perf_pct` always 100 | Idle temps below trip point; XML not applied (`ProductName` mismatch); SELinux blocking writes |
| XML ignored | Missing `--config-file`; platform not matched; example `EXAMPLE_SYSTEM` config still installed |
| `--adaptive` fails silently | No INT3400 / `data_vault`; falls back to default engine |

See [TESTING.md](TESTING.md) for step-by-step verification commands.
