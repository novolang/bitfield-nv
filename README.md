# bitfield-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

A register's bits, described once and then read and written by the
description instead of by hand.

Three things, and they build for a Cortex-M:

- **a flag set** — a word that knows how wide it is, with `bitflags`'s
  set algebra on it: `insert`, `remove`, `toggle`, `contains`,
  `intersects`, union, intersection, difference, symmetric difference,
  complement, and the lowest set bit an interrupt dispatcher loops on;
- **a field** — an offset, a width and a signedness, with `extract` and
  `insert` as pure functions over `u8`, `u16`, `u32` and `u64` words;
- **a register** — a word plus a chain of field writes, so the
  read-modify-write sequence every driver writes by hand is one
  expression.

And a fourth module for the host, where the names live: a field with a
label and an enumeration of its values, a register as a list of those,
`validate` to catch a description whose fields overlap, and the printing
and parsing a person needs.

**Nothing here touches memory.** A register's value is a value; the MMIO
read and the MMIO write stay in `hal`, which is what makes this `core`
and what keeps a driver's two memory accesses in the two places that are
allowed to perform them.

## `std.bits` already shifts — what does this add?

[`std.bits`](https://novo-lang.org/docs/stdlib/bits) is seven operators
and eleven functions over a bare `Int`: `&`, `|`, `^`, `~`, `<<`, `>>`,
`>>>` and `bits.get`, `bits.set`, `bits.clear`, `bits.flip`,
`bits.rotate_left`, `bits.rotate_right`, `bits.count_ones`, the two zero
counts and `bits.bit_len`.  Those are the instructions, and everything
here lowers to them.  What this adds is the four things a bare `Int`
does not carry:

| `std.bits` gives you | this adds |
| --- | --- |
| `a & m`, `a \| m`, `a ^ m` | a value that knows its word is 8, 16 or 32 bits, so `complement` answers 255 and not -1 |
| `bits.get(a, i)` | a field that is more than one bit — a width, an offset, and the shift done once instead of at every call site |
| — | **sign extension**: a six-bit trim holding `0b111111` is -1, and a driver that forgot reads 63 and never finds out |
| — | **a description you can check**: `validate` refuses a register whose two fields claim the same bit, which is the bug that otherwise surfaces months later as a peripheral misbehaving |
| — | **names**, host-side: `CONFIG { HWFC: 1, PARITY: Included(7) }` out of a word a probe read |

`bits.rotate_left`, `bits.count_ones` and the zero counts stay the only
spelling for what they do; this package calls them rather than
reimplementing them.

## Which registers in `hal` and `bsp` this describes

The consumers exist and they write these bits by hand today.  Named
concretely, because "a register description" is abstract until you see
the word it replaces:

**`orbit/bsp/nordic/nrf52/nrf52840-dk`** — the nRF52840 UART0 block is
five registers of fields declared as one blob each:

| register | declared today | what it is |
| --- | --- | --- |
| `CONFIG` | `field hwfc : 1 @ 0` | `HWFC:1@0`, `PARITY:3@1`, `STOP:1@4`, `PARITYTYPE:1@8` — four fields, one named |
| `ERRORSRC` | `field src : 4 @ 0` | four independent flags — `OVERRUN`, `PARITY`, `FRAMING`, `BREAK` — written back as the literal `15` to clear them all |
| `ENABLE` | `field en : 4 @ 0` | an enumeration: `0` is Disabled and `4` is Enabled, and both appear as bare numbers in the driver |
| `PSEL_TXD` / `RXD` / `RTS` / `CTS` | `field pin : 6 @ 0` | `PIN:5@0`, `PORT:1@5`, `CONNECT:1@31` — the driver writes `6`, `8`, `5`, `7` and relies on CONNECT being zero |
| `BAUDRATE` | `field rate : 32 @ 0` | an enumeration of twenty-odd constants; the driver writes `0x01D60000` and `0x10000000` |

and the GPIO block is the other two shapes:

- `OUTSET` / `OUTCLR` / `IN` / `DIRSET` / `DIRCLR` are **one bit per
  pin** over a 32-bit word — a `BfFlags(32)`, and `bfflags.mask(pin)`
  is the `1 << pin` the HAL writes;
- `PIN_CNF[32]`, declared `field cfg : 32 @ 0`, is `DIR:1@0`,
  `INPUT:1@1`, `PULL:2@2`, `DRIVE:3@8`, `SENSE:2@16`.  The HAL writes
  the literals `3` and `0` into it, and `3` means "output, standard
  drive, input buffer connected" — which is a sentence the word does not
  say and a `BfRegDesc` would.

**`orbit/bsp/raspberrypi/rp2040`** — the same three shapes on a
different chip, which is the argument that the shapes are the chip's and
not Nordic's:

- `SIO.GPIO_IN` / `GPIO_OUT_SET` / `GPIO_OUT_CLR` / `GPIO_OE_SET` /
  `GPIO_OE_CLR` — one bit per pin again;
- `RESETS.RESET` and `RESET_DONE`, declared `field bits : 32 @ 0` — a
  flag set with one named bit per peripheral block, and a name table is
  exactly what makes a reset-stuck bug readable;
- `IO_BANK0.CTRL[30]`, declared `field funcsel : 5 @ 0` — really
  `FUNCSEL:5@0`, `OUTOVER:2@8`, `OEOVER:2@12`, `INOVER:2@16`,
  `IRQOVER:2@28`, with FUNCSEL an enumeration (`SIO` is 5, `UART` is 2);
- `PADS_BANK0.GPIO[30]`, declared `field cfg : 8 @ 0` — really
  `SLEWFAST:1@0`, `SCHMITT:1@1`, `PDE:1@2`, `PUE:1@3`, `DRIVE:2@4`,
  `IE:1@6`, `OD:1@7`.

**`orbit/hal`** — the traits themselves carry three enumerations in
integer arguments, each documented in a comment above the trait and
nowhere in the type: `GpioOut.mode`'s `dir` (0 input, 1 output, 2
pull-up, 3 pull-down), `GpioInAsync.wait_for_edge`'s `edge` (0 rising, 1
falling, 2 either), and `SpiBus.init`'s `mode` (0..3, which is CPOL at
bit 1 and CPHA at bit 0 — a two-field register in a two-bit word).  A
`BfFieldDesc` with a value enumeration is what those comments become.

