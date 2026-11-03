# Changelog

All notable changes to pid-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-13

**Implemented.** Every body under `src/` is written and the suites pass: 52
tests over five files, 116 documentation examples, and the device probe. The
interface published in 0.0.1 is unchanged — no signature, no type and no
effect row moved — and the `NOT IMPLEMENTED — interface only` stamp, its
README section and the implementation-status table are gone. 106 functions
and 4 private helpers carry real bodies; the manifest moves to
`stability = "experimental"`, `version = "0.1.0"` and `novo = ">= 0.20.2"`.

### Workarounds recorded here

- **`fixedpoint-nv` is wired by `path` in `novo.toml`, and its range moved
  from `^0.0.1` to `^0.1.0`.** The two halves of one edit: a `0.0.x` range
  admits exactly that one release, and the registry's 0.0.1 is an interface
  release whose every call panics, while the 0.1.0 this package needs is
  landed in this checkout and not on the registry. The `path` line is
  development wiring and **must be removed before a publish**:
  `novo pkg publish` refuses a path dependency, so `novo pkg publish
  --dry-run` refuses at that line and its shard rows go unmeasured here —
  that is the refusal, not a failing check, and `../PATH-DEPENDENCIES.md` is
  the standing note on it. `novo.lock` follows the manifest and now records
  `source = "path"`.

### Added

- **`pid`** — the `Float` loop: the six steps of a step (elapsed time, the
  error and its proportional term, the derivative on measurement or on error
  with a first-order filter, the integral, the anti-windup policy, the
  output clamp and the terms behind it), `hold`, `reset`, both bumpless
  transfers, the two policies as `Int` discriminants, and `check`.
- **`pidfx`** — the same loop in `FxQ16_16`, saturating at every operation,
  with the `_raw` entry points a firmware drives it through.
- **`pidtune`** — the relay experiment's cycle detection and the five
  published tuning rules.
- **`piderr`** — the nine-message `Error` implementation.
- **Two private helpers in `pid` and two in `pidfx`.** `clamp_output` is the
  one place each module's clamp is written down, so `step` and
  `with_manual_output` cannot disagree about it; `finite` in `pid` and
  `at_limit` in `pidfx` are what the two `check`s decide "a number at all"
  with; and `divide` in `pidfx` is the module's whole arithmetic trap, which
  the design note below is about.

### Design notes

**`fxq16_16.sat_div` cannot divide by a `dt`, and `pidfx.step` does not use
it.** `sat_div` guards the DIVISOR rather than the quotient: it answers
`max_value()` whenever the divisor's raw word is below 65536, which is every
`dt` a control loop uses — a 10 ms sample is 655 in Q16.16, so a saturating
divide by it would pin the derivative at the top of the range on every step.
`divide` in `pidfx.nv` calls `fxq16_16.from_ratio(raw(a), raw(b))` instead,
which divides the two raw words and clamps the quotient, which is the same
no-trap policy applied where the overflow can happen. `fxq16_16.div` would
be right and traps, and rule 8 of the README forbids a trapping operation
here.

**An unrecognized policy code behaves as clamping.** The code is stored as
given, so `with_windup_code(p, 9)` leaves a field that `check` reports as
`PidUnknownPolicyCode`; `windup_of` answers the policy IN FORCE, which is
clamping, and `step` asks `windup_of`. A code nobody meant must not switch
anti-windup off. This is the reading the 0.0.1 suite's two assertions
require — `windup_of` on the 9-code is `PID_WINDUP_CLAMP` while `check` of
the same controller is `Some` — and one sentence of that suite and one rule
of the README were made false by it and updated (below).

### Verification

- **The mutation battery left six survivors, and each one is now a test.**
  Fifteen behavioural mutations across the four modules: nine went red in the
  shipped suites, six stayed green and each one named a claim nothing was
  checking — equal limits (`pid`), the integral's rescale when a gain change
  moves `ki` (`pid`), the fixed-point derivative's divide by a `dt` below one
  (`pidfx`), the `dt` ceiling (`pidfx`), the integral frozen under a probe
  that saturates before its output does (`pidfx`), and a relay experiment's
  completed cycles (`pidtune`). Each survivor has a case in
  `tests/pid_edge_tests.nv`, the file added to carry them, and each mutation
  re-run under `novo test --isolate` fails there **by name**.
