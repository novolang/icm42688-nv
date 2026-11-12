# Changelog

## Unreleased

Moves off its stand-ins onto devices-nv 0.4.0 and embedded-hal-nv 0.4.0, which
are path dependencies until they are published.

- `ImuControl` is implemented for `Icm42688` and `Icm42688Spi`: `imu_ready`
  reads the shared data-ready flag, `imu_set_rate_hz` and `imu_rate_hz` set and
  read the rate of both sensors, and `imu_standby` puts either in standby (the
  accelerometer, which has none, is turned off).
- `Accelerometer.accel_fifo_config(mode, watermark)` and
  `accel_fifo_watermark_reached` replace the inherent `fifo_begin`. Behaviour
  change: the gyroscope is batched when it is running and not on request, and
  the watermark is 1 to 32 packets and refused outside that (`fifo_begin`
  limited it to what the FIFO holds). `AccelFifoStop` is new: the part's
  stop-on-full mode. `icm42688.fifo_start` is unchanged.
- `GyroRange125dps`, `GyroRange62dps`, `GyroRange31dps` and `GyroRange15dps`
  replace the inherent `gyro_set_fs_code`: `gyro_set_range` takes them and
  `gyro_range` answers them, where 0.1.0 answered `GyroRange250dps` for the
  four narrow codes. `GyroRange4000dps` is refused. `gyro_fs_code_write` is
  gone.
- The inherent `accel_rate_mhz`, `set_accel_rate_mhz`, `accel_mode` and
  `set_accel_mode` are the trait's `accel_rate_mhz`, `accel_set_rate_mhz`,
  `accel_mode` and `accel_set_mode`, because an inherent method of the name of
  a trait method breaks the build. `accel_mode` answers `?AccelMode`: `None`
  while the accelerometer is off, `AccelLowPower` or `AccelNormal` otherwise.
  `Icm42688Mode` and `accel_mode_read` and `accel_mode_write` stay.
- The waits go through the bus's `delay_us`. The `timer` field of both types is
  gone; `attach` is unchanged.

## 0.1.0

The first release: a driver for TDK InvenSense's ICM-42688-P six-axis IMU
on an I2C or an SPI bus, from the data sheet DS-000347 revision 1.9.

- `Icm42688` (I2C) and `Icm42688Spi` (SPI), the types a board's device
  table builds from the part's wiring, implementing devices-nv's
  `Accelerometer`, `Gyroscope`, `Imu`, `Thermometer` and `Identifiable`.
- The register banks: REG_BANK_SEL, a visit to bank 1 or bank 2 that puts
  bank 0 back also when the work failed, and `bank_reg` and `set_bank_reg`
  for the registers the driver has no function for.
- The bring-up: the supply wait, the soft reset with the poll of
  RESET_DONE_INT, WHO_AM_I, the interface settings of § 12.3 for I2C and
  SPI, INT_ASYNC_RESET cleared, the pins push-pull, active high and
  pulsed, and both sensors in low-noise mode at 100 Hz, ±2 g and ±250 dps.
- The accelerometer in milli-g and the gyroscope in milli-degrees per
  second at the sensitivities of Tables 1 and 2, the ranges (the
  gyroscope's four narrow ranges by code), the rates by mode with the
  500 Hz code out of order, the power modes of Table 12 with the 200 µs
  wait and the mode-change rule of § 12.1, the temperature of § 4.13, and
  the data-ready flag.
- The anti-alias filters of both sensors from the 63-row table of § 5.3,
  and the order and bandwidth of the UI filters.
- The interrupt routes to INT1 and INT2, the pins' polarity and output
  type, and the pulse timing for 4 kHz and above.
- The FIFO in stream mode with the 16-bit packets of § 6.1 (16 bytes for
  both sensors, 8 for one), the count and the watermark in packets, a read
  of one packet per transaction, and the invalid sample dropped.
- The accelerometer's and the gyroscope's self-tests as the data sheet
  defines them: the response is the output with the self-test enabled
  minus the output with it disabled.
- The device program `examples/imu_reader.nv`, a reading loop. Built for
  the nRF52840 DK it is 16,644 B of flash and for the nRF54LM20 DK 18,124 B.
  `tests/device_build.sh` builds it for both and runs the emulated board's
  image.
- Not verified on hardware: no board on the bench carries the chip. A reply
  from an ICM-42688-P over the Nordic boards' TWIM and SPIM buses is NOT
  VERIFIED until the driver rig.
- Not implemented: the 20-bit FIFO packet, the APEX motion functions, the
  notch filter, the user offsets, the timestamp, FSYNC, CLKIN, I3C,
  three-wire SPI, and the async form of the traits.

### Known

- The data sheet states no limits for the self-test response, and the
  driver does not judge it.
- The data sheet does not say how a burst read behaves at FIFO_DATA. The
  driver reads one packet per burst, as Zephyr's driver reads the FIFO.
- The accelerometer's 1.5625 Hz is reported as 1563 in thousandths of a
  hertz.
