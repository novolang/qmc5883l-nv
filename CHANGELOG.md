# Changelog

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
