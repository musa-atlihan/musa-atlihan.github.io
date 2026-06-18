---
layout: post
title: "What I Learned Building an Agent OS for Agent-Driven Development"
date: 2026-06-07
categories: [agent-driven-development]
tags: [agents, markdown, adr, engineering-process]
excerpt: >-
  Lessons from building a Markdown-based agent workflow with planning gates,
  ADRs, acceptance matrices, review rules, and durable project memory.
---

Agent-driven development is still early.

Most of us are experimenting with better prompts, stronger models, faster
coding loops, and more automation. Those things matter. But while building my
own project, I learned that the bigger unlock is not simply asking an agent to
write code.

The bigger unlock is giving agents an operating system.

By "agent OS", I mean a structured way of working where the project itself
teaches agents how to plan, audit, implement, review, and avoid drifting away
from the product.

For me, that operating system is mostly Markdown:

- an `AGENTS.md` file that routes agents into the right workflow,
- milestone plans that define scope and non-goals,
- ADRs, or Architecture Decision Records, that preserve important decisions,
- gates that stop risky work before implementation,
- acceptance matrices that turn vague intent into concrete behavior,
- testing rules that map back to those matrices,
- review rules that make agents report bugs before writing fixes,
- and a human-in-the-loop process where I stay involved at the right decision points.

This has changed how I build.

It did not make the project magically easy. It made the work more controlled,
reviewable, and repeatable.

## The project that forced the lesson

I am currently building Recon Core, an open-source Reconciliation as Code
framework.

The goal of Recon is to help data teams define whether two data sources should
match, run repeatable checks, and generate evidence about the result. In plain
terms: it is about making data reconciliation explicit, testable, and
trustworthy instead of relying on scattered scripts and tribal knowledge.

That kind of project has many places where small mistakes can become expensive:

- public configuration files,
- command-line behavior,
- generated reports,
- data privacy,
- external integrations,
- compatibility promises,
- error messages,
- and future package or plugin ecosystems.

That made Recon a good stress test for agent-driven development.

If an agent only optimizes for "make the test pass", it can easily make the
wrong product or architecture decision. If it changes behavior without updating
the docs, the next agent may read stale instructions. If it fixes one example
but misses a similar case, the same bug returns in another form.

I needed a way to let agents help deeply without letting them silently steer the
project away from its direction.

That is where the agent OS became real.

## Agents need rails, not just prompts

A simple agent workflow often starts like this:

> Here is the task. Please implement it.

That can work for small isolated changes. It breaks down as the project grows.

For serious work, an agent needs to know more than the task. It needs to know:

- what the product is,
- what it is not,
- which rules are non-negotiable,
- which work is high-risk,
- what must be reviewed before coding,
- what docs must change with behavior,
- what tests prove the behavior,
- and when to stop and ask the human.

This is why I moved from prompting the agent to operating the agent through
project rules.

The `AGENTS.md` file became the root router. It stays compact. It does not try
to hold every detail. Instead, it sends the agent to the right workflow based on
the work:

- audit,
- implementation,
- milestone planning,
- documentation and ADR updates,
- public behavior changes,
- security and release rules,
- code review.

That small shift matters.

Without routing, every agent starts from a blank mental model. With routing, the
agent enters the right process before it touches the code.

## Milestones need prework

One of the most useful changes I made was requiring lightweight prework for
every milestone.

Not only the obviously risky milestones. Every milestone.

The lightweight version is simple:

- scope,
- non-goals,
- expected behavior,
- affected docs,
- required tests,
- security, privacy, and compatibility impact,
- Definition of Done.

This prevents a common failure mode: starting implementation before the shape of
the work is clear.

For high-risk milestones, the lightweight version is not enough. Then I require
an acceptance or conformance matrix.

The matrix forces the agent and me to spell out:

- what situations must be handled,
- what the expected behavior is,
- which tests prove it,
- which docs or decisions must change,
- and which cases are intentionally out of scope.

This sounds formal, but it is practical.

It catches the "we tested one example, but missed the neighboring case" problem.

Imagine a team is building a payment workflow. It is not enough to test one
successful card payment and call the feature done. A useful matrix asks:

- What happens when the payment succeeds?
- What happens when the card is declined?
- What happens when the network times out?
- What happens if the user retries?
- What happens if the payment succeeds but the confirmation email fails?
- What happens if the amount is zero, very large, or in a different currency?
- What should the customer see?
- What should support teams see?
- What should never appear in logs or reports?

That is the kind of thinking I want agents to help with.

They stop implementing only the happy path and start mapping the behavior space.

## High-risk work should be allowed to stop

This was a major mindset shift.

Sometimes the best agent output is not code.

Sometimes the best output is:

> Do not implement this yet. The decision is not written down. The test matrix
> is missing. The risk is bigger than this milestone. This work should be split.

I want agents to be able to say that.

In fact, I now treat it as a feature of the workflow.

If a milestone is too broad, touches too many risky areas, or cannot be made
safe as one implementation unit, the agent should recommend splitting it. A
large milestone can become smaller sub-milestones, each with its own scope,
tests, and Definition of Done.

That is not bureaucracy. That is project control.

It lets the system slow down before it creates expensive confusion.

