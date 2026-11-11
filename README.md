# amg8833-nv

A driver for Panasonic's AMG8833, one of the Grid-EYE infrared array sensors: a
thermopile array of 8 by 8 pixels with a thermistor, read over I2C. The part
outputs the temperature of each pixel at 10 or 1 frames a second. The package
reads a frame into a buffer the caller owns, as 64 temperatures in millidegrees
Celsius, and implements `Thermometer` of
[devices-nv](https://novo-lang.org/packages/devices-nv) for the thermistor. The
reference is "Specifications for Infrared Array Sensor Grid-EYE AMG88**"
(Panasonic Corporation, Automation Controls Business Unit, dated Aug. 30, 2011,
marked "REFERENCE ONLY").

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
sensor, so the driver has run only against a double of the register file, in the
host suite. The device program below builds for the nRF52840 DK and the
nRF54LM20 DK and has not been flashed. The orientation of the array, pixel 1 at
the bottom right, is the specification's drawing and is NOT VERIFIED on a part.

## Install

```sh
novo pkg add amg8833-nv
```

## The device program

`examples/thermal_reader.nv` brings the sensor up at 10 frames a second and prints a
line ten times a second over the board's console:

```
ambient 25 C  hottest 31 C at row 3 col 4
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardTimer }
use dev_thermo.{ TempMilliC, Thermometer }
use dev_bus
use amg8833.{ Amg8833, AmgFrame }

// Writes `n`, 0 or more, in decimal.  Formatting into a string would allocate,
// which @tier(embedded) refuses, so each digit goes out as a byte.  An Int has
// at most 19 decimal digits, which bounds both loops.
@tier(embedded)
fn digits(n: Int) [hw]
    var div = 1
    for _k in 0..18
        if div <= n / 10
            div = div * 10
    for _k in 0..19
        if div > 0
            hal.uart.write_byte(48 + n / div % 10)
            div = div / 10

// Writes a temperature in whole degrees, rounded toward zero, with a minus sign.
@tier(embedded)
fn degrees(milli_c: Int) [hw]
    var m = milli_c
    if m < 0
        hal.uart.write("-")
        m = -m
    digits(m / 1000)

// The index of the hottest pixel of the frame.
@tier(embedded)
fn hottest(var frame: AmgFrame) -> Int
    var best = 0
    for i in 1..64
        if frame[i] > frame[best]
            best = i
    best

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    let sensor = Amg8833.attach(BoardI2c { id: 0 }, BoardTimer { id: 0 }, dev_bus.wiring_i2c(0, 0x69))
    // The part needs 50 ms after its supply before it answers; `begin` waits for
    // the first frame as well.
    hal.timer.delay_ms(50)
    if not sensor.begin()
        hal.uart.write("no answer from the sensor at 0x69\n")
    var frame: AmgFrame = Vec.new()
    while true
        if sensor.frame(var frame)
            match sensor.temp_read()
                Some(t) =>
                    hal.uart.write("ambient ")
                    degrees(t.milli_c)
                None => hal.uart.write("ambient ?")
            let h = hottest(var frame)
            hal.uart.write(" C  hottest ")
            degrees(frame[h])
            // The index counts back from pixel 64 at the top left.
            hal.uart.write(" C at row ")
            digits((63 - h) / 8)
            hal.uart.write(" col ")
            digits((63 - h) % 8)
            hal.uart.write("\n")
        else
            hal.uart.write("no answer\n")
        hal.timer.delay_ms(100)
```

The program is `examples/thermal_reader.nv`. A board package that depends on this
package builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the two boards below and for the emulated board, where it runs the image
and checks that the program reports that the sensor does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program |
|---|---|---|
| `nrf52840-dk` | 14,400 B | 3,746 B |
| `nrf54lm20-dk` | 15,864 B | 4,182 B |

The numbers are from the build's memory report. Flash also holds the runtime and the
compiler's support library. The arena, the runtime's heap, is reserved whether or not
the program uses it; the driver's own buffers are on the stack, and the one list it
touches is the bus's answer to a 128-byte read, released when the read returns.

Wiring: the breakout's SDA and SCL to the board's first I2C bus and 3.3 V to its
supply. The AD_SELECT pin on VDD puts the part at 0x69 and on GND at 0x68. The Nordic
DKs drive I2C with a TWIM, on the pins `docs/hal/README.md` names under "The Nordic
DKs' buses". A reply from the sensor over a DK's bus is NOT VERIFIED until the driver
rig carries one.

## What the package contains

| Module | Holds |
|---|---|
| `amg8833` | The `Amg8833` type, the `AmgPort` trait the register functions run over, the conversions, and the functions for the power modes, the resets, the frame rate, the moving average, the interrupt and the frame. |

## The frame

`Amg8833.frame(var out)` reads registers 0x80 to 0xFF in one transaction and fills
`out`, a `Vec[Int; 64]` (`AmgFrame`), with the pixels in the order of the registers,
pixel 1 first, each in millidegrees Celsius. A pixel is a 12-bit two's complement count
of 0.25 degrees. The specification draws pixel 1 at the bottom right and pixel 64 at
the top left, so `pixel_index(row, col)` gives the index of the pixel `row` rows from
the top and `col` columns from the left. `Amg8833.frame_period_us` answers the time
between frames, 100,000 or 1,000,000.

The thermistor is 12 bits, a sign and a magnitude, of 0.0625 degrees; `temp_read` answers
it in millidegrees, rounded to the nearest.

## The rules a user needs

1. A frame is one transaction. The specification says that reading 0x80 to 0xFF at once
   never mixes old and new pixels (section 4-10 (11)). The bus must carry a transfer of
   128 bytes.
2. After the supply the part answers within 50 ms, and its output settles after 15 s
   (section 4-5). A frame read before the first conversion is zeros. `begin` waits 100 ms
   after it sets the frame rate.
3. The frame rate is 10 or 1 a second. `rate_set` refuses any other value.
4. `set_power` takes `POWER_NORMAL`, `POWER_SLEEP`, `POWER_STANDBY_60S` or
   `POWER_STANDBY_10S`. A write in sleep mode takes effect only on the return to normal mode
   (section 4-10 (1)).
5. `reset_flags` clears the status, the interrupt flag and the interrupt table; `reset_all`
   also restores the registers (section 4-10 (2)).
6. The moving average (twice moving average) is set by writing three unlock values to a
   reserved register first (section 4-10 (7)). `set_moving_average` does it.
7. The interrupt levels are two's complement counts of 0.25 degrees. `set_levels` takes
   the upper level, the lower level and the hysteresis in millidegrees, and refuses a value
   outside -512 to 511.75 degrees. It writes the six registers one at a time. In the
   absolute-value mode the INT pin goes low when a pixel is above the upper level or below the
   lower one, and the interrupt table (`interrupt_table`, `table_flag`) flags the pixels.
8. The part has no identification register, so the type implements no `Identifiable`.

## What is not included

- The register-by-register reads of the Arduino library: a frame is one transaction.
- A frame-ready event from the INT pin. The pin signals the interrupt of the pixel levels
  (section 4-8) and not a new frame.
- The low-gain parts' temperature range: the conversions are the same for the AMG8831,
  AMG8832, AMG8851 and AMG8852, and only the AMG8833 is named in the tags.

## Related packages

- [mlx90640-nv](https://novo-lang.org/packages/mlx90640-nv) reads a thermal array of 32 by
  24 pixels.
- [devices-nv](https://novo-lang.org/packages/devices-nv) defines `Thermometer`;
  [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) defines `I2cBus` and
  `Timer`.

## Tests

`tests/amg8833_tests.nv` checks the thermistor and pixel conversions against the tables
the specification prints in section 4-10 (the thermistor's 0x7D0, 0x190, 0x004, 0x804
and 0x940; the pixel's 0x1F4, 0x064, 0x001, 0xFFF, 0xF9C and 0xF24), the unlock sequence of
the moving average, the six level writes, the frame as one read of 128 bytes, and every
register function against a double of the register file. `bash tests/coverage.sh` measures
the line coverage of `src/`.

## Sources

- Specifications for Infrared Array Sensor Grid-EYE AMG88**, Panasonic, dated Aug. 30, 2011.
- Read for fixes, not ported: Zephyr's `drivers/sensor/amg88xx`, Adafruit_AMG88xx, and the
  Rust crate `grideye` 0.3.0.

## Licence

Apache-2.0.
