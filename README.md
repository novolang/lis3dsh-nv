# lis3dsh-nv

The LIS3DSH is ST's three-axis digital accelerometer: it measures the
acceleration along each of three axes and reports it over an SPI or I2C
bus, as its data sheet (DocID022405) describes.  This package drives the
part on an SPI bus and presents it as
[devices-nv](https://novo-lang.org/packages/devices-nv)'s
`Accelerometer` and `Identifiable`.

## What it is

An **accelerometer** reports acceleration in g, the acceleration of
gravity.  At rest it reads 1 g on the axis that points up and close to 0
on the others, so a reading also tells which way the board is tilted.
The driver answers every reading in **milli-g**, thousandths of g.

The part measures within a **full scale**: ±2, ±4, ±6, ±8 or ±16 g.  A
wider scale reaches larger accelerations and resolves small ones more
coarsely.  Each axis is a 16-bit two's complement count, and the data
sheet's **sensitivity** says what one step of the count is worth at each
full scale: 0.06 mg at ±2 g, 0.12 at ±4 g, 0.18 at ±6 g, 0.24 at ±8 g
and 0.73 at ±16 g (Table 3).

The part samples at one of nine **output data rates**, from 3.125 to
1600 samples a second (Table 23), or is powered down.  It raises a
**data-ready** flag when a sample of all three axes is complete, and can
pulse its INT1 pin at the same moment, so a board sees a new sample
without asking the part.

On **SPI** a transaction is an address byte and the data inside one
chip-select window: bit 7 of the address byte reads, and bits 6 to 0
name the register (§ 5.2).  The address advances after each byte of a
longer transfer while the ADD_INC bit of CTRL_REG6 is set.  The part's
identification byte, **WHO_AM_I** at register 0x0F, is 0x3F.

## Install

```
novo pkg add lis3dsh-nv
```

## Example

A board names the part in the `devices` block of its declaration, as
the STM32F407G-DISC1's does:

```novo norun:fragment
    devices
        accel  Lis3dsh  from "lis3dsh-nv"  on spi0  cs accel_cs  irq accel_int1
```

and a program reads it as an accelerometer:

```novo norun:needs-pkg
use dev_accel.{ Accelerometer }
use lis3dsh

@tier(embedded)
fn main() [hw]
    let accel = bsp.board.accel()
    // Brings the part up: ±2 g, 100 samples a second, data-ready
    // pulsed on INT1.
    if accel.begin()
        match accel.accel_read()
            // The acceleration along z, in milli-g.
            Some(a) => hal.uart.write(str.from_int(a.z))
            None    => hal.uart.write("no sample")
```

## What the package contains

| Module | What is in it |
| --- | --- |
| `lis3dsh` | `Lis3dsh`, the driver: `attach`, `begin`, `scale_g` and `set_scale_g`, `rate_mhz` and `set_rate_mhz`, `irq_level`, `reg` and `set_reg`, and devices-nv's `Accelerometer` and `Identifiable`; and the same operations as functions over any `SpiBus` and `GpioOut`, with the register map's conversions |

## How to choose an entry point

A program on a board uses `Lis3dsh`, which the board's device table
builds.  A program that holds its own bus, or a test with a double of
the part, calls the module's functions, which take the bus, the GPIO
and the wiring as arguments.

## The rules a user needs

1. `begin` must run before a reading.  It answers `false` for a part
   that does not answer 0x3F at WHO_AM_I, the LIS302DL of the earliest
   STM32F4DISCOVERY boards among them (it answers 0x3B).
2. devices-nv's `AccelRange` names ±2, ±4, ±8 and ±16 g.  ±6 g is
   reached through `set_scale_g(6)`; while the part is at ±6 g,
   `accel_range` answers ±2 g, and `scale_g` answers 6.
3. The trait's rates are whole hertz, so the three slowest rates read as
   3, 6 and 12 Hz through `accel_rate_hz`.  `rate_mhz` answers them in
   millihertz: 3125, 6250 and 12500.  A rate between two of the part's
   is rounded up to the next one.
4. A reading is converted at the scale the part is set to, read from
   CTRL_REG5 just before the sample.  The three axes of a sample are
   read in one transaction with block data update on, so they come from
   one sample.
5. After `begin`, INT1 pulses high for about 50 us once a sample (IEL
   set, pulsed, Table 19).  A program that waits on the line takes its
   rising edge.
6. The part listens on its I2C interface while its chip select is high
   (§ 5.2), so traffic on the bus while it is deselected can leave that
   interface mid-frame.  `begin` spends one read before WHO_AM_I for that
   reason.
7. At rest a reading is not exactly 1 g and 0.  The data sheet gives the
   zero-g offset as ±40 mg, typical (Table 3).

## Running on a microcontroller

Every function builds for a microcontroller.  The driver's own code
builds no list: its bytes move through devices-nv's `DevBytes`, a
32-byte buffer held in the caller's frame, and every byte goes through
`SpiBus.transfer`.  The registry shows the package on the host layer,
the layer of the trait packages it depends on.

## What is not included

- The I2C interface: the board that carries the part wires it to SPI.
- The two state machines, the FIFO, the offset registers, the self-test
  and the temperature byte: no trait of devices-nv names them; `set_reg`
  reaches the registers.
- A soft reset at `begin`: the data sheet gives no time for it to take,
  and `begin` writes every register the driver relies on instead.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `Accelerometer` trait this driver implements, the wiring record and
  the SPI helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv)
  names the `SpiBus` and `GpioOut` traits and the board handles.
- lsm303agr-nv and lsm303dlhc-nv drive ST's eCompass modules, whose
  accelerometers are another way into the same trait, over I2C.

## Tests

`novo test tests` runs the module's functions over a double of the
part's register file: the address byte with its read bit and without an
MS bit, the address increment through CTRL_REG6, the sign of each
count, the sensitivity at each scale, the rates, the bring-up and a bus
with no part; and each method of the type `Lis3dsh` over the host's
board handles, which are absent hardware whose every byte reads 0.  The
driver on hardware is measured by the STM32F407G-DISC1's suite in the
novo-lang repository,
`orbit/bsp/st/stm32f4/stm32f407g-disc1/tests/accel.sh`, which reads the
part on the board and the coverage of this module there.

## Licence

Apache-2.0.
