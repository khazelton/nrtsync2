# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`nrtsync2` is a research/writing project (not yet a code project) about **near-real-time (NRT) synchronization of access-related data from Systems of Record / ERPs into IAM/IGA and other provisioned systems**. The driving problem: when access rights depend on user data, stale data causes users to be wrongly granted or wrongly denied access, so the goal is to minimize the lag between a change in a System of Record and its propagation to downstream systems.

The central thesis being developed: although the systems involved vary widely, they can be sorted into a small set of **categories based on their architecture**, and NRT sync approaches map onto those categories.

## Current state

- The entire repository is `memory/prelude.md` — the background-and-motivation section.
- No source code, build tooling, tests, or git history exist yet. There is nothing to build, lint, or run.
- A sibling project `../nrtsync` (v1) exists in the same `devspace`; this is the second iteration.

## Convention: `memory/`

Prose and notes live as Markdown in `memory/`. `prelude.md` establishes the framing; expect additional documents to extend the argument (e.g. the taxonomy of system architectures, per-category sync strategies). Keep new writing consistent with the terminology already set in `prelude.md`: *System of Record / ERP*, *IAM/IGA*, *provisioned systems*, *near-real-time sync*.

## Notes for future instances

- When this grows into a taxonomy or a codebase, update this file with the actual structure and any real commands — do not carry forward assumptions from the sibling `nrtsync` project without checking.
