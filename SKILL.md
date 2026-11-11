---
name: toyota-bz
description: Reading a 2027 Toyota bZ (the renamed bZ4X, North American market) over the OBD port and getting battery level into Home Assistant. Use when connecting a WiCAN / ELM327-style dongle to a 2026+ Toyota bZ, when the published bZ4X 2022–2025 profiles return NO DATA, when writing or debugging a WiCAN vehicle profile for this car, or when wiring its state of charge into MQTT / Home Assistant / EV charging automation.
---

# Toyota bZ (2027) over OBD

Everything under "Verified" was measured on one car: a 2027 Toyota bZ, North
American market, on 2026-11-11, using a WiCAN PRO (hardware `WiCAN-OBD-PRO`,
firmware 4.50) with the car in READY. Anything not measured is under
"Unverified" and says so. Do not promote an unverified item without a new
measurement.

## 1. What does not work (stop trying these)

| Attempt | Result |
|---|---|
| Classic OBD-II service `01` (`0100`, `015B`), 11-bit functional `7DF` | `NO DATA` |
| Classic service `01`, 29-bit functional `18DB33F1` | `NO DATA` |
| WiCAN "Scan PIDs" (standard PID scan), protocol 0, 6 or 7 | empty list |
| Published `wican-fw` profile `vehicle_profiles/toyota/bz4x.json` (2022–2025): `ATSH7D2` `221F5B`, `221F9A`; `ATSH747` `22181A` | `NO DATA` — `7D2` and `747` do not answer at all |
| `221D3E` to `747` (battery capacity, from 2023 bZ4X write-ups) | `NO DATA` |
| OBD-on-UDS request `22F400` on 11-bit functional `7DF` | `NO DATA` |

On 11-bit CAN these addresses answer UDS TesterPresent (`3E00`): `780`, `792`,
`7A1`, `7B3`, `7C0`, `7D0`, `7F1` (positive) and `7C4` (negative response
`7F 3E 11`). All of them reject `221F5B`, `221F9A`, `22181A`, `221D3E` and
`22F45B` with `7F 22 31` (request out of range). `7C4` alone answers `22F400`.
None of the 2022–2025 bZ4X identifiers were found on any of them.

## 2. What works: OBD-on-UDS (SAE J1979-3) on 29-bit CAN

The car implements the newer "OBD on UDS" scheme: the classic service-`01`
parameter IDs are read with UDS service `22` and data identifier `F4xx`, where
`xx` is the old PID number. It only answers on 29-bit identifiers at 500 kbit/s.

- ELM327 protocol: `ATSP7` (ISO 15765-4, 29-bit, 500k)
- Functional (broadcast) header: `ATSH18DB33F1`
- Physical header for control unit `XX`: `ATSH18DAXXF1`; replies come from `18DAF1XX`
- Control units seen replying to functional `22F4x0` requests: `05`, `0B`, `0C`, `49`, `5A`, `5B`, `5C`, `62`
- `5A` and `62` are the two that carry battery data. Use physical addressing to
  one of them; a functional request returns one reply per control unit.

### Supported identifiers (decoded from the `F400`/`F420`/…/`F4C0` bitmaps)

| Control unit | Supported `F4xx` identifiers |
|---|---|
| `5A` | `42`, `46`, `49`, `4A`, `4D`, `4E`, `51`, `5B`, `65`, `9A`, `A6`, `D2`, `D6` |
| `62` | `42`, `46`, `5B`, `9A`, `B2`, `B9`, `D0`, `D2`, `D4`, `D8`, `DA` |
| `5B`, `5C` | `0D`, `31`, `42`, `65`, `9A` |
| `49` | `0D`, `31`, `D1` |

The `F400` and `F420` bitmaps of `5A` and `62` were not captured, so their
rows start at `F441`.

### Verified decodes

Formulas are the standard SAE J1979 ones. `A`, `B`, … are the data bytes after
`62 F4 xx`.

| Request (to `5A`) | Meaning | Formula | Example raw data → value |
|---|---|---|---|
| `22F45B` | Traction battery level | `A × 100 / 255` % | `A1` → 63.1 % |
| `22F442` | 12 V system voltage at the control unit | `(A×256 + B) / 1000` V | `37 DD` → 14.30 V |
| `22F446` | Ambient air temperature | `A − 40` °C | `2F` → 7 °C |
| `22F4A6` | Odometer | `(A B C D as 32-bit) / 10` km | `00 00 12 AC` → 478.0 km |
| `22F49A` | Traction battery voltage (bytes C–D of the 6-byte record) | `(C×256 + D) × 0.015625` V | `0E 00 62 C0 00 30` → 395 V |

"Verified" here means: the request answers, the formula is the published
standard one, and the value is physically plausible. The battery level has
**not** yet been compared against the dashboard at several levels — do that
before trusting it for automation (see §5).

### Unverified (raw captures only — decode before use)

| Request | Raw data bytes | Note |
|---|---|---|
| `22F49A` bytes E–F (`5A`) | `00 30`, later `00 2C`, `00 38` | Probably battery current; scale and sign not confirmed |
| `22F4B9` (`62`) | `0E D8 0E DB` | Two 16-bit values 3800 / 3803; consistent with min/max cell voltage in mV, not confirmed |
| `22F4B2` (`62`) | `FF` | unknown |
| `22F4D0` (`62`) | `01 FF FF 00 00 00 00` | unknown |
| `22F4D2` | `5A`: `02 00 FE`; `62`: `01 FF 00` | unknown |
| `22F4D4` (`62`) | `FF F7 0D 44` | unknown |
| `22F4D6` (`5A`) | `00 0C` | unknown |
| `22F4D8` (`62`) | `01` | unknown |
| `22F4DA` (`62`) | `FF FF FA 35` | unknown |
| `22F4D1` (`49`) | `00 00 00 00` | unknown |
| `22F465` (`5A`) | `02 00` | auxiliary input/output status bits |

