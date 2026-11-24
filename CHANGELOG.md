# Changelog

## 0.3.0 — 2026-11-24

The register reads and writes, and the bus set-up, are devices-nv's `RegPort` and `wiring_*` functions, and `Qmc5883lPort` is smaller by them.  The release depends on devices-nv 0.4.1.

### Breaking

- **`Qmc5883lPort` has the supertrait `RegPort` and no longer declares `read_regs` and `write_reg`.**  A type that implements `Qmc5883lPort` for a bus of its own moves those two methods, unchanged, into an `impl RegPort`.  *Migration:* with `use dev_bus.{ RegPort }`,

  ```novo
  impl RegPort for MyPort
      fn read_regs(self, reg: Int, n: Int, var dst: DevBytes) -> Bool [hw]
          ...
      fn write_reg(self, reg: Int, v: Int) -> Bool [hw]
          ...

  impl Qmc5883lPort for MyPort
      ...                      // the other methods stay where they are
  ```

  The package's own types are migrated.  A caller that holds one of them is unaffected.

### Deprecated

- `i2c_setup`, `irq_level_of`, `reg_read`, `reg_write`, `reg_update`: each is a one-line function over the method or the `wiring_*` function that replaces it, warns W1001 at a call with the replacement named, and is removed in the next release.  `Qmc5883lPort` itself is the driver's trait and stays.

### Changed

- The register helpers the module carried (`reg_read`, `reg_write`, `reg_update` over its port) are devices-nv's `RegPort` methods (`read_reg`, `write_reg_verified`, `update_reg_verified` and the others); the module calls them as `p.read_reg(reg)`.
- The bus set-up and the interrupt-level read are devices-nv's `wiring_i2c_setup`, `wiring_spi_setup` and `wiring_irq_level`, called with the part's rate and mode.
- Image size on nrf52840-dk, `.text` plus read-only data, before and after: 14,964 bytes before and after on the read-out (nrf52-qemu: 8,000 and 8,000).

## 0.2.0 — 2026-11-12

The release depends on devices-nv 0.4.0 and embedded-hal-nv 0.4.0.

- `Qmc5883l.range()` and `set_range(r)` are now the `Magnetometer` methods `mag_range()`
  and `mag_set_range(r)`, which answer and take `MagRange2G` and `MagRange8G`; any other
  `MagRange` answers `false` and writes nothing.  `mag_range()` answers `None` when the
  part does not answer, where `range()` answered ±2 gauss.  The module functions
  `range_read` and `range_write` stay.
- `Qmc5883l.overflowed()` is now `Magnetometer.mag_status()`: `MagOverflow` when OVL is
  set, `MagNotReady` when DRDY is clear, `MagValid` otherwise.  After a data read DRDY is
  clear, so the answer is `MagNotReady` until the next sample.  `overflow_read` stays.
- `Thermometer.temp_kind()` answers `TempRelative`.
- New module functions `mag_range_read`, `mag_range_write`, `mag_status_read`, `status_of`,
  `mag_range_of` and `qmc_range_of`.

## 0.1.0

The first release: a driver for QST's QMC5883L three-axis magnetometer on an I2C bus.

- `Qmc5883l`, the type a board's device table builds, implementing devices-nv's
  `Magnetometer`, `Thermometer` and `Identifiable`, with its own methods for the range,
  the mode, the rate, the oversampling ratio, the overflow flag and the DRDY pin.
- The register access as functions over `Qmc5883lPort`, a trait that a type holding a
  bus implements, and the framing as functions over `I2cBus`.
- Readings in nanotesla at 12000 counts per gauss at ±2 gauss and 3000 at ±8, converted
  with the range read in the same burst as the data.
- The status flags (data ready, overflow, skipped data), the four data rates, the four
  oversampling ratios and the DRDY pin, each set without touching the other fields of
  its register.
- The die temperature as a change, 100 counts to the degree, from the two registers in
  one transaction.
- One function for a host's scheduler that parks the task for the data-ready flag
  (`mag_next`), and `irq_arm` and `irq_disarm` for the DRDY pin through an `AsyncEdge`
  source.
- The soft reset and a bring-up that checks the chip id, resets the part, writes the
  SET/RESET period 0x01 and starts continuous mode.
- The suite runs over a double of the part and covers every line of `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries a QMC5883L.
- The data sheet read is QST's Rev. 1.0 as reproduced in a module maker's data sheet
  (GY-271); the LCSC copy that the brief named could not be fetched.  It has no document
  number.  Its register map lists 0x0C and 0x0D as reserved; the chip id 0xFF at 0x0D is
  the value the `qmc5883l` crate reads.
- `MagnetometerAsync` and `ThermometerAsync` are not implemented: a `@value` struct takes
  no `async fn` (SPEC section 14.5).  `AsyncDeviceReady` is not implemented either, since
  no board's GPIO implements `AsyncEdge`; `irq_arm` is the part that does not depend on a
  board.
- The temperature has no absolute zero, and the `Thermometer` trait has no way to say so;
  a reading is a count times ten milli-degrees from an unknown offset.
- The device program builds for the nRF52840 DK, the nRF54LM20 DK and the FRDM-MCXN947,
  with images of 14,856, 15,840 and 16,832 bytes of flash.  A reply from a QMC5883L over
  the Nordic buses is NOT VERIFIED until the driver rig.
- The part has no self-test and no SPI interface; the pointer roll-over is not used.
