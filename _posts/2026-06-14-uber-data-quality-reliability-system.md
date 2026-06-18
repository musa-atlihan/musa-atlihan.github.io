---
layout: post
title: "How Uber Turns Data Quality Checks Into a Reliability System"
date: 2026-06-14
categories: [data-quality]
tags: [data-engineering, reliability, monitoring, uber]
excerpt: >-
  A look at how freshness, lineage, reconciliation, alerting, and
  auto-resolution turn data quality checks into reliability infrastructure.
---

Most people have experienced surge pricing at some point.

You leave a concert, a stadium, an airport, or any crowded place where everyone
is trying to get home at the same time. You open Uber, check the price, and the
ride is suddenly more expensive than usual.

From the user side, it looks like a simple multiplier. Behind the scenes,
though, that number depends on a real-time marketplace trying to understand
driver supply, rider demand, location, timing, marketplace pressure, and whether
the data feeding the system is reliable enough to support the decision.

That last part is what I found most interesting while studying Uber's data
quality architecture.

Surge pricing is not only a pricing problem. It is also a data trust problem. If
the data is delayed, incomplete, duplicated, or inconsistent, the system can
still continue running while the decisions quietly become worse. The app may
work, dashboards may refresh, and machine learning models may keep returning
predictions, but the truth underneath those decisions may already be drifting.

That is one of the hardest failure modes in data systems. Not the obvious
failure where everything crashes. The quieter one, where the system looks alive,
but the data has stopped being dependable.

What makes Uber's approach interesting is the way they turned data quality from
a set of checks into a reliability system. Their internal platform, UDQ, is not
just about watching tables and sending alerts. It is about defining what healthy
data means, generating checks at scale, using lineage to keep coverage relevant,
reconciling data across systems, reducing noisy alerts, and closing the loop
when incidents recover.

The individual parts are useful, but the real value is in how they connect.

That is the lesson I took from it: data quality becomes much more powerful when
it is treated as an operating model, not just a collection of validations.

## Defining what good data means

One of the strongest parts of Uber's approach is that they make data quality
specific.

In many organizations, people say "the data quality is bad", but that phrase can
hide many different problems. It might mean the data arrived late, a dataset is
missing records, duplicates are inflating metrics, or two systems disagree about
the same business event. Without a shared definition, every incident becomes
harder to diagnose because people are not always talking about the same failure.

Uber grouped historical data quality incidents into four measurable dimensions:

- freshness,
- completeness,
- duplicates,
- cross-data-center consistency.

That classification matters because it gives the system something concrete to
test. Instead of treating quality as a vague feeling, each dimension becomes a
type of reliability question:

- Is the data recent enough to use?
- Is enough of it present?
- Are duplicate records distorting the truth?
- Does the same data match across systems?

Freshness is the clearest example.

A weak freshness check only asks whether something updated recently. That can be
useful, but it does not always tell you whether the data is ready for real use.
Uber's definition goes deeper: freshness is the delay after which a dataset is
expected to be 99.9% complete.

That difference is important. A dataset can receive new records and still be
incomplete. A pipeline can finish and still not be safe for downstream
consumers. A model can receive inputs that look recent but are not complete
enough to trust.

So the better question is not only, "Did data arrive?" It is, "Has enough of the
expected data arrived for this dataset to be used safely?"

That turns freshness into a reliability threshold. It creates a clearer point
where the system can decide whether downstream users, dashboards, or models
should trust the data yet.

## The hard part is keeping checks useful

Adding a few data quality checks is usually not the hardest part.

The harder part is keeping those checks useful as the data ecosystem changes.
Pipelines evolve, tables get deprecated, SLAs shift, new downstream consumers
appear, and ownership changes. A check that made sense six months ago can slowly
become outdated if the system around it has changed.

That is where many monitoring efforts lose their value. The checks still exist,
but they no longer reflect the real shape of the system. Over time, the quality
layer can become noisy, incomplete, or stale.

Uber's approach is interesting because test generation is connected to metadata,
SLAs, partition keys, and lineage. That means the quality layer is not only a
manually maintained list of checks. It can use context from the data ecosystem
to generate and update validations more intelligently.

This is important because quality coverage has to move with the system. If a
dataset has upstream and downstream dependencies, the monitoring should
understand that. If an obsolete table no longer matters, it should not continue
creating noise. If a critical dataset changes, the quality expectations around
it need to stay aligned.

At the scale Uber described, their Test Execution Engine supported roughly
100,000 daily executions across thousands of tests. The number is impressive,
but the more interesting point is the pressure it creates. Once testing reaches
that level, the challenge is no longer just running checks. The challenge
becomes coverage, ownership, noise, root cause clarity, and incident workflow.

A data quality system does not only need to detect problems. It also needs to
remain trusted by the people who operate it.

## A flexible way to execute assertions

One technical detail I liked was Uber's use of Abstract Syntax Trees, or ASTs,
for test assertions.

The reason this is useful is that a large data quality platform cannot rely on
custom logic for every possible validation rule. Over time, that would become
difficult to maintain. By representing assertions in a structured form, the
execution engine can parse and evaluate many different kinds of rules
consistently.

That gives the system flexibility as the number and variety of tests grows.

This is not the loudest part of the architecture, but it is the kind of design
choice that matters over time. Many systems do not fail because the first
version is bad. They fail because every new requirement adds another special
case until the platform becomes fragile.

