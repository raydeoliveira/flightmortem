---
name: flightmortem
description: >-
  Inspect recorded ArduPilot DataFlash, PX4 ULog (.ulg or .ulog) and
  Betaflight Blackbox logs over hosted MCP. Use when an agent needs sample
  counts, battery resistance or vibration measurements from recordings.
---

# FlightMortem

## Start with the hosted tools

Connect to `https://flightmortem-mcp.raymond-deoliveira.workers.dev/mcp` and call `tools/list` for the current arguments and pricing. Use `fm_inspect_log` to list channels in a recorded DataFlash, ULog or Blackbox file. Use `fm_power_health` when the log includes voltage and current or the caller has normalized `telemetry.battery`. Both calls are free and require no credentials.

Start with the [synthetic PX4 ULog fixture](https://github.com/raydeoliveira/flightmortem#inspect-a-px4-ulog-file). Its four readings cover 3 seconds. The hosted parser returned `px4_ulog` with 4 battery samples, and the battery analysis returned `r_int_mohm: 25`. A hosted request sends its supplied bytes to FlightMortem. Use the fixture or choose a log you can share.

## Input formats

| Input | How to send it |
|---|---|
| ArduPilot DataFlash `.bin`, PX4 ULog v1 `.ulg` or `.ulog`, Betaflight Blackbox `.bbl` | Pass standard base64 in `log_base64`. The decoder reads format markers in the file bytes. |
| Normalized telemetry | Pass a `telemetry` object. For battery analysis use `telemetry.battery`, with `t_s`, `voltage_v` and `current_a` in each row. |

Supply one input form per request. The hosted binary limit is 1,572,864 decoded bytes. A successful tool call returns its measurements in `result.structuredContent`. Check `isError` and `warnings` before interpreting a result.

See the [Quick Start](https://github.com/raydeoliveira/flightmortem#quick-start) for the browser and Apify samples. Read the [hosted tool catalog](https://flightmortem-mcp.raymond-deoliveira.workers.dev/.well-known/mcp/server-card.json) for prices before using metered tools.
