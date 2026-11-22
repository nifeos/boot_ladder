# The stranger cadence

The release's whole claim is that a stranger, given this repository and nothing else, can build
the crate and run its proofs from the README alone. A real agent runs that check on a
cadence, and the weekly `stranger cadence` job fails when the
last run is past 28 days.

What a run is:

1. Clone the repository fresh.
2. Follow the README and nothing else: build, test, and run the proofs.
   The README must name the build and test commands (with any feature a test target needs), the
   bare-metal build with `rustup target add`, the Kani install before the first `cargo kani`, how
   long the proofs take and how to run one harness, what a passing run prints that looks alarming,
   how to run and revert one falsification patch, and how to check the minimum Rust version.
   Each one it lacks is a finding.
3. Note every place the README lied, omitted, or assumed. Those are findings against the README,
   not against the stranger.
4. Record the date on the line below.

Last run: 2026-11-22

## What the last run did

A fresh clone and the README, on a machine that already had Rust, the three bare-metal targets,
the 1.88 toolchain and Kani 0.68.0. It followed "Building and testing", "Running the proofs" and
"Minimum Rust version" command by command.

- `cargo test` passes: 19 unit tests and 2 documentation tests. `cargo build --lib` passes for
  `x86_64-unknown-none`, `aarch64-unknown-none-softfloat` and `riscv64imac-unknown-none-elf`.
- `cargo kani` ends with `Complete - 2 successfully verified harnesses, 0 failures, 2 total.`
  after 481 seconds on a busy host. One harness by its full path takes 12 seconds.
- The README's falsification commands do what it says. With the
  `a_count_is_read_without_overflow` patch applied, the harness exits 1 and ends with
  `Complete - 0 successfully verified harnesses, 1 failures, 1 total.`, and the reversed patch
  leaves `git status` clean.
- At Rust 1.88, the stated minimum, the tests and the three bare-metal builds pass.

The run found two places where the README's description fell short of the output, and this
version corrects both. The proofs took longer than the five and a half minutes stated, and the
falsified run printed a second failed check, `attempt to add with overflow`, beside the one the
README names.

Result: pass.
