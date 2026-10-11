# Hive Mind — Knowledge Compiler Architecture

Source-agnostic markdown knowledge layer: compile → lint → verify → serve.
Standalone version, cost-optimized for solo dev on local hardware (MSI GF75 Thin, 16GB RAM) before moving to a cheap VM.

## Full architecture

```text
════════════════════════════════════════════════════════════════════════════
  STANDALONE ARCHITECTURE — full map (dev on GF75, cost-optimized)
════════════════════════════════════════════════════════════════════════════

 LOCAL DEV/PILOT MACHINE (MSI GF75 Thin, 16GB RAM)  ──or──  $6-12/mo VM later
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│  ┌────────────────────── SOURCE ADAPTERS (pluggable) ──────────────────┐ │
│  │                                                                       │ │
│  │   ①  markdown/git   ②  Notion         ③  Confluence   ④ (future)     │ │
│  │   BUILD FIRST        build second      build last       ...          │ │
│  │   (no auth,          (clean API,       (messiest:                    │ │
│  │    no rate limit)     native webhooks)  v1/v2 API split)              │ │
│  │                                                                       │ │
│  │   each implements:  .list()  .fetch()  .watch()  .permissions()      │ │
│  └───────────────────────────────┬──────────────────────────────────────┘ │
│                                   ▼                                        │
│                    ┌──────────────────────────┐                           │
│                    │   HASH-DIFF CHECK         │  ← FREE. Skip anything    │
│                    │   (content checksum vs.   │    unchanged since last   │
│                    │    last-seen state)        │    sync. This is the     │
│                    └────────────┬─────────────┘    #1 cost lever.         │
│                                 ▼ (only new/changed docs pass)             │
│                    ┌──────────────────────────┐                           │
│                    │  CANONICAL DOCUMENT       │  ← deterministic parser,  │
│                    │  { id, source, title,     │    no LLM, no cost        │
│                    │    blocks[], metadata,    │                           │
│                    │    permissions[], links[]}│                          │
│                    └────────────┬─────────────┘                           │
│                                 ▼                                          │
│         ┌───────────────────────┴────────────────────────┐               │
│         ▼                                                  ▼               │
│  ┌─────────────────┐                              ┌──────────────────┐   │
│  │ LOCAL EMBEDDINGS │  ← Ollama, always local        │ RULE-BASED       │   │
│  │ (nomic-embed /   │     $0 marginal cost,          │ CHECKS           │   │
│  │  bge-small)      │     runs fine on GF75 GPU      │ - timestamp diff │   │
│  └────────┬─────────┘                                │ - numeric diff   │   │
│           ▼                                           │ - frontmatter    │   │
│  ┌─────────────────┐                                 │   mismatch       │   │
│  │ pgvector         │  ← inside Postgres, one box,     └────────┬─────────┘  │
│  │ (dedup/cluster)  │     no managed vector DB                  │            │
│  └────────┬─────────┘                                          │            │
│           │                                     catches ~80% of conflicts   │
│           │                                     for free; residual escalates│
│           └──────────────────┬───────────────────────────────┘            │
│                               ▼                                            │
│                  ┌─────────────────────────────┐                          │
│                  │   MODEL ROUTER (dev vs prod) │                          │
│                  └──────────┬──────────┬───────┘                          │
│                             ▼           ▼                                  │
│              ┌───────────────────┐  ┌────────────────────────────┐        │
│              │ OLLAMA (local)     │  │ CLAUDE API — Haiku, batch   │        │
│              │ llama3.1:8b /      │  │ ONLY for:                   │        │
│              │ qwen2.5:7b         │  │  • multi-doc synthesis       │        │
│              │                    │  │  • residual conflict calls  │        │
│              │ used for:          │  │  called ONLY on hash-diffed│        │
│              │  • dev/testing     │  │  new/changed content →      │        │
│              │  • simple y/n      │  │  small $/mo even in prod    │        │
│              │    classification  │  │                              │        │
│              └────────┬───────────┘  └──────────────┬───────────────┘      │
│                       └───────────────┬──────────────┘                     │
│                                       ▼                                    │
│                        ┌──────────────────────────┐                       │
│                        │     MARKDOWN WIKI          │  ← flat files,       │
│                        │  concept/entity pages,      │    git or R2/S3      │
│                        │  tiered index.md,            │    (zero egress      │
│                        │  frontmatter + provenance     │    on R2)           │
│                        │  + staleness flag per page    │                    │
│                        └─────────────┬────────────────┘                    │
│                                      ▼                                     │
│                        ┌──────────────────────────┐                       │
│                        │  SEARCH / SERVE            │  ← SQLite FTS5 or     │
│                        │  llms.txt + BM25/vector      │    Postgres FT,      │
│                        │  search, permission-filtered  │    same box         │
│                        └─────────────┬────────────────┘                    │
│                                      ▼                                     │
│                        ┌──────────────────────────┐                       │
│                        │  Claude Code / Copilot /    │                     │
│                        │  agents query this — never   │                    │
│                        │  raw sources directly          │                  │
│                        └─────────────┬────────────────┘                    │
│                                      │                                     │
│           feedback loop: answered queries filed back                       │
│           into wiki as new/updated pages (dashed ↑ to synthesis)           │
│                                                                            │
└──────────────────────────────────────────────────────────────────────────┘
```

## Cost summary at this stage

```text
  hosting .......... $0 (GF75) → $6-12/mo (VM, once running 24/7)
  embeddings ........ $0 (Ollama, local, always)
  vector db ......... $0 (pgvector, same box)
  search ............ $0 (SQLite FTS5 / Postgres FT)
  dev/test LLM calls  $0 (Ollama local)
  prod synthesis .... pennies-few $/mo (Haiku, batch, hash-diffed only)
  storage ........... $0-1/mo (Cloudflare R2)
  ────────────────────────────────────────────────────────────────
  TOTAL: near-$0 through dev, low single-digit $/mo once piloting
```

## Build order

1. **Markdown/git connector** — no auth, no rate limits, prove the loop end to end.
2. **Hash-diff + canonical document layer** — deterministic, no LLM, this is the #1 cost lever.
3. **Local embeddings (Ollama) + pgvector** — dedup/clustering, always free.
4. **Rule-based conflict checks** — timestamp/numeric/frontmatter diffing, catches most conflicts for free.
5. **Model router** — Ollama for dev/simple classification, Claude Haiku (batch, hash-diffed only) for synthesis and residual conflict resolution.
6. **Markdown wiki + search/serve layer** — flat files with provenance and staleness flags, SQLite FTS5 or Postgres full-text.
7. **Notion connector** — second source, reuses everything downstream untouched.
8. **Confluence connector** — last, since it's the messiest API surface (v1/v2 split, storage format).

## Key design principle

Everything above the canonical document layer is source-specific and swappable.
Everything below it is source-agnostic and should never change when a new connector is added.
The model router is the only thing that changes between dev and prod, or between solo-project and enterprise scale.