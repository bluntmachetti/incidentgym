---
layout: post
title: "Watch the blast radius go from 46% to 10%."
date: 2026-06-25 09:00:00 +0000
description: "A computed readiness score is honest but invisible. So we render it. The Blast-Radius Replay animates an incident across the service-dependency graph for your environment, persisted in Amazon Aurora PostgreSQL — red where it spread, cyan where you saved — turn by turn. Here's how the pure graph-cascade resolver works, and why a pure engine lets the debrief re-run your unmade moves with no model call."
log: "004"
read: "7 min"
summary: "A deterministic score is defensible but it isn't visceral — so we render it. The Blast-Radius Replay plays the incident across the service-dependency graph for your environment, persisted in Amazon Aurora PostgreSQL, red where it spread and cyan where you contained it, turn by turn. Because the resolver is pure, the debrief re-runs the moves you didn't make to show the best alternative and a why-each-number-moved causal chain — all computed-on-read, no LLM, no writes, and reproducible because the same inputs always produce the same score."
flags:
  - text: "Deterministic, computed on read"
    kind: ok
  - text: "Counterfactuals, no model call"
    kind: ok
  - text: "Graph persisted in Amazon Aurora"
    kind: ok
---

Two logs ago I argued that a security product can't grade you on vibes — that the outcome of a decision has to be *computed* from what would happen in a model of your environment, not judged by a language model on how plausible your prose sounds. We built that: a pure graph-cascade resolver that owns the numbers while the LLM is confined to narrative and personas. It works. It's auditable. And on screen, a bare number like a blast-radius percentage is completely inert. A CISO doesn't *feel* a decimal. So this log is about the other half of the job — making the defensible number visceral without letting the rendering touch the math.

The answer is the **Blast-Radius Replay**: we take the environment's real service-dependency graph — the same one the resolver reads — and animate the incident moving across it, turn by turn. Red spreads down the dependencies that fell. Cyan lights up what your decision saved. The run opens with the blast spreading from the origin and, turn by turn, contracts as your moves contain it — from 46% of the criticality-weighted estate on turn one down to 10% by turn three. Same engine that scores you, now drawing you a map of the damage.

> The number says you were better off. The replay shows you *which services* you saved, and which call did it.

## The resolver is a graph cascade, and that's the whole trick

Every environment in IncidentGym is a service-dependency graph — nodes are services, edges are "depends on," and each node carries a criticality weight. When an incident lands, the resolver propagates the decision through that graph: starting at the compromised node, the cascade walks the dependency edges to everything downstream that is now exposed. Sum the criticality weights over what's affected, normalize against the whole environment, and you get the criticality-weighted blast radius — the headline number.

Your moves are graph operations on that cascade. **Contain** cuts downstream dependencies: a contained service stops propagating, so the subtree hanging off it drops out of the exposed set on the next turn. That pruning is exactly what the animation renders as a node going from red back toward cyan. The resolver doesn't have a separate "score formula" bolted onto a separate "visualization" — the blast radius *is* a property of how far the cascade reaches, and the replay *is* that same cascade drawn over time. One source of truth, two faces.

From that cascade, the engine reports three scoring axes — `technical`, `speed`, and `riskMitigation`. How fast you contain the right node and how much criticality-weighted capacity stays up across the incident come straight off the turn-by-turn cascade. Contain the right node early and recovery moves in your favor; chase a low-criticality service while a payments dependency burns and it doesn't. The axes aren't vibes about your judgment — they're arithmetic over the graph you actually changed.

## Why "pure" is the load-bearing word

The resolver is **pure** in the engineering sense: no hidden randomness, stable ordering, same inputs always produce the same outputs. I keep saying it because almost every nice property in this post falls out of that one constraint.

The most obvious payoff is auditability — a CISO can put the score in front of a board and an auditor can re-derive it. But purity buys something I didn't fully appreciate until we shipped the debrief: **you can re-run the engine over hypotheticals for free.** If the function has no side effects and no hidden randomness, then evaluating "what if I had contained the database instead of failing over the gateway" is just another call with different arguments. No new game, no new LLM turn, no write to Aurora. Those counterfactual re-runs happen in-memory inside the serverless function — zero writes back to Aurora, zero LLM API calls. The engine is a function, so the debrief gets to ask it questions.

## The debrief asks the questions you didn't

That's where the counterfactuals come from. When you finish a turn, the debrief re-runs the resolver over the moves you **did not make** and surfaces the **best alternative for that turn** — the single decision that would have contained the most criticality-weighted blast. Not an LLM speculating about what a better responder might have done; the actual deterministic engine, scoring the road not taken with the same arithmetic that scored you.

Paired with that is the **"why each number moved" causal chain**: for every metric that changed turn-over-turn, the debrief traces it back to the graph operation that caused it. Blast dropped because the contain cut a high-criticality subtree out of the cascade. Recovery improved because the failover kept a dependency online. It reads like a diff for your incident.

Here's the contained run as a readout — the three turns the replay animates:

| Turn | Move | Criticality-weighted blast | What the replay shows |
|---|---|---|---|
| 1 | incident lands | **46%** | red cascades from the origin down the dependency graph |
| 2 | contain the origin | **26%** | a high-criticality subtree drops out; nodes flip toward cyan |
| 3 | contain the exposed dependency | **10%** | the cascade collapses; most of the graph is back to cyan |

And the part that matters for trust: **every one of those numbers, every counterfactual, every causal-chain entry is computed on read.** The debrief is not loading stored verdicts the LLM wrote earlier. It re-derives them from the persisted decisions and the topology pinned to that session — each session pins its topology at start, so reopening it reproduces the debrief identically even if the live environment graph has since changed. No model is in the loop, nothing is written back. The ledger in Aurora persists the committed per-turn *facts* — the action you took, the metrics, the causal effects per turn, scoped to your org by the composite-FK invariant from Log 003 (`Turn(sessionId, orgId)`, cross-tenant write rejected as Postgres `23503`). The counterfactuals and the replay frames are not stored; they're re-derived on read as a pure projection of those persisted facts.

## Determinism makes the whole debrief reproducible

The last piece is determinism end to end. Because the resolver has no hidden randomness and produces the same outputs from the same inputs, the entire debrief — your run, the best-alternative counterfactual, the causal chain, the frame-by-frame blast values the replay animates — reconstructs with **zero model calls**. Because each session pins its topology at start, reopening the same session tomorrow runs the resolver over the identical inputs and the replay draws the identical cascade. Hand the same pinned inputs to a teammate and they reconstruct your exact debrief on their own machine. Reproducing a finished incident review costs a graph walk, not an API bill.

And it doesn't have to be a person in the seat. Point an autonomous responder at the same incident and the engine scores its run, replays its blast radius across the same graph, and shows the better move it left on the table — the same reproducible evidence, whether the responder is a human or a bot. The more of the response that gets automated, the more that's the line between *trusting* an agent and *measuring* one.

That's the line we've been holding since the start of this build, restated for the debrief: the LLM writes the *story* — the situation, the CEO and security-lead and payments-lead personas in the room, the pressure. The engine owns the *numbers*, and now it owns the *picture* of the numbers too. The replay is the deterministic engine's output made visceral, and because the engine is pure, the most persuasive thing in the debrief — *here's the better move you didn't make, and here's exactly why your score moved* — costs nothing and reproduces forever.

A computed score is the honest foundation. Rendering it on the real dependency graph is what finally makes someone *feel* the distance between the blast that spread and the blast you contained.

Live demo: **<https://incidentgym.redoubtlabs.dev>**