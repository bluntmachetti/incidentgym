---
layout: post
title: "Why another cyber incident-response training tool?"
date: 2026-06-21 09:00:00 +0000
description: "Cyber incident-response training is broken two ways — it's episodic, and the new AI trainers grade you on how plausible your prose sounds — at exactly the moment AI is accelerating the attack side. IncidentGym computes the outcome of a decision from a model of your environment: a readiness score a board and an auditor can re-derive."
log: "001"
read: "7 min"
summary: "The attack side is speeding up — security agencies and frontier labs now warn that AI is raising the volume and velocity of cyberattacks — while IR training is still an annual tabletop, or an AI grading plausible prose. We built IncidentGym to score readiness a different way: deterministically, grounded in your environment's dependency graph, auditable. The honest 'why us' isn't 'another chatbot.' It's how we score."
flags:
  - text: "AI is accelerating the attack side"
    kind: warn
  - text: "Training: episodic + vibes-graded"
    kind: warn
  - text: "Computed, not graded on vibes"
    kind: ok
  - text: "Board-defensible readiness"
    kind: ok
---

There are already a lot of ways to "train" an incident-response team, so the first honest question to answer is the one in the title: why build another one? I didn't set out to add a tool to a crowded shelf. I set out because the two dominant ways teams rehearse for a breach both have the same flaw — they produce a feeling of readiness without a number you can stand behind. This is the opening dispatch of IncidentGym's field log, so before the technical posts go deep on the engine, I want to be precise about the problem we're actually solving and the bet we're making. To be clear about what this is: I started IncidentGym as my entry for the **#H0Hackathon** (Vercel + AWS Databases), and this field log is its public build journal.

## Why this matters now: the attack side is speeding up

