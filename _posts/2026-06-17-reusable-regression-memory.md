---
layout: post
title: "Beyond TDD: Giving Coding Agents Reusable Regression Memory"
date: 2026-06-17
categories: [agent-driven-development]
tags: [agents, testing, regression, data-engineering, data-quality]
excerpt: >-
  A practical pattern for linking high-value regression tests to bug context,
  pytest markers, validation scripts, and future agent carryover gates.
---

When I work with coding agents, I increasingly care less about whether they can
write a test once and more about whether the project can preserve why that test
matters.

This article is about a small pattern I have been using while building Recon
Core: a Regression Capture Manifest. It is a lightweight YAML-backed layer that
links important regression tests to their bug context, pytest markers,
validation scripts, pre-commit checks, and future carryover gates.

The point is not to replace TDD, code review, or normal test design. The point
is to reduce avoidable rediscovery in goal-driven review and fix loops. When a
future agent starts working on the same surface, I want it to begin with a
validated catalog of important regressions and edge cases instead of relearning
everything from scattered tests, old comments, and past review threads.

## The reconciliation problem that shaped this

I came to this from data engineering, where the recurring need was
reconciliation. Teams often need to check whether a target table still matches a
source system, whether a migration preserved row counts, whether keys are
missing, whether duplicates appeared, whether nulls broke a join, or whether a
Snowflake view matches the backend data it is supposed to represent.

But the way this work happens is often fragmented.

Across projects, I kept seeing the same shape. A QA analyst might have a
spreadsheet. An analyst might have a SQL file. A data engineer might have a
Python script, a notebook, a local folder of comparison queries, or a screenshot
that proved something was correct on a particular day.

These checks were useful, but they were usually personal, local, one-time, and
hard for the rest of the team to discover or reuse. Months later, when the same
kind of validation was needed again, the old context was easy to forget.

That became very visible to me on a recent project where we were replicating
multiple backend database sources into Snowflake. Reconciliation was not
optional. We needed repeatable checks proving that Snowflake tables and views
matched their backend sources, and we needed the workflow to be usable by the
team, not locked inside one person's local scripts.

With the help of Claude Code, I built a reusable reconciliation framework for
that team. The useful part was not that it was a framework for its own sake. The
useful part was that it standardized how reconciliation tests were written,
reviewed, reused, and run across the team.

