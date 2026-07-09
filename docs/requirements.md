---
title: Requirements Catalog — NRT Data Synchronization
date: 2026-07-09 11:07
draft: 0.1.0
author: Keith Hazelton, Claude Fable 5
---

# Requirements Catalog — NRT Data Synchronization: ERP/SaaS → IAM/IGA Registry

Status: draft v0.1 · 2026-07-08
Scope: near-real-time synchronization of identity-relevant data (workers, accounts, org attributes, lifecycle events) from authoritative sources (ERP, HR, SaaS) into an IAM/IGA registry.

Requirement IDs are stable; do not renumber. Approach references use the seven architectural families (**01–07**) defined in `nrt-sync-survey.md`.

---

## 1. Latency & Freshness (LAT)

**NOTE**: Bolded Requirement IDs apply regardless of which approach carries the data.

| Requirements: R- | Requirement |
|------------------|-------------|
| LAT-1 | Latency SLOs are defined **per event class**, not globally (see Appendix A for the event-class taxonomy and default targets). Leaver/revocation: seconds–minutes. Joiner: before first workday. Routine attribute change: hours acceptable. |
| LAT-2 | Leaver propagation is treated as a **security control**: measured, alerted on breach, reported to security stakeholders. |
| **LAT-3** | Freshness is observable: every registry record carries the source-system timestamp and the sync timestamp; staleness is queryable per record and per population. |
| LAT-4 | Latency SLOs hold under burst load (e.g. reorg producing 10k changes/hour), not only steady state. |

## 2. Completeness & Correctness (COR)

| Requirements: R- | Requirement |
|------------------|-------------|
| COR-1 | No permanent drift: every missed or lost event is eventually repaired. A reconciliation mechanism exists independent of the event path. |
| COR-2 | Deletes and terminations are detected even when the source emits no delta record for them. The tombstone/deletion-detection strategy is explicit per source. |
| COR-3 | The registry can be compared against full source truth on demand ("prove the registry matches HR"). |
| **COR-4** | Drift is a standing metric: count and age of divergent records, per source. |
| COR-5 | No lost updates at delta boundaries: watermark queries use overlap windows; concurrent changes during a sync window are not skipped. |

## 3. Delivery Semantics (DEL)

| Requirements: R- | Requirement |
|------------------|-------------|
| DEL-1 | All delivery is assumed at-least-once; every registry write is idempotent. |
| **DEL-2** | Idempotency/merge keys are immutable source identifiers (e.g. employee ID) — never mutable attributes (email, display name). |
| DEL-3 | Out-of-order delivery is tolerated: stale updates are rejected by version/sequence/source-timestamp comparison, not arrival order. |
| DEL-4 | Per-person event ordering is preserved where it matters (rehire-after-terminate); processing is serialized per person identifier. |
| DEL-5 | Failed events are never dropped: dead-letter queue with replay, under the same idempotency guarantees. |
| **DEL-6** | Replay is safe: reprocessing old events cannot regress newer registry state. |
| DEL-7 | Thin-payload events ("record X changed") trigger an authoritative read-back; read-after-event races are tolerated via retry with backoff. |

## 4. Identity & Data Model (IDM)

| Requirements: R- | Requirement |
|------------------|-------------|
| **IDM-1** | Correlation rules across sources (ERP worker ↔ directory account ↔ SaaS user) are deterministic, versioned, and testable. |
| **IDM-2** | Ambiguous matches go to a human exception queue; no silent auto-linking. Orphaned/unmatched accounts are surfaced, not ignored. |
| **IDM-3** | Identifier lifecycle is handled: email change, name change, identifier recycling. |
| **IDM-4** | Multi-source attribute precedence is defined per attribute (e.g. HR owns legal name, IT owns email) and documented. |
| **IDM-5** | The registry schema exceeds SCIM core where IGA needs it (contract dates, cost center, entitlement metadata); extensions are defined, not improvised. |
| IDM-6 | Initial load is a separate, designed path (bulk pagination → cutover at a recorded watermark) with no gap or overlap versus the incremental stream. |

## 5. Ingestion Contract (ING)

