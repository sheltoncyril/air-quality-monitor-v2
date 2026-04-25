# Air Quality Monitor v2

## Project Overview
ESP32-C3 air quality monitor running ESPHome with a custom BME690 BSEC3 component. Displays data on a 128x128 SH1107 OLED. Publishes to Home Assistant via ESPHome API.

## Hardware
- **MCU**: ESP32-C3-DevKitM-1 (RISC-V, single core)
- **Framework**: ESP-IDF (not Arduino)
- **CO2**: SenseAir S88 (NDIR, Modbus over UART, GPIO7/8, 9600 baud)
- **Particulate**: PMS5003 (UART, GPIO20/21, 9600 baud)
- **Gas/Env**: BME690 (I2C 0x77, BSEC3 Sel_IAQ library)
- **Light**: VEML7700 (I2C 0x10)
- **Display**: SH1107 128x128 OLED (I2C 0x3C, 180 rotation)
- **Buzzer**: Passive piezo on GPIO10 (LEDC PWM)
- **Logger**: USB CDC (frees both hardware UARTs)

## Repository Structure
- `air-quality-monitor.yaml` — main ESPHome config (single file, everything)
- `secrets.yaml` — WiFi, API keys, BSEC library path
- Fork: `github.com/sheltoncyril/esphome` branch `dev` — custom `bme690` component

## BME690 Component (Fork)
- Located at: `github.com/sheltoncyril/esphome` branch `dev`, path `esphome/components/bme690/`
- Uses BSEC 3.3 Sel_IAQ variant (NOT plain IAQ)
- BSEC library binary: `bsec_v3-3-0-0/release_bin/Sel_IAQ/bin/esp/esp32_c2c3/libalgobsec.a`
- Config blob: `bsec_config_selectivity[2001]` from `bme690_sel_33v_3s_4d`
- `BSEC_OUTPUT_BREATH_VOC_EQUIVALENT` is NOT supported by Sel_IAQ — do not subscribe to it
- Restricted to ESP32 variants with prebuilt BSEC binaries (ESP32, S2, S3, C2, C3)
- BSEC positive return values (e.g. 100 = timing warning) are warnings, not errors

## Display Architecture
- **Font rules**: font_big (20px) = hero values only. font_mid (12px) = titles, single items. font_sm (8px) = data rows with label+value pairs.
- **180 rotation caveat**: Face pixel art mouth curves and eye shapes must be drawn inverted — smiles need bar below corners, frowns need bar above corners. The rotation flips the visual.
- **Page cycling**: Tick-based in interval lambda (not `show_next`), face page gets 2 ticks (16s), data pages get 1 tick each (8s).

## Boot Self-Test
- 4 phases: 0=Waiting, 1=Testing, 2=Validating, 3=Ready
- Sensors must provide consecutive valid readings (2 normal, 5 if previously failed)
- 90s timeout (11 ticks x 8s interval) triggers reboot
- 3 consecutive boot failures → hardware error screen (frozen, counter resets for next power cycle)
- Per-sensor fail flags persisted in NVS across reboots
- `hw_error` bool (non-persisted) keeps error screen sticky within a session

## Buzzer
- Alert sounds gated on `buzzer_alert` switch (check `id(buzzer_alert).state` before playing)
- Switch turn_on/turn_off sounds suppressed during first 30s (`millis() > 30000`)
- Schmitt trigger hysteresis on all thresholds (~10% band between trip and clear points)
- `delay()` in lambdas blocks the main loop and kills RTTTL playback — avoid delays after `play()`

## SenseAir S88
- ABC (auto baseline calibration) drifts readings if sensor never sees 400ppm — disable for rooms that stay occupied
- Factory calibration: Modbus command 0x7C02 to HR2. Frame: `FE 06 00 01 7C 02 6D 04`
- Factory cal button has 60s boot guard — prevents accidental execution on startup
- The yellow LED flashing inside is the NDIR IR source — normal operation

## Build & Deploy
```bash
# Clean rebuild (needed when fork changes)
rm -rf .esphome/external_components/ .esphome/build/air-quality-monitor-2/.pioenvs

# Build and flash via OTA
esphome run air-quality-monitor.yaml --device air-quality-monitor-2.local --no-logs

# Check logs
esphome logs air-quality-monitor.yaml --device air-quality-monitor-2.local
```

## Versioning
- Version in `esphome.project.version` — shown on splash screen and in HA device info
- Bump version on significant changes
- Create GitHub release with `gh release create vX.Y.Z`

## Common Issues
- **BSEC error -35**: Config/library/header mismatch. Must all be Sel_IAQ variant.
- **BSEC error -34**: Config version mismatch (e.g. BSEC 2.x config with 3.x library).
- **BSEC warning 100**: Timing violation — harmless, logged as warning.
- **Build cache stale**: Delete `.esphome/external_components/` and `.pioenvs` for clean rebuild.
- **Boot logs missed via API**: API connects after setup(), so init errors are lost. Use `dump_config()` fields or persisted fail reasons.
- **NaN in C++**: `int(NaN)` is undefined behavior. Always check `isnan()` before casting.
- **ESPHome filter `{}`**: Use `if (!cond) return {}; return x;` not ternary `cond ? x : {}`.
- **`ESP.restart()` not available on ESP-IDF**: Use `App.safe_reboot()` instead.
