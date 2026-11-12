# qmc5883l-nv

A driver for QST's QMC5883L, a three-axis magnetometer with an I2C interface,
specified in QST's data sheet "QMC5883L: 3-Axis Magnetic Sensor", Rev. 1.0.  The
driver implements the `Magnetometer`, `Thermometer` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv) and reads the field in
nanotesla.

## Install

```sh
novo pkg add qmc5883l-nv
```

## Example

A device program reads the part over I2C.  The breakout's SDA and SCL go to the
board's I2C pins; the part's address is fixed at 0x0D.  The program brings the part
up in continuous mode at 10 Hz and ±2 gauss and prints one line per sample over
RTT.  The Earth's field is between 25 000 and 65 000 nanotesla.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_mag.{ MagNanoT, Magnetometer }
use qmc5883l
use qmc5883l.{ Qmc5883l }

// The part's address.
const ADDR = 0x0D

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

@tier(embedded)
fn main() [hw]
    let part = Qmc5883l.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    if not part.begin()
        hal.uart.write("qmc5883l: no answer\n")
        return
    loop
        match part.mag_read_new()
            Some(m) =>
                hal.uart.write("qmc5883l")
                field(" x=", m.x)
                field(" y=", m.y)
                field(" z=", m.z)
                hal.uart.write("\n")
            None    =>
                hal.timer.delay_ms(5)
