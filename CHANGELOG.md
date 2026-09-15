# Changelog

All notable changes to pid-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
