# 3D Architectural Grounding — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [A.I. Mistakes Writers Must Stop Making](https://www.youtube.com/watch?v=3kf9rRztJgA)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: The Flat Page and the Missing Third Axis

**The concept.** Every specification you hand to a model is a *surface*. It has two axes: **what the thing does** (behavior, signature, happy path) and **how it should read** (idiom, vocabulary, library choice, tone). A language model is the best surface interpolator ever built — it has read more surfaces than any human alive. It has read almost no *substrates*. Nothing in the training distribution is "the twelve minutes your service spent half-up during a failover," "the double-charge that happened because two workers replayed the same message," or "the memory that grew 40 MB/hour for six days."

A prompt supplies a page. Reality is a volume. The gap between the two is where AI-generated code dies.

**"Korean Chicken Fluency" (KCF).** In the lecture's terms: an AI can produce a recipe for *Korean chicken* that is flawlessly fluent — right vocabulary, right register, confident steps, correct-looking technique — and that no cook would recognize as food that exists. Transposed to software: **code whose every surface token is correct and whose substrate is absent.** It compiles. It uses the right decorators. It passes the happy-path test the same model wrote. It is *superficially valid and production-invalid*.

The correct diagnosis is not "the model is dumb." The model answered the question it was asked. The question was 2D.

**The four detection signals of KCF.** Recognize it before you review it:

| Signal | What it looks like |
|---|---|
| **Asymmetric precision** | Library names, method signatures, and constants are exact; *lifetimes* are unstated. Every noun is precise, every duration is missing. |
| **Defective verb tense** | "This should handle…", "will be cached…", "can be retried…" — modal verbs where a guarantee, a bound, or a failure path belongs. |
| **No failure vocabulary** | Zero mention of timeout, partial write, replay, duplicate delivery, backpressure, restart, or migration. The happy path is the only path. |
| **No simultaneity vocabulary** | Zero mention of who else is touching the same state. Ordering, atomicity, and lock ownership are absent because the author never imagined a second actor. |

**The three orthogonal axes of depth.** Grounding is not an adjective. Saying "production-grade, robust, handle edge cases, act as a senior engineer" adds paint to the page — it does not add thickness. Depth must be *supplied as constraints*, in three independent dimensions:

- **Z1 — Memory Lifecycle.** Where does every byte live? Who allocates it, who owns it, how long does it survive, in which process, across restarts, across a deploy, across a schema migration, across an account deletion request? What is the eviction rule, the growth bound, the tombstone policy?
- **Z2 — Concurrency Model.** Who may touch this state at the same time? What is the atomicity boundary? Who owns the lock, and who can hold it while calling out to a network? What is the ordering guarantee, the read-your-writes guarantee, the backpressure rule, the idempotency key?
- **Z3 — Failure Modes.** What happens when each dependency is *slow*, *down*, *lying*, *duplicated*, *reordered*, or *partially applied*? What detects it, how fast, how large is the blast radius, what is the recovery, who owns it at 03:00?

```
            2D — A PAGE WITH NO THICKNESS                3D — A VOLUME

      Y: how it reads (idiom)                            Z1  memory lifecycle
            ▲                                          ────────►  who allocates, how long,
            │                                                  what evicts, what survives restarts
            │        · · · · · · ·                              Z2  concurrency model
            │      ·   fluent   ·  ◄── every token               ────────►  atomicity, ordering,
            │    ·     plausible ·      is correct                 idempotency, backpressure
            │    ·     output    ·                                 Z3  failure modes
            │      ·   compiles  ·                                ────────►  fault → detect →
            │        · · · · · · ·                                            contain → recover
            │                                                              → owner
            └────────────────────────────────►
                    X: what it does (behavior)
                  Z = 0. Nothing behind the page.
```

**Why this matters specifically in software engineering.** In prose, a fluent-but-ungrounded sentence costs a reader one raised eyebrow. In software, KCF is *load-bearing*: the artifact executes. Four structural reasons make the failure systematic rather than accidental:

1. **Plausibility inverts scrutiny.** The more idiomatic the output, the fewer questions a reviewer asks. Fluency is not merely uncorrelated with correctness here — it is *anti-correlated with the attention paid to it*. The broken code reads the best.
2. **The tests inherit the blind spot.** If the same 2D prompt produces the implementation and its tests, both share the missing axis. Green tests then function as a false certificate. A passing suite is evidence about the surface, not the substrate.
3. **The cost is deferred, not avoided.** A 2D artifact passes demo, ships, and converts a 20-minute specification cost into a multi-hour incident cost, paid by someone else, at the worst hour, with `p99` on fire.
4. **Regeneration destroys grounding silently.** Every re-ask is a fresh interpolation from the same flat page. Ungrounded decisions are not carried forward — they are re-rolled. Grounding written nowhere is grounding lost.

