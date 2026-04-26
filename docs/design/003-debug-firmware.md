# 003 — Debug Mode Firmware

**Date:** 2026-04-26
**Status:** In Progress
**Motivation:** Need a separate firmware variant for testing hardware failure, boot recovery, and scoring verification. Must be visually distinct from production and independently buildable.

---

## Problem

No way to test error states without physically disconnecting sensors or waiting for real bad air. Need to verify:
- Face expressions at each scoring level
- Buzzer tones at each level
- Hardware error screen and recovery
- Boot failure counter and reboot sequence

## Approach: ESPHome Packages + Composition

### File Structure
```
air-quality-monitor.yaml        # Production (add substitutions for name)
debug.yaml                      # Debug variant — packages base + adds debug features
```

`debug.yaml` uses `packages:` to inherit all production config, then adds debug-only components. No code duplication of sensor/display logic.

### What debug.yaml adds
1. **Override name** to `air-quality-monitor-2-debug` via substitutions
2. **Select entity** — "Test Scenario" dropdown in HA:
   - `None` (default — normal operation)
   - `Scoring Walkthrough` — cycles through all 5 face levels (8s each), plays each tone
   - `HW Failure` — sets hw_error flag, shows hardware error screen + beep
   - `Boot Recovery` — sets boot_fails=2, reboots (next boot needs 5 consecutive good readings)
3. **Global** — `test_mode` int to track active test
4. **Interval** — test execution logic (separate from production intervals)
5. **"DEBUG" label** — shown on splash screen via substitution in the splash lambda

### Key Design Decision: Additive, Not Invasive

Debug features drive outputs DIRECTLY (show page, play tone, draw face) rather than overriding sensor values. This means:
- No need to modify production scoring lambdas
- No `!extend` on display pages (avoids lambda duplication)
- Debug interval takes over display during tests, releases when done
- Production code is completely unaware of debug mode

### Scoring Walkthrough Behavior
1. Select "Scoring Walkthrough" in HA
2. Debug interval takes over display cycling
3. Cycles: Excellent(8s) → Good(8s) → Fair(8s) → Poor(8s) → Bad(8s)
4. Each step: shows face expression + level label + plays corresponding tone
5. After full cycle: auto-resets to "None", normal operation resumes

### HW Failure Test
1. Select "HW Failure" in HA
2. Sets `hw_error = true`, `fail_co2 = true`
3. Forces display to splash page (which renders hw error screen)
4. Plays hwfail beep
5. Stays until user selects "None" or reboots

### Boot Recovery Test
1. Select "Boot Recovery" in HA
2. Sets `boot_fails = 2` in NVS
3. Calls `App.safe_reboot()`
4. On next boot: boot_fails=2, sensors need 5 consecutive good readings
5. If sensors healthy: boots normally, clears counter
6. If boot times out: boot_fails becomes 3 → hw_error screen

---

## Production YAML Changes (minimal)

Add substitutions block for name/friendly_name so debug.yaml can override:
```yaml
substitutions:
  device_name: air-quality-monitor-2
  friendly_name: Air-Quality-Monitor-2

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}
```

---

## Files
- `air-quality-monitor.yaml` — add substitutions (2 lines changed)
- `debug.yaml` — new file, ~100 lines

## Verification
1. `esphome compile air-quality-monitor.yaml` — production still builds
2. `esphome compile debug.yaml` — debug variant builds
3. Flash debug to device, verify:
   - Splash shows "DEBUG" or different name
   - Select entity appears in HA
   - Each test scenario works as described
4. Flash production back — normal operation, no debug artifacts
