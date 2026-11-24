# Changelog

All notable changes to `flightmortem` are documented in this file.

## [0.1.0] - 2026-11-24

### Added
- Clean zero-dependency TypeScript implementation across `32` source modules (`5,290` lines, `237,180` bytes) targeting Node `>= 22.18.0`, Cloudflare Workers (`23`-module closure with `0` `node:*` imports), and `node:22-alpine` Apify containers.
- Native binary parsers for ArduPilot DataFlash `.bin` (`0xA3 0x95` + `FMT` schemas), PX4 ULog v1 `.ulog`, Betaflight/INAV `.bbl` Blackbox (single and multi-session `--log-index`), and normalized JSON telemetry.
- Cooley-Tukey Radix-2 Welch Hann-windowed gyro/accel PSD (`fm_fft_spectrum`) with parabolic sub-bin peaks, MAD noise floor, harmonic detection, and notch/LPF targets.
- Battery OLS `R_int` (`mOhm` + `R^2`), auto `1S`–`12S` cell detection, trapezoidal `mAh`/`Wh`, hover throttle, and motor imbalance/saturation analyzer (`fm_power_health`).
- 8-cause crash forensics engine (`fm_crash_diagnose`) producing `< 20 KB` timelines and root-cause fingerprints.
- Bounds-clamped `<= 25%`-step PID/notch/LPF tune diff synthesizer (`fm_tune_diff`) across ArduPilot, PX4, Betaflight, and INAV with apply/rollback scripts and atomic `--out` manifest output.
- A/B regression flight comparator (`fm_compare_flights`), CLI (`inspect`, `power`, `fft`, `diagnose`, `tune`, `compare`, `serve`, `mcp`), MCP `2025-03-26` HTTP `402` server, and Apify Actor / Standby adapter.
- Unit, integration, adversarial QA, E2E subprocess suites (`20` unit + `2` E2E tests), and independent Python `struct.pack` + Welch DFT + OLS oracle verification.
