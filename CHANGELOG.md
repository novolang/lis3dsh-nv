# Changelog

## 0.1.0 (2026-10-08)

The first release: a driver for ST's LIS3DSH three-axis accelerometer on
an SPI bus.

- `Lis3dsh`, the type a board's device table builds from its wiring,
  implementing devices-nv's `Accelerometer` and `Identifiable`, with its
  own methods for the bring-up, the five full scales (±6 g included),
  the rate in millihertz, the INT1 line's level and register access.
- The register access as functions over any `SpiBus` and `GpioOut`,
  which the type's methods call, with every byte moved through
  `SpiBus.transfer`, so the driver builds no list.
- The host suite runs the functions over a double of the part's
  register file and the type's methods over the host's board handles.
