# Plan 2: Smell Calibration & Gas Classification for BME690 in ESPHome

## Background

BME690 (and BME688) support gas scanning with ML-based classification via Bosch's BSEC library. The sensor cycles through up to 10 heater temperature/duration steps per scan, producing a gas resistance "fingerprint." A trained model (exported from BME AI-Studio) maps fingerprints to gas classes or concentrations.

**Two output types:**
- **Classification** (`GAS_ESTIMATE_1-4`): "What gas?" — probability 0.0-1.0 for up to 4 classes
- **Regression** (`REGRESSION_ESTIMATE_1-4`): "How much?" — quantitative concentration for up to 4 targets

The trained model is embedded as a BSEC config blob that replaces the standard Sel_IAQ config.

## How It Works End-to-End

### Training Pipeline (Outside ESPHome)
1. **Hardware**: BME690 dev kit (App Board 3.1) or any BME688/690 breakout
2. **Software**: BME AI-Studio (Windows desktop app, free from Bosch)
3. **Burn-in**: New sensor needs ~24h initial stabilization
4. **Data collection**: Expose sensor to target gases, label samples in AI-Studio
5. **Training**: AI-Studio trains classification/regression model
6. **Export**: Outputs a BSEC config blob (C array `bsec_config[]` + `.config` binary)

### Deployment (In ESPHome)
1. Replace standard `bsec_config_iaq` blob with trained model blob
2. Subscribe to `GAS_ESTIMATE_1-4` or `REGRESSION_ESTIMATE_1-4` outputs
3. BSEC handles heater cycling, feature extraction, and inference internally
4. Outputs arrive with accuracy field: 0=collecting features, 3=prediction ready

## Implementation Plan

### Phase 1: Config Blob Selection Infrastructure

**Goal:** Allow users to choose between standard IAQ config and custom trained configs.

**Changes:**
- `__init__.py`: Add `algorithm_output` config option:
  - `iaq` (default) — standard Sel_IAQ, current behavior
  - `classification` — gas classification mode
  - `regression` — gas quantification mode
- `__init__.py`: Add `bsec_config` optional config:
  - Path to custom BSEC config blob file (exported from AI-Studio)
  - When provided, overrides the built-in `bsec_config_iaq`
  - Validated: must be a file path to a `.config` or `.c` file
- `bme690.h`: Conditionally include either built-in config or user-provided config at compile time

**Config:**
```yaml
bme690:
  bsec_library: !secret bsec_library_path
  algorithm_output: classification  # or regression, or iaq
  bsec_config: "custom_configs/my_trained_model.config"
  sample_rate: LP
```

**Challenge:** BSEC config is a C array compiled into firmware. Options:
- A) Generate a `.c` file from binary `.config` at build time (Python codegen in `__init__.py`)
- B) Store blob in SPIFFS/LittleFS and load at runtime via `bsec_set_configuration()`
- C) User provides a `.h` file with the C array, included via build flag

**Recommendation:** Option A — Python codegen. Most ESPHome-native. `__init__.py` reads the binary config file, converts to C array, writes a generated `.c` file during codegen phase. BSEC3 already loads config via `bsec_set_configuration()` at runtime from a buffer, so we just need to swap which buffer.

---

### Phase 2: Gas Estimate Sensors

**Goal:** Expose classification outputs as ESPHome sensors.

**Changes:**
- `sensor.py`: Add optional sensors:
  - `gas_estimate_1` through `gas_estimate_4` (float 0.0-1.0, probability)
  - `gas_estimate_1_accuracy` through `gas_estimate_4_accuracy` (int 0-3)
- `text_sensor.py`: Add optional text sensors:
  - `gas_class_1` through `gas_class_4` (label name from training)
  - Or a single `detected_gas` that shows highest-probability class
- `bme690.h`:
  - Subscribe to `BSEC_OUTPUT_GAS_ESTIMATE_1-4` (IDs 22-25)
  - Handle outputs: publish probability + accuracy
  - Subscribe to `BSEC_OUTPUT_RAW_GAS_INDEX` (ID 26) for heater step tracking

