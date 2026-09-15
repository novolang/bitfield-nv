# bitfield-nv

A hardware register is a word whose bits are divided into named **fields**:
a one-bit enable here, a three-bit prescaler there, a five-bit pin number
after that. This package describes a register's layout once, as ordinary
values, and then reads and writes the word by that description. It is the
novo-lang counterpart of the Rust crates
[bitflags](https://docs.rs/bitflags) and [bitfield](https://docs.rs/bitfield),
which do the same job with macros.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What it is

Three shapes cover almost every register a datasheet prints, and this package
has one type for each.

A **flag set** is a word with one independent bit per thing: one bit per
general-purpose pin, one bit per pending interrupt source, one bit per
peripheral held in reset. The operations on it are set operations. `BfFlags`
is a flag set that also carries how wide its word is.

A **field** is a run of adjacent bits holding one number. It is described by
three things: the **offset**, which is the index of its lowest bit; the
**width**, which is how many bits it has; and whether it is **signed**, that
is, whether its top bit means the value is negative. `BfField` is those three
numbers. Reading a field out of a word is a shift and a mask. Writing one is
a shift, a mask, a clear and an or, which is the sequence called a
**read-modify-write**.

A **register** is a word and the fields in it. `BfReg` is the word and its
width, and `with` applies one field write to it, so the whole configuration
word is one chain of calls.

The nRF52840's UART `CONFIG` register is an ordinary example. It is one
32-bit word holding `HWFC` at bit 0 with a width of 1, `PARITY` at bit 1 with
a width of 3, `STOP` at bit 4 with a width of 1 and `PARITYTYPE` at bit 8
with a width of 1. A driver that writes the literal 15 into it has written
all four, and the literal says none of that.

The fourth type is the **description**: a field with a label, a register with
a list of labelled fields, and, where the datasheet gives one, an
**enumeration**, which is a list of names for particular values of a field.
A description is what lets a word be printed as `CONFIG { HWFC: 1, PARITY: 7 }`
and lets a test check the layout before any hardware sees it.

**Nothing in this package touches memory.** A register's value arrives as an
ordinary integer the caller already read, and leaves as one the caller will
write. The load and the store stay in the hardware abstraction layer.

| Quantity | Value |
| --- | --- |
| Fields in `BfField` | 2 integers and 1 boolean |
| Fields in `BfFlags`, `BfReg` | 2 integers each |
| Word widths a description may use | 1 to 64 bits |
| Typed word widths with their own functions | 8, 16, 32, 64 |
| Value of `lowest` or `highest` on an empty set | -1 |

## Install

```
novo pkg add bitfield-nv
```

## Example

```novo
use bffield
use bfreg
use bitfield

fn main() [io]
    // Two fields of the nRF52840 UART CONFIG register, each an offset and a width.
    let hwfc = bffield.field(0, 1)
    let parity = bffield.field(1, 3)

    // The read-modify-write, as arithmetic on a 32-bit word the driver read.
    let cfg = bfreg.with(bfreg.with(bfreg.zeroed(32), hwfc, 1), parity, 7)

    // `word` is already masked to the register's width, so it can be written back.
    println("${cfg.word}")

    // The same two fields with names on them, for a host that has to print the word.
    let d = bitfield.reg_desc("CONFIG", 32, 0, [
        bitfield.field_desc("HWFC", hwfc),
        bitfield.field_desc("PARITY", parity)])

    // Refuses a description whose fields overlap or fall outside the register.
    match bitfield.validate(d)
        Ok(covered) => println(bitfield.format_reg(d, cfg.word))
        Err(e)      => println(e.message())
```

Build and test with `novo pkg build` and
`novo test --isolate tests/bitfield_tests.nv`. Today `novo test` fails on
purpose: every test reaches a `not implemented: <module>.<fn>` panic. The
tests are the specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `bfflags` | A set of bits over a word of a known width, with the set operations: insert, remove, toggle, contains, intersects, union, intersection, difference, symmetric difference and complement, plus the counts and the lowest and highest set bit. |
| `bffield` | A field as an offset, a width and a signedness, with `extract` and `insert` over a plain `Int` and over each of `u8`, `u16`, `u32` and `u64`, and the range checks. |
| `bfreg` | A register's word and its width, with one field write per call, the two whole-mask operations, and the check that a write will fit. |
| `bitfield` | The names: a label per field, an enumeration per value, a register as a list of labelled fields, the check that a description is consistent, and the printing and parsing a person reads. |

## How to choose an entry point

**A driver on a device uses `bfflags`, `bffield` and `bfreg`.** These three
speak integers and nothing else. They allocate nothing, and a field
description held as a module-level constant costs three numbers.

**A host tool uses `bitfield` as well.** A register-map generator, a debugger
decoding a word read over a debug port, a start-up log and a test asserting a
layout against a datasheet all need the labels, and labels are strings.

**Use `bfflags` when every bit is independent, and `bfreg` when bits are
grouped into numbers.** A pin-output word is a flag set. A pin-configuration
word is a register. Both are 32 bits and neither is the other.

**The typed functions `extract_u8` through `insert_u64` are the same
arithmetic with the word's width in the type.** Use them where the value came
from or is going to a sized location. Use `bffield.extract` and
`bffield.insert` where it is already an `Int`.

## The rules a user needs

1. **The width belongs to the value, not to `Int`.** novo-lang's `Int` is 64
   bits and a status word is 8, 16 or 32. The complement of the empty set
   over an 8-bit word is 255, and over an `Int` it would be -1. Every
   constructor and every operation masks to the declared width, so `bits` and
   `word` can be read straight out and handed to a hardware write.
2. **A signed field sign-extends when it is read.** A six-bit trim field
   holding `0b111111` reads as -1. Declare it with `signed_field`. A driver
   that declares it with `field` reads 63 and nothing reports the mistake.
3. **`insert` masks the value; it does not refuse it.** A value too large for
   its field is truncated to the field's width. `bffield.fits` asks whether a
   value is in range, `bfreg.accepts` asks that and whether the field is
   inside the register, and `bitfield.write_all` and `write_named` make the
   check for you and answer `BfValueOutOfRange`. The device path is unchecked
   and the host path is checked.
4. **A field that does not lie inside the word reads zero and writes
   nothing.** `bffield.fits_in` is how a caller asks in advance. There is no
   panic and no error value, because a `@value` struct may not be a `Result`
   payload (SPEC section 14.5).
5. **The word's width is the call's argument, not the field's.** One
   `BfField` therefore describes the same layout in thirty-two different
   registers, which is what a per-pin configuration array needs.
6. **`bitfield.validate` is a separate call, and a description is not checked
   until it is made.** It refuses a description whose two fields claim the
   same bit, and one whose field runs past the register's width, naming the
   fields and the bit. It answers how many bits the description covers.
   Running it once in a test is what catches an off-by-one offset before a
   peripheral misbehaves.
7. **`bfreg.residue` is the bits no field explains.** Give it
   `bitfield.covered_mask` and it answers what is left. A non-zero residue in
   a word read from hardware means the description is incomplete.
8. **`bitfield.parse_flags` answers the bits as an `Int`, not a `BfFlags`.**
   A `@value` struct may not be a `Result` payload, so the fallible direction
   comes back as the integer, and `bfflags.of(t.width, bits)` makes it a set
   again.
9. **A set bit with no name prints as a hex mask, and an empty set prints as
   `(none)`.** Nothing is dropped from a printed flag word, because an
   undocumented bit being set is the case somebody is reading the output for.
10. **`bfflags.lowest` and `.highest` answer -1 for an empty set.** An
    interrupt dispatcher loops on `lowest` and stops on -1.

## Running on a microcontroller

The package states that its modules run on a device with no heap allocator,
and the compiler checks that claim on every build. Here it covers `bfflags`,
`bffield` and `bfreg`, which speak `Int` and the four unsigned widths and
nothing else.

`tests/embedded_probe.nv` is that claim as a program that either builds or
does not. It drives the flag algebra, all four typed widths and a chained
register write.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

That command was run against this release. It produces a Cortex-M4
executable, `embedded_probe.elf`. The probe builds; it is not run, because
every function it calls is a `todo()` that would panic on the first line.

The `bitfield` module is outside the claim. It speaks `Str` and lists, and
one host-only function anywhere in a compilation unit is an undefined symbol
at link time on a device, whether or not the firmware calls it.

## What is not included

- **Any memory access.** No function here reads or writes an address. A
  register's value arrives as an integer and leaves as one, which is what
  keeps a driver's load and store in the one place allowed to perform them.
- **A generated type per register.** The two Rust crates this follows are
  macro crates that emit a struct per flag set and per register. A `@value`
  struct takes no type parameter, so what they generate per type this package
  carries once, as data: the flag constants stay the caller's integers and
  the field description is a value.
- **A list of fields on the device side.** A `@value` struct cannot own a
  list, and an owned list would be copied whole on every assignment
  (SPEC section 14.3). The list lives in `bitfield.BfRegDesc`, on the host,
  where it can be printed and checked. On the device a register is the word
  and a chain of `with` calls.
- **A reset value on `BfReg`.** It is on `BfRegDesc`, beside the names, which
  is where `is_reset` and `format_changes` need it.
- **The rotate and population-count primitives.** `std.bits` has
  `rotate_left`, `rotate_right`, `count_ones` and the two zero counts, and
  this package calls them rather than spelling them again.

## Related packages

- `std.bits` in the standard library is the operators and the single-bit
  functions over a bare `Int`: `&`, `|`, `^`, `~`, `<<`, `>>`, `>>>`, and
  `get`, `set`, `clear`, `flip`, the two rotates, `count_ones`, the zero
  counts and `bit_len`. Everything here lowers to those. What this package
  adds is a word that knows its own width, a field wider than one bit, sign
  extension, a description that can be checked, and names.
- [heapless-nv](https://novo-lang.org/packages/heapless-nv) is the
  fixed-capacity containers for the same device. A driver that needs to hold
  several register values without an allocator uses its bounded vector.
- [can-nv](https://novo-lang.org/packages/can-nv),
  [usb-nv](https://novo-lang.org/packages/usb-nv) and
  [modbus-nv](https://novo-lang.org/packages/modbus-nv) are peripheral and
  protocol packages in the same category. Each packs and unpacks bit fields
  of its own, defined by its protocol rather than by a datasheet.

## Tests

```bash
novo test --isolate tests/bitfield_tests.nv   # 38 tests
```

The layouts in the tests are from the nRF52840 Product Specification
(Nordic Semiconductor) and the RP2040 Datasheet (Raspberry Pi Ltd). The set
algebra's names and meanings follow `bitflags`, and the field accessor shape
follows `bitfield`.

The suite asserts that the complement of an empty 8-bit set is 255, that a
word wider than the set is masked on the way in, that `contains` means all of
a mask and `intersects` means any of it, that a dispatcher loop takes the
lowest pending source, that `extract` and `insert` are each other's inverse,
that a signed field sign-extends its top bit, that a field outside the word
reads zero and writes nothing, that the four typed widths compute what the
generic form does, that chaining `with` builds the configuration word, that
`residue` is the bits no field explains, that two fields on one bit are
refused by name, that a field past the register's end is refused by name,
that `write_all` checks the range where `bfreg.with` masks, that a register
prints as its fields, that an enumerated field prints the name and the
number, that `format_changes` prints only what left reset, that a bit the
table does not name prints as a hex mask, and that printing and parsing a
flag word round-trip.

The tests compile today and fail at run, each on the
`not implemented: <module>.<fn>` panic that is its body. That is the expected
state of an interface release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `bfflags.BfFlags`, `bffield.BfField`, `bfreg.BfReg` | declared |
| `bitfield.BfValueName`, `.BfFieldDesc`, `.BfRegDesc`, `.BfFlagNames`, `.BfError` | declared |
| `bfflags.empty`, `.all`, `.of`, `.bit`, `.mask`, `.width_mask` | no |
| `bfflags.insert`, `.remove`, `.toggle`, `.put` | no |
| `bfflags.contains`, `.intersects`, `.test` | no |
| `bfflags.union`, `.intersection`, `.difference`, `.symmetric_difference`, `.complement` | no |
| `bfflags.is_empty`, `.is_all`, `.count`, `.lowest`, `.highest` | no |
| `bffield.field`, `.signed_field`, `.mask`, `.end_bit` | no |
| `bffield.field_max`, `.field_min`, `.fits`, `.fits_in` | no |
| `bffield.extract`, `.insert`, `.clear` | no |
| `bffield.extract_u8` to `.insert_u64`, four pairs | no |
| `bfreg.of`, `.zeroed`, `.with`, `.get`, `.without` | no |
| `bfreg.set_mask`, `.clear_mask`, `.accepts`, `.residue` | no |
| `bitfield.value_name`, `.field_desc`, `.enum_desc`, `.desc_field`, `.reg_desc`, `.flag_names` | no |
| `bitfield.validate`, `.covered_mask` | no |
| `bitfield.read_all`, `.write_all`, `.read_named`, `.write_named`, `.is_reset` | no |
| `bitfield.format_reg`, `.format_field`, `.format_changes` | no |
| `bitfield.format_flags`, `.parse_flags`, `.flag_name` | no |
| `bitfield.BfError.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
