# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`nrtsync2` is a research/writing project (not a code project) about **near-real-time (NRT) synchronization of access-related data from Systems of Record / ERPs into IAM/IGA and other provisioned systems**. The driving problem: when access rights depend on user data, stale data causes users to be wrongly granted or wrongly denied access, so the goal is to minimize the lag between a change in a System of Record and its propagation to downstream systems.

The central thesis: although the systems involved vary widely, they sort into a small set of **architectural families**, and NRT sync approaches map onto those families. The survey develops this as **seven families** read against a small number of axes — pull vs. push, batch vs. per-change, and (most decisively) who owns progress and who absorbs failure. The objective is reframed away from raw "low latency" toward minimizing the window of *decision-relevant staleness* without dropping or reordering the changes that make an access decision correct — weighing the revoke path (a late revoke is a standing exposure) more heavily than the grant path.

## Current state

The project is prose/notes — no source code, build tooling, or tests. There is nothing to build, lint, or run.

Contents:

- `memory/prelude.md` — background and motivation (the framing). Author: Keith Hazelton.
- `docs/nrt-sync-survey.md` — **"Keeping Access Current"**, the survey & critical review. Defines the seven architectural families (numbered **01–07**), an evaluation lens (freshness, delivery guarantee, recoverability, coupling/backpressure, source impact, semantic fidelity), a comparison matrix, a critical synthesis, and implementation tradeoffs. Family 06 (*sink-polled write-ahead log with watermark*) is the marked focus. Authors: Keith Hazelton, Claude Fable 5.
- `docs/nrt-sync-survey.html` — HTML rendering of the survey.
- `docs/requirements.md` — **Requirements Catalog**, keyed to the survey's 01–07 taxonomy. Requirement IDs are grouped by area (LAT, COR, DEL, IDM, ING, SCL, SEC, AUD, OPS, LCM, ORG) plus appendices (event classes & SLO targets; glossary). **Requirement IDs are stable — do not renumber them.**

The seven families (survey ordering):

1. **01** Scheduled full reconciliation
2. **02** Incremental / delta polling
3. **03** Standards-based push provisioning (SCIM)
4. **04** Event-driven webhooks
5. **05** Durable event streaming & log-based CDC
6. **06** Sink-polled write-ahead log with watermark ★ *focus*
7. **07** Continuous shared signals (SSF / CAEP / SCIM Events)

A sibling project `../nrtsync` (v1) exists in the same `devspace`; this is the second iteration.

## Conventions

- **`memory/`** holds framing prose (`prelude.md`) plus local Claude Code settings (`.claude/settings.local.json`). **`docs/`** holds the survey, its HTML render, and the requirements catalog.
- Markdown documents carry YAML frontmatter: `title`, `date`, `draft` (semver, e.g. `0.1.0`), `author`.
- Keep terminology consistent with what `prelude.md` and the survey establish: *System of Record / ERP*, *IAM/IGA*, *provisioned systems*, *near-real-time sync*, and the **01–07** family numbering.
- When editing `requirements.md`, reference approaches by their **01–07** family numbers and keep existing requirement IDs stable.

## Notes for future instances

- If this grows into a taxonomy tool or a codebase, update this file with the actual structure and any real commands — do not carry forward assumptions from the sibling `nrtsync` project without checking.
- To publish to GitHub, use the **sync2github** skill (target: `github.com/khazelton/nrtsync2`).