**The mental model to hold.** *You are not teaching the model to be smart; you are handing it the reality it does not have.* The model brings the entire surface of the corpus. You bring the one thing the corpus cannot contain: **the actual system, its actual lifetimes, its actual contention, its actual failure history.** The unit of work in this protocol is not the prompt. It is the constraint.

---

## 2. Core Transformation Protocols

### Rule 1 — Run the Tri-Axis Interrogation Gate *before* any code generation

Nothing is generated until all three axes have been explicitly answered. "Implicit" = absent. Paste this block and fill it *before* asking for code:

```
GROUNDING PREAMBLE — required before codegen
─────────────────────────────────────────────────────────────────────────
1. MEMORY LIFECYCLE
   - owner:            (who allocates / who frees)
   - residence:        (process memory | request scoped | durable store | cache)
   - TTL / eviction:   (exact rule + maximum growth bound)
   - survival:         (restart? redeploy? schema migration? user deletion?)
   - canonical copy:   (when two copies disagree, which one wins)

2. CONCURRENCY MODEL
   - actors:           (who can touch this state, how many at once)
   - atomicity:        (the boundary that is all-or-nothing)
   - ordering:         (what order is guaranteed; what order is NOT)
   - mutual exclusion: (lock owner, held across what calls)
   - idempotency:      (key / dedupe window for retried work)
   - backpressure:     (what happens on burst; what is shed, what is queued)

3. FAILURE MODES
   - per dependency:   slow | down | lying | duplicated | reordered | partial
   - detection:        (signal + latency to notice)
   - blast radius:     (what else breaks; what stays up)
   - recovery:         (automatic path; manual path)
   - owner:            (who is paged)
   - residue:          (what inconsistent state is left behind)
─────────────────────────────────────────────────────────────────────────
```

A legitimate "not applicable" must be **positive**: for a pure function, concurrency = "inputs are immutable and passed by value; no shared mutable state; safe to call concurrently"; failure = "total over its declared domain, throws only on precondition violation." *"N/A"* typed without a reason is a 2D answer wearing a lab coat.

### Rule 2 — Compile the state inventory before the API

Enumerate every piece of state the module owns, references, or mutates. For each: `name → residence → owner → lifetime → eviction → authority-when-conflicting`. If an entry cannot be filled in, you have not finished specifying — you have only finished describing. The state inventory is what makes the code *have a location in the world*.

### Rule 3 — Name the competing actors. Draw the race, don't hope it away.

For any shared state, write down the interleaving you are worried about — literally two columns of steps. Then state which mechanism prevents it (unique constraint, optimistic version check, single-writer queue, lease + fence token, compare-and-swap) and what happens on the loser's path. "There is only one user" is a claim about production traffic; treat it as an assumption to be recorded, not a fact to be assumed.

### Rule 4 — Deliver failure modes as an FMEA table, not a sentence

Degradation is a specification artifact:

| Dependency | Fault | Detection (signal / latency) | Blast radius | Recovery | Owner | Residue |
|---|---|---|---|---|---|---|
| Primary DB | slow | `p95` latency alert / 60s | all writes queue; reads stale | circuit-break, serve stale reads | `#platform` | queued writes may replay |
| Upstream API | duplicated delivery | dedupe-hit metric / immediate | double-applied side effect | idempotency key short-circuits | `#integrations` | none if key is durable |
| Cache | down | miss-rate spike / 30s | load doubles on DB | fail-open to DB, shed non-critical reads | `#platform` | cold-start stampede |

If the table has one row, the system has one dependency you have not thought about.

### Rule 5 — Replace every adjective with an artifact

Adjectives are surface decoration. Convert them at the point of writing:

| Adjective | 2D reading (unusable) | 3D artifact (checkable) |
|---|---|---|
| "robust" | sounds resilient | named faults + detection latency + recovery path |
| "scalable" | sounds big | bound: *N* ops/s per instance at `p99 < X ms`, memory ≤ *M* B/entry |
| "secure" | sounds safe | threat named, trust boundary drawn, authZ decision point located |
| "performant" | sounds fast | budget table (per-op latency, per-request allocation) |
| "handles edge cases" | sounds thorough | the enumerated edge-case list, each with expected output |
| "production-grade" | sounds serious | the three axes, filled in |

### Rule 6 — Ground against the real substrate, and mark what you could not

