# boot_ladder

[![check](https://github.com/nifeos/boot_ladder/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/nifeos/boot_ladder/actions/workflows/ci.yml?query=branch%3Amain)
[![proofs](https://github.com/nifeos/boot_ladder/actions/workflows/proofs.yml/badge.svg?branch=main)](https://github.com/nifeos/boot_ladder/actions/workflows/proofs.yml?query=branch%3Amain)
[![license](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue.svg)](https://github.com/nifeos/boot_ladder#license)
[![minimum Rust version](https://img.shields.io/badge/rust-1.88%2B-blue.svg)](https://github.com/nifeos/boot_ladder/blob/main/Cargo.toml)

The fixed heads of the lines a kernel and its shell print while booting, and the list of self-test
check names. A program that reads a serial log, a CI transcript or a photograph of a screen then
keys on the same bytes the kernel wrote. `no_std`, no dependencies, no `unsafe`.

Each head is a system's name followed by a fixed shape, such as `<system> on ` for the first line
and `<system> self-test: ` for the verdict. The crate names no system: the caller passes its own
name to `Heads::new`, and the program that prints a line and every program that reads it take the
head from the same `Heads`, so both sides hold the same bytes. A head prints through `Display` and
matches a line in place, so neither side allocates.

## EXAMPLES

Finding the verdict in a transcript, and deciding whether it is red:

```rust
use boot_ladder::{Heads, verdict::ParsedVerdict};

let heads = Heads::new("example");
let line = "\u{fffd} example self-test: 12 of 15 passed, 3 absent: clock, 1 FAILED: entropy";

// The head is found after the noise, and the tail is read through the shared grammar.
assert!(heads.self_test().is_in(line));
let verdict = ParsedVerdict::in_line(heads, line).unwrap();
assert_eq!((verdict.passed, verdict.total), (12, 15));
assert!(verdict.is_red());
assert_eq!(verdict.failed_names().collect::<Vec<_>>(), ["entropy"]);
```

Printing a head:

```rust
use boot_ladder::Heads;

assert_eq!(Heads::new("example").banner().to_string(), "example on ");
```

## What is proven

Two Kani harnesses show that a count never overflows and that a name list holds only plain names,
whatever ASCII text arrives up to their bounds. A replayable patch in `falsifications/` turns each
red. The tests cover the round trip of
the verdict, the invariants of the check list, and the shape of every head.

## The matching rule

A head begins a line, but a serial link may put corrupted bytes in front of it. A reader therefore
finds the head anywhere in a line (`Head::find_in`) and takes the tail from there. Every head but
the shell banner ends in a space or punctuation, so a match is on a boundary. The shell banner
(`Heads::shell_banner`) is matched whole: the shell prints a period after it, and the period is not
part of the head. The architecture after the first line is free text; a reader takes it from the
tail of the machine line (`Heads::machine`).

The tail of a head may change between releases, because it carries numbers. A head never changes.

## The verdict

The self-test prints one verdict line. Its grammar is code, not prose: `verdict::Verdict` prints it
and `verdict::ParsedVerdict` reads it, so the printer and every reader agree by construction.

```text
example self-test: 12 of 15 passed, 3 absent: block_device clock true_random_number_generator
example self-test: 13 of 15 passed, 2 FAILED: userspace entropy
```

A verdict is green unless it carries a `FAILED:` list. An `absent:` list names checks the machine
has no device for; whether an absence is acceptable is the reader's board profile to say. The
total is `SELF_TEST_CHECKS.len()` for a kernel built from the same version, and one of
`EARLIER_SELF_TEST_CHECK_COUNTS` for an older one.

## Version policy

The Cargo version is the contract's version, and the crate carries no separate identity constant:
a console line cannot hold a checkable version without a new line to freeze. A change to a head or
to a check name bumps the minor version and adds a compatibility entry: the earlier-counts list
for a changed total, and a list of renamed checks if a name is ever renamed. A reader built
against one version reads a log from an older kernel by those entries and no other.

## Building and testing

```sh
cargo test
rustup target add x86_64-unknown-none aarch64-unknown-none-softfloat riscv64imac-unknown-none-elf
cargo build --lib --target x86_64-unknown-none
cargo build --lib --target aarch64-unknown-none-softfloat
cargo build --lib --target riscv64imac-unknown-none-elf
```

The crate has no features and no dependencies, so a plain `cargo test` runs every unit test and
doctest, the examples above included.

### Running the proofs

Install Kani first: run `cargo install --locked kani-verifier --version 0.68.0`, then
`cargo kani setup`. Setup downloads a model checker and the Rust toolchain that Kani pins, so it
needs network access, and `cargo install` compiles Kani from source. Then run every harness:

```sh
cargo kani
```

The two harnesses take five and a half to eight minutes, depending on the host's load, and peak
near 3.2 gigabytes of memory, as `/usr/bin/time -v cargo kani` measured on a shared 80-core x86_64
host. Almost all of the time is
`a_segment_is_read_without_panicking`; `a_count_is_read_without_overflow` takes seconds. To run one
harness, give its full path, because `--exact` refuses a bare name:

```sh
cargo kani --harness verdict::verification::a_count_is_read_without_overflow --exact
```

A passing run prints lines that look alarming and mean nothing. They are the `use of an unstable
feature` warning, the `Found the following unsupported constructs` warning, and many `aborting
path on assume(false)` lines, which are paths Kani discards on purpose. The answer is the last
line, `Complete - 2 successfully verified harnesses, 0 failures, 2 total.`, and `cargo kani` exits
0.

### Running one falsification patch

Each file in `falsifications/` applies a small defect that turns its harness red. Apply one,
confirm the proof fails, then undo it with the same patch reversed:

```sh
patch -p1 --no-backup-if-mismatch < falsifications/verdict.verification.a_count_is_read_without_overflow.patch
cargo kani --harness verdict::verification::a_count_is_read_without_overflow --exact
patch -R -p1 --no-backup-if-mismatch < falsifications/verdict.verification.a_count_is_read_without_overflow.patch
```

The failing run prints `Failed Checks: attempt to multiply with overflow`, then
`Failed Checks: attempt to add with overflow`, and `VERIFICATION:- FAILED`. It ends with `Complete - 0 successfully verified harnesses, 1 failures, 1
total.`, and `cargo kani` exits 1. Each patch's opening lines name the check it expects to fail.

## BUGS

- Nothing gates that a program uses `Heads` rather than a head spelled as a literal; review does.
  After the crate leaves its first consumer's repository, only that consumer's own lint can.
- `Heads::shell_banner` is the shell's banner, not its `$ ` prompt, which is too weak to key on in a
  log. A reader that needs the prompt waits for the banner and then looks for `$ `.
- Nothing checks that every program of one system builds its `Heads` from the same name; each
  passes its own, and a reader of that system's logs finds a mismatch as a missing rung.
- The kernel pads a check's name to ten characters in its per-check lines, and the longest names no
  longer line up in that column.

## Minimum Rust version

Rust 1.88. To check it, install the toolchain and its bare-metal targets, then run the tests and
all three bare-metal builds with `+1.88`:

```sh
rustup toolchain install 1.88 --profile minimal
rustup target add --toolchain 1.88 x86_64-unknown-none aarch64-unknown-none-softfloat riscv64imac-unknown-none-elf
cargo +1.88 test
cargo +1.88 build --lib --target x86_64-unknown-none
cargo +1.88 build --lib --target aarch64-unknown-none-softfloat
cargo +1.88 build --lib --target riscv64imac-unknown-none-elf
```

## License

Licensed under either of MIT (LICENSE-MIT) or Apache-2.0 (LICENSE-APACHE), at your option.
