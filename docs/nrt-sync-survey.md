# Keeping Access Current

**A Survey & Critical Review — moving access-relevant change from Systems of Record / ERPs into IAM/IGA and downstream systems, and where each approach quietly breaks.**

- **Scope:** SoR/ERP → IAM/IGA sync
- **Approaches:** 7 architectural families
- **Lens:** freshness × correctness × recoverability
- **Drafted:** 2026-07-09

> **Thesis.** The real objective is not "low latency" in the abstract. It is minimizing the window of *decision-relevant staleness* — the interval in which an access decision could be made against data the System of Record has already superseded — *without* silently dropping or reordering the very changes that make the decision correct.

---

## Framing — why this is hard, restated

Timely, accurate control over who can do what is the core promise of IAM and IGA. When a right to access depends on detailed, current data about a user, stale data produces two failures: a user is granted access they should no longer have, or denied access they now legitimately need.

These two failures are **not symmetric**. A late *grant* is a productivity annoyance; a late *revoke* — the leaver who keeps their entitlements, the transfer who retains the old team's data — is a standing security and compliance exposure. Any honest evaluation of a sync approach has to weigh how it behaves on the revoke path specifically, not just its average propagation time.

The prelude to this project observes that there seem to be as many sync approaches as there are systems, yet they sort into a manageable set of architectural categories. That is the organizing claim here: the seven families below are best read as points on a small number of axes — **pull vs. push**, **batch vs. per-change**, and, most decisively, **who owns progress and who absorbs failure**.

---

## The evaluation lens — eight dimensions that decide fitness

| Dimension | Question it answers |
| --- | --- |
| **Freshness** | Typical propagation delay from committed change to usable downstream state. |
| **Delivery guarantee** | Can a change be lost, duplicated, or delivered out of order? |
| **Recoverability** | If the consumer is down for an hour, does it catch up — or lose the gap? |
| **Coupling & backpressure** | Who absorbs load when the far side is slow: source, broker, or sink? |
| **Source impact & access** | API quotas, load, and whether you can even reach the change log. |
| **Semantic fidelity** | Multi-value attributes, custom fields, and — the perennial trap — deletes. |
| **Operational surface** | Infra to run, schema evolution, and the failure modes you now own. |
| **Governance fit** | Auditability, replay for evidence, and urgency of the revoke path. |

---

## The map — seven families on one axis

Ordered by **control model** (pull · batch · sink-driven ◄——► push · per-change · source-driven), not by maturity:

`01 Full reconciliation → 02 Delta polling → 06 Sink-polled WAL → 03 SCIM push → 05 Streaming CDC → 04 Webhooks → 07 Shared signals`

Note that the sink-polled write-ahead-log approach (**06**) sits on the *pull* side yet delivers per-change freshness — it is the anomaly that most of this review turns on.

---

## Survey & critique, family by family

### 01 · Scheduled full reconciliation
*Pull · Batch · Whole-dataset compare*

- **Freshness:** hours–days
- **Guarantee:** eventually correct
- **Fails when:** you need speed

The baseline every mature deployment still runs: periodically pull the *entire* population from the source and diff it against the target. Okta's Workday connector, for instance, recommends a **full import weekly** precisely to reconcile all users and heal any drift the faster paths missed ([Okta, Workday best practices](https://help.okta.com/en-us/content/topics/provisioning/workday/best-practices.htm)).

Its weakness is exactly its freshness — a full compare is expensive, so it runs rarely. But its strength is underrated: because it observes ground truth in full, it is the only family that can detect *deletes and missed events by absence*. That is why it never fully disappears; it degrades into a correctness backstop rather than a primary path.

> **Keep as:** the audit floor. No per-change scheme is trustworthy without a periodic full reconcile behind it.

### 02 · Incremental / delta polling
*Pull · Per-interval · "changed since T"*

- **Freshness:** = poll interval
- **Guarantee:** window-dependent
- **Fails when:** clocks & deletes

