# Lab 2 Milestones — EtixP

Working notes and the script for recitation. Fork: <https://github.com/EtixP/f26-lab02>

| Milestone | Status |
| --- | --- |
| 1 — Specify the invariant as a property | **Done** (pushed on its own, CI red) |
| 2 — Fix the bug | **Done** (CI green) |
| 3 — Audit the generated suite | **Done** (written up in `README.md`) |

## Setup

- Forked `CMU-17-214/f26-lab02` to `EtixP/f26-lab02`; `origin` points at the fork and
  `upstream` at the course repo.
- Actions enabled on the fork before the first push, so every push is recorded.
- Baseline `mvn test`: **all 7 tests green** (6 examples + 1 provided property).
- Baseline coverage of `AvailabilityCalculator`, from `target/site/jacoco/jacoco.csv`:

  | Metric | Missed | Covered | |
  | --- | --- | --- | --- |
  | Instructions | 0 | 80 | **100%** |
  | Branches | 0 | 8 | **100%** |
  | Lines | 0 | 17 | **100%** |

  A perfect score on every counter JaCoCo reports, over code that is wrong.

## Milestone 1 — the property

Added `everyMinuteIsExactlyOneOfBookedOrFree` to
`src/test/java/edu/cmu/cs214/availability/AvailabilityProperties.java`
(lines 46–59), under the marker comment.

```java
@Property
void everyMinuteIsExactlyOneOfBookedOrFree(@ForAll("scenarios") Scenario s) {
    List<TimeInterval> free = calc.freeSlots(s.dayStart(), s.dayEnd(), s.bookings());
    for (int minute = s.dayStart(); minute < s.dayEnd(); minute++) {
        final int m = minute;
        boolean booked = s.bookings().stream().anyMatch(b -> b.start() <= m && m < b.end());
        boolean reportedFree = free.stream().anyMatch(f -> f.start() <= m && m < f.end());
        assertTrue(booked != reportedFree, /* message elided */);
    }
}
```

It encodes the sentence from `ARCHITECTURE.md` directly: every minute of the business
day is either covered by a booking or reported as free, never both and never neither.
`booked != reportedFree` is the "exactly one of" — it catches double-counting and
dropped time in the same assertion. Checking minute by minute is cheap here because a
day is at most 1440 minutes.

Committed and pushed **on its own, before any fix** (commit `1cdae05`), so the Actions
tab records a red run.

### What to show the TA

**1. The property.** The code above, and the one-sentence invariant it encodes.

**2. The smallest failing sample.** jqwik shrank 9 steps down to:

```
Shrunk Sample (9 steps)
-----------------------
  arg0: Scenario[dayStart=0, dayEnd=1, bookings=[]]
```

*(On re-runs this prints under `Sample` rather than `Shrunk Sample`, because jqwik
replays the previously failing sample first.)*

**3. What the calculator returns for that sample.** `[]` — an empty list. A one-minute
business day with no bookings at all, and it reports zero free time. Minute 0 is neither
covered by a booking nor reported as free.

The unshrunk original sample makes the shape of the bug more obvious, if it helps to
show both: day `[40, 1384)` with a last booking ending at minute 645. Everything from
645 to 1384 — nearly twelve hours — is silently dropped.

**4. Why the provided "no overlap" property still passes on it.**
`freeSlotsNeverOverlapABooking` loops **over the returned free slots**. On this sample
the returned list is empty, so the loop body never executes and the property passes
*vacuously*. That is not bad luck, it is structural: the property quantifies over the
calculator's own output, so it can only ever catch a slot that was wrongly *reported*
free. It is blind by construction to free time that is missing. A calculator hardcoded
to `return List.of();` passes it on every input in the world.

**5. Why mine fails.** Mine quantifies over the **minutes of the business day**, which
come from the input, not the output. The day supplies the minutes whether or not the
calculator returns anything, so absence becomes observable. That is the whole difference
between the two properties: one asks "is what I got back correct?", the other asks "did
I get back everything that should be there?"

### Evidence

- Local failing run: the `mvn test` output showing `Shrunk Sample` (screenshot / saved output).
- CI: the red run on the Actions tab for commit `1cdae05`.

## Milestone 2 — the fix

`src/main/java/edu/cmu/cs214/availability/AvailabilityCalculator.java`. The loop emitted
the gap *before* each booking and advanced `cursor` past it, then returned immediately. It
never emitted the final gap `[cursor, dayEnd)`. Three lines added:

```java
if (cursor < dayEnd) {
    free.add(new TimeInterval(cursor, dayEnd));
}
```

The `cursor < dayEnd` guard is load-bearing, not decoration: `TimeInterval`'s constructor
throws on `start >= end`, so emitting the tail unconditionally would turn the bug into an
exception on any fully-booked or degenerate day.

**One-sentence summary for the TA:** `freeSlots` never emitted the free gap between the
end of the last booking and the end of the business day, so all free time after the last
meeting was silently dropped — and a day with no bookings came back with no free time at
all.

The property was not touched. The fix is in the code.

### Verification

- `mvn test`: **8 tests, 0 failures, BUILD SUCCESS** (6 examples + 2 properties).
- Re-ran both properties at `-Djqwik.tries.default=20000`: 20,000 checks each, green.
- Spot-checked the counterexamples and edge cases directly:

  | Input | Returns |
  | --- | --- |
  | day `[0,1)`, no bookings *(the shrunk sample)* | `[[0,1)]` |
  | day `[40,1384)`, last booking ends 645 *(original sample)* | `[[244,278), [645,1384)]` |
  | day `[540,1020)`, no bookings | `[[540,1020)]` |
  | day `[540,1020)`, booked `[540,1020)` | `[]` |
  | day `[540,1020)`, booking `[900,2000)` runs past close | `[[540,900)]` |
  | day `[100,100)` degenerate, no bookings | `[]`, no exception |
  | day `[540,1020)`, booking `[0,100)` entirely outside | `[[540,1020)]` |

- Coverage after the fix: still 100% instruction, branch and line — see below, this is the
  point of Milestone 3.

**Show the TA:** the Actions tab with the red run (commit `1cdae05`) beside the green run
(commit `0e22d41`), plus the one-sentence summary above.

## Milestone 3 — the audit

Written up in full in `README.md` under "Milestone 3: auditing the generated suite". In
brief, the three weaknesses in `AvailabilityCalculatorTest`:

1. **Controllability** — every value-checking test ends its last booking exactly at
   `DAY_END` (1020), so no trailing gap ever exists to be dropped. The exposing input
   shape is never driven. All six tests also share one hardcoded day.
2. **Controllability** — no test passes an empty booking list. A completely free day came
   back as `[]`, the most trivially wrong output possible, and nothing asked.
3. **Observability** — `returnedSlotsNeverOverlapABooking` *does* drive the exposing input
   (booking `[600,660)`, tail `[660,1020)` dropped) but asserts only non-overlap **over the
   returned slots**, so it cannot see a slot that is missing. It is vacuous on an empty
   list: `return List.of();` passes it on every input.

**Why 100% coverage did not save it:** the defect was a *missing* statement, not a wrong
one. There was no `if (cursor < dayEnd)` block to leave uncovered, so the omission was
invisible to every counter JaCoCo reports. The fix added 11 instructions, 2 branches and 2
lines, and the report read 100% before and after — identical score, opposite correctness.
Weakness 3 is the other half of the lesson: buggy code executing at full coverage credit
under an assertion structurally incapable of failing.

**Show the TA:** the README section, and the before/after coverage numbers.

## Tools and models

See the note at the bottom of `README.md`.
