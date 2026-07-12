# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A Markdown notes workspace, not a codebase. There is no build, lint, or test tooling — every file here is a `.md` document. Contents are ideas, personal notes, experiments, and early-stage design specs, per `README.md`.

Working in this repo means writing/editing Markdown documents, not writing or running code.

## Conventions (from README.md)

- Keep each note focused on one topic.
- Use clear headings and short sections.
- Update entries as ideas evolve rather than letting them go stale.
- Add a new Markdown file when a new thought starts to grow, rather than overloading an existing note.

## Current documents

- `hive-mind.md` — architecture spec for a "Knowledge Compiler": a source-agnostic pipeline (adapters → hash-diff → canonical document → embeddings/rules → model router → markdown wiki → search/serve) that compiles docs from markdown/Notion/Confluence into a lint/verify-able knowledge layer, cost-optimized for solo/local-first development.
- `intel.md` — spec for "Intel", a unified CLI that scans dependencies, config, and logs and surfaces prioritized findings (`intel scan`, `intel deps`, `intel config`, `intel logs`, `intel report`). Defines a canonical `Finding` dataclass, pluggable `Adapter`/`Synthesizer` protocols, and a free-by-default / opt-in-LLM cost model.

Both specs share a design principle worth preserving if either is ever implemented: keep a canonical, source/scanner-agnostic data layer (`Finding` in intel.md, the canonical document in hive-mind.md) as the strict boundary — everything above it is swappable per-source logic, everything below it (storage, reporting, serving) must stay agnostic and never change when a new adapter is added.

## Working with these documents

- These are design docs for projects that don't exist yet as code. If asked to implement one, treat the relevant `.md` file as the spec and confirm scope with the user before generating a project structure — the docs describe multi-stage build sequences (see "Build order" / "Build Sequence" sections) that are meant to be built incrementally, not all at once.
