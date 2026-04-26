# Air Quality Monitor — Task Tracker

## In Progress

### 002 — IAQ Stabilization Gate
**Priority:** High
**Status:** Planning
**Why:** BSEC IAQ starts unreliable (accuracy 0-1) and takes minutes to hours to stabilize. Currently face/buzzer/HA scoring use IAQ immediately, showing misleading expressions and tones before data is trustworthy.
**Design doc:** [002-iaq-stabilization-gate.md](002-iaq-stabilization-gate.md)

## Planned

### 003 — Debug Mode Firmware
**Priority:** Medium
**Status:** Not started
**Why:** Need separate firmware variant for testing hardware failure, boot recovery, scoring walkthrough. Must be visually distinguishable from production.
**Design doc:** TBD

## Completed

### 001 — Unified Air Quality Scoring
**Completed:** 2026-04-26
**What:** Replaced 4 independent additive scoring systems with worst-of category model. EPA/ASHRAE/BSEC-based thresholds.
**Design doc:** [001-unified-air-quality-scoring.md](001-unified-air-quality-scoring.md)
**Release:** v1.2.0
