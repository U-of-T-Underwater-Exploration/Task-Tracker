# 1. DVL Reading

**Goal:** read data from the Kogger Micro DVL and produce a report on what it actually outputs. **Start from branch `feature/dvl-driver`**, which already has the `ID_DVL_VEL` payload parser and the protocol PDF.

| Node | Status | File |
| --- | --- | --- |
| `uuv_dvl_driver` | branch, to be completed | [uuv_dvl_driver.md](uuv_dvl_driver.md) |
| `dvl_profiler` | new | [dvl_profiler.md](dvl_profiler.md) |

## What the protocol PDF says

Source: [Kogger-Protocol](https://github.com/koggertech/Kogger-Protocol), `Kogger SB protocol.pdf`. Product page: [Kogger Micro DVL](https://kogger.tech/product/micro-dvl/).

- **Frame:** `0xBB 0x55 | ROUTE | MODE | ID | LENGTH(0-128) | PAYLOAD | CHECK1 CHECK2`.
  - Checksum is Fletcher-16.
  - Values are little-endian; floats are IEEE754.
  - MODE bits 0:1 give the TYPE: 1 = CONTENT (device to host), 2 = SETTING, 3 = GETTING. Bits 3:5 give the message version.
- **`ID_DVL_VEL` (0x79), version 2, 68 bytes:**
  - `FLAGS` (U4): the bit meanings are not documented.
  - `TIMESTAMP` (U4, ms), `DELTA_TIME` (F4, s), `LATENCY` (F4, s).
  - `VELOCITY_X/Y/Z/Z1/Z2` (F4, m/s), each with a matching `UNCERTAINTY_*` (F4, m/s).
  - `DISTANCE_Z/Z1/Z2` (F4, m).
- **Other IDs the device may also emit:**
  - `ID_DIST` (0x02, mm)
  - `ID_ATTITUDE` (0x04): Euler angles in 0.01°, or a quaternion
  - `ID_TEMP` (0x05, 0.01 °C)
  - `ID_TIMESTAMP` (0x01, ms)
  - `ID_DATASET` (0x10): configures periodic output, but its bitmask does **not** list DVL_VEL.
- **Not documented:** beam angle, beamwidth, the meaning of `FLAGS`, the meaning of Z1 and Z2, and how to enable `DVL_VEL` output.
