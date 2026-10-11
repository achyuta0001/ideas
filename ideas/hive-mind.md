# Hive Mind — Knowledge Compiler Architecture

Source-agnostic markdown knowledge layer: compile → lint → verify → serve.
Standalone version, cost-optimized for solo dev: build on the Mac, pilot on the always-on MSI GF75 Thin, then move to a cheap VM. See [premise corrections](#premise-corrections).

> **Built.** This design shipped as [hive](https://github.com/achyuta0001/hive) (Python, last commit
> 2026-07-18). This note is the original design; where the two differ, the code is the truth.
> Main differences: all state lives in stdlib SQLite (no Postgres or pgvector); Claude Code
> reads the wiki over MCP and Copilot as plain markdown; permissions are captured at ingest but
> not enforced yet; rule checks are hints the LLM must confirm or dismiss, never auto-resolved;
> Confluence is tested offline only.

## Verified facts (probed 2026-10-11, not assumed)

```text
  Mac        Apple M5, 16 GB unified, arm64, macOS 27.0.1
             sqlite3 3.54.0 CLI and Python's sqlite3 (3.53.4), both with FTS5
             Docker.app installed, so Postgres + pgvector can run as a
             container with no host install

  Not present on Mac: ollama, postgres (no psql / pg_config), pgvector

  GF75       NOT PROBED. tailscaled was not running on the Mac, so peer state
             is unknown. Last seen by pocket-agent (2026-08-22): achyutasmsi,
             Windows, offline for 3 days. GPU SKU and RAM unverified; the
             "16GB RAM" in the first draft is unconfirmed.
```

## Premise corrections

**1. Develop on the Mac. The GF75 is the always-on pilot host, not the inference box.**
The first draft ran everything on the GF75, including the 7-8B Ollama models.
[pocket-agent](pocket-agent.md) later worked out that the GF75's GPU has 4 or 6 GB of VRAM
depending on SKU, and the SKU is still unknown. `llama3.1:8b` and `qwen2.5:7b` at Q4 are about
5 GB before any context, so they spill into system RAM on a 4 GB card and are tight on 6 GB.
The M5 has roughly 10 GB usable for weights and runs them comfortably. Embedding models are
under 300 MB and fit anywhere.

```text
  dev     Mac M5. Runs the whole stack, 7-8B models included.

  pilot   GF75 over Tailscale. Connectors, Postgres + pgvector, embeddings,
          search/serve. None need much GPU; all of them benefit from
          staying on while the laptop is closed. For y/n classification
          run a 3-4B model there, or route it to Haiku.

  prod    $6-12/mo VM. No GPU, 1-4 GB RAM. Embeddings on CPU, all LLM
          work on Haiku. Only the model router changes.
```

**2. "Rule checks catch ~80% of conflicts" is a guess, not a measurement.**
This note never measured it. hive sidesteps the question: it feeds rule checks to the LLM as
hints instead of letting rules settle conflicts alone. The number only matters if you want rules
to skip LLM calls; then hand-label a sample of real conflicts and measure the catch rate first.
The "pennies-few $/mo" Haiku line in the cost summary depends on it and on how much content
changes per day.

## Full architecture

```text
════════════════════════════════════════════════════════════════════════════
  STANDALONE ARCHITECTURE — full map (Mac dev, GF75 pilot, cost-optimized)
════════════════════════════════════════════════════════════════════════════

 DEV: Mac M5  ·  PILOT: MSI GF75 Thin (always-on)  ──or──  $6-12/mo VM later
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
│  │  bge-small)      │     fits Mac, GF75 or CPU      │ - timestamp diff │   │
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
│              │ OLLAMA (Mac, dev)  │  │ CLAUDE API — Haiku, batch   │        │
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

## Open questions

- **Search engine.** SQLite FTS5 or Postgres full-text? Postgres is already on the box for
  pgvector, so one engine means one thing to back up. FTS5 is zero-ops and already works on
  the Mac. Decide before step 6. *hive: stdlib SQLite for all state; Postgres never arrived.*
- **Wiki storage.** git or R2? git gives history, diffs and provenance for free. R2 only earns
  its place if the serve layer runs somewhere without a checkout.
- **Permissions across sources.** One synthesized page can merge an open Notion page with a
  restricted Confluence one. Who may read the result? The intersection of source ACLs is the
  safe default but hides a lot. And whose identity does a querying agent carry: yours, or a
  service account's? *hive: captured per page at ingest, enforced nowhere yet (one reader).*
- **Feedback-loop trust.** Answered queries get filed back as wiki pages. What stops a wrong
  answer from becoming a source? At minimum a provenance tag and a lower trust tier; maybe a
  review step.
- **Serve interface.** Plain files plus llms.txt that Claude Code reads directly, or an MCP
  server? Files are simpler, but an agent reading files sees everything, so permission
  filtering can only be enforced behind something like MCP. *hive: both. MCP tools for Claude
  Code, plain markdown for Copilot.*
- **Webhook reachability.** Notion's servers have to reach the webhook endpoint, and the GF75
  is only on the tailnet. Tailscale Funnel, a small relay, or poll instead?
- **VM sizing.** Does a $6-12/mo VM hold Postgres + pgvector + CPU embeddings for the corpus
  you expect? Measure on the pilot before paying for it.

## Related

- [pocket-agent](pocket-agent.md) — the same compile/serve idea at personal scale: `pocket ask`
  (its build step 7) searches a drive-resident vault instead of a team wiki.