The pragmatic default: ask the source for records changed since the last high-water timestamp. Okta runs **incremental imports as often as hourly** for exactly this ([Okta Workday provisioning](https://help.okta.com/en-us/content/topics/provisioning/workday/workday-provisioning.htm)); for HR sources without richer interfaces, corma.io notes platforms simply "periodically check the HR system through REST APIs for new or updated records" ([corma.io](https://www.corma.io/blog/best-practices-for-provisioning-in-identity-and-access-management-with-examples)).

Freshness is bounded cleanly by the interval, and it needs no infrastructure beyond a cron and a cursor. The traps are subtle: changes that land *during* a poll can slip a window if the cursor is advanced naively; server/client **clock skew** silently drops rows; and a plain "what changed" query usually **cannot see hard deletes** — a deleted row simply isn't in the result set, so a leaver can survive indefinitely until a full reconcile (01) catches them.

> **Watch:** tune the interval down and you approach real-time — but you also multiply source load, and you never fix the delete-detection gap.

### 03 · Standards-based push provisioning (SCIM)
*Push · Per-change intent · Connector protocol*

- **Freshness:** often 20–40 min
- **Guarantee:** best-effort
- **Fails when:** rich attributes

SCIM is the lingua franca of provisioning: the IdP *pushes* user create/update/deactivate to downstream apps over a standard REST schema. Conceptually per-change and near-real-time — but real deployments disappoint. Reports put common SCIM sync cadence at **every 20–40 minutes rather than truly live** ([Stitchflow, AWS SCIM](https://www.stitchflow.com/scim/aws)), and the data model bites: many implementations support **only single-value attributes, with no custom fields** — awkward for roles, departments, or project assignments ([Zluri](https://www.zluri.com/eye-on-identity/google-workspace-scim-provisioning-limitations)). Practitioners catalog a long tail of interop pain ([Evolveum, "SCIM troubles"](https://github.com/Evolveum/docs/blob/master/iam/iga/identity-provisioning/scim-troubles.adoc)).

The deeper critique: standardizing the *wire format* did not standardize the *delivery semantics*. SCIM says nothing binding about ordering, retry, or replay, so "push" in practice is still a connector on a timer. It is essential for interoperability and useless as a latency guarantee.

> **Reality:** excellent as a common target interface; do not mistake its per-change API shape for per-change freshness.

### 04 · Event-driven webhooks
*Push · Per-change · Source-initiated HTTP*

- **Freshness:** seconds
- **Guarantee:** at-least-once, unordered
- **Fails when:** the sink is down

When the source supports it, a webhook fires the instant a record changes and the provisioning workflow starts "almost instantly" ([corma.io](https://www.corma.io/blog/best-practices-for-provisioning-in-identity-and-access-management-with-examples)). This is the freshest of the source-driven options and the one most people mean by "real-time."

It also inherits every hazard of source-initiated push. The source must write its record *and* emit the event — the classic **dual-write problem**, where a crash between the two silently loses the event or announces state that never committed ([dual-write problem](https://dev.to/aloknecessary/event-driven-architecture-the-dual-write-problem-and-how-to-solve-it-5266)). Delivery is **at-least-once and unordered**, so consumers must be idempotent and tolerate reordering ([Cockroach Labs](https://www.cockroachlabs.com/blog/idempotency-and-ordering-in-event-driven-systems/)). Worst of all for correctness: if the sink is offline when the webhook fires, **the event is gone** — there is no cursor to resume from. Webhooks buy freshness by putting recovery entirely on the source's retry policy, which you rarely control.

> **Caveat:** fast and cheap, but the least recoverable. Never run webhooks without a reconcile (01/02) to catch what was dropped while you were down.

### 05 · Durable event streaming & log-based CDC
*Push · Per-change · Broker-mediated (Kafka/Debezium)*

- **Freshness:** ms–seconds
- **Guarantee:** ordered, replayable, at-least-once
- **Fails when:** you count the ops cost

Change Data Capture reads the database's own transaction log and emits every insert/update/delete as an event, typically onto a durable bus like Kafka via Debezium ([Confluent, CDC](https://www.confluent.io/learn/change-data-capture/)). Because it reads the commit log rather than polling tables, it is low-latency, low-impact on the OLTP source, and **ordered by commit** — with the broker's retention giving you **replay** for free. Pairing CDC with the transactional [outbox pattern](https://www.decodable.co/blog/revisiting-the-outbox-pattern) closes the dual-write gap that undoes webhooks.

The honest counterweight is operational gravity. You now run and secure a broker, connectors, schema registry, and serialization evolution; delivery is still **at-least-once**, so consumers must dedupe; and practitioners increasingly warn against reaching for this machinery when the change volume doesn't justify it ([squer.io, "stop overusing the outbox"](https://www.squer.io/blog/stop-overusing-the-outbox-pattern)). It is the strongest general answer and the heaviest — most identity estates don't have Workday-scale change volume to amortize it.

> **Strong:** the reference architecture when you own the source database and change volume is real. Overkill for a few thousand HR events a day.

### 06 · Sink-polled write-ahead log with watermark ★ *focus*
*Pull · Per-change · Consumer-owned cursor*

- **Freshness:** seconds
- **Guarantee:** ordered, no-loss, resumable
- **Fails when:** you can't reach the WAL

Here the **sink** tails the source's write-ahead / commit log directly — Postgres WAL, MySQL binlog, DynamoDB streams — and **persists its own watermark** (an LSN, binlog position, or offset) marking exactly how far it has consumed ([microservices.io, transaction log tailing](https://microservices.io/patterns/data/transaction-log-tailing.html)). This is log-based CDC's engine, but with the control inverted: no broker sits in the middle, and progress is owned by the data's destination rather than its origin.

That inversion is the whole point, and it resolves the failure that dogs families 03–04. Because the watermark lives with the sink, **the sink can go dark for an hour and resume from exactly where it stopped** — the source did nothing special, buffered nothing, and lost nothing; the WAL is the buffer. You inherit log-order (correct ordering), no-loss delivery, and natural **backpressure**: a slow sink simply advances its cursor more slowly. The cost is *coupling* — the sink must understand the source's log format and schema, needs privileged log access, and must make the watermark durable and its own writes idempotent (log tailing is at-least-once, and the canonical warning is that it is "tricky to avoid duplicate publishing"). Bootstrapping a new sink also needs a consistent snapshot stitched to the live tail without gaps or dupes — the problem the DBLog **watermark-based snapshotting** algorithm exists to solve, now standard in Debezium and Flink CDC ([CDC watermarking guide](https://medium.com/@sami.alashabi/unlocking-real-time-data-synchronisation-a-guide-to-change-data-capture-cdc-with-kafka-and-356c0ba083b7)).

> **Best tradeoff:** the most robust freshness-with-correctness balance when you can get log access — per-change latency, log-order, and, uniquely among the fast paths, recovery that the *consumer* controls.

### 07 · Continuous shared signals (SSF / CAEP / SCIM Events)
*Push · Per-event · Standardized security signals*

- **Freshness:** near-instant
- **Guarantee:** signal, not bulk sync
- **Fails when:** used as the whole pipe

The emerging standards layer. The OpenID Foundation's **Shared Signals Framework** carries security-relevant events between systems in real time using signed Security Event Tokens; **CAEP** profiles events like "session revoked," "credential change," or "assurance-level change," while **SCIM Events** extend the same token format to lifecycle events like user creation, suspension, and deletion ([OpenID, SSF blueprint](https://openid.net/shared-signals-framework-the-blueprint-for-modern-iam-part-1-of-4/); [OpenID, SCIM provisioning](https://openid.net/juggling-with-fire-made-easier-provisioning-with-scim/)).

Read correctly, this is not a competitor to families 01–06 but a **dedicated fast lane for the asymmetric-risk events** — the urgent revoke, the compromised credential — that shouldn't wait for the bulk sync cadence. Its limits are maturity and scope: interop is still stabilizing ([SGNL, CAEP best practices](https://sgnl.ai/whitepaper/caep-best-practices/)), and it is a thin signalling channel, not a mechanism for reconciling full population state. Use it to cut revoke latency to seconds; keep a real sync underneath it for everything else.

> **Emerging:** promising as a governance fast-path for high-urgency events. Not yet, and not designed to be, your primary provisioning pipe.

---

## Comparison matrix

| Family | Freshness | Delivery guarantee | Recovery if sink down | Owns progress | Ops surface |
| --- | --- | --- | --- | --- | --- |
| **01** Full reconciliation *(pull · batch)* | hours–days | Correct by construction; sees deletes | N/A — self-healing on next run | Sink (cursor-free) | minimal |
| **02** Delta polling *(pull · interval)* | = interval | Window-bounded; misses hard deletes | Good — resumes from cursor | Sink (timestamp) | low |
| **03** SCIM push *(push · connector)* | 20–40 min | Best-effort; no ordering/replay spec | Depends on connector retry | Source/IdP | medium |
| **04** Webhooks *(push · HTTP)* | seconds | At-least-once, unordered, dual-write risk | **Poor — events lost while down** | Source | low |
| **05** Streaming CDC *(push · broker)* | ms–seconds | Ordered + replayable; at-least-once | Strong — broker retention | Broker (offsets) | high |
| **06** Sink-polled WAL ★ *(pull · consumer cursor)* | seconds | Log-ordered, no-loss; at-least-once | **Strong — sink owns watermark** | Sink (LSN/offset) | medium |
| **07** Shared signals *(push · SET)* | near-instant | Signalling only; not full-state sync | Varies (SSF stream) | Source/receiver | medium |

Ratings are directional, not benchmarks — real numbers depend on volume, source capabilities, and tuning. The one column that most separates the families in practice is **"recovery if sink down."**

---

## Critical synthesis — what the survey actually argues

No single family wins outright, and the ones marketed hardest as "real-time" (webhooks, SCIM) are among the least trustworthy on the correctness axis. Three conclusions fall out of the comparison.

**1 · The deciding variable is who owns progress and who absorbs failure.**
Source-driven push (webhooks) is fresh but fragile: it puts recovery on the source and loses events whenever the sink blinks. Broker-mediated streaming fixes this by making the broker own retention — at real operational cost. The **sink-polled WAL with a consumer-owned watermark (06) is the quiet winner**: it earns log-based CDC's ordering and no-loss guarantees while letting the destination control its own recovery, with no broker to run. When you can get log access, it is the best freshness-versus-correctness trade in the set — which is why it deserves first-class billing rather than being folded into "CDC."

**2 · Every fast path is at-least-once and can drift — so reconciliation is not optional.**
Webhooks, SCIM, streaming, and log tailing all deliver at-least-once and none reliably observe hard deletes on their own. That makes a periodic **full reconciliation (01) a permanent structural requirement**, not a legacy leftover: it is the only family that detects loss by absence. The mature pattern is layered, not singular.

**3 · Deprovisioning deserves its own fast lane.**
Because a late revoke is a security hole while a late grant is an inconvenience, the urgent-revoke event is worth pulling out of the bulk cadence entirely. This is exactly the niche **Shared Signals / CAEP (07)** fills — seconds-latency signalling for "session revoked" and "credential compromised" — layered over a slower, more complete sync beneath.

### A defensible reference stack

- **A · Fast path** — sink-polled WAL / watermarked log tailing (06) where you own the source DB; streaming CDC (05) only where change volume justifies a broker; webhooks (04) as a stopgap when the source is HTTP-only.
- **B · Urgent-revoke lane** — Shared Signals / CAEP (07) for compromise and termination events that must not wait for the fast path's queue.
- **C · Correctness backstop** — scheduled full reconciliation (01), with delta polling (02) as the fallback for sources that expose no log at all.
- **D · Interop boundary** — SCIM (03) as the standard *target* interface to downstream apps, understood as a contract for shape, not a promise of speed.

Read against the prelude's thesis, the categories do collapse onto a few axes — and the most useful axis turns out not to be pull-vs-push or batch-vs-streaming, but **where the recovery cursor lives**. Put it at the sink, and you get freshness you can actually trust.

---

## Implementation tradeoffs

Choosing a family (01–07) settles the shape of the pipe. It does not settle the six engineering decisions below — and these are where a "near-real-time" design quietly turns lossy, out-of-order, or stale in production. Each is a genuine tension with a defensible default, not a solved problem.

### Exactly-once is a trap · *delivery semantics*
Every fast path here is **at-least-once**; true exactly-once delivery across a network boundary is effectively unachievable. Effort spent chasing it is better spent making re-delivery **harmless**.
- ✗ *Chase exactly-once* — fragile, slow, still leaks dupes on failover
- ✓ *At-least-once + idempotent* — upsert keyed by entity + version
- **Default:** assume duplicates. Make the sink idempotent — apply keyed on `(entity_id, change_version)` so replays converge instead of corrupt. *(bears on 04 · 05 · 06)*

### Watermark timing = loss vs. rework · *when to commit the cursor*
The order of "apply change" and "advance watermark" decides your failure mode. Commit the cursor **before** the write lands and a crash **loses** the change; commit **after** and a crash **reprocesses** it.
- ✗ *Commit early* — gaps: silent data loss
- ✓ *Commit after apply* — at-worst reprocessing
- **Default:** advance the watermark only after the apply is durable, and lean on the idempotency above so the reprocessing is a no-op. Never trade loss for a little less rework. *(bears on 02 · 06)*

### Order per entity, not globally · *ordering & parallelism*
Strict global ordering is correct but serial — it caps throughput at one worker. Full parallelism is fast but reorders events, so a stale "disable" can land **after** a newer "enable" and re-grant access.
- ✗ *Global order* — correct, single-threaded, slow
- ✓ *Unordered* — fast, but races on the revoke path
- **Default:** partition by a stable key (the user/identity id): strict order *within* an identity, parallel *across* identities. Carry a version/LSN so the sink can drop out-of-order stragglers. *(bears on 04 · 05 · 06)*

### Bootstrapping without a gap · *initial load*
A new sink must load a consistent snapshot *and* stitch it to the live tail with no missing or duplicated change in the seam. Lock the source and you block it; snapshot naively alongside the stream and you get gaps or dupes.
- ✗ *Lock & dump* — consistent but blocks the source
- ✓ *Naive concurrent* — gaps/dupes at the seam
- **Default:** use **watermark-based snapshotting** (the DBLog algorithm in Debezium / Flink CDC): chunk the snapshot and interleave it with the live log, deduping by log precedence. Lock-free and seamless. *(bears on 05 · 06)*

### Deletes are the silent leak · *deletes*
A "what changed since T" query cannot return a row that no longer exists, so **hard deletes are invisible** to most fast paths — exactly the leaver whose access must be revoked. Absence is not an event.
- ✗ *Trust the feed* — misses hard deletes indefinitely
- ✓ *Reconcile by absence* — catches them, but costly
- **Default:** prefer sources that emit tombstones / soft-delete events (log-based CDC does — 05/06). Where they don't, a periodic full reconciliation (01) is the *only* reliable delete detector — treat it as mandatory, not optional. *(bears on 01 · 02 · 06)*

### Buy freshness, then measure it · *freshness vs. cost*
Freshness has a price paid in source load: tighter polling burns API quota and adds DB pressure; wider batching goes stale. And latency you don't measure will drift — a healthy-looking pipe can silently fall an hour behind.
- ✗ *Poll harder* — fresh, but quota & source strain
- ✓ *Batch wider* — cheap, but stale decisions
- **Default:** get freshness from event/log capture (04–06), not from cranking the poll interval; reserve tight polling for HTTP-only sources. Set an explicit end-to-end **staleness SLO** and alarm on consumer lag — the freshness you can't see, you don't have. *(bears on 02 · 04 · 06)*

> **The thread through all six.** Five of the six defaults lean on the same two primitives: an **idempotent, versioned apply** at the sink and a **durable watermark** advanced after that apply. Get those right and the remaining decisions — ordering, bootstrap, deletes — become tuning rather than correctness risks. This is the concrete, code-level reason the survey keeps returning to the sink-owned cursor (06): it is where these primitives naturally live.

---

## Sources consulted

**SCIM, provisioning & polling cadence**
- [Stitchflow — AWS IAM Identity Center SCIM: pricing & limitations](https://www.stitchflow.com/scim/aws) (20–40 min sync cadence)
- [Zluri — Google Workspace SCIM provisioning: limitations and alternatives](https://www.zluri.com/eye-on-identity/google-workspace-scim-provisioning-limitations) (single-value / custom-attribute limits)
- [Evolveum Docs — "SCIM troubles"](https://github.com/Evolveum/docs/blob/master/iam/iga/identity-provisioning/scim-troubles.adoc)

**HR-driven & event-driven provisioning**
- [corma.io — Best practices for IAM provisioning (2025)](https://www.corma.io/blog/best-practices-for-provisioning-in-identity-and-access-management-with-examples) (webhooks vs. periodic REST checks)
- [Identity Defined Security Alliance — Best practices for real-time IGA](https://www.idsalliance.org/blog/best-practices-to-ensure-successful-real-time-iga/)
- [SAP — Reference architecture for HR-driven identity lifecycle (Recruit-to-Retire)](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/sap-reference-architecture-for-iam-hr-driven-identity-lifecycle-management/ba-p/13534571)

**Connector reconciliation (Workday / Okta / SailPoint)**
- [Okta — Workday provisioning best practices & FAQ](https://help.okta.com/en-us/content/topics/provisioning/workday/best-practices.htm) (full import weekly, incremental hourly, Real-Time Sync)
- [Okta — Workday provisioning overview](https://help.okta.com/en-us/content/topics/provisioning/workday/workday-provisioning.htm)

**CDC, streaming & log-based capture**
- [Confluent — What is Change Data Capture?](https://www.confluent.io/learn/change-data-capture/)
- [Decodable — Revisiting the Outbox Pattern](https://www.decodable.co/blog/revisiting-the-outbox-pattern)
- [Alashabi — CDC with Kafka & Debezium; watermark-based snapshotting (DBLog)](https://medium.com/@sami.alashabi/unlocking-real-time-data-synchronisation-a-guide-to-change-data-capture-cdc-with-kafka-and-356c0ba083b7)

**Transaction-log tailing, watermarks & delivery semantics**
- [microservices.io — Pattern: Transaction log tailing](https://microservices.io/patterns/data/transaction-log-tailing.html) (WAL/binlog; "tricky to avoid duplicate publishing")
- [squer.io — Stop overusing the outbox pattern](https://www.squer.io/blog/stop-overusing-the-outbox-pattern)
- [Cockroach Labs — Idempotency and ordering in event-driven systems](https://www.cockroachlabs.com/blog/idempotency-and-ordering-in-event-driven-systems/)
- [Alok — Event-driven architecture: the dual-write problem](https://dev.to/aloknecessary/event-driven-architecture-the-dual-write-problem-and-how-to-solve-it-5266)

**Shared Signals / CAEP / SCIM Events**
- [OpenID Foundation — Shared Signals Framework: the blueprint for modern IAM (Pt. 1)](https://openid.net/shared-signals-framework-the-blueprint-for-modern-iam-part-1-of-4/)
- [OpenID Foundation — Provisioning with SCIM & SCIM Events](https://openid.net/juggling-with-fire-made-easier-provisioning-with-scim/)
- [SGNL — CAEP best practices](https://sgnl.ai/whitepaper/caep-best-practices/)

*Sources were consulted via web search on 2026-07-09. Latency figures are vendor/practitioner estimates that vary widely with configuration and change volume; treat them as directional. Several cited pages are vendor or practitioner blogs and carry the usual commercial framing.*

---

*nrtsync2 · Survey & critical review · Draft 2026-07-09 · Reads best alongside `memory/prelude.md`. An HTML version with the same content lives at `docs/nrt-sync-survey.html`.*
