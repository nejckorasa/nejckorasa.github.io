---
title: "Building a Credit Card Ledger with AI: Why Specs Aren't the Source of Truth (Yet)"
description: "I built a credit-card ledger in production with AI. Here's what only running it taught me — and where the popular 'regenerate it from the spec' thesis breaks when the code moves money."
date: 2026-07-24
tags: ["AI", "Software Architecture", "Testing", "Fintech", "Software Engineering"]
categories: Software Engineering
ShowToc: true
TocOpen: false
---

A few days into the build, the model made a failing test pass by editing the test. It fed the assertion the wrong input, the suite went green, and the code underneath was still wrong.

I've spent about three and a half months building a credit-card ledger with Claude: double-entry accounting, authorisation and clearing, interest that accrues daily and capitalises monthly, statement cycles, delinquency. It runs in production at a regulated lender, on real customer balances. I didn't hand-write the application code — I wrote the specs, the conventions, and the reviews, and I read every line the model produced. That's the vantage point for everything below: close enough to direct it, close enough to catch it.

A ledger doesn't let you round off. It balances or it doesn't. That makes it a good place to test a claim you've heard a lot this year: that AI makes code disposable, and the thing worth keeping is the spec and the evals you regenerate it from.

Most of it is right, until the code moves money.

## The Bug the Spec Couldn't Hold