- **The tier annotations are load-bearing, and measured on the binary
  route.** Stripping `@tier(app)` from any of the six host-face functions is
  refused by the package's staged device build (`novo build
  --target=nrf52-qemu src/zz_probe_tmp.nv`, E4000); keeping them is what
  lets that image link. The measurement has to be that route: `novo pkg
  build --target=nrf52-qemu` runs no embedded tier check at all for a
  library, and accepts the stripped annotation.
- **Coverage is 388/388 instrumented lines (100.0%), and nothing is
  excused.** `pid` 150, `piderr` 11, `pidfx` 141, `pidtune` 86, counted as
  the intersection of the per-suite uncovered sets and measured after `novo
  fmt`. The shipped suites left 55 of those lines unrun — every one a
  `return` whose condition the oracle only ever evaluates false, or a path
  its fixtures never take — so `tests/pid_cover_tests.nv` was added to run
  them. There is no `cov: skip` marker in the package.

### Changed

- **Eleven `x == false` sites in the shipped suites are now `not x`.** The
  `redundant-bool` lint rule is right about each: five in
  `tests/pid_tests.nv`, two in `tests/pidfx_tests.nv`, two in
  `tests/pidtune_tests.nv` and two in `tests/embedded_probe.nv`, all
  confirmed pre-existing with `git show HEAD:`. No assertion changed
  meaning, and the lint pass over the ten files of the package then reports
  count 0.
- **Three test files lost their release stamp.** The `RED ON PURPOSE`
  blocks in `tests/pid_tests.nv`, `tests/pidfx_tests.nv` and
  `tests/pidtune_tests.nv` said every assertion reaches a `not implemented`
  panic; no body does. Deleted, which is what landing means.
- **One sentence in `tests/pid_tests.nv`.** The case read "an unknown code
  leaves the FIELD alone"; the field holds the code, and it is the policy in
  force that is left alone. The assertions under it are untouched, and no
  other line of any shipped suite was edited.
- **README rule 6 says the same thing**, and the sections that described the
  interface release — the status stamp, the "tests fail on purpose"
  paragraph, the implementation-status table and the "the device build
  fails" section — are replaced by what is measured now.

### Known

- **`tests/embedded_probe.nv` compares its own total against 19, and every
  one of its checks holds.** The file has twenty `n = n + 1` statements and
  a fully passing run scores 20, so the shipped firmware prints
  `FAIL: pid-embedded` while nothing is wrong with it — a false failure,
  which is the direction that costs a reader the most. Measured rather than
  counted, by building an instrumented variant of the probe and running it
  under `qemu-system-arm`: `SCORE 20`. The literal and the comment above it
  agree with each other (19) and disagree with the function they describe
  (20); it was left alone rather than edited, because the rule that protects
  a shipped suite's assertions applies, and this package does not decide it
  alone.
- **A device build from `tests/` resolves no dependency.** `novo build
  --target=nrf52-qemu tests/embedded_probe.nv` warns in its first line that
  the target file is not the package's `main`, so it loads the modules under
  `src/` and not fixedpoint-nv, and then names 205 type errors — every one
  of them a name it did not load (`FxQ16_16`, and `PidError` from a module
  the probe does not `use`). Staged into `src/`, the same file assembles six
  modules from fixedpoint-nv 0.1.0, links, and runs; the README's device
  section gives both commands, and that staged build is the one this
  release's device claim is measured with. The route is the toolchain's
  documented one — it warns before it fails — and the filing is about the
  205 errors rather than the route:
  `bugs/diagnostics/a-non-main-build-tells-a-module-to-add-the-use-line-it-already-has`
  records that they advise adding a `use` line `src/pidfx.nv` already
  carries, in files the warning has just said the build loads.
- **`pidfx.check` reports a saturated value as `PidNotFinite`.** `FxQ16_16`
  has no NaN and no infinity, so a value sitting on either end of the
  container — which is how `from_float(40000.0)` arrives — is "not finite"
  in the only sense this form has: the caller asked for a number this type
  cannot hold. The variant is shared with the `Float` form, where the
  sentence is literal.
- **Cascade control, feedforward, gain scheduling and a loop manager** are
  still outside, as 0.0.1 said.

This release requires novo 0.23.0 or later.

## 0.0.4 — 2026-10-28

The dependency requirement is raised: `fixedpoint-nv` from ^0.0.1 to ^0.0.3.  The newest version of each builds with no migration warning under the 0.22.0 compiler, and the older one drew E2050, E2052 or E2055 warnings into this package's build.  No signature and no behaviour changed.

## 0.0.3 — 2026-10-25

A file that names a sibling module's type, function or constant now has the `use` line for that module (E2033, E2003).  No signature and no behaviour changed.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `pid` — `PidF`, `PidGains` and `PidLimits`, the step, `hold`,
  `reset`, both bumpless transfers, the two anti-windup policies, the
  two derivative sources, the derivative filter, the `dt` ceiling, the
  eight readers and `check`.
- `pidfx` — the same surface in fixedpoint-nv's `FxQ16_16`, plus a raw
  integer path — `controller_raw`, `step_raw`, `output_raw` — for
  firmware with gains in flash and no float anywhere.
- `pidtune` — the Åström–Hägglund relay experiment, the five published
  tuning rules, `PidUltimate` as a value worth storing between runs,
  and the two checks that say why an experiment produced nothing.
- `piderr` — one error type for the setup, with nothing for the loop,
  because `step` cannot fail.

### Known

- **The load-bearing interface is that the state carries its own
  output.** A `@value` cannot be a tuple element, so `step` answers one
  value and every diagnostic is a reader — which is the row a person
  tuning a loop wants and which a `-> Float` controller throws away.
- **A `@value` struct cannot hold an enum at all** (E2011), so the
  policies are `Int` discriminants with `PidAntiWindup` and
  `PidDerivative` as the vocabulary around them. The two forms share
  the codes.
- **`pidtune.gains_from` answers ZERO GAINS** on a failed experiment or
  an unknown rule, because a `@value` cannot be a `Result` payload
  either — and zero gains is a controller that does nothing, which is
  the only safe answer. `check_rule` and `check_experiment` are the
  refusals.
- **The clock is the caller's**, which keeps this `core` and makes a
  field failure reproducible from a log. A non-positive `dt` holds and
  a `dt` past the ceiling is clamped.
- **`hold` is the function a real sensor needs**, and un-priming the
  derivative is the important half of it.
- **The device claim covers BOTH forms** and
  `tests/embedded_probe.nv` builds them for a Cortex-M4.
- **What sensorhub would take is six calls**, and the README names
  them.
- Cascade control, feedforward, gain scheduling and a loop manager are
  deliberately outside.

### Design notes

**The device build does not succeed, and the 0.0.1 entry above is wrong
about it.**  `tests/embedded_probe.nv` covers both forms, and
`novo build --target=nrf52-qemu tests/embedded_probe.nv` answers
seventeen E2011 errors of the form "@value struct `PidFx`: field
`integral` has type `FxQ16_16` — type `FxQ16_16` is not in the v1
@value field set".  `FxQ16_16` is a `@value` struct and `novo pkg
build` accepts every one of those fields; the device build refuses them
because it resolves no dependencies at all — it loads the local modules
under `src/` and nothing else, so fixedpoint-nv is absent.  A probe that
names `fxq16_16` directly fails earlier, with "no module named
'fxq16_16' in scope (not stdlib, not a local module, not a dep)".  The
defect is in the toolchain rather than in this package, and the README
now says the claim rests on the code.
