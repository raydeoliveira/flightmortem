---
name: flightmortem
description: Deterministic ArduPilot .bin, PX4 .ulog, and Betaflight/INAV .bbl flight-log parser, Welch Hann gyro/accel FFT analyzer, OLS battery R_int estimator, 8-cause crash root-cause fingerprinter, and <=25%-step PID/notch diff synthesizer via CLI (--json/--describe), HTTP 402 Streamable HTTP MCP, and Apify Actor / Standby MCP.
---

# flightmortem

Zero-dependency TypeScript UAV blackbox forensics and FFT/PID tuning oracle (`node >= 22.18.0`).

## CLI Discovery & Invocation

- Inspect machine-readable commands, flags, MCP tool pricing, and exit codes:
  `node src/adapters/cli.ts --describe`
- Pass `--json` for structured stdout `{ schema_version: 1, ok, command, data, error }`.
- Validate parameters and preview outputs without writing `--out` manifests or binding HTTP ports via `--dry-run`.
- Pass `--file <path>` for binary `.bin`/`.ulog`/`.bbl` logs up to `52,428,800` bytes (`50 MiB`), `--input <json|->` for inline JSON, or `--baseline <path> --candidate <path>` for `compare`.

Subcommands (`8`):
- `inspect` (`fm_inspect_log`) — Free: Parse `.bin`/`.ulog`/`.bbl`/JSON sample counts, duration, sample rate, dropouts, events, and parameters.
- `power` (`fm_power_health`) — Free: Auto cell count (`1S`–`12S`), resting/min cell voltage, sag `%`, OLS `R_int` (`mOhm` + `R^2`), trapezoidal `mAh`/`Wh`, motor spread/saturation `%`.
- `fft` (`fm_fft_spectrum`) — Free: Cooley-Tukey Welch Hann PSD, MAD noise floor, parabolic sub-bin peak frequency, harmonic tagging, biquad notch & LPF targets.
- `diagnose` (`fm_crash_diagnose`) — Free: Rank 8 failure causes (motor saturation/loss, brownout, EKF divergence, high vibe/clipping, PID/D-term resonance, attitude loss, dropout, failsafe) + timeline in `< 20 KB` fingerprint.
- `tune` (`fm_tune_diff`) — Metered (`$0.015`): Bounds-clamped `<= 25%`-step PID/notch/LPF diff across `ardupilot`, `px4`, `betaflight`, `inav` with `apply_commands`, `rollback_commands`, and atomic `--out` manifest.
- `compare` (`fm_compare_flights`) — Metered (`$0.010`): Before/after A/B regression delta across gyro noise dB, attitude RMS error, vibration, motor imbalance, and battery sag.
- `serve` — Start local Node HTTP `402` Streamable HTTP MCP server (`--host`, `--port 0..65535`).
- `mcp` — Run stdio JSON-RPC 2.0 MCP server (`2025-03-26`, billing bypassed locally).

## Streamable HTTP MCP (`2025-03-26`)

- **Hosted endpoint**: `https://flightmortem-mcp.raymond-deoliveira.workers.dev/mcp`
- **Polar Pro Checkout (`$29` license)**: `https://buy.polar.sh/polar_cl_qBGivPZfy8sSjg81OkKqtWmXZqGorfs4jLAQC4QojVO`
- `GET`/`HEAD /health` — Health status, version, protocol version, tool count (`6`).
- `GET`/`HEAD /llms.txt` — Plain-text agent quickstart with free/metered tools, `claude mcp add`, and payment headers.
- `GET`/`HEAD /.well-known/mcp/server-card.json` (and `GET`/`HEAD /`) — Tool catalog, pricing tiers, `pricing`, `quickStart`, auth methods, and `polarCheckoutUrl`.
- `POST /mcp` — JSON-RPC 2.0 (`initialize`, `ping`, `tools/list`, `tools/call`; flat object schemas without `oneOf`/`anyOf`/`allOf` and read-only/idempotent MCP `annotations`; non-POST returns `405` + `Allow: POST, OPTIONS`; body limit `2,097,152` bytes; decoded `log_base64` limit `1,572,864` bytes).

Authentication for metered tools (`fm_tune_diff`, `fm_compare_flights`):
1. `PAYMENT-SIGNATURE` (or `X-PAYMENT` / `Authorization: Payment`) — x402 v2 verify + settle (`PAYMENT-RESPONSE` header on `200`; failed tool execution never settles payment).
2. `X-Polar-License-Key` — Polar.sh `/validate` (`status === "granted"`).
3. `Authorization: Bearer <token>` — Constant-time SHA-256 comparison against `API_KEYS_CSV`.

Missing or invalid credentials fail closed with HTTP `402`, `PAYMENT-REQUIRED`, and `WWW-Authenticate`.

## Apify Actor & Standby MCP

- **Actor mode** (`src/adapters/apify_main.ts`): Reads `INPUT` (`{ tool, arguments, chargeEvents? }` or `{ calls: [{ id?, tool, arguments }], chargeEvents? }`), writes Dataset items and Key-Value Store `OUTPUT`, and charges `.actor/pay_per_event.json` events only for successful metered calls.
- **Standby mode** (`APIFY_META_ORIGIN=STANDBY` or `FM_APIFY_MODE=standby`): Listens on `ACTOR_WEB_SERVER_PORT`/`PORT` (`4321` default, `"0"` preserved), answers `GET /` with `x-apify-container-server-readiness-probe` HTTP `200`, and serves `/mcp`.

## Exit Codes

`0` success | `1` runtime error | `2` usage / validation error | `3` I/O or symlink error | `4` corrupt binary log format | `5` payload limit or permission error | `130` `SIGINT`/`SIGTERM`