The case for *measuring* readiness instead of feeling it gets sharper every quarter, because the other side is accelerating. The UK's National Cyber Security Centre assessed that AI will [almost certainly increase the volume and heighten the impact of cyber attacks](https://www.ncsc.gov.uk/report/impact-of-ai-on-cyber-threat) — lowering the barrier for less-skilled actors and sharpening reconnaissance and social engineering. And it isn't only a forecast: in late 2025 Anthropic reported [disrupting the first documented *AI-orchestrated* espionage campaign](https://www.anthropic.com/news/disrupting-AI-espionage), in which an agentic system ran reconnaissance, vulnerability discovery, exploitation, and data exfiltration across roughly thirty organizations — with the AI doing an estimated 80–90% of the hands-on work.

Whatever you make of the exact figures, the direction is hard to argue with: incidents are getting faster and more frequent, and the gap between "something's off" and "it's everywhere" is compressing. That's the world an annual tabletop can't keep up with — and, with no small irony, a strange moment to let *plausible-sounding AI prose* be the thing that tells a security team it's ready. The faster the attack, the more a readiness number has to be real.

## The first break: training that's episodic

Most IR rehearsal is a tabletop. A consultant flies in once a year, walks your team through a scenario over an afternoon, and leaves you a slide deck and a warm feeling. It's not worthless — it surfaces gaps, it gets people in a room. But it tells you almost nothing about whether your team is ready *today*. Readiness isn't a thing you have on the day of the workshop and keep for twelve months; it drifts as people leave, as the environment changes, as the runbook quietly goes stale. An annual snapshot of a moving target is not a measurement. It's a memory.

What a CISO actually needs is continuous and comparable: a way to drill regularly and watch a number move, so "are we more ready than last quarter?" has an answer instead of an anecdote.

## The second break: graded on vibes

The newer wave of tools fixes the cadence problem by putting an AI in the loop — drill any time, get instant feedback. Good instinct. But look closely at *how a lot of them score you*: you write your response in prose, and a language model judges how plausible that prose sounds. The grade is a measure of how convincing your writing is.

For most domains that's a fine approximation. For a security product it's a credibility hole, because the whole job of an attacker is to make the wrong thing look right.

> Plausible prose is not a contained breach.

A confident, well-structured paragraph about isolating the affected service can score beautifully and still describe a move that, in your actual environment, would have taken down the payment path it depended on. The model grading you doesn't know your dependency graph. It's rewarding fluency. And a readiness score built on fluency is exactly the score an auditor — or an attacker — should not trust.

## What we're actually building

IncidentGym is incident-response training that's continuous, measurable, and above all *defensible*. The design goal, stated as plainly as I can: produce a readiness score a CISO can put in front of the board, and an auditor can't wave away.

The thing that makes that possible isn't a better prompt or a sterner AI judge. It's a different architecture for where the number comes from. The outcome of a decision in IncidentGym is **computed** — propagated through a model of your environment's service dependencies — not *graded*. When you isolate a service, fail over, or escalate, a deterministic engine works out what would happen in a model of your environment: what cascades, what's contained, which criticality-weighted services you saved and which you lost. Same decision, same environment, same score, every time. That reproducibility is the entire point. A number you can re-derive is a number you can audit; a number that depends on a model's mood that afternoon is not.

So the honest "why us" is not "we added another chatbot to incident response." There's a language model in IncidentGym, and it does real work — it narrates the unfolding incident, it voices the advisor personas you argue with under pressure, the CEO and the security lead and the comms officer in the room. But it does not own the score's deterministic core — the axes a board and an auditor can re-derive. (One axis, communication, is model-assisted; we fence it and label it openly, and a later dispatch shows exactly how.) The story is generated; the defensible score is computed. Keeping that line clean — narrative on one side, deterministic resolver on the other — is the part I'm proudest of, and it's what makes the readiness number mean something.

## What the next dispatches go deep on

This is the opener, so I'm framing the bet, not unpacking the machinery. Three threads run through everything that follows, and each one gets its own dispatch:

- **The LLM narrates, but it never judges.** The split between the model that tells the story and the pure engine that scores the decision is the spine of the product. A future log walks through exactly where that boundary sits and why "same inputs, same score" is non-negotiable for a security tool.
- **Tenant isolation enforced by the database, not by hope.** IncidentGym is multi-tenant, and we didn't want isolation to be a `where` clause someone could forget. It's a composite foreign-key invariant in the schema, so a cross-tenant write is rejected by PostgreSQL itself — a database error, not a passed application check. A dispatch covers how that turns "trust us" into something you can demonstrate live.
- **The blast-radius replay over the real dependency graph.** The debrief doesn't just hand you a number. It animates the incident spreading and being contained across your environment's service-dependency graph, turn by turn — red along the dependencies that fell, cyan for what you saved — next to the counterfactual showing what a different call would have done. It's the deterministic engine's output, made visceral. That one earns a full post.

## Where this is today

A field log that blurs shipped and not-shipped isn't worth much, so: today you drill **solo** against a set of **built-in, fictional environments** — the dependency graphs are ours, not yet yours. Two things are in testing and not live: **bring-your-own-topology**, so you can drill on a model of *your* real estate instead of a stand-in, and **team-vs-incident multiplayer**, so a whole response team can run the same authoritative engine together. When they ship, this log keeps the receipts.

## The forward bet

Here's the part I'm most willing to be held to. Because the scoring is deterministic and grounded in a topology rather than in a model's opinion, the same engine doesn't only have to grade people. Point it at an autonomous IR agent and it answers a question the industry is about to need badly: *how good is our agent, on our environment, against this incident?* As response gets more agentic, "is the bot any good?" needs a reproducible answer, and a ruler that's already environment-grounded and auditable is the natural place to get one.

And this isn't hypothetical for us. It's the same conviction behind a sibling project — [Aftershock](https://bluntmachetti.github.io/aftershock) — where a society of AI agents responds to a simulated disaster and is scored on a deterministic, reproducible ruler instead of graded on how good its plan *sounds*. Different domain (a disaster in a simulated town, not a breach in your estate), same refusal: if an agent is going to run the response, you measure it with a number you can re-derive.

That's the wager this whole field log is here to test in public. The dispatches that follow keep the receipts — what shipped, what we got wrong, what we walked back. If there's one line that justifies building another incident-response tool, it's this: don't grade the prose, compute the outcome.

Live demo: **<https://incidentgym.redoubtlabs.dev>**