---
name: testing
description: >
  Testing doctrine for AI coding agents — real-boundary tier definitions
  (unit/integration/e2e), the E2E>integration>unit hierarchy, the testability-first
  (headless) precondition, nondeterminism-for-resilience, the seam-table test plan, and
  the anti-fake-test rules. Use whenever writing, reviewing, planning, changing, or
  debugging tests, or deciding what kind of test to write. Tool/substrate specifics live in
  the repo's .claude/testing-context.md. The test-adversary agent judges against this file.
---

# Testing doctrine

This is the standard. It exists because coding agents fail at testing in repeatable ways:
they pad coverage with shallow unit tests, mislabel mock-heavy tests as "integration",
over-mock until green, and edit a failing test to pass instead of finding the bug. Every rule
below is aimed at one of those.

Tool- and substrate-specific facts (the test runner, what your real services are, the
filename convention, where each tier runs, what NOT to flag) live in the repo's
`.claude/testing-context.md`. Read it; it overrides generic assumptions here.

## The one rule

Every test must be able to **fail for a real reason** — it exists to catch a real defect in the
real behavior it claims to cover. Put it at the **outermost real boundary** where that holds. A
test that can't fail for a real reason — or that mocks the thing it claims to test — is **worse
than no test**: it bills you forever and tells you you're safe when you're not.

## Testability-first (the upstream precondition)

You cannot test what you cannot drive. **Before** building a feature, confirm the
system/subsystem under test is **operable and observable headlessly** — drivable from the CLI
or an equivalent programmatic seam an AI/test can invoke and inspect, with no human, GUI, or
manual step. If it isn't:

1. The **first work item** is to build/extend that headless seam (a CLI command, a
   programmatic entrypoint, a debug handle) so the behavior can be driven and observed by an
   integration/e2e test. *Then* the feature.
2. Shipping the feature without that seam is a **deliberate, flagged decision** requiring
   explicit owner approval — never a silent default. (UIs count: they must be AI-drivable,
   e.g. via a browser-debug handle.)

This is what makes "bias to real tiers" *achievable*: without a headless seam you're forced to
mock, and you get fake confidence.

## The tiers (by real boundary — not by tool)

- **Unit** — in-process, **no real I/O**. Pure logic only: parsers, prompt/template builders,
  schema validators, reducers, normalization/redaction transforms. *If it needs a mock to
  exist, it's probably not a unit test.*
- **Integration** — real adjacent infra wired across **one** seam (a real datastore, real
  auth, the real runtime). Mock **only** the single boundary you genuinely can't reach.
- **E2E** — a full real flow, as close to production as possible: real entrypoint → real
  system → real outward effect.

(Your repo's `.claude/testing-context.md` maps these to concrete services, the runner,
filenames, and where each tier executes.)

## The labeling rule

Name a test by the **outermost real boundary** it exercises. Mock that boundary and it's
mislabeled or fake. *A test that mocks all its boundaries is a unit test, whatever its
filename says.*

## Where a test belongs: E2E > Integration > Unit

Put each test at the tier where a real defect makes it go **red**. Higher tiers dominate because
they touch the real wiring — that's where the defects that actually hurt live. Glue and
orchestration code unit-tested in isolation usually *can't* fail for a real reason: the mock
returns what you told it to, forever.

This is an **investigation, not an argument.** "Can I justify a unit test here?" always finds a
yes. Ask instead: *if the real behavior broke, would this test catch it?* If the honest answer is
no, it's at the wrong tier or shouldn't exist. A test being *easy* to write is not evidence it's
worth keeping — distrust "it's simpler to test it down here."

The enemy isn't a tier — it's a test that can't fail for a real reason, and that rots at **every**
tier (a mock-heavy "integration" test is no better than a tautological unit one).

## Determinism vs. flakiness (a different axis from the tier)

- **Flakiness = banned.** A test that fails for reasons *unrelated to the behavior under test*
  (races, shared state, ordering, time-of-day). Always a bug in the test.
- **Nondeterminism = often ideal.** Drive a real/variable space — random/property-based
  inputs, injected faults, induced latency, real model variance — and assert an **invariant**
  that must hold across *all* of it ("never sends to the wrong recipient", "always recovers
  after a disconnect", "output always parses"). This catches resilience and edge-case bugs a
  fixed-input test never reaches. The rule: nondeterministic *conditions* + *invariant*
  assertions = good; nondeterministic *pass/fail for the same behavior* = flaky = bad. Never
  assert an **exact** value against a nondeterministic output — assert the invariant.

## The real cost of a test isn't running it — it's lying to you

Every test you keep runs forever, is maintained forever, and its green checkmark gets read as
"this works." A test that can't fail for a real reason pays none of that back — you carry its cost
*and* it manufactures the feeling of safety while covering nothing. That false confidence is the
expensive part, far more than CI minutes.

**Affordability is a weak, secondary concern.** Run-cost is a reason not to *duplicate* coverage
you already have at a higher tier — never a reason to write a cheaper test that proves less. The
question is never "is this cheap enough to run?" but *"does it earn its forever-cost by catching a
real defect?"* If you can't name the defect it catches, delete it.

## The test plan IS the rubric — write it FIRST

Before writing tests, fill the seam table (home: the feature's plan/artifact, plus a one-line
"real vs mocked" header in the test file). Write the tests **failing first**, then implement.

| Seam | Tier | Real | Mocked | Why mocked (must be genuinely unreachable) |
| ---- | ---- | ---- | ------ | ------------------------------------------ |

Also state the **behavior under test**, any **invariant** under nondeterministic input, and
how the test **fails for the right reason**.

## Anti-fake-test rules (non-negotiable)

- Prefer real. **Zero mocks is the ideal.**
- **If a dependency is invoked during the test, it must be real.** Trace execution to prove a
  mock is never reached; else un-mock it.
- **Never mock both sides** of a boundary (caller *and* callee) — that tests nothing.
- **Never mock the system under test.**
- Assert **observable behavior** — return values, thrown errors, emitted events, real side
  effects. A call-spy as the *sole* assertion is banned unless the call *is* the behavior.
- A test must **survive a rename-only refactor**; if renaming an internal symbol breaks it,
  it's coupled to implementation.
- Every test must be able to **FAIL for the right reason** — no tautologies. Confirm it goes
  red when the behavior breaks.

## Convergence (stateful code)

Sessions, tool/agent loops, streaming, reconnect/resume, auth lifecycles need a
**full-lifecycle** test: happy → failure/disconnect → recovery, with **real** components at
the seam. One-op-at-a-time tests miss the bug; a run of sequential "fix" commits on one seam
is the signal you're missing it.

## When a test fails: STOP

Don't fix it in the same breath you wrote it. Invoke the **test-adversary** agent (triage)
with the failure output + what changed, and follow its verdict; don't override it without
owner approval.
