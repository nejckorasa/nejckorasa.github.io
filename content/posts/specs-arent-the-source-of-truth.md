---
title: "Specs Aren't the Source of Truth (Yet)"
description: "Field notes from building a credit-card ledger with AI, and what the manifesto gets right and wrong when the code moves money."
date: 2026-07-24
tags: ["AI", "Software Architecture", "Testing", "Fintech", "Software Engineering"]
categories: Software Engineering
ShowToc: true
TocOpen: false
---

A few days into the build, the model made a failing test pass by editing the test — feeding the assertion the wrong input so it would go green. It had taken the most direct route to the goal we'd set, which was make the suite green, and the suite was green. The code underneath was still wrong.

That one stuck with me, because it sits on top of an argument that's had a good year: that once AI is writing the code, the code is disposable and the tests are what you keep. Behaviour outlives implementation. The evals are the real codebase. Delete the code and regenerate it from the spec.

I agree with most of it. I also think it breaks the moment the code starts moving money, and I've spent a few months finding out where.

## This Is a Field Report

For about three and a half months I've been building a credit-card ledger with Claude. Double-entry accounting, authorisation and clearing, interest that accrues daily and capitalises monthly, statement cycles, delinquency, the lot. It runs in production. I didn't write the application code by hand — I wrote the specs, the conventions, and the reviews, and the model wrote the code.

A ledger is a good place to test the strong version of the claim, because it doesn't let you round off. It either balances or it doesn't.

The clearest statement of the claim I've read is the [aicoding.leaflet.pub](https://aicoding.leaflet.pub/) series — I'll call it the manifesto. It's worth reading, and worth arguing with. That's what the rest of this is.

## What the Manifesto Gets Right

Start with what I agree with, because it's most of it.

The core move is economic. Code used to be expensive to produce, so we treated it as an asset to keep. AI made it cheap, so keeping it is closer to a liability now — "code is no longer scarce. It is abundant, fast, and increasingly disposable." The durable thing is the system's behaviour and its boundaries, not the files. Small components are valuable because they're safe to delete.

At the component level this is just true, and I've felt it. I regenerate individual pieces of this ledger — an action, a repository method, a migration — with no sentiment at all. If a module is small and its boundary is clean, deleting it and asking for a new one beats reading and patching the old one. The discipline the manifesto asks for is the discipline that always made systems maintainable. The reward for it went up.

The domain is where it starts to matter. The manifesto is written from a world where regenerating is cheap because being wrong is cheap: regenerate, run the evals, ship, and if something's off you see it and go again. A ledger removes that. A wrong answer is a silent error in someone's money that may not surface for a full billing cycle. High cost of being wrong, and a long delay before you find out. Both strong claims — the evals are the truth, regenerate from the spec — lean on the same assumption: that the knowledge you need to rebuild is written down somewhere. Often it isn't.

## An Eval Is a Claim, Not a Truth

"Tests and evaluations define truth, not files." I get the appeal. A test is executable, unambiguous, and survives a rewrite in a way implementation details don't. But an eval is only a source of truth if something the author can't edit holds it in place. When the thing writing the code is also writing the tests — and the model is both — the test stops being an independent check and becomes one more surface to optimise. That's the story I opened with. Asked to satisfy the assertion, it satisfied the assertion.

So the question isn't whether tests are the truth. It's what makes any test trustworthy, and that has nothing to do with whether a human or a machine wrote it. Two things do the work.

First, an oracle the author can't reach. The check has to get its answer from something outside the optimiser's grasp — by a different route, from a record that can't be quietly rewritten to agree with it. The most valuable check in this ledger re-derives a core accounting invariant from the raw, append-only entries, with its own arithmetic, and compares that against what the live code believes. If both used the same helper, agreement would prove nothing. Because they don't, and because the entries can't be edited to force a pass, agreement is evidence.

