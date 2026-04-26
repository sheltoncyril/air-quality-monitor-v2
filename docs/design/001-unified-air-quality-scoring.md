# 001 — Unified Air Quality Scoring System

**Date:** 2026-04-26
**Status:** Proposed
**Motivation:** Face expression, buzzer tones, HA status text, and ventilation alert all use separate scoring systems with different thresholds. A single bad metric (e.g., CO2 >2000) doesn't trigger an appropriately bad face because the additive scoring dilutes individual severity.

---

## Problem

Four independent scoring systems exist, each with different thresholds:

### Face Score (display)
Additive — 11 thresholds across CO2/PM2.5/IAQ. Max 11 points mapped to 6 expressions.

| Condition | Points |
|-----------|--------|
| CO2 > 800, 1000, 1500, 2500, 5000 | +1 each (max 5) |
| PM2.5 > 12, 35, 55 | +1 each (max 3) |
| IAQ > 100, 150, 250 | +1 each (max 3) |

| Score | Expression |
|-------|-----------|
| 0 | ^_^ ecstatic |
| 1 | :) happy |
| 2 | :\| neutral |
| 3 | :( worried |
| 4+ | >:( angry |
| 5+ | X_X dead |

**Problem:** CO2 at 2000 ppm (dangerous) only scores 3 points = worried face. Meanwhile CO2=900 + PM=15 + IAQ=110 (all mildly elevated) also scores 3 = same worried face.

### Buzzer Score (60s interval)
6 Schmitt-trigger thresholds with hysteresis. Only triggers on score change.

| Metric | Trip | Clear | Points |
|--------|------|-------|--------|
| CO2 | >1000 | <900 | +1 |
| CO2 | >1500 | <1350 | +1 |
| PM2.5 | >35 | <30 | +1 |
| PM2.5 | >55 | <48 | +1 |
| IAQ | >150 | <130 | +1 |
| IAQ | >250 | <220 | +1 |

### HA Air Quality Text
5 thresholds: CO2 >1000/1500, PM >35, IAQ >150/250. Maps to Excellent/Good/Fair/Poor/Bad.

### Ventilation Needed Binary
CO2 >1200 OR PM >35 OR IAQ >200.

---

## Proposed Design: Worst-Of Category Scoring

### Core Principle
Each pollutant sensor maps independently to a 0–4 severity level. Overall air quality = **worst individual level** (not additive). One bad metric = bad result.

### Per-Sensor Category Levels

Based on EPA 2024 AQI, ASHRAE ventilation guidelines, WHO PM2.5 guidelines, and Bosch BSEC official IAQ scale.

#### CO2 (SenseAir S88 — direct NDIR measurement)

| Level | Range (ppm) | Basis |
|-------|-------------|-------|
| 0 Excellent | < 800 | Well-ventilated, outdoor-adjacent |
| 1 Good | 800–1000 | ASHRAE adequate ventilation zone |
| 2 Fair | 1000–1500 | Ventilation declining, some drowsiness |
| 3 Poor | 1500–2000 | Inadequate ventilation, cognitive effects |
| 4 Bad | > 2000 | Significant health impact |

#### PM2.5 (PMS5003)

| Level | Range (µg/m³) | Basis |
|-------|---------------|-------|
| 0 Excellent | < 9 | EPA 2024 "Good" |
| 1 Good | 9–35 | EPA "Moderate" |
| 2 Fair | 35–55 | EPA "USG" |
| 3 Poor | 55–150 | EPA "Unhealthy" |
| 4 Bad | > 150 | EPA "Very Unhealthy" |

#### IAQ (BSEC3 Sel_IAQ)

| Level | Range | Basis |
|-------|-------|-------|
| 0 Excellent | < 51 | Bosch "Excellent" |
| 1 Good | 51–100 | Bosch "Good" |
| 2 Fair | 101–150 | Bosch "Lightly Polluted" |
| 3 Poor | 151–250 | Bosch "Moderately/Heavily Polluted" |
| 4 Bad | > 250 | Bosch "Severely Polluted" |

### Overall Score
```
overall_level = max(co2_level, pm25_level, iaq_level)
```

### Face Expression Mapping

| Level | Expression | Rationale |
|-------|-----------|-----------|
| 0 Excellent | ^_^ ecstatic | Air is great |
| 1 Good | :) happy | Normal indoor air |
| 2 Fair | :\| neutral | Starting to degrade |
| 3 Poor | :( worried | Open a window |
| 4 Bad | >:( angry | Take action |
| — hw_error | X_X dead | Reserved for hardware failure only |

### Buzzer Mapping

Buzzer triggers on **level transitions** (not raw threshold crossings). Schmitt trigger hysteresis applied per-sensor at level boundaries.

| Level | Tone | RTTTL |
|-------|------|-------|
| 0 Excellent | Rising major triad | `d=16,o=6,b=180:c,e,g` |
| 1 Good | Two rising notes | `d=16,o=6,b=180:e,g` |
| 2 Fair | Repeated note | `d=16,o=5,b=160:a,a` |
| 3 Poor | Descending minor | `d=16,o=5,b=160:e,d,c` |
| 4 Bad | Low double beep | `d=8,o=4,b=120:c,p,c` |

Hysteresis: each level boundary has ~10% band. E.g., CO2 trips to Fair at 1000, clears back to Good at 900.

#### Hysteresis Thresholds

| Metric | Level | Trip | Clear |
|--------|-------|------|-------|
| CO2 | 1 Good | >800 | <720 |
| CO2 | 2 Fair | >1000 | <900 |
| CO2 | 3 Poor | >1500 | <1350 |
| CO2 | 4 Bad | >2000 | <1800 |
| PM2.5 | 1 Good | >9 | <7 |
| PM2.5 | 2 Fair | >35 | <30 |
| PM2.5 | 3 Poor | >55 | <48 |
| PM2.5 | 4 Bad | >150 | <130 |
| IAQ | 1 Good | >51 | <45 |
| IAQ | 2 Fair | >101 | <85 |
| IAQ | 3 Poor | >151 | <130 |
| IAQ | 4 Bad | >251 | <220 |

### HA Text Sensor
Derives directly from overall level: Excellent / Good / Fair / Poor / Bad.

### Ventilation Needed Binary
`overall_level >= 2` (Fair or worse).

---

## Sensors NOT Included in Composite Score (and why)

| Sensor | Reason |
|--------|--------|
| **Temperature** | Comfort, not toxicity. Bad temp doesn't mean bad air. |
| **Humidity** | Same — affects comfort, not air quality per se. Humidity already factors into BSEC IAQ (25% weight). |
| **CO2 Equivalent** (BSEC) | Redundant with IAQ. Also less reliable than direct SenseAir CO2. |
| **Gas Percentage** (BSEC) | Internal BSEC metric, redundant with IAQ output. |
| **Gas Resistance** (raw) | Already consumed by BSEC to produce IAQ. Using both = double-counting. |
| **PM1.0, PM10** | PM2.5 is primary health-relevant metric. PM10 less useful indoors. Could add PM10 later if needed. |
| **Light** | Environmental, not air quality. |
| **Pressure** | Weather, not air quality. |

### Potential Comfort Indicator (separate from AQ score)
Temperature + humidity could drive a **separate** comfort face or indicator on the environment page, not mixed into air quality. This is out of scope for this design.

---

## Gaps Identified

### Current Setup
1. **No averaged PM2.5** — EPA AQI uses 24-hour average, but PMS5003 gives instantaneous readings. Short spikes (cooking, etc.) trigger alerts that don't reflect sustained exposure. Consider adding a sliding window average (e.g., 10-minute or 1-hour) alongside instantaneous.
2. **BSEC calibration period** — IAQ reads 0-25 during initial 4-day calibration. Scoring should ignore IAQ when `iaq_accuracy < 2` (BSEC "medium" accuracy).
3. **No PM10 scoring** — PM10 is available from PMS5003 but unused. In most indoor environments PM2.5 is sufficient, but dust-heavy environments (workshops, renovations) would benefit from PM10 tracking.

### Hardware Gaps (for future versions)
4. **No dedicated VOC sensor** — BSEC IAQ includes VOC via gas resistance, but a dedicated TVOC sensor (e.g., SGP41) would give calibrated ppb readings. Current BME690 gas resistance is relative, not absolute.
5. **No formaldehyde sensor** — Important for new furniture/paint off-gassing. Would need dedicated sensor (e.g., Dart WZ-S).
6. **No NO2/O3 sensor** — Relevant near roads or with gas stoves. Would need electrochemical sensor.
7. **No radon** — Significant indoor health hazard in some regions. Requires dedicated alpha particle detector.

---

## Files to Modify
- `air-quality-monitor.yaml` — all changes in single file:
  - Face score lambda (~lines 275-286)
  - Face expression drawing (~lines 288-331) — no pixel art changes needed, just score mapping
  - Buzzer alert lambda (~lines 490-531)
  - `air_quality` text sensor (~lines 667-677)
  - Ventilation needed binary sensor (~line 719)

## Verification
1. Build: `esphome compile air-quality-monitor.yaml`
2. Manual threshold testing: inject known values via HA service calls or sensor overrides
3. Verify face matches expected expression at each level boundary
4. Verify buzzer only fires on level transitions
5. Verify HA text sensor updates correctly
6. Verify hysteresis prevents rapid toggling near boundaries
