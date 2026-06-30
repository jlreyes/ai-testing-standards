# ai-testing-standards

A Claude Code plugin that makes AI coding agents write **real** tests instead of fake ones.
Agents fail at testing in specific, repeatable ways — they pad coverage with shallow unit
tests, mislabel a mock-heavy test as "integration", over-mock until the suite goes green
without proving anything, and — when a test fails — edit the *test* to pass instead of finding
the bug. This plugin counters all four: a doctrine the test-writer loads, and an independent
adversary that judges the result.

## What's in it

### 1. `testing` skill — the doctrine
Auto-loads whenever tests are written, reviewed, or changed. The rules a strong model already
knows but doesn't apply under momentum:

- **Real-boundary tiers.** Unit = in-process, no real I/O. Integration = real adjacent infra
  across one seam. E2E = a full real flow. Name a test by the **outermost real boundary** it
  exercises; mock that boundary and it's mislabeled or fake.
- **Hierarchy & bias: E2E > Integration > Unit.** Each lower tier must justify why the tier
  above can't cover it. Bias up; the enemy isn't unit tests, it's *low-value* ones.
- **Testability-first (the headless precondition).** You can't test what you can't drive.
  Before a feature, the system must be operable **and observable headlessly** — from the CLI
  or an equivalent programmatic seam an AI/test can invoke. If it isn't, building that seam is
  the **first** work item, flagged — not skipped by over-mocking.
- **Nondeterminism ≠ flakiness.** Kill *flakiness* (fails for reasons unrelated to the
  behavior). **Embrace** nondeterminism — drive a real/variable space (timing, injected
  faults, real model variance) and assert an **invariant** that holds across all of it. That's
  how you test resilience and edge cases a fixed-input test can't reach.
- **"Affordable" is defined.** Cost (time, money, setup) weighed against the marginal
  confidence a test uniquely adds. A test is unaffordable only when a lower tier already proves
  the same behavior — never because it's slow or nondeterministic. Cost forbids duplication,
  never licenses faking.
- **Anti-fake-test rules.** If a dependency is invoked, it must be real. Never mock both sides
  of a boundary, or the system under test. Assert observable behavior, not call choreography.
  Every test must be able to **fail for the right reason**.

### 2. `test-adversary` agent — the enforcement
Sonnet 5, runs in **its own context** with no access to the writer's reasoning, so it can't
inherit the writer's rationalizations. Two modes:

- **Review** a test plan + diff before "done" → reports issues, tagged
  `KILL` / `RELABEL` / `DE-MOCK` / `PROMOTE` / `SEAM-MISSING` / `WEAK-ASSERTION`. It **never
  approves** — silence means clean.
- **Triage** a failing test → a binding verdict (`FIX_IMPLEMENTATION` / `UPDATE_TEST` /
  `ARCHITECTURAL_ISSUE`) with confidence. This is the cure for hack-it-green: the agent that
  wrote the code doesn't get to decide whether its failing test is "outdated".

## Install

```bash
claude plugin marketplace add jlreyes/ai-testing-standards
claude plugin install ai-testing-standards@ai-testing-standards
```

## Configure for your repo

The doctrine is framework-agnostic. Tell it about *your* stack with a
`.claude/testing-context.md` at your repo root — the agent reads it and honors it over generic
assumptions (it's where you map the tiers to real services, set the filename convention, say
where each tier runs, and list the house patterns it must **not** flag):

```markdown
# Testing context (read by the test-adversary agent)

## Runner & tiers → filenames
- Runner: <vitest | jest | pytest | go test | …>. Tier by filename:
  `*.test.ts` unit · `*.integration.test.ts` integration · `*.e2e.test.ts` e2e.

## What each tier means here
- Integration = <your real datastore + auth + runtime, across one seam>.
- E2E = <your real entrypoint → system → outward effect>.

## Where tiers run
- Unit → local. Integration / e2e → <your CI / remote runner>.

## Headless seam status
- <Is the system drivable headlessly yet? If not, what's the precondition to build first?>

## Do NOT flag (house choices)
- <intentional patterns the agent must not report — skip-guards, deliberate mocks, etc.>
```

## Use the agent

- **Review (gate before done):** *"Use the test-adversary agent to review the test changes on
  this branch."*
- **Triage (on a red test):** *"A test failed — use the test-adversary agent to triage it,"*
  and pass it the failure output plus what changed.

## License

MIT — see [LICENSE](./LICENSE).
