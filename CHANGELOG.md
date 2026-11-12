# Changelog

## Unreleased

The package takes devices-nv and embedded-hal-nv by path until 0.4.0 is published.

- `Amg8833` implements devices-nv's `ThermalImager`: `thermal_cols`, `thermal_rows`,
  `thermal_start`, `thermal_read` and `thermal_rate_mhz`.  The inherent `frame(var out)` is
  gone; `thermal_read(var out)` replaces it and fills a `ThermalFrame` (`Vec[i32; 768]`)
  with the same 64 pixels in the same order.  *Migration:* `var frame: ThermalFrame`
  for `AmgFrame`, and `frame[i] as Int` where an `Int` is wanted.
- The aliases `AmgBytes` and `AmgFrame` are gone.  The 128-byte buffer is devices-nv's
  `DevBurst`, read with `wiring_read_burst_into`, and `AmgPort.read_regs`, `table_get` and
  `table_flag` take it.
- New: the constant `ROWS` and the function `rate_mhz`.

## 0.1.0

The first release: a driver for Panasonic's AMG8833 Grid-EYE 8 by 8 infrared array sensor.

- `Amg8833`, the type a program builds from the board's I2C bus, timer and the part's
  wiring (`wiring_i2c(0, 0x69)` or `0x68`). `frame` reads the 64 pixels in one transaction
  of 128 bytes into a `Vec[Int; 64]` the caller owns, in millidegrees Celsius, pixel 1
  first. The thermistor is devices-nv's `Thermometer`.
- The frame rate of 1 or 10 a second, the four power modes with the 50 ms to communication,
  the flag reset and the initial reset with their 2 ms, the twice-moving-average output with
  the three-byte unlock, the interrupt control, the upper, lower and hysteresis levels in
  millidegrees, the status and its clear, and the interrupt table.
- The conversions follow the tables printed in section 4-10 of the specification: a pixel
  is a 12-bit two's complement count of 0.25 degrees; the thermistor is a sign and a
  magnitude of 0.0625 degrees, rounded to the nearest millidegree; the levels are two's
  complement counts of 0.25 degrees.
- The device program `examples/thermal_reader.nv`, a thermistor and hottest-pixel reader.
  Built for the nRF52840 DK it is 14,400 B of flash and for the nRF54LM20 DK 15,864 B.
- Line coverage of `src/` is 100 %, measured with `tests/coverage.sh`.

### Known

- Hardware verification: NOT VERIFIED. No board on the bench carries the sensor.
- The specification is marked "REFERENCE ONLY" and carries no revision number; the copy read
  is dated Aug. 30, 2011. It does not say whether the pixel figure of section 4-7 is seen from
  the lens or the board side.
- The specification does not say that the registers auto-increment on a write, so the six
  level registers are written one at a time.
- The 128-byte read needs a bus that carries 128 bytes in one transfer. `dev_bus.DevBytes`
  holds 32, so the package has its own `AmgBytes`.
- `devices-nv` 0.3.3 has no trait for a thermal array; the package defines its own frame API.
- The `[tier]` of the manifest is `embedded`, the strictest, so the module also checks at
  `rt` and `system`.
