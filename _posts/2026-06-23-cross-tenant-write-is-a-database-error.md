---
layout: post
title: "A cross-tenant write should be a database error, not a code review."
date: 2026-06-23 09:00:00 +0000
description: "App-code-only multi-tenant isolation is one forgotten WHERE clause from a leak. In IncidentGym we pushed isolation into the schema: every row carries orgId, every parent declares @@unique([id, orgId]), every child binds (sessionId, orgId) as a composite foreign key into the parent's (id, orgId). A cross-tenant write is rejected by Postgres itself — error 23503 — not by a reviewer. And it's provable live in the audit console."
log: "003"
read: "6 min"
summary: "Multi-tenant isolation enforced only in application code is one forgotten WHERE clause from a cross-tenant leak. So we stopped trusting code review for it. In IncidentGym every row carries an orgId, every parent declares @@unique([id, orgId]), and every child binds (sessionId, orgId) as a composite foreign key into the parent's (id, orgId) — making child.orgId == parent.orgId a Postgres guarantee. A cross-tenant write isn't a missed review, it's a foreign-key error 23503. The CISO audit console has a one-click proof you can run live."
flags:
  - text: "Isolation by construction"
    kind: ok
  - text: "Postgres rejects it · 23503"
    kind: ok
  - text: "Provable live in /audit"
    kind: ok
---

The scariest line in any multi-tenant codebase is the one that *isn't* there. A `where: { orgId }` that some future commit forgets to add. A new query path that ships without it. A clever join that quietly reaches across tenants because nobody on the review thread held the whole schema in their head at 1 a.m. In a B2B security product, that omission isn't a bug — it's a breach, and it's the kind a CISO will ask you about before they ask about anything else. So when we built IncidentGym's tenancy, I refused to let "a careful reviewer noticed the missing clause" be the thing standing between two customers' data. I wanted the *database* to refuse.

This log is about how we made cross-tenant isolation a structural property of the schema instead of a discipline we have to maintain in application code — and how that lets us put a button in the product that *proves* it, live, in front of an auditor.

## The failure mode we refused to ship

The default way to do multi-tenancy is: stamp every row with an `orgId`, and add `where: { orgId }` to every query. It works right up until the moment one query doesn't. The enforcement lives entirely in code you have to remember to write, on every read and every write, forever, including in code that doesn't exist yet. The guarantee is therefore exactly as strong as your least careful future diff. "We're isolated" really means "we're isolated as long as nobody forgets," which is not a sentence you want to say to a security buyer.

There's a second, sneakier version of the same hole. Even if every *top-level* query is scoped, child rows can still drift. A `Turn` belongs to a `GameSession`. If you write a `Turn` with `sessionId` pointing at session `S` but stamp it with the *wrong* `orgId` — through a bug, a confused caller, a malicious request that lies about its tenant — a naive schema is perfectly happy to store it. Now you have a row that claims to belong to org A while hanging off a parent owned by org B. The application-code check that "should" have caught it is, again, just code someone wrote and someone else might not.

I didn't want to be one forgotten clause away from that. So we moved the invariant down a layer.

## Make the relationship carry the tenant

The core idea is small and, once you see it, almost obvious: **don't let a child row reference a parent by id alone — make it reference the parent by (id, orgId) together.**

Concretely, in our Prisma schema:

- Every tenant-owned table carries an `orgId` column. Nothing is tenant-ambiguous.
- Every *parent* table declares a composite unique key on `@@unique([id, orgId])`. The id is already unique on its own; this extra key exists purely so the pair `(id, orgId)` is a thing a foreign key can point at.
- Every *child* table binds `(sessionId, orgId)` as a **composite foreign key** into that parent's `(id, orgId)`.

That last line is the whole game. The relationship from `Turn` to `GameSession` is no longer "this `sessionId` exists" — it's "a session with *this id and this orgId* exists." Which means `child.orgId == parent.orgId` is no longer something application code asserts and hopes for. It's something Postgres *checks on every insert and update*, because it's the referential-integrity rule the constraint encodes.