```

The program is `tests/device_read.nv`, and `tests/device_build.sh` builds it for the
boards below.  The nRF52840 DK's I2C bus is TWIM0 on SDA P0.26 and SCL P0.27, and the
nRF54LM20 DK's is TWIM22 on SDA P1.03 and SCL P1.04 (`docs/hal/README.md` in the
novo-lang repository, "The Nordic DKs' buses").

```sh
novo build --target=nrf52840-dk tests/device_read.nv
novo build --target=nrf54lm20-dk tests/device_read.nv
novo build --target=frdm-mcxn947 tests/device_read.nv
```

**Hardware: NOT VERIFIED.**  No board carries a QMC5883L yet.  The program builds for
the three boards, the image for the emulated board runs under QEMU and reports that
the part does not answer, and the driver's functions run over a double of the part
that follows the data sheet's register map and bus protocol.  A reading from a part on
a board has not been taken, and a reply from a QMC5883L over the Nordic buses is NOT
VERIFIED until the driver rig exists.

### The tier, the boards and the footprint

The package is checked at the embedded tier (`[tier]` in `novo.toml`), so every
function in it is also valid at `rt` and `system`; no migration knob is on.  Each row
is the memory report of `novo build` for the program above, with the driver's reading
path in the image.

| Board | Flash | Of which the program | `.data` and `.bss` | Static | Arena provable |
|---|---|---|---|---|---|
| nRF52840 DK | 14,856 bytes | 4,300 bytes | 3,252 bytes | 510 bytes | 208 bytes |
| nRF54LM20 DK | 15,840 bytes | 4,470 bytes | 3,236 bytes | 510 bytes | 208 bytes |
| FRDM-MCXN947 | 16,832 bytes | 6,014 bytes | 2,748 bytes | not recorded | not recorded |

The rest of each image is the runtime, the compiler's support library and tables.  The
driver keeps no state: every buffer is a fixed-size `Vec` in the caller's frame, and
`MagNanoT` and `TempMilliC` are `@value` structs.  "Arena provable" is the compiler's
count of allocations at initialisation.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and nothing else.
The driver performs no input or output of its own.  The bus functions of devices-nv
build one short frame per transaction inside a `system` block and release it when the
call returns.  The awaiting function, `mag_next`, declares `[hw, async]` and is
`@tier(rt)`.  The program above declares `[hw]` for the same reason.

## What it is

A magnetometer measures the magnetic field along three axes.  The Earth's field at the
surface is between about 25 and 65 microtesla, and a magnetometer that reads it tells a
program which way the board faces, as a compass does.  Iron and current-carrying wires
near the part add their own field.

The QMC5883L measures within a range of ±2 or ±8 gauss.  Each axis is a 16-bit two's
complement count, 12000 counts per gauss at ±2 gauss and 3000 at ±8 gauss (Table 2); the
counts saturate at -32768 and 32767, and OVL in the status register says so.  One gauss
is 100 000 nanotesla, so a count is 8.33 nT at ±2 gauss and 33.3 nT at ±8 gauss, and the
driver rounds toward zero.

The part has two modes, standby (the state after power-on and reset) and continuous.  In
continuous mode it measures in the background at 10, 50, 100 or 200 Hz, with an
oversampling ratio of 512, 256, 128 or 64 that trades noise against current (Table 18),
and sets DRDY in the status register when a new set of data is ready.  It can drive the
same event onto its DRDY pin.  It also has a temperature sensor, whose gain is
calibrated at about 100 counts per degree and whose offset is not: only changes in
temperature are meaningful.

| Field of CTRL1 (0x09) | Bits | Codes |
|---|---|---|
| Oversampling | 7 and 6 | 00 is 512, 01 is 256, 10 is 128, 11 is 64 |
| Range | 5 and 4 | 00 is ±2 gauss, 01 is ±8 gauss |
| Data rate | 3 and 2 | 00 is 10 Hz, 01 is 50 Hz, 10 is 100 Hz, 11 is 200 Hz |
| Mode | 1 and 0 | 00 is standby, 01 is continuous |

The data sheet's own example sets 0x1D: 512 samples, ±8 gauss, 200 Hz, continuous.

The bus is I2C at address 0x0D, up to 400 kHz.  The address advances by itself during a
read, and a read is a write of the register and a repeated start.

## What the package contains

| Module | What is in it |
|---|---|
| `qmc5883l` | The type `Qmc5883l` (I2C), `QmcRange`, `QmcMode`, the trait `Qmc5883lPort`, the bus framing, the register access, the conversions, the settings, the temperature, the awaiting function and the bring-up |

## How to choose an entry point

A program on a board uses `Qmc5883l`, built by `attach` with the board's bus, its GPIO
and the wiring, as a board's device table builds it.  It implements `Magnetometer`,
`Thermometer` and `Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr`; nothing is sent |
| `begin()` | brings the part up, see below |
| `mag_read()`, `mag_ready()`, `mag_read_new()` | `Magnetometer`: the latest sample in nanotesla, the data-ready flag, a sample not read before |
| `mag_rate_hz()` | the rate in continuous mode; 0 in standby |
| `temp_start()`, `temp_read()` | `Thermometer`: the part converts with its measurements, so a start answers 0, or -1 in standby; a reading is a change, see the rules |
| `mag_range()`, `mag_set_range(r)` | `Magnetometer`: the full scale as `MagRange2G` or `MagRange8G`, changed without touching the other fields of CTRL1; any other range answers `false` and writes nothing |
| `mag_status()` | `Magnetometer`: `MagOverflow` when OVL is set, `MagNotReady` when DRDY is clear, `MagValid` otherwise; STATUS is read on its own |
| `temp_kind()` | `Thermometer`: `TempRelative`, since the reading has no fixed zero |
| `device_id()` | `Identifiable`: the chip id, 0xFF |
| `mode()`, `set_mode(m)` | standby or continuous |
| `set_rate_hz(hz)` | the slowest rate at or above `hz`, answering the rate taken |
| `oversampling()`, `set_oversampling(n)` | the ratio, and the smallest ratio at or above `n` |
| `set_drdy_pin(on)` | the DRDY pin on or off |
| `reg(r)`, `set_reg(r, v)` | one register, and a write that is read back |

A program that holds its own bus, or a test with a double of the part, implements
`Qmc5883lPort` over its bus and calls the module's functions, which take the port:
`qmc5883l.sample_read(port)`, `qmc5883l.start(port)`.  The framing is `i2c_read_into` and
`i2c_write`, generic over `I2cBus`.  `status_read(port)` answers STATUS whole and
`status_ready`, `status_overflow` and `status_missed` test its bits.

### Waiting for a sample

`MagnetometerAsync` cannot be implemented by `Qmc5883l`: a `@value` struct takes no `async
fn` (SPEC section 14.5).  The package gives the wait as a function for a host's
scheduler: `mag_next(part, timer, period_us, tries)` parks the task between polls of the
data-ready flag, taking a `TimerAsync`.  The part measures in the background in
continuous mode, so the task is free in between.

A program at `@tier(embedded)` waits on the DRDY pin instead.  `irq_arm(edge, w, f)` and
`irq_disarm(edge, w)` arm and release a rising-edge wait on the pin of the wiring through
any `AsyncEdge` source.  No board's GPIO implements `AsyncEdge` yet, so this path is NOT
VERIFIED on a board.

## The rules a user needs

1. `begin` has to run before a reading: the part is in standby at power-on.  It checks
   the chip id, resets the part, writes the SET/RESET period register (0x0B) with 0x01,
   which the data sheet says it is to hold and which the reset clears, and starts
   continuous mode.
2. The chip id is not in Rev. 1.0, which lists 0x0C and 0x0D as reserved.  The driver
   reads 0xFF from 0x0D, as the `qmc5883l` crate does, and `begin` refuses a part that
   answers otherwise.  A module may carry another chip: the QMC5883P answers at 0x2C and
   the HMC5883L at 0x1E, and neither is this part.
3. A read of any of the six data registers clears DRDY and DOR and starts data
   protection, in which the data registers are not updated until 0x05 has been read.
   The driver always reads all six, in one burst with STATUS, the temperature and CTRL1.
   The ready flag is read from STATUS alone, which does not clear it.
4. STATUS in the sample burst is read after the data, so its DRDY and DOR bits are
   already clear there.  `mag_status()` and `status_read` read it on its own.  A data read
   clears DRDY, so `mag_status()` after a `mag_read()` answers `MagNotReady` until the next
   sample completes; OVL is not cleared by that read, so an overflow stays visible.
5. CTRL2's pointer roll-over (ROL_PNT) makes a read that begins at 0x00 to 0x06 wrap from
   0x06 to 0x00, so a burst that goes on to the temperature and CTRL1 would return the
   data twice.  The driver leaves it clear.
6. A sample is converted at the range it was taken in, because CTRL1 is in the same
   burst.  A saturated count is converted as the count it saturated at: the field is
   beyond the range, and `mag_status()` says `MagOverflow`.
7. The temperature is the count times ten, in milli-degrees, with no fixed zero: only a
   difference between two readings is a temperature change.  `temp_kind()` answers
   `TempRelative` for it.
8. CTRL2's INT_ENB is 1 for the pin disabled and 0 for enabled, as Rev. 1.0 words it.  The
   reset leaves the pin enabled, and so does `begin`; `set_drdy_pin(false)` disables it.
9. After a soft reset every register is back at its reset value: standby, ±2 gauss, 10
   Hz, 512 samples and the SET/RESET period 0.

## What is not included

- A self-test: the part has none.
- The pointer roll-over (ROL_PNT) as a feature: the driver reads in bursts that do not
  need it, and leaves it clear.
- An SPI interface and a second address: the part has I2C only, at 0x0D; the data sheet
  says other addresses are available from the factory.
- The variants that look alike and are not this part: the QMC5883P (a different chip at
  another address; Adafruit's "QMC5883" board carries it), the HMC5883L (Honeywell, at 0x1E),
  and the undocumented "A" version of the QMC5883L that flight-controller code mentions.
  Modules sold as GY-271 carry one or the other.
- A conversion of the temperature to an absolute value.

## Where the traits fall short

The gaps that remain for this driver:

- `MagnetometerAsync` cannot be implemented by a `@value` driver (SPEC section 14.5) and
  the embedded tier refuses `async fn`, so `mag_next` stays a free `@tier(rt)` function
  over a `TimerAsync`.
- No board's GPIO implements `AsyncEdge`, so `irq_arm` and `irq_disarm` have no source.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the traits this driver
  implements, the wiring record and the register helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the `I2cBus`,
  `GpioOut`, `TimerAsync` and `AsyncEdge` traits and the board handles.
- [lis3mdl-nv](https://novo-lang.org/packages/lis3mdl-nv) drives the LIS3MDL, a
  magnetometer of ST on I2C or SPI.
- [bmm150-nv](https://novo-lang.org/packages/bmm150-nv) drives the BMM150, a magnetometer
  of Bosch Sensortec whose output the host compensates.

## Tests

`novo test` runs the module over a double of the part that answers on an I2C bus: the
register and the burst, the data registers and the flags that a read of them clears,
the pointer roll-over, the conversions at both ranges, the settings that must not touch
each other's fields, the temperature, the soft reset that clears the SET/RESET period, the
bring-up and its refusals, the awaiting function over a timer double, and the type over
the host's board handles.  The log of bytes sent is compared for each entry point.  The
suite covers every line of `src/`.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are the data sheet above
(the copy fetched reproduces its text in a module maker's data sheet, GY-271; the copy
named in the brief, LCSC's, could not be fetched) and the `qmc5883l` crate 0.0.1.  There
is no Zephyr driver for the part.  Each fix kept is named in `src/qmc5883l.nv` where it is
kept:

- the crate's range constant is bit 3 and its mask clears bit 3, but the range is bits 5
  and 4 and bit 3 is the high bit of the data rate, so setting ±8 gauss raises the rate and
  leaves the range alone; every setting here changes only its own field;
- the crate reads the temperature in two transactions, so a new measurement between them
  gives a count of two samples; the driver reads both bytes at once;
- the crate's `enable_interrupt` writes CTRL2 whole; `set_drdy_pin` changes INT_ENB only;
- the crate relies on the pointer roll-over to read STATUS and the data in one burst from
  0x06, which loses the range and the temperature from the burst; the driver reads from
  0x00 with the roll-over clear;
- the SET/RESET period is rewritten after every reset, since the reset clears it.

## Licence

Apache-2.0.