Cite the actual version and the actual documented behavior. Where the model asserts runtime behavior you have not verified (an API default, a timeout, a retry policy, a lock's reentrancy, an ORM's transaction scope), tag it inline:

```
# [UNGROUNDED] assumes driver retries idempotently on connection reset —
#              verify against <driver> <version> docs before relying on it
```

Visible uncertainty is a reviewed decision. Silent uncertainty is an incident. Never let a plausible default pass as a verified one.

### Rule 7 — Freeze the contract before implementation; re-ground after every mutation

The three axes are part of the interface, not the implementation notes. Any change to a lifetime, an atomicity boundary, or a failure behavior is a **breaking change to the grounding contract** and must be re-stated and re-reviewed — even if no signature changed. A refactor that preserves the signature but widens the failure mode is not a refactor.

### Rule 8 — Version the grounding document with the code

Store the state inventory, the race table, and the FMEA beside the module (same commit). When the module is regenerated or re-prompted, the grounding document is the *input*, never the output. Ungrounded regeneration is the single largest source of regression in AI-assisted codebases: the model re-rolls decisions that were once deliberate.

### Worked micro-example — "add a rate limiter" (2D vs 3D)

```
2D REQUEST  →  "Write a rate limiter for our API."
2D OUTPUT   →  in-process dict keyed by IP, counter incremented per request,
               reset by a background timer. Fluent. Correct-looking.
               FAILS: per-instance counters (limit multiplies by replica count),
               unbounded key growth (memory leak via unique IPs),
               counter lost on restart (limit trivially resettable),
               no atomic increment under threads, timer drift,
               no defined behavior when the store is unavailable.
```

3D-requested version answers Z1 (keys are TTL-evicted, growth bounded, counters live in the shared store, survival across restart is *intended*), Z2 (atomic increment in the store, single source of truth, burst policy defined), Z3 (store unavailable → fail-open vs fail-closed is an explicit decision with an owner and an alert). Same signature. Different system.

---

## 3. Engineering Application Scenarios

### Scenario A — Code Reviews

**Purpose:** use the review as the grounding gate it should have been, without rewriting the author's work for them.

Protocol:

1. **Classify the artifact first.** Before reading line by line, determine which axes are *specified*. If Z1–Z3 are absent from the PR, the review's first output is not a style comment — it is the three questions. Do not spend review attention on naming and formatting while the artifact's substrate is undefined; that is polishing the page of a book with no third dimension.
2. **Tag every finding to an axis.** `[MEMORY]` (unbounded growth, missing TTL, unowned buffer, stale copy treated as canonical), `[CONCURRENCY]` (check-then-act, lock held across I/O, missing dedupe key, ordering assumed), `[FAILURE]` (no timeout, silent catch, partial write with no repair path, missing detection), `[PLAUSIBLE-OK]` (surface-level observation that is genuinely non-blocking).
3. **Separate blocking from advisory explicitly.** A `[PLAUSIBLE-OK]` comment is advisory by construction. An unaddressed `[MEMORY]`/`[CONCURRENCY]`/`[FAILURE]` finding is blocking, regardless of how clean the diff reads.
4. **Prefer the interrogative to the imperative when the axis is thin.** "What happens to this entry when the client never returns?" produces a grounded fix and a smarter author. Rewriting the function yourself transfers the substrate knowledge to nobody.
5. **Verify the evidence claim.** "Tests pass" answers the surface. Ask which failure was *exercised*: a test that never injects a timeout, a concurrent writer, or a restart has not tested these axes. Route the review's authority from the diff to the exercised failure.

Review comment template:

```
[MEMORY] `sessionCache` has no eviction rule and is keyed by user id.
         Lifetime = process lifetime → unbounded growth under signup traffic.
         Needs: TTL/`maxsize`, maximum growth bound, and the authority rule when
         the in-process copy disagrees with the store.
```

### Scenario B — PR Descriptions

**Purpose:** the PR description is the grounding artifact. If it cannot state the three axes, the change is not ready to merge.

Template (mandatory sections; empty is not an option):

```
## Grounding Contract
### Z1 Memory Lifecycle
- New state:        <name → residence → owner → TTL → eviction → survives restart?>
- Change to existing lifetimes: <before → after, and why>
### Z2 Concurrency Model
- Actors touching this state: <n / kind>
- Atomicity boundary:         <what is all-or-nothing>
- Ordering / idempotency:     <guarantee, key or dedupe window>
### Z3 Failure Modes
- Exercised failure:  <which fault was actually simulated, and how>
- New failure paths:  <introduced by this change; detection + owner>
- Behavior when a dependency is down: <explicit, with latency to detection>

## Evidence
- <test/command that exercised each row above — "green suite" is not evidence for Z1–Z3>
## Ungrounded residue
- <assertions still resting on unverified runtime behavior, with the follow-up>
```

