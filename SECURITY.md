# Security Policy — Flightmortem

Report vulnerabilities privately via GitHub Security Advisories for `raydeoliveira/flightmortem`.

## Defensive Parsing Bounds

- HTTP `/mcp` requests reject bodies exceeding `2,097,152` bytes (`413`).
- Remote/MCP base64 or inline log payloads are capped at `1,572,864` decoded bytes; CLI `--file` is capped at `52,428,800` bytes.
- Binary frame parsers (`ardupilot_bin`, `px4_ulog`, `betaflight_bbl`) enforce strictly monotonically advancing offsets, bounded LEB128 varint widths (`<= 10` bytes), zero `eval`, zero dynamic code execution, and fail-closed `FmValidationError` handling.
