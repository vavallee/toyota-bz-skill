# toyota-bz-skill

A Claude Code skill and a WiCAN vehicle profile for reading a **2027 Toyota bZ**
(the renamed bZ4X, North American market) over the OBD port.

The published bZ4X 2022–2025 profiles do not work on this car. It answers
OBD-on-UDS (SAE J1979-3): UDS service `22`, identifiers `F4xx`, on 29-bit CAN
at 500 kbit/s.

| File | Contents |
|---|---|
| `SKILL.md` | What works, what does not, decode formulas, WiCAN traps, Home Assistant notes |
| `profiles/wican_bz_2027_car_data.json` | WiCAN vehicle profile: battery level, 12 V voltage, ambient temperature, odometer, pack voltage |

## Quick test with any ELM327-compatible adapter

```
ATSP7
ATSH18DA5AF1
22F45B
```

A reply ending `62 F4 5B xx` gives the battery level as `xx × 100 / 255` percent.

## Using the skill

Copy this directory to `.claude/skills/toyota-bz/` in a project, or to
`~/.claude/skills/toyota-bz/`.

## Status

Measured on one car on 2026-11-11 with a WiCAN PRO, firmware 4.50. The battery
level has not yet been calibrated against the dashboard; see `SKILL.md` §5.