A structured assertion model gives the system more room to grow without turning
every validation into custom execution logic.

## Row counts are useful, but they are not proof

Another problem Uber has to solve is consistency across data centers.

A fast first check is row count comparison. If Data Center A has one million
rows and Data Center B has one million rows, that gives the system a useful
signal. If the counts do not match, something is clearly wrong.

But row counts only tell part of the story.

Two datasets can have the same number of rows and still contain different
records. That means row counts are useful as a first layer, but they are not
enough to prove the data actually matches.

That is where deeper reconciliation becomes necessary.

Uber also uses Bloom filter-based checks for this problem. A Bloom filter gives
the system a compact way to test whether records from one large dataset appear
in another without running a full expensive comparison every time.

The simple mental model is this: one side has the expected list of records, and
the other side can be checked against a compact representation of that list.
Because Bloom filters are probabilistic, they come with tradeoffs such as
possible false positives. But the practical advantage is that they can move the
investigation from "these datasets disagree" toward "these records may be
missing."

That distinction matters. A vague mismatch tells engineers where to start
looking. A diagnostic signal points them closer to what actually needs to be
fixed.

## Alerting is easy. Preserving signal is hard.

When a system runs a large number of tests, alerts become their own problem.

A monitoring system can be technically correct and still operationally painful.
If it creates too much noise, engineers eventually stop trusting it. Once that
happens, even the important alerts lose power because they arrive through a
channel people have learned to ignore.

Uber handles this with patterns that are designed to preserve signal.

One of them is the sustain period. If a test fails because a pipeline is
temporarily delayed, the system does not have to escalate immediately. It can
mark the dataset as a warning and wait to see whether the issue persists. If the
data catches up within the acceptable window, the incident does not need to
become a full page. If the failure continues, then it escalates.

That is a more practical way to handle real data systems. Not every temporary
failure deserves human attention. Some failures recover naturally. Some are
symptoms of an upstream issue. Some are short delays that should be watched but
not treated as emergencies.

Uber also uses dependency-aware alerting. If an upstream freshness issue
explains downstream completeness failures, engineers should not receive separate
incidents for every symptom. Suppressing downstream alerts in that case is not
hiding the problem. It is protecting the signal.

One root cause should lead to one clear investigation path, not a pile of
duplicate tickets.

That is the difference between a monitoring system people trust and one they
eventually mute.

## Closing the loop matters

Another practical part of Uber's approach is automated incident resolution.

When an alert is created, the system does not simply open a ticket and stop. It
continues rerunning checks with exponential backoff. If the data catches up and
the test starts passing again, the incident can be resolved automatically.

This may sound like a small operational detail, but it matters.

Monitoring systems create work. Every alert, ticket, follow-up, and stale
incident adds operational load. If a system can detect an issue but cannot help
retire the issue when it is no longer relevant, it leaves work behind for humans
to clean up.

Auto-resolution is part of making the quality system sustainable. Engineers
should spend more time investigating problems that need judgment and less time
closing tickets for systems that already recovered.

## The bigger lesson

In Uber's write-up, they reported that UDQ detected roughly 90% of data quality
incidents before those incidents impacted downstream systems or business
metrics.

That number is impressive, but the system design behind it is the more useful
takeaway.

A freshness check by itself is small. A row count by itself is small. A
duplicate rule by itself is small. An alert suppression rule by itself is small.

But when those pieces are standardized, generated automatically, connected to
lineage, reconciled across systems, filtered through alert logic, and tied to
incident resolution, they become something much more valuable.

They become infrastructure for trust.

That is the difference between having data quality checks and having a data
quality reliability system.

And I think that distinction matters. Data quality is not just about whether
dashboards are green. It is about whether people can trust the data enough to
make decisions with it. It is about whether a model is receiving reliable
inputs, whether a business metric reflects reality, and whether teams can stop
arguing about which number is correct.

## Why this matters beyond Uber

This is not only an Uber story.

Any company that depends on machine learning, real-time analytics, automated
decisions, or executive dashboards has some version of this problem. The scale
may be different, but the pattern is familiar.

Data can be late. It can be incomplete. It can be duplicated. It can disagree
across systems. And the failure may not be obvious at first.

A model becomes slightly less accurate. A metric shifts in a way nobody
understands. A stakeholder asks why two reports do not match. A team starts
keeping a private spreadsheet because the official source no longer feels
dependable.

That is how trust starts to erode.

And once people stop trusting the data, they build workarounds. They ask
multiple teams for the same number. They maintain side reports. They make
decisions based on instinct because the system of record no longer feels
reliable.

That is why data quality is not only a technical problem. It is also an
organizational trust problem.

The real question is not whether a company has checks somewhere. The better
question is whether those checks are part of a system that can actually protect
trust.

Can the system detect bad data before downstream consumers feel it? Can it prove
that critical data movement is correct? Can it separate a temporary delay from a
real incident? Can it avoid creating ten alerts for one root cause? Can it close
the loop when the issue resolves?

In small systems, you can sometimes get away with hope. In large systems, hope
does not scale.

You need definitions, tests, reconciliation, lineage, alert logic, and incident
workflows working together. Most of all, you need a system that keeps asking
hard questions before the business pays for easy assumptions.

That is my biggest takeaway from Uber's approach.

Bad data is not just something to clean up later. It is something to defend
against before it turns into bad decisions.

And trust needs more than green dashboards.

Trust needs evidence.
