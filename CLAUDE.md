# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A Markdown notes workspace, not a codebase. There is no build, lint, or test tooling — every file here is a `.md` document. Contents are ideas, personal notes, experiments, and early-stage design specs, per `README.md`.

Working in this repo means writing/editing Markdown documents, not writing or running code.

## Layout

- `README.md` — the index: one row per idea with its status. Status lives only there.
- `ideas/` — one note per idea, plus list notes (`system-design-ideas.md`, `backlog.md`).
- `ideas/_template.md` — starting point for a new note. Copy it, then add a row to the index.

## Conventions (from README.md)

- Keep each note focused on one topic.
- Use clear headings and short sections.
- Update entries as ideas evolve rather than letting them go stale.
- Add a new Markdown file when a new thought starts to grow, rather than overloading an existing note.
- Date verified facts ("probed YYYY-MM-DD") and keep guesses out of that section.

## Working with these documents

- Check the status in the README index before treating a note as unbuilt. Some ideas already exist as code in their own repos (hive-mind as hive, intel as tripwyre). There the code is the truth and the note is the original design.
- For an idea that isn't built yet, treat its note as the spec and confirm scope with the user before generating a project structure. The notes describe multi-stage build orders that are meant to be built incrementally, not all at once.
- The repo is public. Keep secrets, IP addresses and other private details out of notes.
- hive-mind and intel share a design principle worth keeping in new designs: a canonical, source/scanner-agnostic data layer (`Finding` in intel.md, the canonical document in hive-mind.md) as the strict boundary. Everything above it is swappable per-source logic; everything below it (storage, reporting, serving) stays agnostic and never changes when a new adapter is added.