| Requirements: R- | Requirement |
|------------------|-------------|
| ING-1 | A single normalized ingest contract at the registry boundary (e.g. SCIM 2.0 + extensions), regardless of source transport. |
| **ING-2** | Source schema drift is non-fatal: contract tests, unknown-field tolerance, quarantine to DLQ rather than pipeline crash. |
| ING-3 | Source API constraints are respected by design: rate limits, pagination, token refresh, vendor maintenance windows. |
| **ING-4** | A per-source capability matrix exists (events? deltas? delete visibility? replay?) with an explicit fallback strategy per gap. |

## 6. Scale & Flow Control (SCL)

| Requirements: R- | Requirement |
|------------------|-------------|
| SCL-1 | A queue decouples ingest from the registry write path; bursts are absorbed, not dropped. |
| **SCL-2** | Registry write throughput is known; queue-depth alerting fires before SLO breach. |
| SCL-3 | Full reconciliation runs are time-bounded and do not starve the NRT lane. |
| **SCL-4** | Population growth headroom (e.g. M&A doubling) does not require architectural rewrite. |

## 7. Security (SEC)

| Requirements: R- | Requirement |
|------------------|-------------|
| **SEC-1** | Least-privilege access per source: read-only, scoped to required objects and attributes. |
| **SEC-2** | PII minimization: only attributes the registry needs are synced; no full-record hoarding in queues or logs. |
| **SEC-3** | Encryption in transit and at rest, including queues, DLQs, and any intermediate files. |
| SEC-4 | Inbound events (webhooks, SETs) are authenticated: signature verification, endpoint auth, replay protection. |
| **SEC-5** | The pipeline itself is a privileged target: its credentials are vaulted, rotated, and monitored. |
| **SEC-6** | No secrets in GUI-configured middleware or plaintext connector settings. |

## 8. Audit & Compliance (AUD)

| Requirements: R- | Requirement |
|------------------|-------------|
| AUD-1 | Every registry mutation is traceable to its source event: who/what/when/which pipeline run. |
| **AUD-2** | Event history retention matches the compliance horizon (certifications, investigations); the retention policy is explicit. |
| AUD-3 | Reprocessability: a window of events can be replayed to reconstruct a state timeline during incident review. |
| **AUD-4** | Registry state history is kept, not just current state — certifications and SoD analysis need historical access views. |
| AUD-5 | Pipeline logic is reviewable and versioned as code; black-box GUI logic requires an export/versioning workaround or is rejected. |
| **AUD-6** | Data residency and processor obligations are met: any middleware handling PII is under DPA, region-pinned where required. |

## 9. Operability (OPS)

| Requirements: R- | Requirement |
|------------------|-------------|
| **OPS-1** | End-to-end observability: source event → registry write, with per-stage latency and per-source health. |
| OPS-2 | Silent source death is detected: heartbeat/verification events or expected-volume alarms on event streams. |
| **OPS-3** | The DLQ has an owner, a runbook, and an SLA — it is not a write-only graveyard. |
| OPS-4 | Per-source pause/resume without losing position (persistent watermarks/offsets). |
| OPS-5 | Backfill and replay are first-class tooling, not incident-time improvisation. |
| **OPS-6** | Dry-run mode: preview registry mutations before applying, especially after adapter or rule changes. |
| OPS-7 | Blast-radius guard: sanity thresholds (zero-record feed, >N% of population changed) halt the pipeline and page a human. An empty feed must never mass-deactivate the workforce. |

## 10. Lifecycle & Change (LCM)

| Requirements: R- | Requirement |
|------------------|-------------|
| **LCM-1** | Vendor API deprecations are tracked per source; adapters upgrade without downtime. |
| LCM-2 | Onboarding a new source is configuration plus an adapter, not an architecture change. |
| LCM-3 | A new consumer (SIEM, downstream app) attaches without modifying source integrations. |
| **LCM-4** | Correlation/precedence rule changes are versioned with effect-preview before rollout (see R-OPS-6). |
| LCM-5 | Standards adoption path: SSF/CAEP-style signed event fast lanes are pluggable later without redesign. |

## 11. Organizational (ORG)

