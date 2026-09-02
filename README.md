# Lab 2 Starter: Availability Calculator

A small reservation component. Given a room's bookings and the day's business hours,
`AvailabilityCalculator.freeSlots` computes when the room is free. It is the code you
work in for Lab 2.

It ships with a generated test suite that passes, and a property-based test harness
(jqwik) with one example property. Everything is green. Your job in Lab 2 is to decide
whether green actually means correct.

**Read `ARCHITECTURE.md` before the code.**

## Build and test

```
mvn test
```

`mvn test` runs both files, the ordinary example-based tests (`AvailabilityCalculatorTest`)
and the property-based tests (`AvailabilityProperties`). A code-coverage report is written
to `target/site/jacoco/index.html`.

## Continuous integration

This repository has CI configured in `.github/workflows/ci.yml`. GitHub disables workflows on a
fresh fork, so enable them once on your fork (the handout shows where). After that, every
push runs `mvn test`. You will watch the gate go red when your new property finds the bug, then
green once you fix it.

## Where things are

- Component: `src/main/java/edu/cmu/cs214/availability/`
- Example-based tests: `src/test/java/edu/cmu/cs214/availability/AvailabilityCalculatorTest.java`
- Property-based tests: `src/test/java/edu/cmu/cs214/availability/AvailabilityProperties.java`
- Setup: `SETUP.md`

See the Lab 2 handout on the course page for the three milestones you show a TA.

## Lab 2 work

Milestone notes and the full walkthrough for recitation are in `MILESTONES.md`.

### Milestone 3: auditing the generated suite

`AvailabilityCalculatorTest` held 100% instruction, branch, and line coverage of
`AvailabilityCalculator` while `freeSlots` never emitted the trailing gap
`[cursor, dayEnd)`. Three concrete weaknesses let that happen.

**1. Every value-checking test pins the last booking to end exactly at `DAY_END`
(controllability).** The five tests that assert on the returned list use last bookings of
`[540,1020)`, `[720,1020)`, `[900,1020)`, `[900,1020)` and `[660,1020)` — all ending at
1020, the close of business. When the last booking ends at `DAY_END` there is no trailing
gap to drop, so the buggy line is not merely unasserted, it is unreachable as a defect.
The suite never drives the one input shape that produces a wrong answer. All six tests
also share a single hardcoded day (`540`/`1020`), so `dayEnd` is never varied either.

**2. No test passes an empty booking list (controllability).** `freeSlots(540, 1020,
List.of())` returned `[]` — a completely free day reported as having no free time, the
most trivially wrong output this component can produce. No test asks. The same blind spot
covers bookings that fall entirely outside business hours, which clip away to the same
empty list.

**3. `returnedSlotsNeverOverlapABooking` runs the buggy path but cannot see the result
(observability).** This is the one test that *does* drive the exposing input: booking
`[600,660)` leaves `[660,1020)` to be dropped, and it was dropped, during this test. But
the assertion iterates **over the returned slots** checking non-overlap, so it can only
catch a slot wrongly reported free — never one that is missing. It is worse than weak, it
is vacuous: the loop body does not execute on an empty list, so a `freeSlots` hardcoded to
`return List.of();` passes it on every input. It bought full coverage credit for the buggy
line while asserting nothing capable of failing.

### Why high coverage did not save it

The defect was a **missing** statement, not a wrong one. Coverage instruments code that
exists; there was no `if (cursor < dayEnd)` block to leave red, so the omission was
invisible to every counter JaCoCo reports. The fix added 11 instructions, 2 branches and
2 lines — and the report read 100% both before and after. Identical score, opposite
correctness.

That is the general lesson. Coverage measures which lines *ran*, not whether the lines
that *should* exist do, and not whether anything checked the result. Weakness 3 is the
proof of the second half: buggy code executing at full coverage credit under an assertion
structurally incapable of failing. Green plus 100% says the tests ran and nothing they
checked was wrong. Whether the code is correct is a separate question, and it is answered
by the specification you write down, not by the metric.

The property in `AvailabilityProperties.java` closes all three gaps at once: jqwik
generates the days and bookings the examples never drove (controllability), and quantifying
over the *minutes of the day* rather than over the returned slots makes missing output
observable (observability).

## Tools and models used

Claude Code (Claude Opus 5) was used for this lab: to run the baseline build and
coverage report, to draft the `everyMinuteIsExactlyOneOfBookedOrFree` property in
`AvailabilityProperties.java`, and to write `MILESTONES.md`. All work was reviewed
and is defended by me.

## Lab 2 work

Milestone notes and the walkthrough for recitation are in `MILESTONES.md`.