## ADRs make project memory durable

Agents have context windows. Projects need memory.

ADRs are how I make important decisions durable.

When a decision affects public behavior, data formats, validation rules,
integrations, security, privacy, compatibility, or architecture, it should not
live only in a chat thread.

It needs to be written down.

This helps humans, but it also helps future agents. A future agent can compare
the current implementation against the recorded decision, find drift, and
recommend what to fix.

That is one of the most useful parts of agent-driven development:

The agent is not only a coder.

It can become an auditor of the project's documented intent.

## Docs are not a cleanup step

I used to think of docs as something to update after implementation.

With agents, that is risky.

If the code changes but the docs do not, the next agent may read stale guidance
and make the wrong move. Documentation drift becomes implementation drift.

So the workflow has to say clearly:

> When behavior changes, update the relevant docs in the same work session.

That includes user docs, planning docs, architecture docs, compatibility docs,
testing plans, changelogs, and ADRs when needed.

This keeps the repo self-explaining.

It also makes reviews much stronger. The agent can compare:

- what the docs say,
- what the tests prove,
- what the implementation does,
- and what the milestone intended.

That comparison produces better findings than a simple code scan.

## Review and fixing should be separate

Another useful rule: when I ask for a review, the agent should not fix anything.

It should take a code-review stance:

- findings first,
- ordered by severity,
- grounded in specific evidence,
- focused on bugs, regressions, missing tests, privacy risks, and behavior drift.

This separation matters.

When review and fixing are mixed too early, the agent can rush into a patch and
hide the reasoning. I prefer to first understand the issue, decide whether it is
real, decide whether it should be fixed now or gated for a future milestone, and
then ask for the fix.

That keeps me clearly in the loop.

## What success looks like

Success does not mean the agent never makes mistakes.

Success means the system catches mistakes earlier and turns them into better
process.

A good example is when a feature handles one case correctly but misses a nearby
case. In a weaker workflow, that becomes a bug report later. In this workflow,
it can become:

- a regression test,
- a code fix,
- a docs update,
- a future milestone gate,
- and a shared requirement for similar work later.

That compounds.

The lesson becomes part of the project. Future agents do not need to rediscover
it from scratch.

## My practical playbook

If you are experimenting with agent-driven development, this is the version I
would recommend starting with.

### 1. Create a compact root agent instruction file

Do not put everything in it. Use it as a router. Define the product mission,
non-negotiable rules, and where agents should go for different workflows.

### 2. Classify work before coding

Separate ordinary tasks from high-risk work. If the task affects public
behavior, generated outputs, security, privacy, compatibility, or external
integrations, treat it as high-risk.

### 3. Require milestone prework

Before implementation, write down scope, non-goals, expected behavior, affected
docs, required tests, risks, and Definition of Done.

### 4. Use matrices for risky behavior

Do not rely on examples only. Enumerate the important situations and neighboring
cases. Map each case to tests, docs, or an explicit out-of-scope decision.

### 5. Let agents recommend splits

If a milestone is too large, let the agent say so. Splitting a milestone before
implementation is much cheaper than untangling a broad half-finished change.

### 6. Keep ADRs close to real decisions

Use ADRs when the decision changes public behavior, architecture,
compatibility, security, privacy, or long-term project direction.

### 7. Make docs part of done

If behavior changed, docs changed. If docs did not change, ask why.

### 8. Separate review from fixing

Use agents as serious reviewers. Ask for findings first. Then decide what to
fix.

### 9. Keep the human in the loop

The agent can audit, recommend, challenge, and implement. But the human should
own product direction, tradeoffs, prioritization, and acceptance of risk.

## What I would avoid

These are the patterns I now try to avoid:

- vague prompts for large tasks,
- one giant milestone with many risky surfaces,
- coding before docs and gates are aligned,
- tests that cover examples but not neighboring cases,
- decisions that live only in chat history,
- agent reviews that immediately patch without explaining the bug,
- and treating the conversation as the source of truth.

Chat is useful, but the repo needs to carry the memory.

## The human role changes, but it does not disappear

The most interesting part of this process is that my role changed.

I am still deeply involved, but not by manually typing every line of code.

My work is more about:

- setting direction,
- asking sharper questions,
- approving or rejecting tradeoffs,
- deciding when something should become a gate,
- deciding when a milestone should split,
- reviewing the agent's reasoning,
- and making sure the project stays aligned with its mission.

That is a different kind of leverage.

The agent becomes much more effective because it is not guessing the process.
The process is in the repo.

## The biggest lesson

Agent-driven development works better when the project is designed for agents
to succeed.

Not by giving them unlimited freedom.

By giving them structure.

Markdown planning, ADRs, milestone gates, matrices, testing rules, and compact
agent instructions may sound simple. Together they create an operating system
for the project.

That operating system helps agents:

- understand the product,
- avoid drift,
- stop before unsafe implementation,
- audit against documented intent,
- find missing cases,
- update docs with behavior,
- and carry lessons forward.

This field is still young. The patterns are still being discovered.

But for me, this approach is already working.

It has made agent-driven development less like asking for code and more like
building with a disciplined engineering partner inside a growing project.

That is the direction I am most excited about.
