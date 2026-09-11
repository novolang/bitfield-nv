# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
