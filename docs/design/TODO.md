# Air Quality Monitor — Task Tracker

## Planned

### 003 — Debug Mode Firmware
**Priority:** Medium
**Status:** Not started
**Why:** Need separate firmware variant for testing hardware failure, boot recovery, scoring walkthrough. Must be visually distinguishable from production.
**Design doc:** TBD

## Completed

### 002 — IAQ Stabilization Gate
**Completed:** 2026-04-26
**What:** Gate IAQ scoring on BSEC accuracy >= 2. Face shows "Cal." during warmup. CO2+PM2.5 drive scoring alone until BSEC stabilizes.
**Design doc:** [002-iaq-stabilization-gate.md](002-iaq-stabilization-gate.md)
**Release:** v1.3.0

### 001 — Unified Air Quality Scoring
**Completed:** 2026-04-26
**What:** Replaced 4 independent additive scoring systems with worst-of category model. EPA/ASHRAE/BSEC-based thresholds.
**Design doc:** [001-unified-air-quality-scoring.md](001-unified-air-quality-scoring.md)
**Release:** v1.2.0