| Requirements: R- | Requirement |
|------------------|-------------|
| **ORG-1** | Source-of-truth ownership per attribute is agreed (HR vs IT vs manager) before build; disputed authority is an org problem tech cannot fix. |
| **ORG-2** | The correlation exception queue is staffed by people with a mandate to resolve. |
| **ORG-3** | The cost model is understood at production volume (per-task iPaaS pricing, API quotas, streaming infra), not pilot volume. |
| **ORG-4** | HR data-quality dependencies are explicit: an operational agreement with HR covers e.g. timely termination dates. |

---

## Highest-risk implicit requirements

Most projects specify latency and forget these; each has caused production incidents in the field:

1. **R-COR-2** — deletes invisible to delta mechanisms → terminated users retain access.
2. **R-DEL-3 / R-DEL-4** — ordering: rehire/terminate swap → active account for a terminated person.
3. **R-OPS-7** — empty or truncated feed interpreted literally → mass deactivation.
4. **R-ORG-1** — disputed attribute authority → sync "bugs" that are actually governance disputes.

---

## Appendix A — Event classes & default SLO targets

Taxonomy referenced by R-LAT-1/2/4. Targets are defaults for negotiation, to be ratified per deployment and recorded alongside this catalog; the revocation row is a monitored security control per R-LAT-2, not a service courtesy.

| Event class | Examples | Default target (source commit → registry effective) | Rationale |
|-------------|----------|-----------------------------------------------------|-----------|
| Revocation / leaver | Termination, security disable, credential compromise signal (CAEP) | ≤ 5 minutes | Open access window is direct security exposure; measured and alerted per R-LAT-2. |
| Joiner / provisioning | Hire, contract start, rehire | Before start of first workday | Productivity risk, not security risk; overnight batch can satisfy it. |
| Mover / entitlement-relevant change | Department, manager, cost center, job profile change | ≤ 1 hour | Drives birthright recalculation, approval routing, SoD evaluation. |
| Routine attribute change | Preferred name, phone, location, display attributes | ≤ 24 hours | Cosmetic; correctness matters, freshness barely does. |
| Org-structure / bulk change | Reorg, mass cost-center remap, M&A load | ≤ 24 hours, burst-tolerant | Volume dominates speed; the R-LAT-4 burst case. Must not preempt the revocation lane while draining. |

Classification is performed at ingest (adapter or transform stage), not by the source: sources emit changes, the pipeline assigns the class that selects the SLO and the processing lane.

## Appendix B — Glossary

| Term | Meaning here |
|------|--------------|
| NRT | Near-real-time: seconds-to-minutes propagation, as distinct from scheduled batch. |
| Registry | The IAM/IGA system's identity store — the sync target and system of record for access decisions. |
| Watermark | Persisted high-water mark (timestamp, sequence, or vendor delta token) marking how far a poller has consumed a source. |
| Delta / delta token | Vendor mechanism returning only records changed since a prior token — the basis of family **02** (incremental / delta polling). |
| Tombstone | Explicit record that an entity was deleted, compensating for sources whose delta feeds omit deletions. |
| CDC | Change data capture: reading a database's transaction log to emit row-level change events. |
| DLQ | Dead-letter queue: durable parking for events that failed processing, retained for repair and replay. |
| Idempotency key | Immutable source identifier that makes reapplying the same event a no-op (R-DEL-1/2). |
| Correlation | Deterministic linking of records for the same person across sources (R-IDM-1). |
| Reconciliation | Periodic full comparison of registry state against source truth, independent of the event path (R-COR-1/3). |
| SCIM | System for Cross-domain Identity Management — standard schema and REST API used here as the registry's ingest contract. |
| SET | Security Event Token (RFC 8417): signed JWT carrying a security event. |
| SSF / CAEP | OpenID Shared Signals Framework / Continuous Access Evaluation Profile: standardized SET streams for revocation and risk signals. |
| iPaaS | Integration-platform-as-a-service middleware (e.g. Workato, Boomi, MuleSoft). |
| Blast-radius guard | Sanity threshold halting the pipeline when a change set is implausibly large or a feed implausibly empty (R-OPS-7). |
