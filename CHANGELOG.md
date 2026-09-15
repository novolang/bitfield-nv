# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Four modules.  `bfflags` is a flag word that knows how wide it is,
  with `bitflags`'s set algebra on it; `bffield` is a field described as
  an offset, a width and a signedness, with `extract` and `insert` as
  pure functions over `u8`, `u16`, `u32` and `u64`; `bfreg` is a
  register's value with read-modify-write chained on it; and `bitfield`
  is the names — a label per field, an enumeration per value, a
  register as a list of those, and the printing and parsing.
- **Nothing here touches memory.**  A register's value is a value, and
  the MMIO read and write stay in `hal`.  That is what makes the package
  `core` and what keeps a driver's two memory accesses in the two places
  that may perform them.
- **The width is a field on the value.**  `Int` is 64 bits and a status
  word is 8, 16 or 32, and `complement` is where that shows: over an
  8-bit word the complement of the empty set is 255, not -1.
- **A signed field sign-extends.**  A six-bit trim holding `0b111111` is
  -1; a driver that forgot reads 63 and never finds out.
- `bitfield.validate` refuses a description whose two fields claim the
  same bit, naming both and the bit.  That is the bug this package
  exists to catch: an off-by-one offset writes one field and clobbers
  another, and nothing fails until a peripheral misbehaves months later.
- The device path masks and the host path checks: `bffield.insert`
  truncates a value too large for its field, and
  `bitfield.write_all` answers `BfValueOutOfRange` with the field's name
  and range.

**The device claim is built.**  `tests/embedded_probe.nv` compiles
`bfflags`, `bffield` and `bfreg` to a Cortex-M4 ELF for
`--target=nrf52-qemu`, driving the flag algebra, all four typed widths
and a chained register write.  `bitfield` is outside the probe on
purpose: it speaks `Str` and lists, and one host-only function anywhere
in a compilation unit is an undefined symbol at embedded link time.

**No list of fields on the device side.**  The plan's row asks for a
register description as a list of fields; a `@value` struct cannot own
one, so the list is host-side in `BfRegDesc` and the device side is the
word plus a chain of `with` calls — which is the sequence a driver would
have written anyway, with the shifts named.

### Design notes

The consumers this was drawn from, recorded here because the README no
longer carries them.  In `orbit/bsp/nordic/nrf52/nrf52840-dk` the
nRF52840 UART0 block declares five registers as one blob each:
`CONFIG` is `HWFC:1@0`, `PARITY:3@1`, `STOP:1@4`, `PARITYTYPE:1@8`;
`ERRORSRC` is four independent flags cleared by writing 15; `ENABLE` is
an enumeration where 0 is Disabled and 4 is Enabled; `PSEL_*` is
`PIN:5@0`, `PORT:1@5`, `CONNECT:1@31`; and `BAUDRATE` is an enumeration
of some twenty constants the driver writes as raw hexadecimal.  The
GPIO block is the other two shapes: `OUTSET`, `OUTCLR`, `IN`, `DIRSET`
and `DIRCLR` are one bit per pin, and `PIN_CNF[32]` is `DIR:1@0`,
`INPUT:1@1`, `PULL:2@2`, `DRIVE:3@8`, `SENSE:2@16`.

`orbit/bsp/raspberrypi/rp2040` has the same three shapes on a different
chip: `SIO.GPIO_*` one bit per pin, `RESETS.RESET` and `RESET_DONE` a
flag set with one bit per peripheral block, `IO_BANK0.CTRL[30]` as
`FUNCSEL:5@0`, `OUTOVER:2@8`, `OEOVER:2@12`, `INOVER:2@16`,
`IRQOVER:2@28`, and `PADS_BANK0.GPIO[30]` as `SLEWFAST:1@0`,
`SCHMITT:1@1`, `PDE:1@2`, `PUE:1@3`, `DRIVE:2@4`, `IE:1@6`, `OD:1@7`.

`orbit/hal` carries three enumerations in integer arguments documented
only in comments: `GpioOut.mode`'s `dir`, `GpioInAsync.wait_for_edge`'s
`edge`, and `SpiBus.init`'s `mode`.  Those comments are what a
`BfFieldDesc` with a value enumeration becomes.

The `bsp` / `peripheral` / `register` / `field` block stays as it is.
It is a compile-time declaration whose offsets are baked into the
firmware, and it cannot be built at run time, passed to a function,
printed or validated against a datasheet by a test.  This package is
the run-time value of the same idea, and nothing here depends on the
block changing.
