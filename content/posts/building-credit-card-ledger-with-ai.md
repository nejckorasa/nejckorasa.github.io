---
title: "Building a Credit Card Ledger with AI: Why Specs Aren't the Source of Truth (Yet)"
description: "I built a credit-card ledger in production with AI. Here's what only running it taught me about where the truth actually lives when the code moves money."
date: 2026-07-24
tags: ["AI", "Software Architecture", "Testing", "Fintech", "Software Engineering"]
categories: Software Engineering
ShowToc: true
TocOpen: false
---

A few days into the build, the model made a failing test pass by editing the test. It fed the assertion the wrong input, the suite went green, and the code underneath was still wrong.

I've spent about three and a half months building a credit-card ledger with Claude: double-entry accounting, authorisation and clearing, interest that accrues daily and capitalises monthly, statement cycles, delinquency. It runs in production at a regulated lender, on real customer balances. I didn't hand-write the application code — I wrote the specs, the conventions, and the reviews, and I read every line the model produced. Close enough to direct it, close enough to catch it.

There's an appealing idea this year, argued most sharply in the [aicoding.leaflet.pub](https://aicoding.leaflet.pub/) essays: code is cheap now, so treat it as disposable, keep the spec and the evals, and regenerate the code beneath them at will. Building a ledger — which either balances or it doesn't — is a good way to find where that holds. Most of it does. It breaks where the money is.

## The Bug the Spec Couldn't Hold

At the component level, disposable code is simply true, and I've felt it: I throw away and regenerate an action, a repository method, a migration without a second thought. The durable thing is the behaviour and the boundaries. The files are scaffolding.

It breaks on one word: *regenerate*.

The bugs that actually hurt are emergent. Two correct rules collide in a state nobody pictured; an assumption holds for every account except the ones created before some migration; an ordering turns ambiguous only at a resolution finer than anyone wrote down. You don't design these in. You grow them, by running real data through the system for months.

One we hit: an account carries a counter meant to be non-zero only while it's delinquent. One path out of delinquency didn't reset it, so an account could read as fully healthy while still holding the marker of a delinquent one — the state and its own audit counter flatly disagreeing. The spec was right; it said reset on exit. The model implemented exit on the paths it thought of and missed one, reachable only after a specific multi-cycle sequence. Re-reading the spec never surfaces that. A live account, ageing day by day through the sequence, does.

Delete the code, rebuild from that same correct spec, and you regenerate the same gap. The fix was never in the spec to regenerate from. You don't even get the same code back: set the sampling temperature to zero and the next model version or a reworded prompt still hands you a different implementation, with a different set of places to be wrong. Regeneration doesn't reproduce the bug you fixed. It clears the board and deals a fresh hand of the emergent kind — the ones nothing checks for yet.

## "Just Update the Spec, Then"

The obvious reply, and the manifesto's real answer to *does the regenerated code match the spec*: run the evals. Write down what production teaches you, and the eval set fills in until regeneration is safe.

Half right. The delinquency fix is one line — the counter is non-zero if and only if the account is delinquent — and that single invariant constrains every possible implementation without caring how the code is written. Invariants stay small; the spec never swells into code-in-prose.

The trouble is *which*. You only knew to write that invariant after an account broke it in production. Every check in the net earned its place through a collision, an incident, a near-miss, so the set is compressive but never complete, and it's thin in exactly the directions production hasn't taken you yet. The manifesto half-builds the fix. It has the detector — evaluation that runs "continuously against reality" — and concedes the loop never closes: "intent and reality can diverge even when all explicit tests pass." What it never adds is the actuator, the step that writes each hard-won lesson back into intent. That still happens by hand, after the fact, by someone who watched it break.

So on this project the specs are frozen. Each is a dated record of why we chose something; a new invariant gets a new spec, never an edit to an old one. What tracks the system as it stands is a living reference generated from the code. Truth runs two ways that never meet — spec → code when we build, code → reference after — with the code in the middle as the source of truth. The thing you'd regenerate from is neither.

## You Can Burn the Code, Not the Data

Suppose you had every invariant. You still couldn't regenerate this system, because it has been running.

The manifesto guards the wire — the API and event schemas other services depend on — and rightly calls them the thing you can't regenerate in isolation. Then it stops at the boundary and never looks at the floor: the data already written down. A ledger that's run for months holds millions of rows and events in a specific shape — column types, enum spellings, the exact layout every past record was serialised into. Regenerate the code and it still has to read that history back, precisely, or it can't tell you yesterday's balance.

Schema, migrations, storage internals are pure implementation detail — the "how" the disposable-code view waves away. They are also permanent, deterministic, and load-bearing, because the data they describe outlived the code that wrote it. You can burn the implementation and rebuild it; the database is there in the morning, and the new code has to match it. A spec complete enough to regenerate a system that still reads its own history has to pin all of that — at which point it isn't a spec. It's the schema.

## What I'd Keep

So what survives, if it's neither the code nor the spec? The design, the conventions, and the checks that watch them.

The spec was only ever a slice. It carries the business rules — what interest accrues, when a balance reclassifies — and says nothing about how the code is structured, tested, deployed, or how it behaves the first time production surprises it. A model writes plausible architecture and plausible tests and gets them subtly wrong in ways only a standard, enforced on every change, will catch. That half lives in conventions and review.

The heaviest check is reconciliation, old and boring enough that bookkeepers had it centuries before computers. That's the point: an old, boring control is the one that holds against a code-writing model, where cleverer checks fold. Every operation keeps a running balance as it goes; separately, afterwards, another process recomputes what the balance should be from the immutable log, with its own arithmetic, and compares. The log can't be edited to force agreement, so agreement is evidence. It catches what no per-operation rule would — the broken write, the botched migration, the hand-edited row that still leaves the books wrong.

Why it holds against a model is worth naming. The gamed test fell because the model owned both sides, the code and its check. Reconciliation's answer comes from data the model didn't write, and it's built independently — derived from the record and first principles, ideally in a pass that never read the write path, so a shared blind spot can't excuse a shared bug. The other half is a person who owns the contract, who decides what a check should *mean* rather than whether it's merely green. Matt Pocock puts the AI-era version well: ["You own the interface. AI owns the implementation. Tests keep it honest."](https://www.aihero.dev/how-to-make-codebases-ai-agents-love)

No single check proves the code means what we meant. The confidence is the overlap: seam tests, the independent recompute, a human on the contract, production surfacing the rest. Conformance isn't a gate you pass once. It's a net, and it only tightens.

## What's Permanent, and What's Just "Yet"

Most of my disagreement is a "yet". Evals may get robust enough to stop being gameable, specs expressive enough to hold what today lives only in code, tooling good enough to shorten the loop. Fuzzing and property tests already reach states no human would — but they only fail on a property you thought to assert. A fuzzer would have caught that delinquency sequence in minutes if the invariant had existed to check against; without it, it sails through. The permanent gap is the unwritten check, and you learn which one was missing by shipping without it.

One honest note, because the productivity is real and easy to oversell. None of the safety came from trusting the model; it came from out-writing it where it counts. There's more test code in this project than application code, and those checks exist because the model is confidently, fluently wrong often enough to need them. Sell the speed without the judgement and you've shipped a green test suite, not a correct ledger.

The disposable-code view has the headline right: the code was never the asset. Building this changed one line of it for me — neither is the spec. Keep the design, the conventions, and the handful of checks a model can't talk its way past, the ones that recompute the truth from a record it never got to write. The rest you can regenerate.
