# icm42688-nv

A driver for the ICM-42688-P, a six-axis inertial measurement unit (IMU)
from TDK InvenSense with a three-axis gyroscope, a three-axis
accelerometer and a temperature sensor, on an I2C or an SPI bus,
specified in TDK's data sheet "ICM-42688-P: High Precision 6-Axis MEMS
MotionTracking Device", document DS-000347, revision 1.9
([invensense.tdk.com/products/6-axis/icm-42688-p](https://www.invensense.tdk.com/en-us/products/6-axis/icm-42688-p)).
The driver implements the `Accelerometer`, `Gyroscope`, `Imu`,
`Thermometer` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv). It reads the
accelerometer in milli-g and the gyroscope in milli-degrees per second.

**Hardware: NOT VERIFIED.** No board on the bench carries an ICM-42688-P
yet. The device program below builds for the nRF52840 DK and the
nRF54LM20 DK, and the driver's functions run over a double of the part
that follows the data sheet's register banks and bus protocols. A reading
from a part on a board has not been taken, and the program has not been
flashed.

## The device program

`examples/imu_reader.nv` reads an ICM-42688-P on a breakout board ten
times a second and prints the acceleration in milli-g, the rate of turn in
milli-degrees per second and the temperature in thousandths of a degree
Celsius over the board's UART. The breakout's SDA and SCL go to the
board's I2C pins, its CS pin to the supply, which selects I2C, and its
AP_AD0 pin to ground, which puts the part at 0x68. With the board flat and
still, `acc` shows about 1000 on its third axis:

```
acc mg 12 -4 1003  gyr mdps 8 -15 3  temp mC 26125
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_imu.{ Imu }
use dev_bus
use icm42688.{ Icm42688 }

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    // The part answers at 0x68 on the board's first I2C bus (AP_AD0 low).
    let imu = Icm42688.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 },
                              dev_bus.wiring_i2c(0, 0x68))
    // Resets the part, checks WHO_AM_I and starts both sensors.
    if not imu.begin()
        hal.uart.write("icm42688 did not start\n")
    while true
        match imu.imu_read()
            Some(s) =>
                // s.accel.z is close to 1000 with the board flat.
                hal.uart.write(str.from_int(s.accel.z))
                hal.uart.write("\n")
            None => hal.uart.write("no answer\n")
        hal.timer.delay_ms(100)
```

The full program, with the gyroscope and the temperature, is
`examples/imu_reader.nv`. A board package that depends on this package
builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the two boards below and for the emulated board, where it runs
the image and checks that the program reports that the part did not start:

```sh
bash tests/device_build.sh
```

| Image | Flash | Of which program | Of which tables and data | RAM (`.data` and `.bss`) |
|---|---|---|---|---|
| `nrf52840-dk` | 16,644 B | 6,040 B | 856 B | 3,236 B |
| `nrf54lm20-dk` | 18,124 B | 6,486 B | 1,822 B | 3,236 B |

The numbers are from the build's memory report. Flash also holds the
runtime and the compiler's support library. The nRF52840 DK's I2C bus is
TWIM0 on SDA P0.26 and SCL P0.27, and the nRF54LM20 DK's is TWIM22 on SDA
P1.03 and SCL P1.04 (`docs/hal/README.md` in the novo-lang repository,
"The Nordic DKs' buses"), so both images carry the reading path. A reply
from an ICM-42688-P over those buses is NOT VERIFIED until the driver rig.
`BoardI2c` reaches the bus that the board's package names with
`@hal(i2c)`; on a board that names none, the bus is absent hardware, every
transfer is refused, and the program prints `icm42688 did not start`.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else. The driver performs no input or output of its own and
allocates nothing of its own: every buffer is a fixed-size `Vec` in the
caller's frame. The bus functions of devices-nv build one short frame per
transaction inside a `system` block and release it when the call returns.
The waits go through the bus's `delay_us`, which is `[hw]` too. The package
checks at every tier, `embedded` included.

## What it is

The ICM-42688-P holds a 16-bit three-axis gyroscope, a 16-bit three-axis
accelerometer and a temperature sensor in a 2.5 x 3 mm package. The
gyroscope measures within a full scale of ±2000, ±1000, ±500, ±250, ±125,
±62.5, ±31.25 or ±15.625 degrees per second (dps), the accelerometer
within ±16, ±8, ±4 or ±2 g (1 g is the acceleration of gravity, 9.81
m/s²). Each axis is a signed 16-bit count, high byte first, and the
sensitivities are the data sheet's (Tables 1 and 2):

| Accelerometer | ±16 g | ±8 g | ±4 g | ±2 g |
|---|---|---|---|---|
| Counts per g | 2,048 | 4,096 | 8,192 | 16,384 |

| Gyroscope | ±2000 | ±1000 | ±500 | ±250 | ±125 | ±62.5 | ±31.25 | ±15.625 |
|---|---|---|---|---|---|---|---|---|
| Counts per dps | 16.4 | 32.8 | 65.5 | 131 | 262 | 524.3 | 1048.6 | 2097.2 |

The full-scale code runs from the widest range to the narrowest.
`GyroRange` has the four narrow ranges as `GyroRange125dps`,
`GyroRange62dps`, `GyroRange31dps` and `GyroRange15dps` (the names give the
whole degrees per second), and `gyro_set_range` takes them. `GyroRange4000dps`
is refused, since the part tops out at ±2000 dps. The readings are converted
at the sensitivity of the range set.

The registers are in five banks. REG_BANK_SEL (0x76) is in every bank and
selects the bank. Bank 0 holds the data, the configuration and the FIFO;
bank 1 the gyroscope's anti-alias filter and the interface settings; bank 2
the accelerometer's anti-alias filter. A part left in another bank answers
other registers for every later read, so each function that visits a bank
puts bank 0 back, also when the work failed, and answers `false` if either
the work or the restore failed. `begin` selects bank 0 first.

Each sensor has its own power mode in PWR_MGMT0 (Table 12): the
accelerometer is off, in low-power mode (duty-cycled) or in low-noise
mode; the gyroscope is off, in standby or in low-noise mode. The
accelerometer's rate depends on the mode (§ 14.38):

| Code | 0001 | 0010 | 0011 | 0100 | 0101 | 0110 | 0111 | 1000 | 1001 | 1010 | 1011 | 1100 | 1101 | 1110 | 1111 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Rate (Hz) | 32k | 16k | 8k | 4k | 2k | 1k | 200 | 100 | 50 | 25 | 12.5 | 6.25 | 3.125 | 1.5625 | 500 |

Codes 0001 to 0110 run in low-noise mode only, 1100 to 1110 in low-power
mode only, and 0111 to 1011 and 1111 in either. The gyroscope has the
codes 0001 to 1011 and 1111. The code 1111 is 500 Hz, out of order between
1 kHz and 200 Hz.

The temperature is `TEMP_DATA / 132.48 + 25` degrees Celsius (§ 4.13),
which the driver computes as `25000 + TEMP_DATA * 100000 / 13248`
milli-degrees. The part updates it at the faster of the two sensors' rates.

The part has two bus interfaces on the same pins; the chip-select pin
chooses.

| | I2C | SPI |
|---|---|---|
| Address | 0x68, or 0x69 with AP_AD0 high | chip select low |
| Several registers in one read | the address advances | the address advances |
| Read | register, repeated start, data | control byte with bit 7 set (R/W), then data, with no dummy byte |
| Write | register, then the data | control byte with bit 7 clear, then one byte |
| Clock | 400 kHz asked, 1 MHz at most | 4 MHz asked, mode 3 (mode 0 or 3), 24 MHz at most |

## Install

```sh
novo pkg add icm42688-nv
```

## What the package contains

| Module | What is in it |
|---|---|
| `icm42688` | The types `Icm42688` (I2C) and `Icm42688Spi` (SPI), `Icm42688Mode`, the trait `Icm42688Port`, the bus framing, the register access and the banks, the conversions, the rates and power modes, the filters, the FIFO, the self-tests and the bring-up |

## How to choose an entry point

A program on a board uses `Icm42688` for a part on I2C and `Icm42688Spi`
for a part on SPI. Each is built by `attach` with the board's bus, its
GPIO and the wiring, as a board's device table builds it. Both implement
`Accelerometer`, `Gyroscope`, `Imu`, `ImuControl`, `Thermometer` and
`Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` (I2C) or with the chip select `w.cs` (SPI); nothing is sent |
| `begin()` | brings the part up, see below |
| `accel_read()`, `gyro_read()`, `imu_read()` | `Accelerometer`, `Gyroscope`, `Imu`: milli-g, milli-degrees per second, and both from one burst of twelve registers |
| `accel_ready()`, `gyro_ready()`, `imu_ready()` | DATA_RDY_INT of INT_STATUS, one flag for both sensors, cleared by the read |
| `accel_range()`, `accel_set_range(r)`, `gyro_range()`, `gyro_set_range(r)` | the full scales, changed without touching the rate; the gyroscope's eight ranges from ±15.625 to ±2000 dps |
| `accel_rate_hz()`, `accel_set_rate_hz(hz)`, `gyro_rate_hz()`, `gyro_set_rate_hz(hz)` | the rates in whole hertz, and the slowest rate at or above `hz` that the sensor's mode runs; 0 turns the sensor off |
| `accel_rate_mhz()`, `accel_set_rate_mhz(mhz)`, `gyro_rate_mhz()`, `set_gyro_rate_mhz(mhz)` | the same in thousandths of a hertz |
| `accel_mode()`, `accel_set_mode(m)` | `AccelLowPower` and `AccelNormal` (low noise); `AccelHighResolution` sets low noise too, and `accel_mode()` answers `None` while the accelerometer is off |
| `gyro_mode()`, `set_gyro_mode(m)`, `set_temp_enabled(on)` | the gyroscope's power modes and TEMP_DIS |
| `imu_set_rate_hz(hz)`, `imu_rate_hz()`, `imu_standby(accel, gyro)` | `ImuControl`: one rate for both sensors (a request of 0 is the slowest rate, not off), and standby for either; the accelerometer has no standby mode, so `true` turns it off |
| `accel_aaf(delt)`, `accel_aaf_off()`, `gyro_aaf(delt)`, `gyro_aaf_off()` | the anti-alias filters, DELT 1 to 63, 42 Hz to 3979 Hz |
| `set_ui_filter_order(a, g)`, `set_ui_filter_bw(a, g)` | the order (1st to 3rd) and the bandwidth code of the UI filters |
| `temp_read()` | `Thermometer`: the die temperature |
| `device_id()` | `Identifiable`: WHO_AM_I, 0x47 |
| `accel_fifo_config(mode, n)`, `accel_fifo_watermark_reached()`, `fifo_end()` | turn the FIFO on in stream or stop-on-full mode with the watermark at `n` packets (1 to 32), whether the count has reached it, and the FIFO off |
| `accel_fifo_count()`, `accel_fifo_read(dst)`, `fifo_take_imu(accel, gyro)` | the FIFO: the packet count, the accelerometer's samples, and both sensors' |
| `route_int1(mask)`, `route_int2(mask)`, `configure_pins(active_low, open_drain)`, `set_int_timing(fast)`, `status()` | the interrupt pins and INT_STATUS |
| `accel_test()`, `gyro_test()` | the change in milli-g and in milli-degrees per second that each self-test makes |
| `reg(r)`, `set_reg(r, v)`, `bank_reg(bank, r)`, `set_bank_reg(bank, r, mask, v)` | one register, a write that is read back, and a register of another bank |

`begin` waits 3 ms for the supply, selects bank 0, resets the part with
DEVICE_CONFIG and polls INT_STATUS for RESET_DONE_INT, checks WHO_AM_I,
sets the FIFO count to packets and the data to big-endian order, turns the
I2C block off on SPI, sets the slew rate and the I3C block of § 12.3 on
I2C, clears INT_ASYNC_RESET (§ 12.6), puts the pins push-pull, active high
and pulsed, puts the data-ready flag on INT1, and starts both sensors in
low-noise mode at 100 Hz, ±2 g and ±250 dps. It then waits 50 ms for the
sensors to start. It answers `false` for a part that does not answer 0x47
and for a register that does not hold what was written.

A program that holds its own bus, or a test with a double of the part,
implements `Icm42688Port` over its bus and calls the module's functions,
which take the port: `icm42688.imu_read_of(port)`, `icm42688.start(port)`.
The framing of each bus is `i2c_read_into`, `i2c_write`, `spi_read_into`
and `spi_write`, generic over `I2cBus` and `SpiBus`.

### The FIFO

The FIFO is an option. It is 2 KB (§ 6.3) and `begin` leaves it off.
`accel_fifo_config` puts it in stream mode (`AccelFifoStream`), which
overwrites the oldest packet when it is full, or in stop-on-full mode
(`AccelFifoStop`), which keeps the oldest. It batches the accelerometer, and
the gyroscope too when the gyroscope is running, and puts the watermark on
INT1 in place of the data-ready flag. `AccelFifoOff` is `fifo_end`. The
packets are the 16-bit ones of § 6.1: 16 bytes when both sensors are
batched (a header, the accelerometer's six bytes, the gyroscope's six, the
temperature byte and a 16-bit timestamp), and 8 bytes for one sensor (a
header, six bytes and the temperature byte). The count is in packets
(FIFO_COUNT_REC) and the watermark is in packets, 1 to 32 through the trait; the
module's `fifo_start` takes up to 127 packets for 16-byte ones and 255 for
8-byte ones. `accel_fifo_watermark_reached` compares the count with the
watermark registers, because FIFO_THS_INT is in INT_STATUS, whose read clears
the data-ready flag with it. The reading takes
one packet per transaction from FIFO_DATA and decodes its header: bit 7 set
is an empty FIFO, bits 6 and 5 say which sensors the packet holds, and bit
4 marks the 20-bit packet, which is not read.

`accel_fifo_read` is the trait's and answers the accelerometer's samples;
it takes every packet out and drops the gyroscope's half. A program that
batches both reads with `fifo_take_imu`, which fills two lists and stops,
without taking a packet, as soon as either is full. A sample of -32768 is
the part's mark for an invalid sample (§ 14.34), which a sensor with a
lower rate than the other writes when it has no new data; it is taken out
and not put in a list.

## The rules a user needs

1. Wait for the supply before `begin`: the part answers registers 1 ms
   after power-up and the supply ramps in at most 3 ms (§ 3.3.2). `begin`
   waits 3 ms itself.
2. `begin` has to run before a reading: both sensors are off after a
   reset, and the data registers read -32768 until a sensor's first sample
   (§ 14.34). A reading of -32768 on any axis is no sample, so
   `accel_read`, `gyro_read`, `imu_read` and `temp_read` answer `None`.
3. After a change of mode, wait 200 µs before the next write (§ 14.36).
   The functions that change a mode wait through the bus's `delay_us`. The
   gyroscope has to stay on for 45 ms after it is turned on before it is
   turned off; the driver does not track that.
4. Reading INT_STATUS clears its flags. `accel_ready`, `gyro_ready` and
   `imu_ready` read the one DATA_RDY_INT flag, so the second of two calls in
   a row answers `false` until the next sample. A program that reads both
   sensors polls `imu_ready` once and reads `imu_read` (`dev_imu.imu_poll`
   does so). A program that polls for FIFO flags with `status()`
   loses the data-ready flag the same way.
5. The accelerometer in low-noise mode does not run rates below 12.5 Hz,
   and in low-power mode it does not run rates above 500 Hz (§ 12.1);
   `accel_set_mode` moves the rate to the nearest rate of the new mode. The
   anti-alias and notch filters work in low-noise mode only (§ 12.4).
6. The rates in whole hertz are rounded to the nearest, so 12.5 Hz reads
   as 13 and 1.5625 Hz as 2. Use the thousandths of a hertz for the exact
   rate; 1.5625 Hz is 1563 there.
7. The FIFO is configured from bypass with the interrupt routes off,
   flushed first, and FIFO_RESUME_PARTIAL_RD is set so that a read that
   does not take the whole FIFO can go on from where it stopped (§ 14.45).
   The data sheet does not say how a burst read behaves at FIFO_DATA; the
   driver reads one packet per burst, as Zephyr's driver reads the FIFO.
8. The self-tests need the board still. The data sheet defines the
   response as the output with the self-test enabled minus the output with
   it disabled (§ 4.9) and names the enable bits (SELF_TEST_CONFIG,
   § 14.57); it gives no limits and no procedure. The driver puts both
   sensors in low-noise mode at 1 kHz, the accelerometer at ±4 g and the
   gyroscope at ±250 dps, passes ten samples, averages five with the
   self-test off and five with it on, and answers the difference. The
   limits are TDK's to state; the driver does not judge the response. It
   sets SELF_TEST_CONFIG back to 0 afterwards, as § 14.57 asks, and puts
   the modes and the ranges back.
9. On SPI the part boots in SPI mode and `begin` turns the I2C block off
   (UI_SIFS_CFG in INTF_CONFIG0). SPI mode 0 and mode 3 both work, and the
   driver asks for mode 3. On I2C, the CS pin is held high.
10. The pins for rates of 4 kHz and above need `set_int_timing(true)`
    (§ 14.50); `begin` leaves the pulse timing at the 100 µs of the slower
    rates.
11. `device_id` answers `None` for a bus that no part drives: an SPI bus
    reads 0xFF, or 0x00 when held low.

## What is not included

- The 20-bit high-resolution FIFO packet (FIFO_HIRES_EN, packet 4 of
  § 6.1), with 19-bit gyroscope data and 18-bit accelerometer data.
  NOT IMPLEMENTED. A FIFO found in that format reads as empty.
- The APEX motion functions of § 8: the pedometer, tilt detection, tap
  detection, wake on motion and significant motion detection. NOT
  IMPLEMENTED. The registers are reachable through `reg`, `set_reg`,
  `bank_reg` and `set_bank_reg`.
- The gyroscope's notch filter, the user offset registers, the timestamp
  register, FSYNC, the external clock input (CLKIN and RTC_MODE), and the
  FIFO's FSYNC timestamp. NOT IMPLEMENTED.
- The I3C interface, and the three-wire SPI mode (INTF_CONFIG4).
- The ICM-42688-V and the other parts of the family (ICM-42686, IIM-42653).
- The async form of the traits. NOT IMPLEMENTED: devices-nv's
  `AccelerometerAsync`, `GyroscopeAsync` and `ThermometerAsync` are
  `async fn` traits, which `@tier(embedded)` refuses (E4002) and which take
  no `@value` struct, and the board handles are `@value` structs. The part
  completes nothing in the background that a program waits for: it samples
  on its own clock and the program reads the data-ready flag or the line.
  The interrupt line as `AsyncDeviceReady` needs an edge source on
  `BoardGpio` that embedded-hal-nv does not have.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  traits this driver implements, the wiring record and the register
  helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus`, `SpiBus` and `GpioOut` traits and the board handles.
- bmi270-nv and lsm6dso-nv drive six-axis IMUs from Bosch Sensortec and
  ST, with the same traits.
- mpu6050-nv drives TDK InvenSense's older MPU-6050.

## Tests

`novo test` runs the module over a double of the part with five register
banks that answers on an I2C bus and on an SPI bus: the control byte, the
address increment and the FIFO port that does not advance it, WHO_AM_I,
the bank selection and its restore after a failure, every conversion at
every range of both sensors, the invalid sample, the temperature, the rate
codes with the mode they depend on, the power modes and the 12.1
transitions, all 63 rows of the anti-alias table, the UI filters, the
interrupt routes, the FIFO with 16-byte and 8-byte packets, empty headers,
invalid samples and a full list, the self-tests with each write refused in
turn, the bring-up from stale settings with each write refused in turn, and
the two types over the host's board handles. The log of bytes sent is
compared for each entry point. A second port refuses a chosen write or
read of a chosen register, which reaches the refusals a read-only register
cannot. The suite covers every line of `src/`, and `tests/coverage.sh`
measures it:

```sh
NOVO=bin/novo bash tests/coverage.sh
```

**Hardware: NOT VERIFIED.** See the device program section.

The sources read for this driver, for their fixes and not ported, are the
data sheet above, Zephyr's `drivers/sensor/tdk/icm4268x` and the Rust
crate `icm426xx` 0.4.0 (`ProfFan/icm426xx`). Embassy and esp-hal hold no
driver for the part. Each fix kept is named in `src/icm42688.nv` where it
is kept:

- Zephyr waits 3 ms for the supply, resets with DEVICE_CONFIG, waits 2 ms
  where the data sheet says 1 ms, and reads RESET_DONE_INT from INT_STATUS
  before it reads WHO_AM_I; so does `begin`, polling for the flag.
- Zephyr waits 250 µs after PWR_MGMT0 (the data sheet says 200 µs) and 50
  ms after the rates (accelerometer 10 ms, gyroscope 30 ms); so does
  `begin`.
- Zephyr switches the interrupt sources off, puts the FIFO in bypass and
  flushes it before it configures; so does `accel_fifo_config`.
- The data sheet asks INT_ASYNC_RESET to be cleared (§ 12.6) and gives the
  I2C and SPI settings (§ 12.3); `begin` follows both.
- The data sheet's use note § 12.1 on the accelerometer's mode changes, and
  the invalid sample of § 14.34, which `icm426xx` and Zephyr's decoder
  also handle, are kept by `accel_set_mode` and by the readers.

## Where the traits fall short

- `AccelerometerAsync`, `GyroscopeAsync` and the other async forms are not
  implemented: `async fn` is refused at the embedded tier, and a `@value`
  driver cannot implement them. The part completes nothing in the background
  that a task would wait for.
- A trait method gets a plain `self`, so the driver keeps no state between
  calls; the full scales and the modes are read from the part in the
  transaction before each sample.
- `ImuControl` has one rate, and the part has two: `imu_set_rate_hz` sets both
  sensors and answers the accelerometer's, and an accelerometer in low-power
  mode can run slower than the gyroscope.
- `Accelerometer.accel_fifo_config` takes no choice of sensors, so the
  gyroscope is batched whenever it is running. The 20-bit packet is not read.

## Licence

Apache-2.0.