Rules for the author:

- **"No behavior change" must be defended on all three axes** for refactors: prove lifetimes are unchanged, prove the lock/ordering discipline is preserved, prove no failure mode was widened. A refactor that keeps the signature and changes the failure surface is a behavior change.
- **Never open with the diff.** The diff is the surface; the grounding contract is the meaning.
- **Leave the residue visible.** A PR that ends with three honest `[UNGROUNDED]` items is more mergeable than one that ends with zero and no evidence — the first is quantified, the second is decoration.
- **The deferral argument, stated plainly in the description:** an ungrounded PR does not remove this work; it moves it to whoever is paged.

### Scenario C — Architecture RFCs / ADRs

**Purpose:** make the decision record falsifiable. Extend both formats with three mandatory sections; if they are missing, the RFC is a 2D document that will be implemented as one.

```
## Decision
## Context
## Grounding (mandatory)
### Z1 — Memory Lifecycle of this design
   State inventory across services: residence, owner, TTL, growth bound,
   survival across deploy / migration / deletion.
### Z2 — Concurrency Model of this design
   Actors, atomicity boundaries, ordering guarantees, single-writer points,
   what the design does NOT provide (e.g. no cross-region linearizability).
### Z3 — Failure Modes of this design
   Per dependency: fault → detection latency → blast radius → recovery → owner.
   Include the failure mode that is *accepted* — accepted ≠ unexamined.
## Alternatives Considered (each answered on the same three axes)
## Consequences (stated as lifetime / concurrency / failure deltas)
## Validation Plan
   Which assumption in Z1–Z3 is load-tested, chaos-drilled, or observed in
   staging — and what result would falsify it.
```

Rules for the architect:

- **Rejected alternatives must be rejected on the axes, not on taste.** "Simpler" is a 2D reason. "Alternative B makes the write path non-idempotent, so a single duplicated message double-charges, and we have no dedupe window" is a 3D reason that survives five years of re-litigation.
- **Write down what the design does *not* guarantee.** Explicit non-guarantees are the most valuable lines in an ADR; they are what stops a future engineer from building on an assumption the system never made.
- **Treat vendor documentation and AI-drafted RFCs as KCF-suspect until grounded.** Marketing prose and model prose share the same defect class: fluent surfaces, missing substrates. Demand the timeout, the consistency model, the retry semantics, the price-per-operation bound — in numbers.
- **Route the consequences to an owner.** Every Z3 row needs a name, or the detection configuration is theater.
- **Re-ground at every architecture change.** The ADR is an input to generation, never a byproduct of it. Cross-reference the review-layer enforcement in `../../kirby-open-code-review/SKILL.md` when the RFC carries review obligations.

---

## 4. Verification Checklist

- [ ] **Axis completeness, before generation.** The request/PR/RFC contains an explicit, filled answer for Memory Lifecycle, Concurrency Model, and Failure Modes — with the state inventory, the named race with its prevention mechanism, and an FMEA table whose detection and owner columns are non-empty. Any "N/A" carries a positive justification; any adjective ("robust", "scalable", "production-grade") has been converted into a number, a bound, or a named enemy.
- [ ] **Failure modes exercised, not asserted.** For every Z1–Z3 claim, I can point to the test, drill, or observation that *exercised* it — a timeout injected, a concurrent writer run, a restart performed, a duplicate delivered. A green happy-path suite alone is not evidence and is not accepted as such.
- [ ] **No silent ungrounded residue.** Every assertion resting on unverified runtime behavior (driver retries, transaction scope, lock reentrancy, default timeouts, ordering) is tagged `[UNGROUNDED]` inline and tracked with an owner. Count is zero *or* every counter is listed — I did not achieve zero by not looking.
- [ ] **Contract-and-code in one commit.** The grounding document (state inventory + race table + FMEA) is versioned with the module it describes, and no lifetime, atomicity boundary, or failure behavior changed without that document being updated in the same change. Regeneration was fed the grounding document as input; nothing was re-rolled silently.
- [ ] **KCF spot-audit passed on the diff.** Reading the final artifact cold, I can answer "where does this state live, who else touches it, and what happens when it breaks?" for every non-trivial block — and the answers are true. If a block still reads as fluent with no answer behind it, the diff is 2D and does not ship.