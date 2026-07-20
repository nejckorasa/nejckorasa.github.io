---
title: "Temporal in Production: Sharp Edges & Good Practices"
description: "If your team is moving into distributed, asynchronous, event-driven systems, Temporal is a good place to learn the habits the shift demands — its sharp edges and good practices, learned from running it in production."
date: 2026-07-17
tags: ["Temporal", "Distributed Systems", "Durable Execution", "Software Architecture", "Microservices", "Event Driven Architecture", "Asynchronous Processing"]
categories: Software Engineering
ShowToc: true
TocOpen: false
---

When a team moves from a monolith into microservices and event-driven, asynchronous systems, it inherits a class of problems that used to be someone else's: work that fails halfway through, steps that must not run twice, calls that return before the work is done. [Temporal](https://temporal.io) is a durable execution engine that handles a lot of this - you define a multi-step process, and it guarantees the process runs to completion even when workers crash in the middle.

I've spent the better part of a decade building distributed systems in the money-movement core of banks - ledgers, payments, credit cards - a lot of it on Temporal, from short request-triggered workflows to ones that stayed open for weeks. This is the high-level guide I'd give a team making that jump: the principles worth internalising before you ship, not a full tutorial. Most of them aren't really about Temporal. They're the habits the async shift demands - Temporal just punishes you quickly when you skip one.

## Durable Execution: The Problem It Solves

Distributed work fails in the middle. You call service A, it succeeds. You call B, it times out. The pod dies before C. Now you have half-finished work and no memory of how far you got. The usual fix is a pile of status columns, a cron job to find stuck rows, and retry logic hand-rolled for every step.

