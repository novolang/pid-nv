# pid-nv

A **PID controller** holds a measured quantity at a chosen value by adjusting
something that affects it. It combines three terms: one proportional to the
present error, one proportional to the accumulated error, and one
proportional to how fast the error is changing. This package implements one
in novo-lang, in two number formats, with a relay experiment for finding the
gains beside it. The reference implementations are the Rust crate
[pid](https://docs.rs/pid) and Brett Beauregard's
[Improving the Beginner's PID](http://brettbeauregard.com/blog/2011/04/improving-the-beginners-pid-introduction/)
series. The fixed-point form is built on
[fixedpoint-nv](https://novo-lang.org/packages/fixedpoint-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What a PID controller is

The **setpoint** is the value you want. The **measurement** is what the
sensor says. The **error** is the setpoint less the measurement. The
**output** is what the controller asks the actuator for: a heater's duty
cycle, a valve's position, a motor's current.

The three terms are named for what each one is proportional to.

- The **proportional** term is the gain `kp` times the error. On its own it
  leaves a steady error, because an output of zero needs an error of zero.
- The **integral** term is `ki` times the error accumulated over time. It is
  what removes that steady error.
- The **derivative** term is `kd` times the rate of change. It damps the
  approach.

Four problems separate a working controller from the three-line formula, and
each has a name.

**Integral windup** happens when the actuator is already at its limit and the
error persists. The integral keeps growing, and when the error finally
reverses the controller takes a long time to unwind. The two cures here are
**clamping**, which stops accumulating while the output is at a limit, and
**back-calculation**, which feeds the difference between the wanted and the
delivered output back into the integral at a rate `kt`.

**Derivative kick** happens when the setpoint moves. The error jumps, so its
rate of change is momentarily enormous, and the derivative term sends a spike
to the actuator. Differentiating the **measurement** instead of the error
removes it, because the measurement did not move.

**Bumpless transfer** is changing the gains, or switching from manual to
automatic control, without the output jumping. `with_gains` and
`with_manual_output` both preserve it.

**Sample time** is the elapsed seconds since the last step. It is an argument
here, not something the controller measures.

The **relay experiment**, published by Åström and Hägglund in 1984, finds the
gains. It drives the actuator between two values around the setpoint and
watches the plant oscillate. The amplitude and the period of that oscillation
give the **ultimate gain** and the **ultimate period**, and a published table
turns those two numbers into the three gains.

| Quantity | Value |
| --- | --- |
| Terms in the controller | 3 |
| Anti-windup policies | none, clamp, back-calculation |
| Derivative sources | the error, or the measurement |
| Tuning rules | 5 |
| Range of the fixed-point form | -32768 to 32767.99998 |
| Step of the fixed-point form | 1.5e-5 |

## Install

```
novo pkg add pid-nv
```

## Example

```novo
use pid

fn main() [io]
    // Proportional, integral and derivative gains, an output range, and a
    // setpoint of sixty degrees.
    var p = pid.controller(pid.gains(2.0, 0.5, 0.1), pid.limits(0.0, 100.0), 60.0)

    // Clamp the integral while the output is at a limit, so it does not
    // wind up while the actuator has nothing left to give.
    p = pid.with_windup(p, PidWindupClamp)

    // One step. The measurement and the elapsed seconds are both the
    // caller's: nothing here reads a sensor or a clock.
    p = pid.step(p, 58.5, 0.1)

    // The state carries the answer and every term that made it, which is
    // the row a person tuning a loop wants to see.
    println("${pid.output(p)} ${pid.p_term(p)} ${pid.i_term(p)} ${pid.d_term(p)}")

    // No reading this time. `hold` keeps the integral and un-primes the
    // derivative, so the next real sample does not difference across the gap.
    p = pid.hold(p)
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails
on purpose: every test reaches a `not implemented: pid-nv.<module>.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `pid` | The controller in `Float`: the state, `step`, `hold`, the builders that change one setting at a time, and the readers that report the output and every term behind it. |
| `pidfx` | The same surface in fixed point, over `FxQ16_16`, with a second set of entry points that take and answer plain integers. |
| `pidtune` | The relay experiment, the ultimate gain and period, and the five published tuning rules. |
| `piderr` | The error type the two check functions answer with. |

## How to choose an entry point

**`pid` on a machine with a floating-point unit.** A Cortex-M4F, a Linux
service, a simulation. One instruction per multiply.

**`pidfx` on a machine without one.** A Cortex-M0+ turns a floating-point
multiply into a call to a software routine, and a loop doing four of them per
step misses its deadline. The fixed-point multiply is one multiply and one
shift.

**`pidfx`'s `_raw` functions when the numbers are already integers.**
`controller_raw`, `step_raw`, `setpoint_raw` and `output_raw` take and answer
plain integers, so a firmware whose gains live in memory as words and whose
output goes into a timer register never writes a fixed-point value in its own
source.

**The two modules are the same surface, function for function**, with `Float`
replaced by `FxQ16_16`. Porting a loop from a service to a device changes the
module name and the number type and nothing else.

**`pidtune` when the gains are unknown.** A program running the experiment
has taken the loop out of service: the tuner drives a square wave and tracks
no setpoint.

## The rules a user needs

1. **The caller supplies the elapsed time.** `step(p, measurement, dt)`.
   Nothing here reads a clock. Replaying a measurement sequence with its
   elapsed-time sequence gives the same outputs on a laptop as on the device,
   so a loop that misbehaved in the field can be reproduced from a log.
2. **Three cases of `dt` are defined rather than refused.** A `dt` of zero or
   below holds, as `hold` does, because dividing by it would send an infinite
   derivative to the actuator. A `dt` above `dt_max` is clamped to `dt_max`,
   so a task preempted for five seconds does not integrate a gap nobody
   observed. Anything else is integrated as given.
3. **Call `hold` when there is no measurement.** A sensor that did not
   answer, a bus acknowledgement that did not come, a reading the range check
   rejected. Stepping with a stale value makes the derivative see no change
   where there was one and the integral accumulate an error nobody measured.
4. **`hold` leaves the integral alone and un-primes the derivative.** A
   missing measurement is not a new experiment, so a heater that was sixty
   per cent on stays sixty per cent on. The next real sample starts the
   derivative afresh rather than differencing across the gap.
5. **The state carries the answer, and every reader asks the state.** A
   `@value` struct may not be part of a tuple or a `Result` payload
   (SPEC section 14.5), so `step` cannot answer a state and a number. It
   answers the state, and `output`, `error`, `p_term`, `i_term`, `d_term`,
   `integral`, `is_saturated` and `is_primed` read it.
6. **A policy is stored as a number and read as one of a named set.** A
   `@value` struct cannot hold an enum field at all (E2011), so `PidF` stores
   an integer. `with_windup(p, PidWindupClamp)` is the ordinary call, and
   `with_windup_code` is for a policy that came out of a configuration word.
   An unknown code leaves the field alone, and `check` is what reports it.
7. **The two forms share the policy numbers**, so a policy travels between
   the floating-point controller and the fixed-point one unchanged.
8. **The fixed-point form saturates everywhere.** A loop that trapped would
   take the firmware down on a transient. One that wrapped would drive the
   actuator to the opposite extreme, which is worse. Saturating fails as the
   output sitting at its limit, which an operator can see.
9. **A gain too large for the fixed-point range is clamped silently on the
   way in, and `pidfx.check` is what reports it.** The range is -32768 to
   32767.99998. Discovering the clamp from a loop's behaviour is the wrong
   way to find out.
10. **A failed tuning experiment answers zero gains.**
    `pidtune.gains_from` cannot answer a `Result`, for the reason in rule 5,
    so it answers a controller that does nothing, which is the only safe
    value. `check_rule` and `check_experiment` are the refusals, and a caller
    that ignores both gets a loop that sits still rather than one driven by
    numbers derived from noise.
11. **Hysteresis in the relay is not optional on a real sensor.** Without it
    the relay switches on every noise sample that crosses the setpoint, which
    chatters the actuator and measures a nonsense period. It costs accuracy:
    the estimated ultimate gain comes out low. That is the trade the method
    is built on.
12. **`PID_RULE_TYREUS_LUYBEN` is the better default, not Ziegler–Nichols.**
    Ziegler–Nichols aims for a quarter-decay ratio, which means visible
    overshoot and two or three oscillations. It is the wrong choice for
    anything a person watches or anything overshoot can damage.
13. **When to stop the experiment is yours.** `relay_cycles` and
    `relay_settled` are the facts. Whether four cycles agreeing within five
    per cent is enough is a judgement about a particular plant.
14. **`with_gains` does not move the output**, which is what lets a tuner
    hand its answer to a running loop.

## Running on a microcontroller

The package states that its modules run on a device with no heap allocator.
**That claim is not checked by a build today, and the device build fails.**

`tests/embedded_probe.nv` exists and covers both forms. Running it against
this release gives seventeen type errors, all of the same kind:

```
$ novo build --target=nrf52-qemu tests/embedded_probe.nv
./src/pidfx.nv:198:0: type error: @value struct `PidFx`: field `integral`
  has type `FxQ16_16` — type `FxQ16_16` is not in the v1 @value field set
  (primitives, nested @value structs, fixed-size [T; N] of those) —
  keep the struct boxed [E2011]
```

`FxQ16_16` *is* a `@value` struct, and `novo pkg build` accepts every one of
those fields. The device build refuses them because it does not resolve the
package's dependencies at all: it loads the local modules under `src/` and
nothing else, so fixedpoint-nv is absent and its type is unknown. A probe
that names a dependency's type directly fails earlier still, with
`no module named 'fxq16_16' in scope (not stdlib, not a local module, not a
dep)`.

The claim therefore rests on the code rather than on a build. What the code
says is that both controllers are `@value` structs of machine words in the
caller's frame, that no function in `pid`, `pidfx`, `pidtune` or `piderr`
declares an effect, and that nothing allocates. That is a reading of the
source, not a measurement, and this section will say something different when
the toolchain resolves a dependency for a device build.

## What is not included

- **A clock, a sensor or an actuator.** The measurement arrives as a number
  and the output leaves as one.
- **Cascade control, feedforward and gain scheduling.** Each is a structure
  built out of controllers rather than a feature of one, and each is composed
  from what is here.
- **A manager that owns several loops.** That is the program's shape.
- **One generic controller over both number types.** A type cannot carry a
  numeric parameter here, and there is no trait a `@value` arithmetic type
  can implement that a `@value` struct could be generic over. The two forms
  are written out, which is the same choice fixedpoint-nv made for its four
  formats.
- **A decision about when a tuning experiment has finished.** See rule 13.

## Related packages

- [fixedpoint-nv](https://novo-lang.org/packages/fixedpoint-nv) is the
  `FxQ16_16` type the second form is written in. Taking its type rather than
  carrying raw integers is the point of that form: a controller written in
  raw integers is one where the shift is somebody's to remember, and a gain
  65536 times too large is the bug that follows.
- [bitfield-nv](https://novo-lang.org/packages/bitfield-nv) writes the output
  into a timer's duty-cycle field.
- [modbus-nv](https://novo-lang.org/packages/modbus-nv) is how a controller
  on one machine reads a sensor on another.
- [can-nv](https://novo-lang.org/packages/can-nv) is the same for a vehicle
  or a machine-tool bus.
- `std.time` in the standard library is where the elapsed seconds come from
  on a host.

## Tests

```bash
novo test                             # 38 tests
novo test tests/pid_tests.nv          # 18: the terms, the policies, the dt rules
novo test tests/pidfx_tests.nv        # 11: the same in fixed point
novo test tests/pidtune_tests.nv      #  9: the relay and the five rules
```

The tuning tables are Ziegler and Nichols 1942, Tyreus and Luyben 1992 and
Pessen 1954, and they are written out in `pidtune`'s documentation comments
so that a reader can check the arithmetic. The relay method is Åström and
Hägglund 1984.

No test reads a clock or a sensor. Every measurement and every elapsed time
is a number the test writes out.

The tests compile today and fail at run, each on the
`not implemented: pid-nv.<module>.<fn>` panic that is its body. That is the
expected state of an interface release. A few assertions pass, because they
are about published constants rather than about functions.

## Implementation status

| Item | Implemented |
| --- | --- |
| `pid.PID_WINDUP_*`, `.PID_DERIV_*`, `pidtune.PID_RULE_*` | yes (they are constants) |
| `pid.PidF`, `.PidGains`, `.PidLimits`, `.PidAntiWindup`, `.PidDerivative` | declared |
| `pidfx.PidFx`, `.PidFxGains`, `.PidFxLimits` | declared |
| `pidtune.PidRelay`, `.PidUltimate`, `.PidTuneRule` | declared |
| `piderr.PidError`, and its `Error` implementation | declared |
| `pid`: 43 functions | no |
| `pidfx`: 42 functions | no |
| `pidtune`: 18 functions | no |
| `piderr.describe` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
