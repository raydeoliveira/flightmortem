# FlightMortem

FlightMortem is a drone flight log analyzer for ArduPilot DataFlash, PX4 ULog and Betaflight Blackbox. Inspect a recorded file to see what it contains, or use its battery and IMU readings for resistance and vibration analysis.

## Quick Start

[Run the free battery sample in your browser](https://www.raymonddeoliveira.com/software#flightmortem-example). Open **Try a free battery analysis** and click **Analyze sample**. Four synthetic readings give an estimated pack resistance of **25 mΩ**.

[Try FlightMortem on Apify](https://apify.com/raydeoliveira/flightmortem-mcp) to run the same prefilled sample. Its single dataset row gives `data.r_int_mohm` of `25` and `data.sample_count` of `4`. The sample uses zero paid Actor events.

You can also [inspect a synthetic PX4 ULog file](#inspect-a-px4-ulog-file) to try the binary parser.

<details>
<summary>Run the free battery sample with curl</summary>

Send these four voltage and current readings to FlightMortem’s hosted MCP endpoint.

```bash
curl -fsS https://flightmortem-mcp.raymond-deoliveira.workers.dev/mcp \
  -H 'Content-Type: application/json' --data-binary @- <<'JSON'
{
  "jsonrpc": "2.0", "id": 1, "method": "tools/call",
  "params": {"name": "fm_power_health", "arguments": {"telemetry": {"battery": [
    {"t_s": 0, "voltage_v": 25.0,  "current_a": 0},
    {"t_s": 1, "voltage_v": 24.75, "current_a": 10},
    {"t_s": 2, "voltage_v": 24.5,  "current_a": 20},
    {"t_s": 3, "voltage_v": 24.25, "current_a": 30}
  ]}}}
}
JSON
```

Look for `r_int_mohm: 25` and `sample_count: 4` under `result.structuredContent`.

</details>

## Inspect a PX4 ULog file

[Download the synthetic PX4 sample](https://raw.githubusercontent.com/raydeoliveira/flightmortem/main/examples/px4-battery.ulg) and save it as `px4-battery.ulg` in your working directory. It contains 255 bytes and four battery readings.

```bash
log_base64=$(base64 < px4-battery.ulg | tr -d '\r\n')
printf '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"fm_inspect_log","arguments":{"log_base64":"%s"}}}\n' "$log_base64" |
  curl -fsS https://flightmortem-mcp.raymond-deoliveira.workers.dev/mcp \
    -H 'Content-Type: application/json' --data-binary @-
```

The hosted release returned `format: "px4_ulog"`, `duration_s: 3`, `sample_counts.battery: 4` and an empty `warnings` list. For a battery estimate from the same file, change `fm_inspect_log` to `fm_power_health`. The same four readings give `r_int_mohm: 25`.

To inspect your own file, replace `px4-battery.ulg` with its quoted path. Use the same inspection request for an ArduPilot DataFlash `.bin`, a PX4 ULog v1 `.ulg` or `.ulog`, or a Betaflight Blackbox `.bbl`. The decoder identifies the format from the file bytes. Inspect the sample counts first to see whether a log contains the channels needed for an analysis. For battery estimates, supply a binary log with voltage and current readings or send normalized JSON as `telemetry.battery`. The hosted endpoint receives the supplied file bytes and accepts files up to 1.5 MiB decoded.

## What It Does / What It Does Not Do

| Surface / Tool | Tier / Price | What It Does |
|---|---|---|
| `fm_inspect_log` / `flightmortem inspect` | Free (`$0`) | Parses ArduPilot DataFlash `.bin`, PX4 ULog v1 `.ulg` or `.ulog`, Betaflight/INAV `.bbl` (multi-session `log_index`), or JSON telemetry into sample counts, duration, sample rate, dropouts, events, and parameters |
| `fm_power_health` / `flightmortem power` | Free (`$0`) | Auto battery cell count (`1S`–`12S` or `BAT_N_CELLS`/override), resting vs min cell voltage, sag `%`, OLS internal resistance `R_int` (`mOhm` + `R^2`), trapezoidal `mAh`/`Wh`, hover throttle, motor spread `%`, and saturation `%` |
| `fm_fft_spectrum` / `flightmortem fft` | Free (`$0`) | Cooley-Tukey Radix-2 Welch Hann-windowed PSD (detrended 50%-overlap segments), MAD noise floor, parabolic sub-bin peak interpolation, `2x`/`3x`/`4x` harmonic tagging, and biquad notch / gyro LPF / D-term LPF targets |
| `fm_crash_diagnose` / `flightmortem diagnose` | Free (`$0`) | Ranks 8 failure causes (motor saturation/loss, battery brownout sag, EKF innovation divergence, high vibration/clipping, PID/D-term resonance oscillation, attitude tracking loss, logging dropout, failsafe/error events) + chronological timeline in a `< 20 KB` fingerprint |
| `fm_tune_diff` / `flightmortem tune` | Metered (`$0.015`) | Synthesizes ArduPilot, PX4, Betaflight, and INAV PID/notch/LPF adjustments clamped to `max_step_pct <= 25%` and parameter safety bounds, emitting executable `apply_commands`, `rollback_commands`, and atomic `--out` rollback manifests |
| `fm_compare_flights` / `flightmortem compare` | Metered (`$0.010`) | A/B before/after comparison returning `IMPROVED`/`REGRESSED`/`MIXED`/`UNCHANGED` across gyro noise floor dB, attitude RMS error, vibration RMS, motor imbalance, and battery sag |
| HTTP `402` & stdio MCP | MCP `2025-03-26` | Hosted Streamable HTTP MCP at `https://flightmortem-mcp.raymond-deoliveira.workers.dev/mcp` and local `flightmortem mcp` stdio server |
| Apify Actor & Standby MCP | PPE / Standby | Single (`{ tool, arguments }`) and batch (`{ calls }`) Dataset/`OUTPUT` runner plus HTTP Standby `/mcp` server |

**What It Does Not Do**:
- FlightMortem reads recorded logs and normalized JSON. DJI and MAVLink `.tlog` files are outside the supported binary formats.
- Does not synthesize or flash full parameter dumps; parameter changes are emitted as clamped `<= 25%`-step diffs with explicit rollback commands requiring pilot review.
- Inline MCP `log_base64` inputs are capped at `1,572,864` decoded bytes inside a `2,097,152`-byte JSON-RPC body.
- PX4 ULog nested struct decoding and Betaflight Blackbox frame support target standard telemetry streams (`sensor_combined`, `vehicle_attitude`, `actuator_outputs`, `battery_status`, `estimator_status`, `gyroADC`, `motor`, `vbatLatest`); vendor-proprietary encrypted logs are rejected fail-closed.

## Install & Uninstall

Use the public hosted service with curl or an MCP client. To disconnect an agent, remove FlightMortem from its MCP configuration. The local CLI and stdio server require a separate source checkout with Node.js 22.18 or later. The [v0.1.2 release notes](https://github.com/raydeoliveira/flightmortem/releases/tag/v0.1.2) cover the hosted version.

## CLI and MCP examples

### Hosted MCP

From your project directory, run `npx skills add raydeoliveira/flightmortem` to install the [agent reference](SKILL.md) for recorded flight logs and battery resistance.

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

## License

Commercial Software License (see `LICENSE` — all rights reserved).
- Commercial Pro license key (`$29`, up to 3 devices/clients) and hosted pay-per-call access available via Polar.sh (`https://buy.polar.sh/polar_cl_qBGivPZfy8sSjg81OkKqtWmXZqGorfs4jLAQC4QojVO`) and Base Mainnet USDC x402.
