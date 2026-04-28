# Plan 1: Core Calibration & Missing Features for BME690 Component

## Context

The custom `bme690` ESPHome component (fork: `sheltoncyril/esphome@dev`) wraps BSEC3 Sel_IAQ but only exposes 3 config options (`bsec_library`, `sample_rate`, `state_save_interval`). Several useful features exist in code but aren't configurable, and some BSEC outputs are subscribed but discarded. ESPHome's official `bme68x_bsec2` component exposes more options that we should match or exceed.

## Changes — Ordered by Impact

### 1. Temperature Offset (HIGH — affects accuracy)

**What:** Expose `ext_temp_offset_` as config option `temperature_offset`.
**Why:** BME690 reads 2-8°C high inside enclosures. Offset corrects both temperature AND humidity via BSEC's compensation. Both official ESPHome BME components expose this.

**Files:**
- `__init__.py`: Add `CONF_TEMPERATURE_OFFSET` optional float, default 0.0
- `bme690.h`: Add `set_temperature_offset(float offset)` setter, wire to `ext_temp_offset_`

**Config:**
```yaml
bme690:
  temperature_offset: 4.2  # measured delta vs reference thermometer
```

---

### 2. Supply Voltage Selection (MEDIUM — affects compensation math)

**What:** Add `supply_voltage` config option (3.3V or 1.8V).
**Why:** BSEC config blob encodes supply voltage for gas resistance compensation. Wrong voltage = wrong IAQ. Currently hardcoded to 3.3V via config blob name `bme690_sel_33v_3s_4d`.

**Files:**
- `__init__.py`: Add `CONF_SUPPLY_VOLTAGE` enum (`3.3V`, `1.8V`), default `3.3V`
- `__init__.py`: Select matching config blob variant (`sel_33v` vs `sel_18v`)
- May need both config blobs included or conditionally loaded

**Config:**
```yaml
bme690:
  supply_voltage: 3.3V
```

**Note:** Only matters if someone runs BME690 at 1.8V. Low priority for our specific hardware (3.3V), but good for library completeness.

---

### 3. Stabilization & Run-in Status Sensors (MEDIUM — aids diagnostics)

**What:** Expose `BSEC_OUTPUT_STABILIZATION_STATUS` and `BSEC_OUTPUT_RUN_IN_STATUS` as binary sensors.
**Why:** Already subscribed but discarded. Would show when gas sensor is warmed up and ready. Directly useful for boot self-test — could replace or augment the current consecutive-readings heuristic.

**Files:**
- `sensor.py` or new `binary_sensor.py`: Add optional binary sensor outputs
- `bme690.h`: Add handler cases in `handle_bsec_outputs()`, publish to binary sensors

**Config:**
```yaml
binary_sensor:
  - platform: bme690
    stabilization_status:
      name: "BME690 Stabilized"
    run_in_status:
      name: "BME690 Run-in Complete"
```

---

### 4. Raw Sensor Outputs (LOW — debugging aid)

**What:** Expose raw temperature, humidity, pressure, gas resistance as optional sensors.
**Why:** Already subscribed. Useful for comparing raw vs BSEC-compensated values. Helps diagnose temperature offset calibration.

**Files:**
- `sensor.py`: Add optional raw sensor configs (raw_temperature, raw_humidity, raw_pressure, raw_gas_resistance)
- `bme690.h`: Add handler cases for raw output IDs

**Config:**
```yaml
sensor:
  - platform: bme690
    raw_temperature:
      name: "BME690 Raw Temperature"
    raw_humidity:
      name: "BME690 Raw Humidity"
```

---

### 5. Compensated Gas Output (LOW — advanced users)

**What:** Subscribe to and expose `BSEC_OUTPUT_COMPENSATED_GAS` (ID 18).
**Why:** Linearized gas resistance corrected for temperature and humidity. More meaningful than raw gas resistance for trend analysis. Not currently subscribed.

**Files:**
- `sensor.py`: Add optional `compensated_gas` sensor
- `bme690.h`: Add to subscription list and handler

---

### 6. Operating Age Config (LOW — niche)

**What:** Add `operating_age` option (`4d` or `28d`).
**Why:** Controls BSEC calibration history window. 4d = faster adaptation to new environments, 28d = more stable long-term. Official BSEC2 component exposes this. Need to verify BSEC3 Sel_IAQ config blobs exist for both variants.

**Files:**
- `__init__.py`: Add enum, select matching config blob

---

### 7. Baseline Tracker Control (LOW — edge case)

**What:** Expose `BSEC_INPUT_DISABLE_BASELINE_TRACKER` as config option.
**Why:** In rooms that never get "clean air," BSEC's auto-baseline drifts IAQ readings upward. Disabling lets you freeze baseline. Very niche — most users want auto-calibration.

**Files:**
- `__init__.py`: Add boolean config
- `bme690.h`: Conditionally push input ID 23 in `push_inputs_to_bsec()`

---

### 8. Self-Test on Boot (LOW — reliability)

**What:** Call `bme68x_selftest_check()` during setup.
**Why:** BME69x driver includes hardware self-test that validates T/P/H/gas against internal thresholds. Could catch hardware failures early and feed into boot self-test system.

**Files:**
- `bme690.h`: Call in `setup()` before BSEC init, log result, set fail flag if `BME68X_E_SELF_TEST`

---

## Implementation Order

1. Temperature offset (quick win, high value)
2. Stabilization/run-in status (medium effort, useful for boot system)
3. Raw sensor outputs (small effort, debugging value)
4. Compensated gas (small effort)
5. Supply voltage (medium effort, library completeness)
6. Self-test (small effort, reliability)
7. Operating age (needs config blob investigation)
8. Baseline tracker (niche, needs testing)

## Not Changing

- **Oversampling**: Hardcoded 16x, BSEC overrides anyway. No user benefit.
- **Filter**: Must stay OFF for BSEC. Not configurable by design.
- **ODR**: Must stay NONE for BSEC forced mode. Not configurable.
- **Initial heater temp/duration**: BSEC overrides immediately. Exposing adds confusion.
- **`BSEC_OUTPUT_BREATH_VOC_EQUIVALENT`**: Not supported by Sel_IAQ variant. Don't add.
