# pid-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

A PID controller as a `@value` state you step with a measurement and a
`dt` you supply — with the parts that separate a control library from
the three-line formula: output limits with two anti-windup policies,
derivative on the measurement rather than the error, a derivative
filter, bumpless transfer on a gain change and on manual-to-automatic,
and a `dt` ceiling for a missed deadline.

Two forms, because the two devices are different machines: `pid` in
`Float` for a core with an FPU, and `pidfx` in fixedpoint-nv's
`FxQ16_16` for one without.  A relay auto-tuner sits beside them.

```
novo pkg add pid-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use pid

// The host reads the sensor and drives the actuator; this package does
// the arithmetic.  `dt` is the elapsed seconds the caller measured.
fn tick(p: PidF, measurement: Float, dt: Float) -> PidF
    pid.step(p, measurement, dt)
```

and around it:

```novo
let p = pid.step(p, measurement, dt)
actuator.drive(pid.output(p))
log.row(pid.error(p), pid.p_term(p), pid.i_term(p), pid.d_term(p), pid.is_saturated(p))
```

## The load-bearing interface: the state carries its own output

`PidF` is a `@value` struct, and SPEC § 14.5 keeps an unboxed struct out
of a `Result`, an optional and a tuple.  So `step` cannot answer
`(PidF, Float)` — and there is no version of this package in which it
could.

What replaces it is better than what it could not have.  `step` answers
**one** value, and every question about the step is a reader on it:
`output`, `error`, `p_term`, `i_term`, `d_term`, `integral`,
`is_saturated`, `is_primed`.  Those are exactly the row a person tuning
a loop wants in their log — is it saturated, is the integral winding,
which term is doing the work — and a `-> Float` controller throws all of
it away.

The same constraint runs through the package.  `pidtune.gains_from`
cannot answer `Result<PidGains, _>` either, so it answers **zero gains**
on a failed experiment or an unknown rule — a controller that does
nothing, the only safe answer a `@value` function can give — and
`check_rule` and `check_experiment` are the refusals.  A caller that
ignored both ends up with a loop that sits still rather than one that
slams an actuator with numbers derived from noise.

### The policies are `Int` fields with an enum vocabulary around them

A `@value` struct cannot hold an enum **at all**, even a payload-free
one: an enum is boxed, and the compiler refuses the field (E2011) with
the hint to spell the choice as a primitive discriminant.  So:

- `PidAntiWindup` and `PidDerivative` are what a caller reads and
  writes, and `with_windup(p, PidWindupClamp)` is the ordinary call;
- `PID_WINDUP_*` and `PID_DERIV_*` are what the struct stores, and
  `with_windup_code` is the device path — a policy that came out of a
  configuration word in flash, where there is no enum to build;
- an unknown code leaves the field alone rather than trapping, and
  `check` is what reports it.

The two forms share the discriminants, so a policy travels between the
`Float` controller and the fixed-point one unchanged.

## The clock is the caller's

`step(p, measurement, dt)` takes the elapsed seconds.  That is what
keeps this package `core` — nothing here reads a clock, a sensor or an
actuator — and it buys the thing a control library most needs: **a
replay of a measurement sequence with its `dt` sequence gives the same
outputs on a laptop as on the device.**  A loop that misbehaved in the
field can be reproduced exactly from a log rather than guessed at.

Three `dt` cases, all defined rather than refused, because a `@value`
function has no refusal to return and the safe behaviour is the only
one worth having:

| `dt` | what happens | why |
| --- | --- | --- |
| zero or negative | **holds**, as `hold` does | dividing by it is an infinite derivative straight into an actuator |
| above `dt_max` | **clamped** to `dt_max` | a task preempted for five seconds would otherwise wind up on a gap nobody observed |
| anything else | integrated as given | |

## `hold`: the function a real sensor needs

A duty-cycle program whose sensor sometimes returns nothing — a
peripheral that did not answer, an I²C NAK, a reading the range check
rejected — **must not** step with a stale measurement.  The derivative
would see no change where there was one, and the integral would keep
integrating an error nobody measured.

`hold(p)` is the honest alternative, and un-priming is the important
half: the next real sample starts the derivative afresh rather than
differencing across the gap.  The integral is left **alone** rather
than reset, because a hold is a missing measurement and not a new
experiment — a heater that was 60 % on before the sensor blinked should
still be 60 % on after.

## What sensorhub would take

`orbit/sensorhub` reads the nRF52's on-die temperature every two
seconds (`DUTY_CYCLE_MS = 2000`) in units of a quarter degree, and on a
board with no TEMP peripheral it returns `NO_READING` rather than
hanging the duty cycle.  It has nothing to close a loop with.

What it would take is `pidfx`, and specifically:

- **`controller_raw`**, because its gains would live in flash as words
  — `131072` for a `kp` of 2 — and there is no float anywhere in that
  program;
- **`step_raw`** on the duty cycle, and **`hold`** on the
  `NO_READING` path, which is exactly the shape its sensor already has;
- **`with_max_dt(4.0)`**, because its duty cycle is two seconds and a
  missed one should not integrate as four;
- **`output_raw`** into a PWM register, because a duty cycle is already
  an integer.

That is the whole integration: six calls, no allocation, and a state in
`.bss`.  `tests/embedded_probe.nv` is those six calls and the `Float`
form's equivalents, built for a Cortex-M4.