**`orbit/sensorhub`** — `I2cBus.write_reg(addr, reg, val)` is the sensor
case: every driver for a real part spends its configuration in one-byte
registers of two- and three-bit fields, and `val` is where the shifts
would otherwise be.

**This is not `embedded-hal-nv`, and it does not absorb `orbit/hal`.**
The owner flagged `hal` on 2026-09-11 for a move and probably a split
when the embedded wave is planned; that is not this package and not this
wave.  What this gives `hal` is a *type to use later* — the register
descriptions above become values, and the trait shapes stay where they
are.

## The `bsp` block already says `field cfg : 32 @ 0` — is this that?

No, and the two are complementary.  The `bsp` / `peripheral` /
`register` / `field` block is a **compile-time declaration**: the
compiler reads it and generates the accessor functions, and its offsets
are baked into the firmware.  It cannot be built at run time, passed to
a function, printed, or validated against a datasheet by a test.

This package is the **run-time value** of the same idea.  A register-map
generator emits `BfRegDesc`s; a debugger decodes a word it read over
SWD; a host-side test asserts that a description's fields do not
overlap.  Longer term the `bsp` block is where a description could be
generated *from* — one source, two consumers — but nothing here depends
on that and the block is unchanged.

## Adding it, and checking it

```bash
novo pkg add bitfield-nv     # into your novo.toml
novo pkg build               # type- and effect-check the package
novo test --isolate tests/bitfield_tests.nv
```

`novo test` is red today and that is the point of the release: all
thirty-eight assertions fail with `not implemented: <module>.<fn>`.
They turn green one at a time as bodies land.

## The one example that will work

```novo
use bffield
use bfreg
use bitfield

fn main() [io]
    // The nRF52840 UART CONFIG register, described once.
    let hwfc = bffield.field(0, 1)
    let parity = bffield.field(1, 3)

    // The read-modify-write, as arithmetic on a word the HAL read.
    let cfg = bfreg.with(bfreg.with(bfreg.zeroed(32), hwfc, 1), parity, 7)
    println("${cfg.word}")          // 15

    // And the same word with names on it, host-side.
    let d = bitfield.reg_desc("CONFIG", 32, 0, [
        bitfield.field_desc("HWFC", hwfc),
        bitfield.field_desc("PARITY", parity)])
    println(bitfield.format_reg(d, cfg.word))
    // CONFIG { HWFC: 1, PARITY: 7 }
```

## The load-bearing interface

`BfField`, in `bffield`:

```novo
pub @value
struct BfField
    offset: Int
    width: Int
    signed: Bool
```

Three integers in the caller's frame, no header and no allocation — a
driver holds one per field as a module-level constant and never
allocates to read a register.  Everything else follows from it:

- `extract(f, word)` and `insert(f, word, v)` are **pure functions on a
  word**.  A field description knows nothing about an address, so this
  package cannot perform a memory access even by accident, and the HAL
  keeps the two lines that can.
