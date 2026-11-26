# flightmortem

Deterministic ArduPilot `.bin`, PX4 `.ulog`, and Betaflight/INAV `.bbl` flight-log parser, Cooley-Tukey Welch gyro/accel FFT spectral analyzer, OLS battery internal-resistance (`R_int`) & motor balance estimator, 8-cause crash root-cause fingerprinter, and bounds-clamped PID/filter diffing utility in zero-dependency TypeScript (`node >= 22.18.0`, Cloudflare Workers, and Apify Actor / Standby MCP).

## Quick Start

Estimate a flight battery's internal resistance from four voltage and current samples. This free hosted call needs only curl and uses the sample readings below.

```bash
curl -sS https://flightmortem-mcp.raymond-deoliveira.workers.dev/mcp \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"fm_power_health","arguments":{"telemetry":{"battery":[{"t_s":0,"voltage_v":25.0,"current_a":0},{"t_s":1,"voltage_v":24.75,"current_a":10},{"t_s":2,"voltage_v":24.5,"current_a":20},{"t_s":3,"voltage_v":24.25,"current_a":30}]}}}}'
```

The response gives `r_int_mohm` of `25` and `sample_count` of `4` under `result.structuredContent`. To keep the example handy, star [FlightMortem on GitHub](https://github.com/raydeoliveira/flightmortem).

## What It Does / What It Does Not Do

| Surface / Tool | Tier / Price | What It Does |
|---|---|---|
| `fm_inspect_log` / `flightmortem inspect` | Free (`$0`) | Parses ArduPilot DataFlash `.bin`, PX4 ULog v1 `.ulog`, Betaflight/INAV `.bbl` (multi-session `--log-index`), or JSON telemetry into sample counts, duration, sample rate, dropouts, events, and parameters |
| `fm_power_health` / `flightmortem power` | Free (`$0`) | Auto battery cell count (`1S`–`12S` or `BAT_N_CELLS`/override), resting vs min cell voltage, sag `%`, OLS internal resistance `R_int` (`mOhm` + `R^2`), trapezoidal `mAh`/`Wh`, hover throttle, motor spread `%`, and saturation `%` |
| `fm_fft_spectrum` / `flightmortem fft` | Free (`$0`) | Cooley-Tukey Radix-2 Welch Hann-windowed PSD (detrended 50%-overlap segments), MAD noise floor, parabolic sub-bin peak interpolation, `2x`/`3x`/`4x` harmonic tagging, and biquad notch / gyro LPF / D-term LPF targets |
| `fm_crash_diagnose` / `flightmortem diagnose` | Free (`$0`) | Ranks 8 failure causes (motor saturation/loss, battery brownout sag, EKF innovation divergence, high vibration/clipping, PID/D-term resonance oscillation, attitude tracking loss, logging dropout, failsafe/error events) + chronological timeline in a `< 20 KB` fingerprint |
| `fm_tune_diff` / `flightmortem tune` | Metered (`$0.015`) | Synthesizes ArduPilot, PX4, Betaflight, and INAV PID/notch/LPF adjustments clamped to `max_step_pct <= 25%` and parameter safety bounds, emitting executable `apply_commands`, `rollback_commands`, and atomic `--out` rollback manifests |
| `fm_compare_flights` / `flightmortem compare` | Metered (`$0.010`) | A/B before/after comparison returning `IMPROVED`/`REGRESSED`/`MIXED`/`UNCHANGED` across gyro noise floor dB, attitude RMS error, vibration RMS, motor imbalance, and battery sag |
| HTTP `402` & stdio MCP | MCP `2025-03-26` | Hosted Streamable HTTP MCP at `https://flightmortem-mcp.raymond-deoliveira.workers.dev/mcp` and local `flightmortem mcp` stdio server |
| Apify Actor & Standby MCP | PPE / Standby | Single (`{ tool, arguments }`) and batch (`{ calls }`) Dataset/`OUTPUT` runner plus HTTP Standby `/mcp` server |

**What It Does Not Do**:
- Does not connect to live flight controllers over USB, MAVLink serial, or MSP radios (`flightmortem` operates offline on recorded logs and telemetry JSON).
- Does not synthesize or flash full parameter dumps; parameter changes are emitted as clamped `<= 25%`-step diffs with explicit rollback commands requiring pilot review.
- Inline MCP `log_base64` inputs are capped at `1,572,864` decoded bytes inside a `2,097,152`-byte JSON-RPC body (use local CLI `--file` for larger logs up to `52,428,800` bytes).
- PX4 ULog nested struct decoding and Betaflight Blackbox frame support target standard telemetry streams (`sensor_combined`, `vehicle_attitude`, `actuator_outputs`, `battery_status`, `estimator_status`, `gyroADC`, `motor`, `vbatLatest`); vendor-proprietary encrypted logs are rejected fail-closed.

## Install & Uninstall

Use the hosted endpoint with curl or an MCP client. The public GitHub repository contains documentation and MCP metadata. The local CLI and tests require a separate source checkout.

The source checkout requires Node.js `>= 22.18.0` (verified on `v25.9.0`; `0` runtime and `0` dev npm dependencies, `237,180` bytes / `231.6 KiB` across `32` TypeScript files and `5,290` lines in `src/`, `300 KB` in `src/` and `432 KB` project total via `du -sk`):

```bash
node --test tests/*.test.ts
node --test e2e/*.test.ts
```

Measured runs on `node v25.9.0` (macOS Arm64):
- Unit & integration (`node --test tests/*.test.ts`): `20` tests across `6` suites in `6` files (`20` pass, `0` fail, `0` skipped).
- End-to-end (`node --test e2e/*.test.ts`): `2` suites (`2` pass, `0` fail, `0` skipped).
- Independent Python oracle (`struct.pack` + independent Welch Hann DFT + OLS `R_int`): `< 1e-3` agreement across rounded FFT peak Hz / dB, OLS `R_int` (`0.0 mOhm` diff), and trapezoidal `mAh`/`Wh`.
- Real upstream firmware logs tested: `PX4/pyulog` (`sample_log_small.ulg`, `921,631 B`, `1,298` IMU, `992` params, `6S`, `204.58 Hz`, `1` dropout `30 ms`; `sample_appended_multiple.ulg`, `486,737 B`, `250 Hz`), `ArduPilot/pymavlink` (`test.BIN`, `65,536 B`, `132` `FMT`, `809` `PARM`, `45` IMU), and `blackbox-log/blackbox-log` (`error-recovery.bbl`, `3,830 B`, `Betaflight 4.2.11`, `114` header params + graceful stop on injected corruption marker).
- Live Cloudflare Worker (`https://flightmortem-mcp.raymond-deoliveira.workers.dev/mcp`), Official MCP Registry (`io.github.raydeoliveira/flightmortem`), and Apify Actor `pvPe9mMC8g9oY376d` (`SUCCEEDED` live run `pWULcpXsFKlwwIJoz`, `cell_count: 6`, `r_int_mohm: 25`, `3s`).

To uninstall locally, remove the `flightmortem` directory.

## CLI and MCP examples

### 1. Local CLI (source checkout)

Inspect capability schema and exit codes:

```bash
node src/adapters/cli.ts --describe
```

Analyze battery internal resistance (`R_int`) and cell sag via inline JSON or a `.bin`/`.ulog`/`.bbl` file:

```bash
node src/adapters/cli.ts power --input '{"telemetry":{"battery":[{"t_s":0,"voltage_v":25.0,"current_a":0},{"t_s":1,"voltage_v":24.4,"current_a":24},{"t_s":2,"voltage_v":23.8,"current_a":48},{"t_s":3,"voltage_v":24.4,"current_a":24}]}}' --json
```

Observed output:

```json
{"schema_version":1,"ok":true,"command":"power","data":{"firmware":"json","sample_count":4,"motor_sample_count":0,"cell_count":6,"cell_source":"auto_inferred","resting_voltage_v":24.4,"min_voltage_v":23.8,"max_sag_v":0.6,"sag_pct":2.46,"min_cell_v":3.9667,"r_int_mohm":25,"r_int_r2":1,"consumed_mah":23.33,"consumed_wh":0.564,"peak_current_a":48,"mean_current_a":24,"hover_throttle_norm":null,"motor_count":0,"motor_means":[],"motor_imbalance_pct":null,"motor_saturation_pct":null,"warnings":[]}}
```

Common workflows on recorded binary logs:

```bash
node src/adapters/cli.ts inspect --file flight.bin --json
node src/adapters/cli.ts fft --file flight.bin --sensor gyro --window-size 512 --json
node src/adapters/cli.ts diagnose --file crash.ulog --cell-count 6 --json
node src/adapters/cli.ts tune --file noisy.bbl --log-index 0 --max-step-pct 15 --out tune_manifest.json --dry-run --json
node src/adapters/cli.ts compare --baseline before.bin --candidate after.bin --json
```

### 2. HTTP `402` Streamable HTTP MCP & Stdio MCP

From a source checkout, dry-run or run the local MCP server:

```bash
node src/adapters/cli.ts serve --dry-run --json
API_KEYS_CSV="demo-key" node src/adapters/cli.ts serve --host 127.0.0.1 --port 8787
node src/adapters/cli.ts mcp
```

Connect Claude Code in one command (4 free tools — inspect, power, Welch FFT, crash forensics — work immediately with zero signup):

```bash
claude mcp add --transport http flightmortem https://flightmortem-mcp.raymond-deoliveira.workers.dev/mcp
```

Machine-readable discovery: `GET /llms.txt` and `GET /.well-known/mcp/server-card.json` expose pricing, free tool names, x402 Base USDC parameters, and Polar Pro checkout links.

For a client that accepts `mcpServers` JSON, configure the free hosted tools with the server URL.

```json
{
  "mcpServers": {
    "flightmortem": {
      "url": "https://flightmortem-mcp.raymond-deoliveira.workers.dev/mcp"
    }
  }
}
```

- Free tools (`fm_inspect_log`, `fm_power_health`, `fm_fft_spectrum`, `fm_crash_diagnose`) work without credentials.
- Metered tools (`fm_tune_diff`, `fm_compare_flights`) accept `X-Polar-License-Key`, `PAYMENT-SIGNATURE` (Base Mainnet x402 v2 USDC), or `Authorization: Bearer <token>`, failing closed with HTTP `402` + `PAYMENT-REQUIRED`.

## Exit Codes

- `0`: success
- `1`: unexpected runtime error
- `2`: CLI usage or input validation error
- `3`: file I/O, missing path, or symlink rejection
- `4`: corrupt or unrecognized flight-log binary format
- `5`: payload size limit exceeded or permission error
- `130`: `SIGINT` / `SIGTERM` shutdown

## License

Commercial Software License (see `LICENSE` — all rights reserved).
- Commercial Pro license key (`$29`, up to 3 devices/clients) and hosted pay-per-call access available via Polar.sh (`https://buy.polar.sh/polar_cl_qBGivPZfy8sSjg81OkKqtWmXZqGorfs4jLAQC4QojVO`) and Base Mainnet USDC x402.