Temporal's promise is that any process you start runs to the end. The runtime picture: there's a Temporal service (its own cluster), and your app runs [**worker**](https://docs.temporal.io/workers) processes that poll it and execute your code. As a workflow runs, Temporal records every step to an event **history**. If a worker dies, another picks the workflow up and *replays* that history to rebuild state, then carries on from where it left off, retrying anything that failed.

The history is the source of truth, and it survives the crash. Most of the rules below fall out of that one fact.

## The Golden Rule: Workflows Decide, Activities Do

There are two kinds of code in Temporal.

- **The workflow** is orchestration. It says *what* runs and in *what order*: call A, then B, wait, then C.
- **Activities** are the actual work. Each does something real - a service call, a DB write, a Kafka publish - and each is retried independently.

The rule: all your business logic, and all contact with the outside world, lives in activities. The workflow stays a boring, readable list of steps.

This falls out of [replay](https://docs.temporal.io/workflows). A worker rebuilds a workflow by re-running its code against the recorded history, so that code must be **deterministic**: re-run against the same history, it makes the same decisions. No raw clock reads, no `rand()`, no network calls in the workflow itself (the SDK gives you deterministic versions of time and randomness). Everything non-deterministic goes into activities, whose *results* are recorded and replayed.

## Keep Temporal Inside One Service

This is the one I'd fight for hardest, because it's the trap Temporal's own marketing walks you into.

Temporal lets you define a workflow in one service and run its activities anywhere - ten services if you like. One workflow orchestrating your whole estate sounds great, and I've seen a big push on exactly that promise.

It's a coupling trap. The moment an activity lives in another service, that service needs a Temporal integration and a shared namespace it otherwise wouldn't. Worse, its activity's input/output schema is now wired into *your* workflow. A colleague changes the shape of their activity, an old-format result in your history no longer deserializes into the new code, and your in-flight workflows break. You've coupled two teams' deploys through a mechanism neither of them can see.

Draw the boundary at the service instead. The workflow and all its activities are owned by one service. When an activity needs another service, it makes a plain HTTP call or emits a Kafka event - the boring, universal thing everyone understands. Temporal becomes an implementation detail of a single service, and no workflow ever spans two teams.

Account creation is a good example. The whole workflow is four activities:

1. **Mark creation started** - write to our own DB.
2. **Register the account with the payments processor** - HTTP.
3. **Create the account in the ledger** - HTTP.
4. **Mark creation completed and emit an `account_created` event** - write to our DB, publish to Kafka.

The workflow just names the order; each activity does the work and loads what it needs by ID, so rich data never has to ride across a service boundary.

## A Workflow Is Async: Announce the Outcome

If you're coming from request/response, this is the mental shift. Starting a workflow returns almost immediately - it does *not* wait for the work to finish. The work runs in the background on a worker, so the caller can't read the result off the response; the outcome has to come back some other way.

That's why the account-creation workflow ends by emitting `account_created`. The event isn't decoration - it's how anyone learns the account was actually created. Whoever needs to react (send the welcome email, provision the card) subscribes to that event rather than blocking on your call. Great for event-driven systems; a bad fit when the caller needs the answer inside the same request.

Announce the failure path too, but only if you have one. With infinite retries a workflow may just keep going until it succeeds, so there's no terminal failure to report. If instead you give up after N attempts, emit something on that path so downstream isn't left waiting on a success that never comes.

## Activities: Idempotent, Retriable, and the Exception Trap

Temporal retries activities automatically, with exponential backoff and unlimited attempts, until they succeed. You inherit that - you don't ask for it - and two things follow.

**First, idempotency is correctness, not hygiene.** Any activity *can* run more than once, and if running it twice does the wrong thing, it's only a matter of time before you hit it. On a ledger this is concrete: an activity that posts a transaction and runs twice moves the money twice. The fix is the ordinary one behind any retrying caller - dedupe the write on a **stable key tied to the operation's identity** (the transaction ID plus the step), never a fresh UUID per attempt, which would defeat the check. Every activity ends up the same shape: load state by ID, return early if that key already landed, do the work, commit.

**Second, how you signal a failure decides whether it retries.** A raised exception means "retry me," so be deliberate:

- **Transient failure** (timeout, 503): let it raise. It retries and recovers. Stuck beats failed.
- **An expected "no"** (declined, already exists): do not raise. Return it as a normal result and let the workflow continue. Throw an exception for a business outcome that will never change, and you've built an infinite loop on a permanent answer. This is the sharpest edge in the whole topic.
- **A genuinely unrecoverable failure** (bad input): raise a *non-retryable* error so it fails fast.

For "stop or retry forever," my default is to start with infinite retries and monitor: if a workflow has to succeed, that plus a good alert is the simplest thing that works, and it self-heals once you deploy a fix. But infinite attempts is only safe with a per-attempt `StartToClose` timeout. An activity that *hangs* rather than crashes never produces a failure to retry on, so it sits forever. That is the real footgun, not infinite retries.

## Workflow IDs Are a Free Concurrency Lock

You set the workflow ID when you start one, and Temporal guarantees only one workflow with a given ID runs at a time. That's a serialization lock you didn't have to build, and the ID scheme sets the granularity: `account_id-create-account` serializes just that operation, while `account_id` alone serializes *every* operation for that account (at the cost of per-account throughput).

I've used this on an account-lifecycle service in front of a ledger. Certain updates had to be strictly serialized, and keying the workflow on the account ID gave us that with no lock table. The ID is the lock. (It holds only while the workflow is running; reuse after it closes is governed by a policy.)

## Temporal Orchestrates; It Doesn't Hold Your State

Temporal runs the process. It is not where your state lives - neither the live state a workflow is carrying, nor the durable record of what happened. Two habits keep that boundary clean.

**Keep the in-flight context minimal.** Don't load a workflow up with data and pass snapshots down to activities. Have the workflow pass **IDs**, and let each activity load what it needs from the DB when it runs and decide how to proceed (its idempotency check included). A snapshot captured early goes stale, and writing it back clobbers a concurrent update; passing IDs sidesteps that, keeps sensitive data out of payloads, and keeps the event history small.

**Keep persistent state in your own DB, not in Temporal.** Temporal keeps history for a retention window, then it ages out - so it's not what you reach for six months later when someone asks what happened to an account. This covers two things. Your **domain state** (balances, statuses, the data behind reporting and emails) obviously belongs in your database. But so does the workflow's own **progress**: the first activity writes `creation_started`, the last writes `creation_completed`. That leaves Temporal doing purely runtime orchestration, with your DB owning the record.

Writing progress down also gives you the alarm that matters. Temporal's built-in latency metric only fires when a workflow *ends*; there's no metric for how long an *open* one has been running. So we kept our own: an hourly job flagged start rows that hadn't reached `completed` within a threshold. A trivial DB lookup - but only because you wrote the records. (Alert on a meaningful SLA per workflow type too, and on any activity stuck on a huge retry count - an infinite retry hiding a failure that won't resolve itself.)

This is the **dual-write problem**: two writes to two systems (here Temporal and your DB) that can't share a transaction, so a crash between them leaves the two out of sync. You can't make them atomic, so order them to pick the failure you can live with. Start the workflow first - Temporal durably records it before returning - and let its first activity write the "started" row; a crash then just means a workflow that retries until the row lands, not an orphan row no workflow will ever process. The DB-and-Kafka version shows up in the Kafka section below, where the outbox pattern can remove it outright.

## Evolving Workflows: Fix Forward, and the Long-Running Trap

The workflow is the expensive part to change, so keep it small: no clever branching, a straight line, complexity pushed down into activities.

This pays off when you fix bugs. **A bug in an activity is painless:** fix the code, deploy, and the next retry runs the new code - the workflow definition never changed, so determinism is never at risk. **A bug in the workflow is the hard case:** in-flight workflows are replaying old history, and new orchestration code can diverge from it. So keep the brains in the activities and design to fix forward.

That hard case turns permanent with **long-running workflows**, the biggest source of operational pain. I've worked on account-closure workflows that ran for weeks - a regulatory hold before a balance could close. The longer the window is open, the more chances for something underneath to change, and the whole time you're maintaining that code path live. A workflow stuck open from 30 days ago is one whose definition you can't safely retire.

So a weeks-long workflow needs a plan for changing it mid-flight: drain the old ones, or patch in place so old runs take the old path. Temporal softens this - [Continue-As-New](https://docs.temporal.io/workflows#continue-as-new) checkpoints a long workflow into a fresh history to keep it bounded and changeable, and durable **timers** and **signals** model the wait natively instead of a polling loop. But the real lesson is simpler: the risk scales with how long a workflow stays open, so keep them short.

## Know When a Kafka Consumer Is Enough

The obvious question is "why not just use Kafka?" And the honest answer: you *can* orchestrate a multi-step process with Kafka - read a message, do the work, commit the offset, emit the next. The catch is that you build all the machinery yourself: the retries, the dead-letter queues, the backoff, the per-step state, the visibility into where something got stuck.

Temporal gives you that, plus **per-activity retry configuration** you'd hand-roll in Kafka: activity A retries every ten minutes with backoff, B is non-retryable, C caps at five attempts. Durable execution is essentially a Kafka consumer with checkpointing between every step, already written for you.

So when is it worth the extra platform? When a flow needs to **wait**, be **watched**, or has **multiple side effects to checkpoint between**. Below that - one side effect, no waiting, retry-then-dead-letter is fine - a consumer is the right tool and Temporal is overhead.

If you do go the consumer route, the order of your commits is the whole game:

```
read msg → do work → commit DB → emit event → ack input (last)
```

Emit after the DB commit (before it, a failed commit leaves a phantom event), and ack the input last (ack-then-crash loses the work). This is the DB-and-Kafka dual write again: ordering picks the lesser failure, but a crash between the commit and the emit still drops the event. To close that gap outright, use the [transactional outbox pattern](https://nejckorasa.github.io/posts/idempotent-kafka-procesing/) - write the event to an outbox table in the same transaction as the state change, and let a relay publish it afterwards. Temporal gives you this durability out of the box; a consumer makes you build it.

None of this makes Kafka the loser. Where Kafka wins is fire-and-forget domain events: you emit "account created" and don't own what downstream does with it. Temporal is for the workflows you own, where you care about every step reaching the end.

## One Alternative: DBOS

If a separate cluster to run and operate is the part that gives you pause, [DBOS](https://www.dbos.dev) is worth a look. It's a lighter take on durable execution: a library that persists workflow state to a Postgres database you already run, rather than a standalone service you stand up and babysit. You trade Temporal's richer tooling and scale for far less operational surface - a reasonable deal when your workflows are modest and you'd rather not own another cluster.

## Where to Start

Everything above is a consequence of one idea: **the history is the source of truth, and a worker can replay it at any time.** Determinism, keeping logic in activities, idempotent retries, keeping domain state in your own DB - they all fall out of that. Get that, and the rest stops feeling like a checklist.

The one process habit that saves the most pain: resist iterating straight in production. Spin up a local dev server (the [`temporal` CLI](https://docs.temporal.io/cli) gives you one in a command), get a workflow running end to end, and start with something small. And lean on the Web UI - inspecting a workflow's full history, then killing or replaying it by hand, is one of the concrete things a queue-plus-dead-letter setup never gives you.