- The word's width is the **call's** argument, not the description's, so
  one `PIN_CNF` layout serves thirty-two registers.  The four typed
  pairs — `extract_u8` … `insert_u64` — are that width made a type.
- `BfReg` is `BfField` applied: a word, its width, and `with` chained
  once per field.
- `BfFlags` is the one-bit case with the shifts already done, because a
  pin mask and a pending-interrupt word are the same shape and neither
  wants a field description per bit.

Three consequences a reviewer should push on:

- **`insert` masks; it does not refuse.** A `@value` function has
  nowhere to put a `Result`, so a value too large for its field is
  truncated. `bffield.fits` is the check, `bfreg.accepts` is the check
  for the pair, and `bitfield.write_all` is the call that makes it on
  the caller's behalf and answers `BfValueOutOfRange`. The device path
  is fast and unchecked on purpose; the host path checks.
- **The register description holds no list on the device side.** The
  plan's row asks for "a register description as a list of fields", and
  a `@value` struct cannot own one — no type parameter, no
  fixed-capacity buffer, and an owned list copied whole on every
  assignment (SPEC § 14.3). So the list is `bitfield.BfRegDesc`,
  host-side, where it can be printed and validated, and `bfreg` is the
  word plus a chain of `with` calls. That chain is the sequence a driver
  would have written anyway, with the shifts named.
- **`BfFieldDesc` spells the three numbers out** rather than holding a
  `BfField`, because it owns a `Str` and a list and is therefore boxed,
  and a boxed aggregate's slots are pointer-shaped. `desc_field` is the
  one call that crosses back.

## The layer, and why

`core`. Shifts and masks over words the caller already holds, and no
function declares an effect.

It carries `tests/embedded_probe.nv`, so the device claim is **built**
rather than asserted: `bfflags`, `bffield` and `bfreg` speak `Int` and
the four unsigned widths and nothing else, and the probe compiles to a
Cortex-M4 ELF for `--target=nrf52-qemu`, driving the flag algebra, all
four typed widths and a chained register write. That claim is the whole
argument for the package — a driver that had to leave the device to
describe a register would go back to writing shifts by hand.

`bitfield` is deliberately outside the probe: it speaks `Str` and lists,
and one host-only function anywhere in a compilation unit is an
undefined symbol at embedded link time whether or not the firmware calls
it.

## The reference implementation

`bitflags` (Rust, MIT/Apache-2.0) for the set algebra's names and
semantics, and `bitfield` (Rust, MIT/Apache-2.0) for the field accessor
shape. Both are macro crates that generate a type per flag set or per
register; there is no macro here and a `@value` struct takes no type
parameter, so what they generate per type this package carries once as a
value — the flag constants stay the caller's bare integers and the field
description is data.

The register layouts in the tests and in the table above are from the
**nRF52840 Product Specification** (Nordic Semiconductor) and the
**RP2040 Datasheet** (Raspberry Pi Ltd), which are the same documents
`orbit/bsp` transcribed its offsets from.

## Status

| function | implemented |
| --- | --- |
| `bfflags.empty`, `.all`, `.of`, `.bit`, `.mask`, `.width_mask` | no |
| `bfflags.insert`, `.remove`, `.toggle`, `.put` | no |
| `bfflags.contains`, `.intersects`, `.test` | no |
| `bfflags.union`, `.intersection`, `.difference`, `.symmetric_difference`, `.complement` | no |
| `bfflags.is_empty`, `.is_all`, `.count`, `.lowest`, `.highest` | no |
| `bffield.field`, `.signed_field`, `.mask`, `.end_bit` | no |
| `bffield.field_max`, `.field_min`, `.fits`, `.fits_in` | no |
| `bffield.extract`, `.insert`, `.clear` | no |
| `bffield.extract_u8` … `.insert_u64` (four pairs) | no |
| `bfreg.of`, `.zeroed`, `.with`, `.get`, `.without` | no |
| `bfreg.set_mask`, `.clear_mask`, `.accepts`, `.residue` | no |
| `bitfield.value_name`, `.field_desc`, `.enum_desc`, `.desc_field`, `.reg_desc`, `.flag_names` | no |
| `bitfield.validate`, `.covered_mask` | no |
| `bitfield.read_all`, `.write_all`, `.read_named`, `.write_named`, `.is_reset` | no |
| `bitfield.format_reg`, `.format_field`, `.format_changes` | no |
| `bitfield.format_flags`, `.parse_flags`, `.flag_name` | no |
| `bitfield.BfError.message` | no |