## Two forms, and the device claim covers both

| module | number type | for |
| --- | --- | --- |
| `pid` | `Float` | a Cortex-M4F, a Linux service, a simulation |
| `pidfx` | `FxQ16_16` | a Cortex-M0+, where a `Float` multiply is a call into a software routine |

The two are the **same surface**, function for function, with `Float`
replaced by `FxQ16_16` — so a caller porting a loop from a service to a
microcontroller changes the module name and the number type and nothing
else, and a reviewer comparing the two files sees only those two
substitutions.

Why two rather than one generic controller: a type cannot carry a
numeric parameter here, and there is no trait a `@value` arithmetic type
can implement that a `@value` struct can be generic over.  Written out
by hand is the honest alternative, and it is the same choice
fixedpoint-nv made for its four Q formats.

**`pidfx` uses fixedpoint-nv's saturating family everywhere.**  A
control loop that *trapped* would take the firmware down on a transient;
one that *wrapped* would drive an actuator to its opposite extreme,
which is worse.  Saturating is the only one of the three whose failure
mode is "the output sat at its limit" — a thing an operator can see.

Q16.16's range is -32768 to 32767.99998, so gains have to fit, and
`pidfx.check` reports one that saturated on the way in:
`fxq16_16.from_float(40000.0)` clamps and says nothing, which is right
for a `@value` constructor and the wrong thing to discover from a loop's
behaviour.

`tests/embedded_probe.nv` builds **both** forms for `nrf52-qemu`, so the
claim is built rather than asserted.

## The tuner

`pidtune` is the Åström–Hägglund relay experiment: drive the actuator
`±amplitude` around the setpoint, watch the plant oscillate, and read
the ultimate gain and period off the oscillation.  Five published
tuning rules turn those two numbers into gains, and
`pid.with_gains` applies them **without moving the output**, which is
why a tuner can hand its answer to a running loop.

It is a separate module with its own state, because a tuner is not a
controller: it drives a square wave, it tracks no setpoint, and a
program running one has taken the loop out of service.

Three things worth knowing before you run one:

- **The hysteresis is not optional on a real sensor.**  A relay with no
  hysteresis switches on every noise sample that crosses the setpoint,
  which is a chattering actuator and a period measurement of nonsense.
  It costs accuracy — the estimated `Ku` comes out low — and that is
  the trade the method is built on.
- **`PID_RULE_TYREUS_LUYBEN` is the better default**, not
  Ziegler–Nichols.  ZN is aggressive by design — a quarter-decay ratio
  means visible overshoot and two or three oscillations — and it is the
  wrong choice for anything a person watches or anything overshoot can
  damage.
- **When to stop is the caller's decision.**  `relay_cycles` and
  `relay_settled` are the facts; "four cycles whose periods agree
  within five per cent" is a judgement about a particular plant, and a
  library that decided for you would be wrong on the plant you have.

## The layer, and why

`core` — no effects.  The measurement arrives as a number, the elapsed
time arrives as a number, and the output leaves as a number.  Nothing
here reads a sensor, drives an actuator, consults a clock or spawns a
task, and everything it does not do belongs to the program that runs it.

One dependency, `fixedpoint-nv`, for `FxQ16_16`.  Taking its type rather
than carrying raw `Int`s is the whole point of the second form: a
controller written in raw Q16.16 integers is one where the shift is
somebody's to remember, and the class of bug that produces — a multiply
that forgot to shift back down, a gain 65536 times too large — is
exactly what a fixed-point type exists to stop.  It is `core` and makes
its own device claim, so `dep-layer` and `core-embedded` both hold.

## The reference implementation

Rust's `pid` crate for the surface — its anti-windup and its
derivative-on-measurement are the shape this follows — and Brett
Beauregard's *Improving the Beginner's PID* series for the four
problems (sample time, derivative kick, windup, bumpless transfer) that
a naive implementation has and this one answers by name.
Åström and Hägglund 1984 for the relay method; Ziegler and Nichols
1942, Tyreus and Luyben 1992 and Pessen 1954 for the tuning tables,
which are in `pidtune`'s doc comments so a reader can check the
arithmetic.

Deliberately outside: cascade control, feedforward, gain scheduling,
and any notion of a loop "manager" that owns several controllers.  Each
is a program's structure rather than a controller's, and each is
composed from what is here.

## Status

Every function is `todo()`.  `novo test --isolate` runs the API suite;
every assertion against a function reaches
`not implemented: pid-nv.<module>.<fn>`, and the few that pass are
assertions about published constants, which are not stubs.

| module | public types | public items | implemented |
| --- | --- | --- | --- |
| `pid` | `PidF`, `PidGains`, `PidLimits`, `PidAntiWindup`, `PidDerivative` | 5 consts, 43 fns | no |
| `pidfx` | `PidFx`, `PidFxGains`, `PidFxLimits` | 42 fns | no |
| `pidtune` | `PidRelay`, `PidUltimate`, `PidTuneRule` | 5 consts, 18 fns | no |
| `piderr` | `PidError` (+ `impl Error`) | 1 fn | no |

Twelve public types, 104 public functions, 10 public constants and one
trait impl.

## Licence

Apache-2.0.