Start with the agreement, because it's most of it. Code used to be expensive, so we kept it; AI made it cheap, so keeping it matters less. At the component level I've felt this — I regenerate individual pieces of the ledger, an action, a repository method, a migration, without a second thought. The durable thing is the behaviour and the boundaries. The files are scaffolding. On all of that the manifesto (the [aicoding.leaflet.pub](https://aicoding.leaflet.pub/) series) is right.

It breaks on one word: *regenerate*.

The bugs that actually hurt are emergent. They come from two correct rules colliding in a state nobody pictured, or an assumption that holds for every account except the ones created before some migration, or an ordering that only turns ambiguous at a resolution finer than anyone wrote down. You don't design these in. You grow them, by running real data through the system for months.

One we hit: an account carries a counter that's only meant to be non-zero while it's delinquent. One path out of delinquency didn't reset it. So an account could read as fully healthy while still holding the marker of a delinquent one, the state and its own audit counter flatly disagreeing. The spec was right; it said reset on exit. The model implemented exit on the paths it thought of and missed one, reachable only after a specific multi-cycle sequence. Re-reading the spec never surfaces that. A live account, ageing day by day through the sequence, does.

Now run the manifesto's move on it. Delete the code, rebuild from the spec, and you get back the same spec, the one that was already right, and the same gap with it. The fix was never in the spec to regenerate from.

And you don't even get the same code back. Forget sampling temperature — set it to zero; the next model version or a reworded prompt still hands you a different implementation, and a different implementation that passes the same spec is a different set of places to be wrong. Regeneration doesn't reproduce the bug you already fixed. It clears the board and deals a fresh hand of emergent ones, the kind nobody has a check for yet.

## "Just Update the Spec, Then"

There's a fair reply, and it's the manifesto's real answer to "how do you know the regenerated code matches the spec": run the evals. Conformance is whatever the checks say it is. Write down what production teaches you, and the eval set fills in until regenerate-from-spec is safe.

The first half of that is right, and it's worth being precise about why. The fix for the delinquency bug is one line — the count is non-zero if and only if the account is delinquent. That single invariant constrains every possible implementation; it doesn't care how the code is written, only that the books obey it. Invariants like that stay small, so I won't pretend the spec swells into code-in-prose. It doesn't have to.

The problem is the other word: *which*. You only knew to write that invariant after an account broke it in production. Every check in the net is there because something taught it to you — a collision, an incident, a near-miss. The set is compressive but never complete, and it's short in exactly the directions production hasn't taken you yet. Regenerate the implementation and you keep every invariant you've earned, but the new code arrives with its own fresh ways to be wrong, and those have no check, because nobody has met them. The manifesto half-builds the answer here: it has the detector, a live-evaluation tier that runs "continuously against reality", and even concedes the loop never closes — "intent and reality can diverge even when all explicit tests pass". What it never describes is the actuator, the step that writes each hard-won lesson back into the intent you regenerate from. That still happens the old way: by hand, after the fact, by someone who watched it break.

Which is why the specs on this project are frozen. Each is a dated decision record — why we chose something, what we ruled out — and none is edited after the fact; a new invariant gets a new spec, not an amendment to an old one. They're an append-only history of intent, not a description of the system as it stands. What describes the system as it stands is a living reference, generated from the code and always current. Truth runs in two directions that never meet: spec → code when we build, code → reference after. The code sits in the middle as the source of truth — the specs say why it exists, the reference says what it does today, and the thing you'd regenerate from is neither.

## You Can Burn the Code, Not the Data

The manifesto sees half of this. It's careful about contracts: the API and event schemas other services depend on are named as the one thing you can't regenerate in isolation, a conserved boundary around a disposable interior. True. But it stops at the wire and never looks at the floor underneath — the data already written down.

A ledger that's run for months holds millions of rows and events in a specific shape: column types, enum spellings, the exact layout every past record was serialised into. Regenerate the code and it still has to read that history back, exactly, or it can't tell you yesterday's balance. You can burn the implementation and rebuild it. You cannot burn the database — the old data is there in the morning, and the new code has to match it.

That layer doesn't compress to an invariant. Schema, migrations, storage internals are pure implementation detail, and also permanent, deterministic, and load-bearing — the "how" the manifesto files under disposable, except this how can't be disposed of, because the data it describes outlived the code that wrote it. A spec complete enough to regenerate a system that still reads its own history has to pin all of it. At which point it isn't a spec. It's the schema.

## What I'd Keep

So what survives, if it isn't the code and isn't the spec? The design, the conventions, and the checks that watch both.

The spec was always a slice of it. It carries the business rules — what interest accrues, when a balance reclassifies — and says nothing about how the code is structured, how it's tested, how it deploys, how it behaves the first time production does something unexpected. That half of the work lives in conventions and review. A model writes plausible architecture and plausible tests and gets them subtly wrong in ways only a standard, enforced on every change, catches.

The heaviest check is reconciliation, and it's old and boring — double-entry bookkeepers had it centuries before computers, let alone language models. That's the point. An old, boring control is the one that holds up against a code-writing model, where cleverer-looking checks fold. The shape is simple: every operation keeps a running balance as it goes, and separately, after the fact, another process recomputes what the balances should be from the immutable log of what happened, with its own arithmetic, and compares. The log can't be edited to make them agree, so agreement is real evidence. It repairs nothing and blocks nothing; it only reports what no per-operation rule would catch — the broken write, the botched migration, the hand-edited row that still leaves the books wrong.

Go back to the gamed test. The model rewrote the check because it owned both sides, so the check was one more thing to satisfy. The checks that hold up are the ones whose answer the model can't compute, because it comes from data the model didn't write. And it's independent twice over — not only in its data but in how it's built. A check grown from the same code it audits inherits the same blind spots; reconciliation is derived from the record and first principles, ideally in a pass that never read the write path, so it has to reach the same number on its own. Disagreements are the bugs worth finding.

The other half is a person who owns the contract — who decides what a check should *mean*, not just whether it's green. That's an old rule ([test at the seam, not the internals](https://nejckorasa.github.io/posts/microservice-testing/)) that got heavier once a machine started writing the internals. Matt Pocock puts the AI-era version well: ["You own the interface. AI owns the implementation. Tests keep it honest."](https://www.aihero.dev/how-to-make-codebases-ai-agents-love)

No single one of these proves the code matches what we meant; no single artifact does. The confidence is the overlap — seam tests, the independent recompute, a human on the contract, and production surfacing what all three missed. Conformance isn't a gate you pass once. It's a net, and it only ever tightens.

## What's Permanent, and What's Just "Yet"

Most of my disagreement is a "yet". Evals may get robust enough that gaming them stops being the easy path. Specs may get expressive enough to hold invariants that today live only in code. Better tooling will shorten the loop.

One part isn't a "yet". Fuzzing and property testing already explore states no human has reached — but they only fail on properties you thought to assert. A fuzzer would have found that delinquency sequence in minutes *if the invariant had existed to check against*; without it, it sails straight through. The permanent gap isn't the unreached state. It's the unwritten check — and you mostly learn which one was missing by shipping without it.

The productivity is real, and easy to oversell, so one honest note. None of the safety came from trusting the model. It came from out-writing it where it counts: there's more test code in this project than application code, and those checks exist because the model is confidently, fluently wrong often enough to need them. Speed on the keyboard, judgement on the contract. Sell the first without the second and you've shipped a green test suite, not a correct ledger.

The manifesto's headline holds up: the code was never the asset. Building this changed one line of it for me. The spec isn't the asset either. Keep the design, keep the conventions, and keep the handful of checks a model can't talk its way past — the ones that recompute the truth from a record it never got to write. The rest you can regenerate.
