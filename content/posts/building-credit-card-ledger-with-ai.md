---
title: "Building a Credit Card Ledger with AI: Why Specs Aren't the Source of Truth (Yet)"
description: "I built a credit-card ledger in production with AI. Here's what only running it taught me — and where the popular 'regenerate it from the spec' thesis breaks when the code moves money."
date: 2026-07-24
tags: ["AI", "Software Architecture", "Testing", "Fintech", "Software Engineering"]
categories: Software Engineering
ShowToc: true
TocOpen: false
---

A few days into the build, the model made a failing test pass by editing the test. It fed the assertion the wrong input so the suite would go green. The suite went green. The code underneath was still wrong.

I've spent about three and a half months building a credit-card ledger with Claude — double-entry accounting, authorisation and clearing, interest that accrues daily and capitalises monthly, statement cycles, delinquency. It runs in production, moving real money. I didn't hand-write the application code; I wrote the specs, the conventions, and the reviews, and the model wrote the code.

A ledger doesn't let you round off. It balances or it doesn't. So it's a good place to test the claim that's had a good year: that AI makes code disposable, and the thing worth keeping is the spec and the evals you regenerate it from.

Most of that is right. It breaks the moment the code moves money. Here's where.

## The Bug That Only Production Found

Start with the agreement, because it's most of it. Code used to be expensive, so we kept it; AI made it cheap, so keeping it is closer to a liability. At the component level this is just true — I regenerate individual pieces of this ledger, an action, a repository method, a migration, with no sentiment at all. The durable thing is the behaviour and the boundaries, not the files. The manifesto — I'll point at the [aicoding.leaflet.pub](https://aicoding.leaflet.pub/) series — is right about all of that.

It breaks on one word: *regenerate*.

**The bugs that actually hurt are emergent.** Two individually-correct rules that collide in one state. An assumption that holds for every account except the ones created before a schema change. An ordering that's ambiguous only at a finer resolution than anyone specifies. You don't design these in. You grow them, by running production data through the system for months.

One example, kept vague because the shape matters more than the mechanics. An account ended up internally contradictory — reported as fully healthy while carrying a counter only a delinquent account should have. The spec was correct; it said in plain words what should happen. The model implemented most of it and dropped one step on one path, reachable only by a specific sequence of events over more than a cycle. No re-reading of the spec surfaces that. Only a live account, moving through real time, does — which is how we found it, in production.

Now run the manifesto's test on it. Delete the code, rebuild from the spec, and you get back the same spec — the one that was already right — so you regenerate the same gap. **The fix was never in the spec.**

And it's worse than a wash, because generation isn't deterministic. Ask twice, get two implementations. Regenerate and you keep the eval net you built, but you throw away the hardening baked into the old code — every guard it grew the hard way that never became a named check. The new implementation arrives with its own fresh emergent bugs, the ones nothing tests for because nobody's hit them yet. In a domain where you find out a billing cycle later, that gap is the whole risk.

## "Just Update the Spec, Then"

The obvious reply: when production teaches you something, write it back into the spec. Do that enough and the spec converges on completeness, and regenerate-from-spec works fine.

Two problems, and the manifesto half-sees both.

**The detection half is fair**, and I won't pretend otherwise. The manifesto builds for exactly this — a live-evaluation tier that "runs continuously against reality rather than periodically against test fixtures." Monitoring catches what tests miss. Granted. But detecting a gap isn't the same as closing the loop, and nowhere does the argument write the lesson back into the intent. Even where you do it by hand, notice what happened: the knowledge came from running, not from specifying. The spec became a transcript of what production taught you, written after the fact. That's a changelog of scars, not a source of truth.

**The second problem ends the fantasy.** Push enough detail back into the spec to actually pin a *correct* ledger — every edge, every ordering rule, every precision detail, every guard — and the spec stops being a spec. It becomes the implementation again, in prose, with worse tooling and no type checker. The manifesto concedes the empirical half of this and declines to follow it home: durable evaluations, it admits, are

> harder than writing the code they specify.

Once your specification is more expensive than your code and still growing, "regenerate from the spec" hasn't removed the hard part. It's renamed it.

## What I'd Actually Keep

Not the code, and not the spec. The design — and the thing that watches it.

Go back to the gamed test. The model edited it because nothing stopped it: it owned the code and the check both, so the check was one more surface to satisfy. **The checks that survive contact with a model are the ones it can't quietly satisfy** — the ones whose answer comes from somewhere it can't reach.

In this ledger that's reconciliation. Every operation keeps a running balance as it goes. Separately, after the fact, another process recomputes what the balances should be — from the immutable log of what happened, with its own arithmetic — and compares. The entries can't be edited to force a pass, so agreement is evidence. It never repairs anything and it can't block an operation; it only reports. That's what catches the subtly broken write, the botched migration, the hand-edited row — the errors that satisfy every per-operation rule and still leave the books wrong.

The other half is a human who owns the contract — deciding what a check should *mean*, not just whether it's green. That's an old rule — [test at the seam, not the internals](https://nejckorasa.github.io/posts/microservice-testing/) — that only got more load-bearing once a machine started writing the internals.

The manifesto has a name for the net: the live-evaluation tier. Fair — reconciliation is exactly that, and I'm not claiming to have found something it missed. I'm saying it's the part it underweights, and the part I'd least want to regenerate. Two things stay true of it. It catches *late* — after the money's moved — so it reports damage, it doesn't prevent it. And it's only ever as complete as the invariants you've thought to encode, so it grows one incident at a time. The hard-won ones live in those checks on purpose, where a model can't simplify them away without a human noticing.

## What's Permanent, and What's Just "Yet"

Most of my disagreement is a "yet." Evals may get robust enough that gaming them stops being the easy path; specs may get expressive enough to carry invariants that today live only in code. Better tooling shortens the loop. One part isn't a "yet," and it's the part carrying the argument: no tool can pre-contain a collision nobody has hit. That isn't a gap in the tooling. It's what *emergent* means, and it doesn't go away.

The honest cost, because leaving it out would sell the trick. None of the safety came from trusting the model. It came from out-writing it where it counts — there's more test code in this project than application code, and those checks exist because the model is confidently, fluently wrong often enough to need them. The productivity is real. So is the judgement it takes to make it safe. Sell the first without the second and you've sold the green suite, not the correct one.

The manifesto's headline is right: the code was never the asset. I'd add the line it stops short of. Neither is the spec. The asset is the design, and the judgement in everything that checks it.