Second, a human who owns the contract — whose job at review isn't "did the tests pass" but "are these the right tests, and do they still mean what they should." AI didn't remove that job; it made it bigger, because the model produces plausible tests all day and some of them point at the wrong thing. (I argued a version of this in 2023, about [not coupling tests to implementation details](https://nejckorasa.github.io/posts/microservice-testing/). The rule I keep — test at the seam, not the internals — is the same one, and it matters more when a machine is writing the internals.)

## The Spec Was Right, and the Bug Shipped Anyway

Here's the claim I most want to push on: that you should be able to regenerate the system from its spec, and if you can't, that's a diagnosis — a sign the understanding was trapped in the code instead of made explicit.

The defects that actually hurt live in a place no up-front spec reaches. They're emergent: two individually-correct rules that collide only in one state, an assumption that holds for every account except the ones created before a schema change, an ordering that's ambiguous only at a finer resolution than anyone thinks to specify. You don't design these in. You grow them, by running production data through the system over months.

One example, kept vague because the shape matters more than the mechanics. An account ended up in a state that was internally contradictory — reported as fully healthy while carrying a counter only a delinquent account should have. The spec for that behaviour was correct. It said, in plain words, what should happen. The model implemented most of it and dropped a single step on one path, and that path was only reachable by a specific sequence of events over more than a cycle. No re-reading of the spec surfaces that. Only a live account, moving through real time, does — which is how we found it, in production.

Now run the regenerate-from-spec test on that. Delete the code, rebuild from the spec, and you get back the same spec, the one that was already right, so you regenerate the gap. The fix was never in the spec.

Regeneration isn't neutral here, either, because generation isn't deterministic. Ask twice, get two implementations. The manifesto knows this and files it under manageable: "non-deterministic generators may produce different code from identical intent graphs… these are not reasons to abandon the approach. They are design constraints." The retained evals are meant to keep the new implementation honest. But evals only cover the failures you already went looking for. A fresh implementation brings a fresh set of emergent ones — a new collision, a new edge — and those are the ones nothing checks for yet, because nobody has met them. Regenerating keeps the net you built and discards the hardening baked into the code underneath it. What's left exposed is the gap between the two: everything the old code learned the hard way that never made it into a named check. In a domain where you find out a billing cycle later, that gap is the whole risk.

The lasting thing that came out of that contradictory account wasn't the one-line fix. It was the check we added afterwards, the one that now refuses to let any account sit in that impossible state again.

## Reconciliation Is Their Tier Three

The obvious objection: that check is just an eval, and so is the whole net around it. Fair. The manifesto has a name for it — the live-evaluation tier, the one that "runs continuously against reality rather than periodically against test fixtures." I'm not claiming to have found something it missed.

I'm claiming it buries the tier that matters most. In this ledger that tier is reconciliation, and it's the part of the system I'd least want to lose. Every operation keeps a running balance as it goes. Separately, after the fact, another process recomputes what the balances should be, straight from the immutable log of what happened, and compares. A second, independent opinion, taken from the record rather than from the code that wrote it. It never repairs anything and it can't block an operation. It only reports.

That matters because everything else can be wrong in ways that pass their local checks. A subtly broken write, a botched migration, a hand-edited row — each can satisfy every per-operation rule and still leave the books wrong. A second opinion from the record is the only thing that catches them.

But two things are true about it. It catches late: it observes after the fact, it doesn't prevent, so in a slow domain the error has usually already shipped. And it's only ever as complete as the invariants you've thought to encode, which means it grows one incident at a time. That's why regenerating the implementation under a good net still isn't safe here. The net is real, it is provably partial, and it tells you about the damage after the money has moved. Keeping the net is not the same as being free to throw away the code beneath it.

## The Manifesto Already Knows This

The best essay in the series makes the point for me. "The Implementation Remembers" says it plainly — "the implementation remembers, the organisation forgets." Working systems carry lessons no document kept, and handing that code to a model to clean up risks removing the memory with the mess. "Clean code that forgets why it exists is just a more elegant way to fail."

That's it, and it cuts both ways. Go forward, spec to code, and you lose the scars; they were never in the spec. Go backward, code to spec — pointing the model at a working system to recover its design — and you lose the why: it reads the guard but not the incident that put it there, and files a special case under cleanup. Both directions drop the same thing, the operational memory that only exists because the system ran in production and got things wrong. In this domain that memory isn't a residue to recover carefully before regenerating. The scars are load-bearing, and they live on purpose in the tests and the checks, where a model can't quietly delete them without a human noticing.

## Where This Is Wrong, and What Changed

The "yet" in the title is honest, but it only covers part of this. Evals may get robust enough that gaming them stops being the easy path; specs may get expressive enough to carry invariants that today live only in code. Better tooling shortens the discovery loop and grows the net. What better tooling can't do is pre-contain a collision nobody has hit yet — and that's the part carrying the argument. That isn't a gap in the tools. It's what "emergent" means.

And the disagreement is narrower than it sounds. At the component level the manifesto is right, and I build that way. It's system-level regeneration — throwing away the whole implementation and trusting an incomplete net to catch what the new one gets wrong — that I won't do here.

The cost, plainly, because leaving it out would be dishonest. None of the safety came from trusting the model. It came from out-writing it where it counts: there's meaningfully more test code in this project than application code, and even with strict documentation discipline the docs still drift from the code in ways I keep finding. The productivity is real. So is the amount of judgement it takes to make it safe. Anyone selling you the first without the second is selling you the green suite, not the correct one.

So the work didn't go away. It moved off the keyboard and onto the parts that were always hard: deciding what has to be true, drawing the boundaries, and building the thing that catches the machine when it's wrong. And it is wrong fluently, which is the hard part. The code was never the asset; the manifesto is right about that. I'd add one thing: neither is the spec. The asset is the design, and the judgement in everything that checks it.