> A cross-tenant write isn't a missed code review anymore. It's a foreign-key violation — Postgres error `23503` — raised by the database before the row ever lands.

Try to write a `Turn` for session `S` (owned by org B) while stamping it with org A's id, and there is no row `GameSession(id = S, orgId = A)` for the composite FK to satisfy. The insert is rejected. Not flagged in a linter, not caught in review, not maybe-caught in a test — *rejected*, at the storage layer, deterministically, every time, including in code paths none of us have written yet. The guarantee stopped depending on anyone's memory.

## Belt, suspenders, and one place to decide tenancy

The composite FK is the floor — the thing that holds even when everything above it is wrong. But we still build the layers above it correctly, because defense in depth means each layer assumes the others might fail.

The first layer is query scoping. We don't sprinkle `where: { orgId }` by hand and pray. Every database call goes through `db.scoped(orgId)`, which injects the `orgId` filter onto every query it issues. There's one place where scoping is decided, not hundreds of call sites where it can be forgotten. A reader can't accidentally see another tenant's rows because the scoped client won't emit an unscoped query in the first place.

The second layer is the one I care about most on the write path: **the finalize step is exactly-once and idempotent.** When a turn is scored and committed, scoring and persistence happen as one resumable operation, so a refresh or a second tab can't double-advance or double-charge a turn. And if a write ever tried to land a child row whose tenant disagreed with its parent's, the composite FK underneath is waiting to reject the mismatch anyway. The structural guarantee doesn't lean on the finalize logic being perfect — it's there precisely for the case where it isn't.

Here's the same idea stated as before/after, because the shift is the whole point:

| | Isolation in app code only | Isolation in the schema |
|---|---|---|
| Enforced by | every `where` clause you remember to write | a composite FK Postgres checks every write |
| A cross-tenant write is | a bug that *might* be caught in review | error `23503`, a foreign-key violation |
| Strength | = your least careful future diff | = a referential-integrity rule |
| How you prove it | "trust us, we scope everything" | run an insert and watch the DB reject it |

The bottom-right cell is what turns this from an architecture decision into a demo.

## Provable, live, in the audit console

A guarantee you can only describe is a claim. A guarantee you can *trigger on stage* is evidence. So the CISO-facing audit console has a **one-click isolation proof**, and it does exactly what a skeptic would do if you handed them a psql prompt: it attempts a cross-tenant write — a child row stamped with the wrong tenant — and shows you what the database says back.

The database rejects it with foreign-key error `23503`, surfaced right there in the console. It's a live demonstration that the isolation boundary is real and is enforced where we say it is: in Aurora, not in a slide. You don't have to take "we scope everything" on faith — you can watch Postgres refuse the write.

And because the boundary lives in the schema, it doesn't weaken in the public demo. The live deployment runs in single-org mode — a shared demo org, no sign-in — so judges can click straight in without signing up, but the *same* composite-FK scoping is in force. The isolation guarantee is identical to the multi-tenant configuration. The demo isn't a watered-down version that "would" be isolated in production. It's the same wall, with one door open for visitors, and the proof works either way.

## Why this was worth the schema care

Composite foreign keys cost something. You have to add the redundant `@@unique([id, orgId])` to every parent, thread `orgId` through every child relation, and resist the urge to take the easy single-column FK. It's more schema to hold in your head than `where: { orgId }`.

But it buys the one property an application-code check can never have: it's true even when the code above it is wrong. The whole pitch of a security product is that you've thought about the failure where *you* are the adversary's path in — the forgotten clause, the confused write, the next engineer who doesn't know the rule. Pushing tenant isolation into a referential-integrity constraint means that path is closed by the database, not by vigilance. The strongest thing I can say about IncidentGym's tenancy isn't "we're careful." It's: a cross-tenant write is a Postgres error, and there's a button that'll show you.

Live demo: **<https://incidentgym.redoubtlabs.dev>**