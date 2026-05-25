# Testing guide

How to validate thermald on **Linux desktop** (upstream workflow) and **Android** (ADB on-device checks).

Implementation details: [ANDROID.md](ANDROID.md). Desktop build instructions: [README.txt](README.txt).

---

## Linux desktop (autotools)

### CI / local build smoke test

GitHub Actions ([.github/workflows/ci.yml](.github/workflows/ci.yml)) runs:

```bash
./autogen.sh
make -j$(nproc)
make clang-tidy
```

Local equivalent:

```bash
sudo apt install autoconf autoconf-archive automake g++ \
  libglib2.0-dev libxml2-dev libupower-glib-dev libevdev-dev gtk-doc-tools
./autogen.sh
make -j$(nproc)
```

### Kernel emulation test suite

Documented in [test/readme_test.txt](test/readme_test.txt):

1. Copy [test/thermald_test_kern_module.c](test/thermald_test_kern_module.c) into kernel `drivers/thermal/`
2. Enable `CONFIG_THERMAL_EMULATION=y`
3. Load the module, then run [test/exec_config_tests.sh](test/exec_config_tests.sh)

Individual scenario scripts (require running thermald + D-Bus on desktop):

| Script | Config | Exercises |
|---|---|---|
| [test/default.sh](test/default.sh) | `default.xml` | Default cpu zone |
| [test/intel_pstate.sh](test/intel_pstate.sh) | `intel_pstate.xml` | intel_pstate cdev |
| [test/rapl.sh](test/rapl.sh) | `intel_rapl.xml` | RAPL cdev |
| [test/cpufreq.sh](test/cpufreq.sh) | `cpufreq.xml` | cpufreq cdev |
| [test/powerclamp.sh](test/powerclamp.sh) | `powerclamp.xml` | powerclamp cdev |
| [test/processor.sh](test/processor.sh) | `processor_acpi.xml` | ACPI processor zone |

Run all:

```bash
cd test && ./run_all_tests.sh
```

These tests use **D-Bus Reinit** and thermal emulation sysfs — they do **not** run on the Android binary.

### Manual desktop checks

```bash
systemctl status thermald
busctl introspect org.freedesktop.thermald /org/freedesktop/thermald
cat /sys/devices/system/cpu/intel_pstate/max_perf_pct
```

---

## Android on-device testing

### Prerequisites

- `adb root` (or userdebug/eng build)
- `thermal-daemon` installed at `/vendor/bin/thermal-daemon`
- Service running as **root**

### 1. Service health

```bash
adb shell getprop init.svc.thermal-daemon
# expected: running

adb shell pidof thermal-daemon
adb shell "tr '\0' ' ' </proc/$(pidof thermal-daemon)/cmdline"
# expected flags include:
#   --ignore-cpuid-check
#   --config-file /vendor/etc/thermal-daemon/thermal-conf.xml
```

If status is `restarting`, capture crash info immediately:

```bash
adb logcat -d -b all | grep -iE 'thermal-daemon|THERMALD|DEBUG' | tail -80
adb shell ls -lt /data/tombstones/ | head -3
```

### 2. Config on device

```bash
adb shell ls -la /vendor/etc/thermal-daemon/
adb shell head -20 /vendor/etc/thermal-daemon/thermal-conf.xml
```

Confirm `ProductName` is `*` (not `EXAMPLE_SYSTEM`) and trip temperature matches expectations.

Quote grep patterns on device (Android `sh` treats `|` as a pipe inside double quotes):

```bash
adb shell "grep -E 'ProductName|Temperature|Type' /vendor/etc/thermal-daemon/thermal-conf.xml"
```

### 3. Hardware prerequisites

```bash
# coretemp present?
adb shell 'grep -H . /sys/class/hwmon/hwmon*/name | grep coretemp'

# Package temp and limits (replace hwmon5 if different)
adb shell cat /sys/class/hwmon/hwmon5/temp1_input
adb shell cat /sys/class/hwmon/hwmon5/temp1_max
adb shell cat /sys/class/hwmon/hwmon5/temp1_crit

# Throttle interface
adb shell cat /sys/devices/system/cpu/intel_pstate/max_perf_pct
adb shell ls /sys/class/powercap/intel-rapl* 2>/dev/null
```

### 4. Manual throttle path test

Verifies kernel + permissions independent of thermald logic:

```bash
adb shell 'echo 50 > /sys/devices/system/cpu/intel_pstate/max_perf_pct && cat /sys/devices/system/cpu/intel_pstate/max_perf_pct'
# expected: 50

adb shell 'echo 100 > /sys/devices/system/cpu/intel_pstate/max_perf_pct'
```

If manual writes fail, fix **SELinux** and/or run the service as root before debugging thermald policy.

### 5. Foreground daemon test

Stop the service and run in the foreground to see init errors on eng builds:

```bash
adb shell stop thermal-daemon
adb shell /vendor/bin/thermal-daemon --ignore-cpuid-check \
    --config-file /vendor/etc/thermal-daemon/thermal-conf.xml --no-daemon
```

In another terminal:

```bash
adb logcat -s THERMALD
```

### 6. Load test + throttle observation

Watch throttle and temperature (adjust `hwmon5` if needed):

```bash
adb shell 'while true; do
  printf "max_perf_pct=%s coretemp=" "$(cat /sys/devices/system/cpu/intel_pstate/max_perf_pct)"
  cat /sys/class/hwmon/hwmon5/temp1_input
  echo
  sleep 2
done'
```

Generate CPU load (Android-compatible shell syntax):

```bash
adb shell 'n=$(getconf _NPROCESSORS_ONLN); i=0; while [ "$i" -lt "$n" ]; do (while true; do :; done) & i=$((i+1)); done; sleep 120; kill $(jobs -p) 2>/dev/null'
```

**Pass criteria:** when coretemp exceeds the passive trip (default **85000** = 85°C in `thermal-conf.android.xml`), `max_perf_pct` drops below 100.

At idle (~50°C), `max_perf_pct=100` is **expected**, not a failure.

### 7. Preference and runtime files

```bash
adb shell cat /data/vendor/thermal-daemon/thd_preference.conf
# 1 = ENERGY_CONSERVE (throttling enabled)

adb shell ls -la /data/vendor/thermal-daemon/
```

### 8. SELinux audit

```bash
adb shell getenforce
adb logcat -d | grep -iE 'avc.*thermal-daemon|thermal-daemon.*denied'
```

---

## Android troubleshooting matrix

| Observation | What to check |
|---|---|
| SIGSEGV in `parser_init()` | Rebuild with `thd_features_parse.cpp` + `thd_util.cpp`; ensure feature-list init fix is present |
| SIGSEGV in `read_thermal_sensors()` | Same as above; check tombstone stack |
| `Non mobile platform, exiting` | Need Android PM profile patch in `thd_engine.cpp` |
| Runs, no throttle under load | Trip temp too high; XML not matched; SELinux blocking pstate writes |
| `max_perf_pct=100` at 50°C idle | Normal — not a bug |
| Config not applied | Missing `--config-file`; wrong `ProductName`; old example XML still installed |
| D-Bus tools fail | Expected on Android — no D-Bus interface in this binary |

---

## Test matrix checklist (Android release sign-off)

Use this before declaring thermal support ready on a new device/image:

- [ ] `init.svc.thermal-daemon` = `running` after boot
- [ ] cmdline includes `--ignore-cpuid-check` and `--config-file`
- [ ] `/vendor/etc/thermal-daemon/thermal-conf.xml` contains `ProductName>*`
- [ ] `coretemp` hwmon present
- [ ] Manual `max_perf_pct` write succeeds as root
- [ ] Under CPU load, coretemp rises
- [ ] Above passive trip, `max_perf_pct` decreases (or RAPL PL1 drops)
- [ ] No recurring SIGSEGV in logcat across 10+ service restarts
- [ ] SELinux enforcing: no denials on pstate/powercap writes (or policy added)

---

## Adding automated Android tests (future)

The repo currently has **no** Android instrumentation tests. Reasonable additions for a product tree:

1. **VTS-style root test** — start daemon, assert pid alive after 5s, parse `/proc/pid/cmdline`
2. **Sysfs probe test** — verify `coretemp` + `intel_pstate/max_perf_pct` exist on x86 products
3. **Policy test** — push a test conf with a low trip + emulated temp if the kernel supports `CONFIG_THERMAL_EMULATION` on Android kernels

Desktop [test/](test/) scripts remain the reference for cooling-device behavior but require the full Linux thermald + D-Bus stack.
