---
title: "Spec-Driven Development in Production: What the Spec Can Actually Own"
description: "Three months building a production credit-card ledger with AI. Which parts of a spec survive, which parts rot, and the step the workflow is still missing."
date: 2026-07-27
tags: ["AI", "Spec-Driven Development", "Software Architecture", "Testing", "Software Engineering"]
categories: Software Engineering
ShowToc: true
TocOpen: false
---

I've spent about three months building a credit-card ledger with Claude - balances, authorizations, interest, statement cycles, delinquency - running in production at a regulated lender. I wrote the specs, the conventions and the reviews. The model wrote the application code.

The workflow is dull: a dated spec, a plan, one slice per merge request, a review before the next slice starts, and the team's conventions in a file every agent loads. It works. Two things I didn't expect are worth writing down.

## What Belongs in a Spec

That spec had everything in it: the domain rules, a table of endpoints, a field list per entity, a diagram of how the entities relate. A few months later I had to add a note at the top saying which parts had gone stale. Endpoints never built, field lists drifted, one relationship drawn backwards.

The domain rules were all still correct. Purchase interest accrues quietly during grace and becomes real debt once the account revolves. Repayments go to the highest rate first, interest before principal. Every word still true.

It isn't that the rules were more important. It's that they existed nowhere else, so there was no second copy to drift from. The endpoint table had a copy - the code - and when the two disagreed, nothing broke. Nobody notices a document going wrong, so it goes wrong.

That gives you a simple test for what to put in a spec: **does the code already say this, exactly?**

If no, write it down. Domain rules, business decisions, the reasons behind them. That's where a model needs you most, because nothing in the codebase implies any of it.

If yes, don't describe it - own the real thing. The API boundary and the DB model are both real decisions, so keep them as a contract you write and generate handlers from, and a model you generate migrations from. We generate both from the code today, which stops them drifting but leaves nothing to review: a generated contract always agrees with whatever the code does. I'd rather write them and have the build complain.

Delete the rest. Endpoint tables and entity diagrams read like documentation and behave like liabilities.

## Tests Aren't the Spec

I nearly wrote that the tests are part of the spec. They're not, and the difference caused our worst bug.

A spec says something about every case: *all* repayments go to the highest rate first. A test says something about one case. Tests check the code against the spec; they aren't the thing being checked against.

Our accounts carry a counter that should only be non-zero while the account is delinquent. Four paths lead out of that state and the two payment ones didn't clear it, so an account could look fully current while still carrying the mark - and that stale number made the next cycle look worse than it was.

The tests covered those paths. They asserted the wrong number, with a comment underneath explaining why the counter stuck around. Fixing the code meant editing the assertions.

So the tests weren't missing. They were wrong in the same way the code was wrong, because the same reader wrote both from the same misreading of an unclear spec. That's worse than an agent gaming its tests, and reviewing harder doesn't fix it - I did review them, and I agreed with them.

Writing the test first helps a little: watching it fail proves it *can* fail. It does nothing about a misreading you and the model share.

What caught it was a rule - the counter is non-zero only when the account is delinquent. One line, true of every account, checked against real data every day. Ours lives in the reconciliation pass, which recomputes balances from the recorded history and compares them to the running totals. Two versions of the same number, produced at different times by different code. It only reports; it never blocks or repairs anything.

## The Missing Step

What's missing is a way to know the code does everything the spec says. Not "ask a model to compare them and report the gaps" - that check can't fail on its own, the same weakness as the prose.

The version that works is boring bookkeeping. Every rule in the spec names the thing that enforces it: a test, a type, a database constraint, a daily check. Then the useful report writes itself - which rules have nothing enforcing them. A model is good at proposing those links; it shouldn't be the judge of them.

{{< mermaid >}}
flowchart LR
  SPEC["Spec<br/>a decision, dated"] --> PLAN["Plan<br/>slices"]
  PLAN --> CODE["Code + tests"]
  CODE --> BIND["Link each rule to<br/>a test · type · check"]
  BIND --> PROD["Production"]
  PROD --> FOUND["Bug, or a new requirement"]
  FOUND -->|"add the rule"| SPEC
  CODE --> REF["Reference<br/>what it does now"]
{{< /mermaid >}}

None of this is new. Regulated industries have tracked requirements to tests for decades, and property-based testing exists for exactly the every-case problem. What's new is that keeping the links current used to be too tedious to bother with, and now a model can do the tedious half.

The loop matters as much as the step. When production finds something the spec never said, it goes back in as a new rule, not just a patch. That's also why we keep two kinds of document: a spec is frozen, a dated record of what we decided and why; a reference doc is live, describing how the system behaves today. The frozen one is safe only because it never claims to describe the present.

## Where This Goes

[codespeak.dev](https://codespeak.dev/docs/programming-system/spec-driven-evolution) is the strong version of all this - "You maintain the spec, not the code" - and worth watching. Its answer to checking the output is tests it writes and repairs itself, which is the same-author problem again.

Anthropic went the other way and [deleted over 80%](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models) of Claude Code's system prompt with no measurable loss, swapping rules for judgment. Both fit the same line: telling a model how to write code is advice it will eventually stop needing. Telling it what your business does is not.

Better models will write better code from the same spec. They still won't tell you which rule you forgot to write down.