Identifiers `B2`–`DA` are in the range SAE J1979-3 added for zero-emission
vehicles. Look them up in the J1979-DA digital annex rather than guessing.

## 3. WiCAN profile

`profiles/wican_bz_2027_car_data.json` is the working vehicle profile. Load it
through the dongle's configuration upload, or POST it to `/store_car_data`, set
`car_model` in the automate configuration to `Toyota: bZ 2027 (OBDonUDS)`, then
reboot the dongle (stored automate data only takes effect after a reboot).

It publishes, every 5 s (30 s for temperature and odometer):

| Topic | Payload key | Unit |
|---|---|---|
| `bz/soc` | `SOC` | % |
| `bz/lv_v` | `LV_V` | V |
| `bz/ambient` | `TMP_A` | °C |
| `bz/odo_km` | `ODOMETER` | km |
| `bz/hv_v` | `HV_V` | V |

### WiCAN traps (each one cost a wrong reading)

1. **Byte numbering differs between the test button and live polling.** With a
   29-bit header, `/autopid/test_pid` (the "Test" button) counts the header as
   four bytes, so the first data byte of a single-frame reply is `B8`. Live
   polling counts it as three, so the same byte is `B7`. A profile tuned with
   the test button reads one byte late when it goes live (battery level showed
   5.49 % instead of 62.75 %). The profile in this repo uses live numbering.
2. **Multi-frame replies keep the second frame's header in the buffer.** For
   `22F49A` in live polling: `B3`=`10`, `B4`=length, `B5..B7`=`62 F4 9A`,
   `B8..B10`=data A–C, `B11..B13`=second-frame header, `B14`=`21`,
   `B15..B17`=data D–F. Hence `HV_V = ((B10*256)+B15)*0.015625`.
3. **Per-parameter Min/Max silently drops out-of-range values.** Useful as a
   garbage filter once the expression is right; while debugging, leave them
   empty or a wrong expression just looks like "nothing publishes".
4. **A parameter only publishes when its destination type is not `Default`.**
5. **Remove the previous car's profile before plugging in.** A profile from
   another vehicle keeps polling and can publish retained garbage on its old
   topics as soon as any control unit answers.
6. **Retained MQTT values outlive the car.** Clear the old vehicle's retained
   topics (`mosquitto_pub -r -n -t <topic>`) before the dongle reconnects, or
   Home Assistant treats the last value from the previous car as current.

## 4. 12 V bus and dongle sleep

- In READY the 12 V bus reads 14.3–14.5 V.
- During AC (Level 2) charging with the car off, the dongle reported **12.90 V**.
  A sleep threshold tuned for a car that holds the bus near 14 V while charging
  (13.1 V in the previous setup) puts the dongle to sleep during a charge, so
  it reports nothing exactly when the charge controller wants a reading.
- Whether the control units `5A`/`62` answer while the car is off and charging
  is **not yet tested**. Test that before lowering the sleep threshold or the
  polling minimum voltage; polling a sleeping car is the usual cause of 12 V
  battery drain with OBD dongles.

## 5. Home Assistant

- AC charging over OCPP (Open Charge Point Protocol) does not carry the car's
  battery level; the wall unit cannot supply it.
- `wican-fw` does not publish MQTT discovery for vehicle-profile parameters, so
  publish a discovery config yourself. Minimal sensor:

```json
{
  "name": "Battery",
  "unique_id": "toyota_bz_soc",
  "state_topic": "bz/soc",
  "value_template": "{{ value_json.SOC }}",
  "unit_of_measurement": "%",
  "device_class": "battery",
  "state_class": "measurement",
  "availability": [{
    "topic": "<dongle status topic>",
    "value_template": "{{ value_json.status }}",
    "payload_available": "online",
    "payload_not_available": "offline"
  }],
  "device": {"identifiers": ["toyota_bz"], "name": "Toyota bZ", "manufacturer": "Toyota", "model": "bZ 2027"}
}
```

- Binding availability to the dongle's status topic hides the retained value
  whenever the dongle sleeps. If automation needs the last known level while
  parked, mirror the last valid value into a helper with a timestamp and a
  maximum age.
- Check calibration before automating on it: read the dashboard percentage at
  three or more levels (for example ~20 %, ~60 %, 100 %) and compare with
  `bz/soc`. Owners of 2026 cars report an OBD-derived level that disagrees with
  the dash and exceeds 100 % in at least one route-planning app; the dash value
  is likely a rescaled "usable" figure. If they differ, fit a linear correction
  rather than assuming the raw value is what the driver sees.

## 6. Method for finding more values

1. Sweep before guessing: send `3E00` to every 11-bit address `700`–`7F7` and
   functional `22F400` on 29-bit; record who answers.
2. Read the support bitmaps `22F400`, `22F420`, … `22F4C0` per control unit.
   Bit 7 of the first byte is identifier base+1; the last bit says the next
   bitmap exists.
3. Read each supported identifier by physical address and keep the raw bytes.
4. Only requests with service `22` (read) and `3E` (tester present) were sent.
   Do not send write, routine-control or session-change services to find data.

## Sources

- `meatpiHQ/wican-fw`, `vehicle_profiles/toyota/bz4x.json` (2022–2025 profile that does not apply)
- `OBDb/Toyota-bZ4X` (community signal definitions for earlier model years)
- SAE J1979 / J1979-3 parameter definitions for the `F4xx` identifiers