That experience is one of the reasons I started [Recon Core](https://github.com/recon-labs/recon-core),
an open-source Reconciliation as Code project. The idea behind Recon is that
reconciliation should be visible, versioned, reusable, schedulable, and
reviewable. Instead of treating comparisons as temporary scripts, the framework
should let data teams define contracts, compile explicit checks, run them
consistently, and produce evidence that explains what was actually compared.

## Tests need memory too

Recon Core is still pre-alpha, and I am building it as a solo developer with
coding agents. That has made me learn a second lesson beyond reconciliation
itself: agents are much more useful when the project gives them stable context,
decisions, rules, and memory.

This is where Andrej Karpathy's [LLM Wiki idea](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
resonated with me. The useful part for my work is the idea of a durable,
agent-maintained knowledge layer that compounds over time instead of forcing the
agent to reconstruct the same context from raw material on every session.

Recon already uses that kind of operating system around the codebase:
`AGENTS.md`, agent instructions, implementation notes, decision logs, gates,
planning docs, and compatibility notes. Those artifacts help agents understand
how to work in the project.

But after enough implementation and review passes, I noticed that tests needed
memory too.

Modern coding agents are good at loops. After a milestone is implemented, the
repeated work is usually not "implement everything again." The loop is narrower:
review the branch, find bugs or missed edge cases, add or correct focused
tests, fix the code, run validation, then review the branch again.

In Codex, I can express this as a goal: continue the review, fix, and validation
loop until a clear exit condition is met. In Claude Code, I think of the same
pattern as a Ralph loop: hand the agent a loop objective, let each review pass
produce fixes, then feed the result into the next pass.

The names differ, but the shape is the same:

- review,
- fix,
- validate,
- review again.

That loop is valuable, but it can also take too many avoidable iterations.

TDD helps because it turns expected behavior into tests before implementation
starts. Unit tests help because they give focused executable checks around a
small behavior or boundary. Regression tests help because they preserve a bug
fix so the same behavior does not break again later.

Still, a test by itself does not always preserve the reason the case mattered,
which future surface must inherit it, or why an agent should consider it before
starting a later change.

This is the gap that led me to the Regression Capture Manifest.

## What the manifest captures

The pattern is simple: when a bug, missed requirement, or edge case is important
enough to become reusable regression memory, capture it in YAML.

The YAML does not replace the test. It records the context around selected
high-signal tests:

- what broke,
- which area it belongs to,
- which executable tests prove it today,
- which future gates must remember it.

Here is a simplified version of the shape:

```yaml
captures:
  - id: source-target-missing-keys-guard
    title: Source keys missing from the target must fail reconciliation
    area: check-engine
    bug_class: missing_keys
    owner_surface: key_safety
    severity: P2
    current_tests:
      - tests/check_engine/test_missing_keys.py::test_missing_source_keys_fail_reconciliation
    carryover_gates:
      - gate: adapter_testkit_regression_carryover
        status: pending
        expected_suite: key_reconciliation
    notes: >
      Row-count equality is not enough. Recon must fail when a source business
      key is absent from the target, and future adapters must preserve the same
      key-level semantics.
```

The `id` is the stable handle for humans, tests, and agents. The `area` and
`bug_class` make inventory and review possible. The `current_tests` field points
to the executable proof that protects the behavior today. The `notes` field
preserves the reason without requiring a future agent to reconstruct the entire
old review.

The field that matters most for future work is `carryover_gates`.

A gate is a named future checkpoint. It says: this bug is tested here today, but
when a related area expands later, that future work is not complete until this
case is reviewed. If a gate is pending, it has not yet been resolved for that
future surface. When that future work starts, the row must become covered,
migrated, deferred with rationale, or `not_applicable` with rationale.

This turns a local regression test into future implementation pressure. The test
is not just protecting today's code. It is also saying:

> When you build the broader version of this system, remember this edge case.

## Linking the manifest to tests

To keep the YAML connected to executable tests, I also link tests back to
capture rows with pytest markers:

```python
@pytest.mark.regression_capture("source-target-missing-keys-guard")
def test_missing_source_keys_fail_reconciliation():
    ...
```

Not every test gets this marker. That is important.

If every unit test becomes a YAML row, the manifest becomes a second test index,
and nobody will read it. The capture layer is for reusable regression or
conformance memory:

- bugs likely to recur,
- public contract behavior,
- risky surfaces,
- future adapter cases,
- diagnostics and evidence behavior,
- CLI behavior,
- parser and compiler behavior,
- similar high-value cases.

The next piece is validation. In Recon Core,
`scripts/check_regression_capture.py` checks the bidirectional link:

- YAML rows must point to real tests.
- Listed tests must carry the matching marker.
- Markers must have matching YAML rows.
- Marked tests must be listed in `current_tests`.

The catalog stays useful because it stays executable.

## Making drift hard

This also fits naturally into pre-commit:

```yaml
repos:
  - repo: local
    hooks:
      - id: regression-capture
        name: regression capture metadata
        entry: python3 scripts/check_regression_capture.py
        language: system
        pass_filenames: false
        files: ^(docs/compatibility/regression-capture/|scripts/check_regression_capture.py|tests/)
```

The hook does not make judgment calls. It catches drift.

If an agent adds a marker without a YAML row, the hook fails. If a YAML row
points to the wrong test, the hook fails. If the marker and `current_tests` list
disagree, the hook fails. `AGENTS.md` teaches the rule up front; pre-commit
enforces the rule when files change.

There is also an advisory side. Recon has a script called
`scripts/check_regression_capture_decisions.py` that looks at changed paths and
maps them to known risky surfaces:

- adapter runtime,
- check-engine semantics,
- diagnostics,
- generated artifacts,
- CLI behavior,
- parser and compiler contracts,
- scan safety,
- other public surfaces.

It does not automatically decide that a capture is required. It asks the
engineering question:

> Did this change create reusable regression memory?

The outcomes are deliberately small. If the answer is yes, add or update a
capture row and link the tests. If the answer is no, record a short rationale
such as `regression_capture_decision: not-required`.

The goal is not ceremony. The goal is to avoid the silent case where an
important bug fix gets a unit test today but no durable memory for the future.

## Why this matters for adapters

One reason this matters in Recon is the future adapter split.

`recon-core` should remain the framework brain: contracts, safety policy,
compiled behavior, and evidence expectations. Future adapter repositories should
provide truthful backend capabilities, metadata, estimates, and execution
primitives for specific systems.

That boundary is important because the framework should decide whether
execution is allowed, while adapters should provide accurate facts and safe
primitives.

Eventually, a shared adapter test kit will be needed so every adapter can prove
it respects the same safety and compatibility expectations. But the test kit
does not have to exist before we start preserving useful cases.

A bug fixed in core today can be captured now, marked as pending for the adapter
test-kit gate, and later reviewed, covered, or migrated when the shared
conformance suite exists.

That is the bridge I wanted: today's local regression tests become tomorrow's
test-kit obligations.

## A workflow other teams can borrow

The workflow is something other teams can borrow without copying Recon's exact
files:

```text
For this bug fix, decide whether it creates reusable regression or conformance
memory. If yes, add a capture row, link current tests, add matching markers, and
run the validator. If no, record regression_capture_decision: not-required with
rationale.
```

In practice, I would put this rule explicitly in `AGENTS.md` and in the review
checklist.

When a bug is fixed, ask whether it is just an ordinary local test or whether it
is reusable regression memory. If it is reusable, add the YAML row, add or update
the focused test, add the marker, run the validator, and let pre-commit catch
drift.

If it is not reusable, leave a short no-capture rationale in the relevant review
note or implementation log.

## How this relates to existing practices

This pattern is not isolated from existing software practices.

It overlaps with traceability matrices because it links important behavior to
tests. It overlaps with test manifests because it adds structured metadata
around tests. It overlaps with conformance suite metadata because it maps cases
to future compatibility gates. It overlaps with regression catalogs because it
remembers fixed bugs as durable cases.

The practical difference is that this version is optimized for coding agents and
long-running repos. It gives an agent a small, validated catalog of fixed-bug
knowledge that is durable, executable, enforceable, and portable into future
shared test suites.

It is not a replacement for TDD, unit tests, CI, review, or judgment. It is a
memory layer around the cases that deserve to survive the current branch.

## Limits

There are limits.

I would not put every unit test in YAML. I would not expect a script to decide
which cases matter. I would not claim this removes review loops.

The useful claim is narrower and stronger: it reduces avoidable rediscovery.
When the next agent works on the same surface, it can read a compact catalog of
past regressions and future obligations before it starts changing code.

That matters because software quality is not only about writing more tests. It
is also about preserving why the important tests exist. In a long-lived
framework like Recon, bugs and edge cases repeat when context disappears,
especially as the project grows toward future repo, package, adapter, and
test-kit boundaries.

If coding agents are going to write and maintain more of our software, they need
memory structures they can actually validate.

A Regression Capture Manifest is one small way to give them that.
