---
layout: post
title: "The LLM writes the story. A pure function writes the score."
date: 2026-06-22 09:00:00 +0000
description: "Most AI graders ask a language model how plausible your answer sounds — and for a security product, plausible prose is not a contained breach. LOG 001 draws IncidentGym's hard line: the LLM narrates the incident and plays the advisors; a pure deterministic resolver computes the score from the environment, reproducibly. Here's why auditable + reproducible is the entire pitch to a regulated buyer."
log: "002"
read: "7 min"
summary: "AI graders that ask an LLM 'how plausible does this answer sound?' have a credibility hole: plausible prose is not a contained breach. So we split the work. The LLM writes the narrative and plays the advisor personas. A pure graph-cascade resolver — no LLM, no clock, no randomness — computes the score from your environment. Same inputs, same score. That's not a feature; for a regulated buyer it's the whole pitch."
flags:
  - text: "LLM narrates, doesn't judge"
    kind: ok
  - text: "Same inputs → same score"
    kind: ok
  - text: "Comms axis: fenced + labeled"
    kind: warn
---

The first design decision in IncidentGym wasn't a UI choice or a database schema. It was a refusal. We were building a trainer that grades how a security team handles an incident, and the obvious path — the path most of the new AI graders take — is to hand the player's answer to a language model and ask it: *how good is this?* The model reads your prose, weighs how plausible it sounds, and returns a number. It's fast, it's cheap, and it is exactly the thing we decided we would not ship. Because for a security product, plausible prose is not a contained breach. A response that *reads* like a competent isolation-and-failover can score beautifully and still be the wrong call for the environment it's running in. If the grader can be talked into a good score, the score is worth nothing to the person who has to defend it.

So we drew a line straight through the middle of the product, and everything else is downstream of it: **the LLM writes the story; a pure function writes the score.**

## What the LLM is allowed to do

Plenty, actually — just not the thing that decides whether you pass. The language model owns the *narrative*: it sets the scene, escalates the situation as the clock runs, and plays the cast of AI advisor personas in the room with you — the CEO who wants a customer statement, the security lead pushing for containment, the payments lead worried about the cutover, the comms officer drafting the holding line. That's real work, and it's work LLMs are genuinely good at: producing fluent, in-character, situation-aware text under pressure. When you isolate a service or escalate or fail over, the advisors react, the world moves, and it *feels* alive.

What the model never touches is the number. It doesn't see a rubric it can be argued into. It doesn't get to decide that a confident-sounding answer "deserves" a higher technical score. The story it writes is flavor and pressure and texture — and flavor, by design, has no vote.

## What a pure function does instead

The score is computed by a pure, deterministic graph-cascade resolver, and the word that matters there is *pure*. No LLM call. No clock. No randomness. It takes two inputs: your decision, and your organization's environment — a service-dependency graph with criticality weights, read from the database as a fixed input to the computation. It propagates the decision through that graph and computes concrete outcomes: blast radius, which services were impacted, what recovered, and a causal trace of exactly what your choice changed. From those it derives the score's deterministic axes: **technical**, **speed**, and **riskMitigation**.

Because the resolver is pure over those inputs, it has a property an LLM grader can never have: **the same inputs produce the same score.** Run the same decision against the same topology a thousand times and you get the same number a thousand times. There's no temperature, no seed drift, no "the model was in a mood." The number isn't an opinion that happened to land near the truth; it's a computation, and you can re-run the computation.

> A score you can re-derive isn't a grade. It's a receipt.

That distinction — grade versus receipt — is the whole reason the line exists. A grade is what someone gives you. A receipt is something you can hand to an auditor and they can check it themselves.

## The one axis we couldn't make pure — and what we did about it

Honesty time, because a build log that only lists wins is just marketing with line numbers. One axis genuinely needs language understanding: **communication**. Whether you told the right stakeholders the right thing at the right moment is a judgment about *text*, and you can't compute it from a dependency graph. So communication is the one axis where an LLM assists the evaluation.

We didn't hide that. We did the opposite — we **fenced it and labeled it.** Communication is the only LLM-assisted axis in the score, and the split is explicit by design: technical, speed, and riskMitigation come out of the pure resolver; communication is the one with a model in the loop. The fence is the point. Instead of one undifferentiated "AI score" where you can't tell signal from vibes, you get three axes you can re-derive and one axis that is openly marked as model-assisted. The buyer decides how much to trust the labeled one. We don't launder it through the deterministic ones.

That's why this carries a `warn` flag and the other two carry `ok`. It's a real caveat, stated plainly, in the place a skeptic would look.

## The split, in one table

Here's the line, stated as a table so it's hard to misread:

| Score axis | Computed by | Reproducible | Provenance |
|---|---|---|---|
| technical | deterministic resolver (pure) | same inputs → same score | deterministic |
| speed | deterministic resolver (pure) | same inputs → same score | deterministic |
| riskMitigation | deterministic resolver (pure) | same inputs → same score | deterministic |
| communication | LLM-assisted | not deterministic | **model-assisted (fenced)** |

One dependency worth naming for the skeptic: the deterministic path needs the topology to be there. The turn it's scoring — `Turn(sessionId, orgId)` — has to resolve to an organization whose environment graph exists, because that graph *is* the input the resolver runs on. No topology, no deterministic computation. The three deterministic axes are only ever computed against a real environment graph, never conjured.

## Why "auditable + reproducible" is the pitch, not a footnote

It would be easy to file all of this under engineering hygiene — nice internals, move on. But for the buyer we care about, it *is* the product. Picture a CISO putting a readiness number in front of a board, or an auditor asked to sign off on it. The first question either of them asks is the same: *where did this number come from, and can you show me?*

If the answer is "a language model read the team's answers and felt they were about an 81," the conversation is over. That number can't be defended, can't be reproduced, can't be distinguished from a number the model would have given to confident nonsense. But if the answer is "three of the four axes are computed by a pure resolver from your own environment, here are the inputs, run it yourself and you'll get the same number — and the fourth axis is explicitly marked as model-assisted" — now you have something a regulated buyer can actually stand behind. The deterministic line isn't there to make the demo look rigorous. It's there because **auditable and reproducible is what a security score has to be**, and an LLM-judges-the-prose grader is structurally incapable of being either.

The whole architecture follows from that one refusal at the start. The LLM makes the incident feel real. The pure function makes the score real. We kept them on opposite sides of a line we can point to — and the one place they had to touch, we fenced, labeled, and put a `warn` on so nobody has to take our word for it.

That's LOG 001. The story is written by a model. The score is written by a function. And only one of those gets a vote on whether you passed.

Live demo: **<https://incidentgym.redoubtlabs.dev>**