**Config:**
```yaml
sensor:
  - platform: bme690
    gas_estimate_1:
      name: "Coffee Probability"
    gas_estimate_2:
      name: "Alcohol Probability"
    gas_estimate_3:
      name: "Smoke Probability"
    gas_estimate_4:
      name: "Clean Air Probability"
```

---

### Phase 3: Regression Estimate Sensors

**Goal:** Expose quantitative gas concentration outputs.

**Changes:**
- `sensor.py`: Add optional sensors:
  - `regression_estimate_1` through `regression_estimate_4` (float, ppm/ppb)
  - Unit of measurement configurable (depends on what was trained)
- `bme690.h`: Subscribe to `BSEC_OUTPUT_REGRESSION_ESTIMATE_1-4` (IDs 27-30)

**Config:**
```yaml
sensor:
  - platform: bme690
    regression_estimate_1:
      name: "Ethanol Concentration"
      unit_of_measurement: "ppm"
```

---

### Phase 4: Gas Scanning Diagnostics

**Goal:** Expose gas scanning state for monitoring and debugging.

**Changes:**
- `sensor.py`: Add `raw_gas_index` sensor (int 0-9, current heater step)
- `bme690.h`: Subscribe to `BSEC_OUTPUT_RAW_GAS_INDEX` (ID 26)
- Optional: Expose per-step gas resistance as a multi-value sensor

**Config:**
```yaml
sensor:
  - platform: bme690
    raw_gas_index:
      name: "BME690 Heater Step"
```

---

### Phase 5: Dual Mode — IAQ + Classification (Advanced)

**Goal:** Run IAQ and gas classification simultaneously.

**Problem:** Standard IAQ and classification use different BSEC config blobs. Unclear if BSEC3 supports both output sets with a single config.

**Investigation needed:**
- Can Sel_IAQ config produce GAS_ESTIMATE outputs? (Likely no — different heater profiles)
- Can a custom trained config also produce IAQ? (Possible — AI-Studio may embed IAQ in trained model)
- Alternative: Run two BSEC instances? (Probably not — single sensor hardware)

**If not possible:** Document as either/or — IAQ mode OR classification mode, not both. User chooses via `algorithm_output` config.

---

## Training Workflow for Users

### Option A: Full AI-Studio Pipeline
1. Get BME690 dev kit or wire sensor to USB-capable MCU
2. Install BME AI-Studio (Windows)
3. Collect labeled gas samples (minimum ~10 samples per class)
4. Train model in AI-Studio
5. Export BSEC config blob
6. Copy to ESPHome project, reference in YAML config
7. Flash and monitor

### Option B: Prebuilt Models (Community)
- Bosch provides some example trained models in BSEC distribution
- Community could share trained configs for common use cases
- Config blob is hardware-agnostic (works across BME688/690 units)

### Option C: On-Device Collection + External Training (Future)
1. ESPHome collects raw gas scanning data (10-step fingerprints)
2. Data exported via HA or MQTT
3. Training done externally (AI-Studio or custom ML)
4. Config blob generated and flashed

---

## Challenges & Risks

| Challenge | Mitigation |
|-----------|-----------|
| AI-Studio is Windows-only | Document alternatives, consider Wine compatibility |
| Training requires physical gas samples | Provide example configs for common scenarios |
| Config blob must match BSEC version | Validate version during config load, log clear errors |
| Classification accuracy varies by environment | Expose accuracy field, recommend re-training per location |
| Flash size increase from multiple configs | Only include selected config, not all variants |
| Sel_IAQ vs custom config mutual exclusion | Clear documentation, config validation |

## File Impact Summary

| File | Changes |
|------|---------|
| `__init__.py` | `algorithm_output` enum, `bsec_config` path, codegen for custom blob |
| `sensor.py` | 8 new optional sensors (4 gas_estimate + 4 regression_estimate + gas_index) |
| `text_sensor.py` | Optional gas class labels |
| `bme690.h` | Subscribe to IDs 22-30, handle outputs, conditional subscription based on mode |

## Implementation Order

1. Phase 1 (config infrastructure) — enables everything else
2. Phase 2 (classification sensors) — most requested use case
3. Phase 4 (diagnostics) — small effort, aids debugging
4. Phase 3 (regression sensors) — same pattern as Phase 2
5. Phase 5 (dual mode) — needs investigation, may not be feasible
