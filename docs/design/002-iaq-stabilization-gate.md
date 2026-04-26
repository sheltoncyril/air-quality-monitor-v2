# 002 — IAQ Stabilization Gate

**Date:** 2026-04-26
**Status:** Proposed
**Motivation:** BSEC IAQ values are unreliable during accuracy levels 0-1 (Stabilizing/Uncertain). Currently IAQ feeds into face expression, buzzer tones, and HA text immediately — before data is trustworthy. This causes misleading faces and false alerts during the first minutes/hours after boot.

---

## Problem

### Current behavior
- Boot self-test checks CO2, PM2.5, and BME temp/humidity — **IAQ accuracy is not checked**
- Face score, buzzer, HA text, ventilation binary all use IAQ via `!isnan()` guard only
- BSEC IAQ starts publishing numbers immediately but accuracy is 0 ("Stabilizing") for ~5 minutes
- Accuracy 1 ("Uncertain") can persist for hours until BSEC builds gas resistance history
- IAQ values during accuracy 0-1 are essentially guesses — can show 25 (Excellent) or 200 (Poor) unpredictably

### BSEC3 accuracy levels

| Numeric | Text | Meaning | Reliable? |
|---------|------|---------|-----------|
| 0 | Stabilizing | Sensor warming up (~5 min) | No |
| 1 | Uncertain | Background history uncertain | No |
| 2 | Calibrating | Found new calibration data | Yes |
| 3 | Calibrated | Fully calibrated | Yes |

### Why not block boot on IAQ accuracy?
BSEC can take 5 minutes to hours to reach accuracy >= 2. Blocking boot would leave the device stuck on splash screen showing no useful data, even though CO2 and PM2.5 are already valid and useful. Bad UX.

---

## Proposed Design

### Approach: Gate IAQ contribution, don't block boot

Boot proceeds as-is (CO2 + PM + BME temp/hum checks). IAQ is **excluded from scoring** until `iaq_accuracy >= 2`. Once stable, IAQ joins the worst-of scoring automatically.

### Changes

#### 1. Add numeric `iaq_accuracy` sensor
The bme690 component already supports it but it's not configured. Add alongside existing text sensor.

```yaml
# In sensor section under bme690 platform
iaq_accuracy:
  name: "IAQ Accuracy Numeric"
  id: iaq_accuracy_num
```

#### 2. Face score lambda
Replace:
```cpp
float iq = id(iaq).state;
int iaq_lv = 0;
if (!isnan(iq)) {
```
With:
```cpp
float iq = id(iaq).state;
int iaq_lv = 0;
bool iaq_ready = !isnan(iq) && !isnan(id(iaq_accuracy_num).state) && (int)id(iaq_accuracy_num).state >= 2;
if (iaq_ready) {
```

#### 3. Buzzer alert lambda
Same pattern — replace `iq_valid = !isnan(iq)` with accuracy check:
```cpp
bool iaq_ready = !isnan(iq) && !isnan(id(iaq_accuracy_num).state) && (int)id(iaq_accuracy_num).state >= 2;
```

#### 4. Air quality text sensor lambda
Same pattern.

#### 5. Ventilation needed binary sensor lambda
Same pattern.

#### 6. Face page display — show calibration status
When IAQ not ready, show "Calibrating" instead of numeric value:
```cpp
it.printf(0, 92, id(font_mid), "IAQ");
bool iaq_ready = !isnan(id(iaq).state) && !isnan(id(iaq_accuracy_num).state) && (int)id(iaq_accuracy_num).state >= 2;
if (!iaq_ready) {
  it.printf(0, 106, id(font_mid), "Cal.");
} else {
  it.printf(0, 106, id(font_mid), "%.0f", id(iaq).state);
}
```

### What stays the same
- Boot self-test — no change, doesn't wait for IAQ
- BME690 IAQ Classification text sensor — already shows "Calibrating" for NaN, useful as-is
- Environment page — already shows `iaq_accuracy_text`, no change needed
- Air Composition page — shows raw IAQ value always (diagnostic page, raw data is fine)

---

## Files to Modify
- `air-quality-monitor.yaml` — all changes in single file:
  - bme690 sensor config (~line 560) — add `iaq_accuracy` numeric sensor
  - Face page lambda (~line 275) — gate IAQ on accuracy + show "Cal." label
  - Buzzer lambda (~line 500) — gate IAQ on accuracy
  - Air quality text sensor (~line 690) — gate IAQ on accuracy
  - Ventilation binary sensor (~line 760) — gate IAQ on accuracy

## Verification
1. `esphome compile air-quality-monitor.yaml` — build succeeds
2. Flash OTA, observe:
   - Face page shows "Cal." for IAQ during first ~5 minutes
   - Face expression based on CO2 + PM2.5 only during calibration
   - Once accuracy hits 2+, IAQ value appears and scoring includes it
3. Check HA — air_quality text shows Excellent/Good/etc. based on CO2+PM only during calibration
4. Verify buzzer doesn't fire IAQ-based alerts during calibration period
