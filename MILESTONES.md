# Lab 2 Milestones — EtixP

Working notes and the script for recitation. Fork: <https://github.com/EtixP/f26-lab02>

| Milestone | Status |
| --- | --- |
| 1 — Specify the invariant as a property | **Done** (pushed on its own, CI red) |
| 2 — Fix the bug | Not started |
| 3 — Audit the generated suite | Not started |

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

## Milestone 2 — fix the bug (not started)

Diagnosis is already in hand from the property's counterexample. In
`src/main/java/edu/cmu/cs214/availability/AvailabilityCalculator.java` (lines 32–38),
the loop emits the gap *before* each booking and advances `cursor` past it, then returns
straight away. It never emits the final gap `[cursor, dayEnd)`. So free time after the
last booking ends is dropped, and an empty booking list produces an empty result instead
of the whole day.

Every one of the six generated examples that could have caught this happens to end its
last booking exactly at `DAY_END`, which is why the tail was never missing in practice.

Remaining: make the fix, get `mvn test` green locally, push, and show the red and green
runs side by side on the Actions tab. Do not weaken the property.

## Milestone 3 — audit the generated suite (not started)

Remaining: name three concrete weaknesses in `AvailabilityCalculatorTest`, each
classified as a **controllability** gap (never drove the input that would expose the bug)
or an **observability** gap (ran the buggy code but the assertions could not see the
wrong result), and write them up in the README along with why 100% coverage did not save
the suite.

## Tools and models

See the note at the bottom of `README.md`.
