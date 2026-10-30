# Estate context for AI coding sessions — handoff

**Written 2026-10-31.** Self-contained. Every capability claim is dated; anything unverified is marked **UNCONFIRMED**. Measurements name the method that produced them so they can be reproduced or challenged.

---

## 1. The problem and who has it

A large firm runs roughly 30–35 microservices and micro-frontends plus 5–7 legacy repositories, with **no distributed tracing** and a bare-bones Kubernetes setup. Documentation lives in Confluence; tickets in Jira; the micro-frontends compose via webpack **Module Federation**. A developer starting a task in one repo has no reliable way to learn what calls them, what they call, which contract they are about to break, or who owns the thing downstream — so either an AI coding agent burns thousands of tokens exploring, or the developer clones repos and reads Confluence, or (most often) both guess. The goal is that a developer working anywhere in that estate gets the right context inside their AI coding session — Claude Code first, other agents without a rewrite — without cloning repos, reading Confluence, or paying for exploration. **Hard requirement: self-hostable.** Enterprise code and docs must not leave the customer's network; the product must deploy on the team's own dev Kubernetes cluster or their own GCP project.

---

## 2. Design ancestry (the author's two prior systems)

These are not background colour; the new design inherits a specific habit from them and upgrades it once.

**hive** (`github.com/achyuta0001/hive`, Python, SQLite). Compiles git + Notion + Confluence through hash-diff into *canonical documents*, then into a linted markdown wiki, served to Claude Code over MCP. The Confluence connector was deliberately built last and is tested offline only. Its design principle, which carries forward verbatim:

> Everything above the canonical document layer is source-specific and swappable; everything below it is source-agnostic and never changes when a connector is added.

**recall** (designed, not built). Local search over Claude Code transcripts, memory files, an Obsidian vault and git commits into SQLite FTS5, indexed at query time with no daemon. Two rules carry forward:

- *Records are the source of truth; the FTS index is derived and can be rebuilt without touching the sources.*
- Deletion asymmetry: **archive** what someone else deletes (transcripts, commits); **mirror** what you delete (notes, memory files).

**The upgrade (settled).** Keep the canonical-boundary habit, but the canonical unit for this product is **`Contract` and `Edge` (each carrying evidence and confidence)**, not `Document`. Rationale: documents are what vendors will own — Atlassian, Notion and Anthropic are all converging on serving documents to agents. **Typed contracts and typed edges with evidence are what no vendor can see**, because they require reading one specific estate's manifests, webpack configs and broker metadata. Build the layer nobody else can build, and let documents be an input to it.

---

## 3. Decision log

Settled decisions are numbered S; open questions are in §8. Each carries its reasoning, and a citation or measurement where one exists.

### Graph construction

**S1. Build the dependency graph from declarations, not traces — there is no tracing. Settled.**
Sources, best to worst, with the confidence tier each produces:

| Source | Tier | Why it ranks here |
|---|---|---|
| Deployment env vars holding service URLs (`*_URL`, `*_HOST`, `*_ENDPOINT`) | `declared` | Present in a bare-bones K8s estate, machine-readable, usually accurate because the pod would not start otherwise |
| **Module Federation `remotes` in webpack config** | `declared` | **Exact, explicit, zero inference.** The strongest signal in the whole estate and the reason the micro-frontend half is tractable at all |
| Kafka consumer-group metadata from broker admin API | `declared` | Real runtime subscription, not an intention; gives produce/consume edges a manifest cannot |
| Ingress rules | `declared` | Edge-of-cluster routing only; says nothing about service-to-service |
| Generated SDK imports | `derived` | Good signal where codegen is used, absent in the legacy repos |
| ingress-nginx access logs | `observed` | A later tier that **substitutes for tracing**: real traffic, no instrumentation, but only what crosses ingress |

**S2. Coverage honesty is a product feature, not a disclaimer. Settled.**
Every impact answer reports its own coverage inline, naming the gaps: *"28/35 services mapped; 7 unmapped (no declared dependencies): legacy-billing, legacy-auth, … impact may be incomplete."* Reasoning: a tool that hides what it cannot see gets one wrong answer and then nobody uses it again. In an estate with 5–7 legacy repos and no tracing, incomplete coverage is the permanent condition, so it must be a first-class field in the output rather than something to apologise for later.

**S3. Freshness is a correctness problem, not a UX nicety. Settled.**
Every card carries `fetched_at` **and the deployed image digest**. The reader compares the card against live state and warns inline when the card predates the running version. The specific failure being designed against: the agent writes code against a contract that changed yesterday, the developer's CI passes because their own repo still compiles, and production breaks at the boundary. Staleness here is indistinguishable from a bug.

**S4. Pin, don't copy. Settled.**
Store identity and pointers — `order-api@a3f9c21:/openapi.yaml` — and derive content on demand. Duplicate only what ranking needs (names, a one-line purpose, keywords). Reasoning: duplication is the primary cause of staleness, and a pointer plus a hash is falsifiable where a copy is not.

**S5. Event-driven beats nightly. Settled.**
A deploy hook, a merge to main, or a Kafka topic change re-extracts **one** service. Hash-diff is the cost lever (inherited directly from hive). A nightly full sweep stays as the floor and the reconciliation pass, not as the primary path.

### Output discipline

**S6. The service proposes, humans merge. Settled.**
The highest-value output is a **reviewable pull request**, not a commit. Two lanes:

- **Mechanical** — image-digest bumps, owner changes, link updates: batched into **one squashed daily bot commit**.
- **Meaningful** — a contract changed, a new edge appeared, a service vanished: a **PR tagging CODEOWNERS**.

Reasoning: 400 bot commits a day makes `git log` useless, and the moment `git log` is useless the humans stop reviewing, at which point the content is unowned and the whole thing rots.

**S7. Generated and hand-written content must be physically separated. Settled.**
The generator owns `services/*.md` and regenerates them wholesale. It **never** touches `overrides/*.md`. Renderers merge the two at build time, overrides winning. Reasoning: without this split the first human correction is silently reverted on the next run, and trust does not recover from that.

**S8. TTL on prose. Settled.**
Hand-written cards carry `reviewed_by`, `reviewed_at`, `ttl: 90d`. Past TTL the card warns in-session and the bot opens a "still true?" PR to the owner. Modelled on the author's existing habit of dating verified facts. Note the limit honestly: a TTL detects *age*, never *wrongness*. Detecting wrongness requires S9.

**S9. Drift detection requires the graph; without it, it is a date field.** Settled as a dependency, not yet built.
"This card claims we call `svc-pricing`; no manifest, env var or Kafka edge supports that edge at HEAD any more; here is a PR deleting the sentence" is only possible with an independent extracted source of truth to contradict the prose. This is the single reason the graph has to exist before the drift engine is worth anything.

### Intent side

**S10. Two boundaries, not one. Settled.**
`Contract`/`Edge` on the **input** side; `Change` on the **intent** side. A new trigger costs exactly one adapter. **Renderers read the store and never a source**, so a PR comment and an MCP call return the same answer by construction — which is also what makes the answers testable.

**S11. Intent sources normalise to a `Change`. Jira is one of seven, not the headline. Settled.**
Ranked by value:

1. **PR diff in CI** — best. No agent, no plugin, no adoption needed; runs on every PR; the output is a comment a human already reads.
2. **Deprecation sweep** — "who still consumes v2?" The question that blocks every migration and that nobody can currently answer.
3. **Incident triage** — reverse traversal: symptom → recent upstream changes.
4. **Release coordination** — what ships together.
5. **Architecture review** — fan-out, cycles, god services.
6. **Onboarding** — the generic case, lowest urgency.
7. **Agent code review** — the case everyone demos and nobody adopts first.

**S12. Ticket-to-brief uses a deterministic ladder; the LLM is the last rung. Settled.**
In order, stopping as soon as a rung produces a confident answer:

1. Structured Jira fields (components, labels, fix versions).
2. **Linked PRs and issues** — a linked PR names the exact repos and files; this is near-free and high-precision.
3. **Named-entity matching against the service inventory.** The vocabulary is **closed** (35 names plus aliases), so dictionary matching beats embeddings on precision, cost and explainability.
4. **History join** — ticket→service pairs learned from past tickets (see `TicketLink`).
5. **Constrained LLM read, last.** Output enum-restricted to the known service list, plus a mandatory **verbatim-quote evidence field used as a hallucination filter**: if the quote is not byte-present in the ticket, the extraction is discarded.

**Empty is a legitimate answer.** A brief that says "no services identified" is correct and useful; a brief that guesses is the −1 failure in §6.

**S13. Two sleeper features nobody else ships. Settled as differentiators.**
- **Prior art** — closed tickets that touched the same services: "how did we do this last time."
- **Conflict detection** — an open PR already editing the same contract.
Both fall out of the data model almost free once `Change` and `TicketLink` exist, and neither appears in any surveyed competitor.

**S14. If impact returns 30 of 35 services, that is a finding, not a result. Settled.**
A fan-out that large means a shared database or a god service. The honest output is *"this change shape is wrong; here is the seam"* — naming the shared node — rather than a 30-item list that the developer will ignore. This is the output that earns architectural credibility.

### Injection and delivery

**S15. One skill, not thirty-five. Settled, forced by a measured platform constraint.**
Claude Code budgets the skill listing at **1% of the model's context window** and, on overflow, **drops descriptions starting with the skills you invoke least**, with no UI warning (debug log only) — [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills), verified 2026-10-30. One skill per service therefore does not merely cost tokens: 35 service skills silently evict each other's descriptions *and the team's other skills' descriptions*, making triggering unreliable in a way that is invisible. Correct shape: **one skill** whose body is a one-line-per-service index plus instructions, with per-service detail in `references/<service>.md` loaded on demand.

**S16. Never inline the catalogue in CLAUDE.md. Settled, measured.**
35 service cards inlined cost **+12,125 tokens on every turn of every session** (§4). `@path` imports do not help: "Imports help you organize a long file but don't reduce its context cost, because imported files load together with the CLAUDE.md that imports them" ([memory](https://code.claude.com/docs/en/memory), 2026-10-30). CLAUDE.md / AGENTS.md gets a **≤40-line routing header** only.

**S17. Delivery rail. Settled after cross-examination.**

| Priority | Rail | Immune to the provider bypass (§4.6)? | Admin cost |
|---|---|---|---|
| 1 | **Repo-level committed `.claude/settings.json`** with `extraKnownMarketplaces` + `enabledPlugins` | **Yes** — not a managed source | **Zero.** Six lines, travels with the repo |
| 2 | `managed-settings.json` (+ `managed-settings.d/` drop-ins) via existing config management | **Yes** | One file, one playbook |
| 3 | Claude apps gateway `managed.policies` | **Yes** | OIDC app + Postgres + container; needed only for per-group |
| 4 | Server-managed settings (claude.ai admin console) | **No** | Lowest, but unusable on Vertex/Bedrock/Foundry |

Rail 1 is primary: it needs nobody's permission, it is immune to the bypass because it is not in the managed tier at all, and for a 35-repo estate the same CI job that builds the plugin can commit the six lines from one template. Caveat from the docs: a repository's `extraKnownMarketplaces` **applies only after the contributor trusts the folder**, and in `-p`/CI runs only where trust was already accepted or `hasTrustDialogAccepted` is set in `~/.claude.json` ([plugins/org](https://code.claude.com/docs/en/plugins/org), 2026-10-30).
Plugin **version = content hash**; marketplace entry sets `autoUpdate: true`.
**Cadence ladder:** daily scheduled CI → event-driven re-extraction (S5) → the `command` plugin source for per-session refresh (it re-runs once per session). **CI must never commit to the branch humans PR against** (S6, S7).

**S18. Agent-agnostic by construction, Claude Code first-class. Settled.**
One source of truth, N thin emitters: `AGENTS.md` + `references/` works today in ~20 tools; a Claude Code plugin (skill + `SessionStart` hook + optional `.mcp.json`); `.cursor/rules/*.mdc`; `.github/instructions/*.instructions.md`. Each emitter is tens of lines. **Do not build an agent-abstraction layer** — the token-budget mechanics differ enough between tools (Claude Code's 1% skill listing vs Cursor's rule precedence) that one abstraction would be wrong everywhere.

**S19. No always-on component in the request path. Settled.**
The artifact path is static files on disk: no cold start, no uptime SLO, no on-call, works air-gapped via `CLAUDE_CODE_PLUGIN_SEED_DIR`, and survives the dev cluster being rebuilt. Extraction and ingestion run as CronJobs/CI jobs off the request path — if they are down for a day, context is a day stale rather than absent. An MCP server is **strictly additive**, for live and per-user-authorised queries only.

**S20. Do not proxy Confluence. Settled, reversed during cross-examination.**
Original plan was a server calling Confluence as the developer via IdP token exchange. **Dropped.** On Atlassian Cloud, point at the **Atlassian Rovo MCP server**: GA, covers Jira/Confluence/Compass/JSM/Bitbucket, OAuth 2.1, and "actions respect users' existing access controls and permissions" ([developer.atlassian.com/cloud/rovo-mcp](https://developer.atlassian.com/cloud/rovo-mcp/), docs updated 2026-06-30). Building our own proxy would re-implement an authorisation boundary we do not own and would need audience-bound tokens per upstream under RFC 8707. Keep **curated ingestion-time mirroring** as the universal fallback — needed anyway for Confluence Data Center (see §8) and as the substrate for S8/S9.

**S21. Per-audience content splitting is for relevance, never for authorisation. Settled.**
Splitting into per-audience plugins so frontend engineers do not carry 20 backend cards: correct, do it, two or three plugins. Using it to enforce an ACL: invalid. Anthropic states the tier "operates as a client-side control, not a security boundary. On unmanaged devices, a user doesn't need admin or sudo access to bypass them" ([server-managed-settings](https://code.claude.com/docs/en/server-managed-settings), 2026-10-30). Plugin files land as plaintext in `~/.claude/plugins/` for everyone who ever matched the policy, with no revocation and no access log; a plugin dropped from `enabledPlugins` stays on disk unless the marketplace sets `forceRemoveDeletedPlugins`. **A server is required when revocation and audit are requirements — not merely when visibility is.**

**S22. The graph is the product; the markdown artifact is its delivery format. Settled late, and it reverses an earlier emphasis.**
Injection is a solved problem: the correct design costs **~550 tokens per session** (§4) and consists of six lines of settings, one skill and a `references/` directory. Distribution was commoditised by Anthropic on **2026-06-29** with the Claude apps gateway. Any competent platform team can build the week-1 artifact in a sprint. The only compounding asset is **correct identity resolution across one specific heterogeneous estate** — the same service appearing as a chart name, a K8s `Service` name, a Kafka `client.id` and a Module Federation remote name, all different; a consumer group matching no service name; a remote declared behind an env var in a webpack config; a URL buried in a Helm value; six legacy repos with no manifests at all. The extractor is a weekend; the **tail** is iterative empirical work and it is different in every estate. Ship the artifact early to get a team using it and to establish baselines, but **staff the graph from week 1**.

---

## 4. Verified platform facts

All verified **2026-10-30** against primary documentation unless stated. Tested against **Claude Code v2.1.306**, model `claude-opus-5-5`, 1M-token context window, macOS. Claude Code documentation moved from `docs.claude.com/en/docs/claude-code/*` to `code.claude.com/docs/en/*` (HTTP 301, observed 2026-10-30).

### 4.1 Measured token costs — method first

**Method, so this is reproducible and challengeable:** `claude -p "Reply with the single word: ok" --output-format json --strict-mcp-config`, summing `input_tokens + cache_creation_input_tokens + cache_read_input_tokens` from the JSON `usage` block. Runs were **bracketed** — a control immediately before and after each treatment — because the environment drifts between batches; within a bracket, control repeatability was **±2 tokens** (40,258 / 40,261 / 40,262 across three runs). Fixtures were 35 synthetic service cards (owner, repo, upstream/downstream deps, Kafka topics and consumer groups, endpoints, runbook, SLO, gotchas) at 695 bytes each. The host machine carried a large pre-existing plugin set, so **absolute totals will differ on a clean install; the within-bracket deltas are the transferable numbers.**

| Configuration | Total input tokens | Δ vs control |
|---|---|---|
| Bare floor — `--setting-sources ''`, no plugins, no MCP | **28,824** | — |
| **Control** — host plugin set + 7-byte CLAUDE.md | **40,260** | 0 |
| + 35 cards **inlined in CLAUDE.md** (24,332 bytes) | 52,385 | **+12,125** |
| + 35 **skills**, 195-char descriptions | 40,673 | **+413** |
| + 35 **skills**, 1,376-char descriptions | 39,332 | **−928** |
| + **5-tool MCP server**, tool search ON (default) | 40,389 | **+129** |
| Control with `ENABLE_TOOL_SEARCH=false` | 63,184 | **+22,924** |
| + 5-tool MCP server, tool search OFF | 63,827 | +643 (vs 63,184) |

Three readings:

- **Dense markdown tokenises at ~2.0 bytes/token** (24,332 ÷ 12,125) because service cards are full of identifiers, backticks, paths and versions. Prose runs ~3.5–4. Either way the inline catalogue is a four-figure per-turn tax.
- **The skill listing is a capped budget, not a per-skill cost.** 35 large descriptions made the total go *down*, because overflow evicted other skills' descriptions. See §4.3.
- **MCP is nearly free at startup under tool search, and expensive without it.** The +22,924 figure is what a large tool surface costs when tool search is unavailable — which happens in real configurations (§4.4).

**Recommended design's always-on cost: ~550 tokens** — one skill description, one 5-tool MCP server at +129, and a ≤40-line `SessionStart` header. On-demand: ~350 tokens per service card read. A three-service task therefore costs ~1,600 tokens against **12,125** for the inline approach.

### 4.2 CLAUDE.md / AGENTS.md

| Fact | Detail |
|---|---|
| Managed CLAUDE.md paths | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`; Linux/WSL `/etc/claude-code/CLAUDE.md`; Windows `C:\Program Files\ClaudeCode\CLAUDE.md` |
| Load order | Root → cwd, concatenated, `CLAUDE.local.md` appended after `CLAUDE.md` at each level; subdirectory files load on demand when Claude reads/writes a file there |
| Imports | `@path`, max depth 4, **do not reduce context cost** |
| Size guidance | "target under 200 lines per CLAUDE.md file" |
| HTML comments | Block-level `<!-- -->` stripped before injection — free maintainer notes |
| Exclusions | `claudeMdExcludes` skips other teams' files in a monorepo |
| **AGENTS.md read directly** | **Since v2.1.277**, but **only when no `CLAUDE.md`/`.claude/CLAUDE.md`/`CLAUDE.local.md` exists in cwd or above**. A `CLAUDE.local.md` silently disables it |
| Override | `/config` → **Project instructions**, or `pluginConfigs` → `cc-plugin-agents-md@builtin` → `{"options":{"instructionFiles":"claude-md-and-agents-md"}}`. Read from **user settings, `--settings`, or managed settings only — ignored in project and local settings** |
| Not read | `AGENTS.local.md`, `AGENTS.override.md`, anything under `.agents/` |
| Difference | `InstructionsLoaded` hooks **do not fire** for an AGENTS.md read via the setting (they do fire for one imported from a CLAUDE.md) |
| Stewardship | Agentic AI Foundation under the Linux Foundation; "over 60k open-source projects" ([agents.md](https://agents.md/)) |
| Listed supporters | Codex, Jules, Factory, Aider, goose, opencode, Zed, Warp, VS Code, Devin, UiPath, Junie, Amp, Cursor, RooCode, Gemini CLI, Kilo Code, Phoenix, Semgrep, GitHub Copilot coding agent, Ona, Windsurf, Augment Code |
| GitHub Copilot | Coding agent reads `.github/copilot-instructions.md`, `.github/instructions/**.instructions.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md` ([changelog 2025-11-05](https://github.blog/changelog/2025-11-05-copilot-coding-agent-supports-organization-custom-instructions/)); **org custom instructions GA 2026-04-02** ([changelog](https://github.blog/changelog/2026-04-02-copilot-organization-custom-instructions-are-generally-available/)) |

### 4.3 Skills

| Fact | Detail |
|---|---|
| Body loading | "a skill's body loads only when it's used, so long reference material costs almost nothing until you need it" |
| **Listing budget** | **1% of the model's context window.** On overflow, descriptions are dropped **starting with the least-invoked skills**; warning goes to the debug log only |
| Tuning | `skillListingBudgetFraction` (default `0.01`), `SLASH_COMMAND_TOOL_CHAR_BUDGET` (fixed char count), `skillListingMaxDescChars` |
| Per-entry cap | combined `description` + `when_to_use` truncated at **1,536 characters** regardless of budget |
| `skillOverrides` | set a low-priority skill to `"name-only"` to free budget |
| Post-compaction | invoked bodies re-injected, capped **5,000 tokens per skill / 25,000 total**, oldest dropped first, **truncated from the top** — put critical instructions first |
| Locations | enterprise (managed settings dir), `~/.claude/skills/`, project `.claude/skills/`, nested `<subdir>/.claude/skills/`, `--add-dir`, plugin `skills/`, claude.ai-synced |
| **Inline shell** | `` !`cmd` `` and fenced ```` ```! ```` blocks execute **at invocation** and splice output into the body. `allowed-tools: Bash(...)` pre-approves; a deny or ask rule **aborts** the invocation; 2-minute Bash timeout; non-zero exit aborts (exception: exit 1 from search/compare commands). Killable org-wide with `disableSkillShellExecution`. **Commands in claude.ai-synced skills do not run on the developer's machine** |
| Diagnostics | `/doctor` estimates listing cost; `/skill-doctor` (shipped 2026-09-04) finds unused skills |

### 4.4 MCP

| Fact | Detail |
|---|---|
| Transports | **HTTP recommended**; **SSE deprecated**; stdio local; WebSocket only via `.mcp.json` / `claude mcp add-json` |
| Tool search | **On by default.** "Only tool names and server instructions load at session start, so adding more MCP servers has minimal impact on your context window" |
| Tool search unavailable | `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` set; the **HIPAA configuration** (and "neither `ENABLE_TOOL_SEARCH` nor managed settings turns it back on"); **Microsoft Foundry on Azure** rejects it server-side |
| `ENABLE_TOOL_SEARCH` | `true` / `false` / `auto` / `auto:N`. `auto` loads tools upfront while their definitions total under **10%** of the window |
| Description limits | server instructions and upfront tool descriptions **4,096 chars**; tool-search-loaded descriptions **16,384 chars**; override both with `CLAUDE_CODE_MAX_MCP_DESCRIPTION_LENGTH` |
| Output limits | warn at **10,000 tokens**; default cap **25,000** (`MAX_MCP_OUTPUT_TOKENS`); spilled to a file above **50,000 characters** |
| Per-server | `alwaysLoad: true` forces upfront; `alwaysLoad: false` keeps everything deferred |
| Scope precedence | local → project → user → plugin → claude.ai connectors; the winning entry is used **whole**, fields never merge |

**Auth (Claude Code side):** OAuth 2.0 via `/mcp` or `claude mcp login <name>`; **Client ID Metadata Documents (CIMD)** auto-discovered; **pre-registered clients** via `{"oauth":{"clientId":"…","callbackPort":8080}}` plus `--client-secret`; static `headers`; **`headersHelper`** (a command run "fresh on each connection", printing a JSON object of string headers); `${VAR}` and `${VAR:-default}` expansion in `command`, `args`, `env`, `url`, `headers`. Credential-looking env vars (names containing `TOKEN`, `SECRET`, `PASSWORD`, `KEY`, `AUTH`) are **stripped** from helpers supplied by a repository or plugin — such helpers must read from a file or credential store. Project/local-scope helpers run only after folder trust (v2.1.238+).

**Auth (spec side), MCP revision 2026-07-28** ([spec](https://modelcontextprotocol.io/specification/latest/basic/authorization)): servers **MUST** implement RFC 9728 Protected Resource Metadata; clients **MUST** send the RFC 8707 `resource` parameter; servers **MUST** validate audience and "**MUST NOT accept or transit any other tokens**" — **token passthrough is prohibited**. Dynamic Client Registration (RFC 7591) is **deprecated**, retained for back-compat, superseded by CIMD. stdio transports **SHOULD NOT** use OAuth and should "retrieve credentials from the environment."

**Central MCP delivery:**

| Mechanism | Capability | Constraints |
|---|---|---|
| `managedMcpServers` (managed settings, **v2.1.259+**) | Remote servers pushed to every user alongside their own; accepts the `oauth` member | **`http`/`sse` only; `https` only** (plain `http`, including localhost, refused); **no** `command`/`args`/`env`/`headersHelper`; **no `${VAR}`**; `headers` values are readable by anyone who can read the managed settings; cached value **withheld until the server confirms the payload**, waiting up to 30 s before connecting |
| `managed-mcp.json` (system path, MDM/GPO only) | **Exclusive control** — only these servers load | Cannot be delivered via server-managed settings; supports stdio and `${VAR}` expansion from each user's environment; `--mcp-config` / `--strict-mcp-config` cause startup exit while present; `allowAllClaudeAiMcps` and `allowClaudeInChromeWithManagedMcp` are the carve-outs |
| `allowedMcpServers` / `deniedMcpServers` / `allowManagedMcpServersOnly` | Allow/deny filtering | `serverName` entries are **explicitly not a security control** — the name is a user-chosen label. Use `serverUrl` or `serverCommand` |

### 4.5 Hooks

33 events, including `SessionStart`, `UserPromptSubmit`, `InstructionsLoaded`, `PreCompact`, `PostCompact`, `ConfigChange`, `Setup`, `SessionEnd`.

**The cap that matters:** "A hook's `additionalContext`, `systemMessage`, and `initialUserMessage` strings, and its plain stdout, are capped at **10,000 characters**." Over the limit, output is written to a file and replaced by the path plus a ≤2,000-character preview — **and Claude is not asked to read the file.** So a hook can deliver roughly 2,500 tokens, once. That makes `SessionStart` a routing header ("you are in repo X on branch Y; services in scope: A, B, C; use the `service-context` skill"), **not** a delivery mechanism.

`SessionStart` re-runs on resume (`source: "resume"`, or `"fork"` with `--fork-session`) and on `compact`, so it can refresh context after compaction. It also accepts `initialUserMessage`, `watchPaths`, `sessionTitle`, `reloadSkills`. Hooks can be delivered by managed settings and by plugins; `allowManagedHooksOnly` restricts them to managed sources.

### 4.6 Settings, delivery and the provider bypass

Precedence, highest first: **managed settings → `--settings` → project local → shared project → user.** List keys **merge** across scopes (`fallbackModel`, `modelPicker`, `availableModels`, `modelSettings` excepted). Verify with `/status` → `Setting sources`; diagnose with `claude doctor`.

**The provider bypass — the single most important delivery fact.** From [server-managed-settings](https://code.claude.com/docs/en/server-managed-settings), Security considerations, verified 2026-10-30:

> **User configures a third-party model provider** | Server-managed settings are bypassed. This includes setting `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_MANTLE`, `CLAUDE_CODE_USE_VERTEX`, `CLAUDE_CODE_USE_FOUNDRY`, `CLAUDE_CODE_USE_ANTHROPIC_AWS`, or a non-default `ANTHROPIC_BASE_URL`.

And under Platform availability:

> If you export a `CLAUDE_CODE_USE_*` provider variable or a non-default `ANTHROPIC_BASE_URL` in your shell, Claude Code **skips the settings fetch** for your sessions… **You can't clear the export with a server-managed `env` block, because the block arrives through the fetch that the export prevents. An endpoint-managed settings `env` block doesn't restore the fetch either: Claude Code checks eligibility before it applies managed `env` blocks.**

Consequences: it is **not recoverable by policy** (the only fixes are removing the shell export, or setting the variable to `""` in **user** settings — i.e. it depends on the developer); and it is a **silent, per-developer, partial** failure — one engineer's `.zshrc` takes them out of the policy regime while teammates stay in it, detectable only via `claude doctor` → `Managed settings (remote)` or `/status` (v2.1.248+). **A team running Claude Code on Vertex in their own GCP project — the likely shape of this customer — cannot be configured from the claude.ai admin console at all.** Anthropic's named remedy is the self-hosted gateway.

Other server-managed-settings limits: Team/Enterprise plans only; Owner/Primary Owner role; requires `api.anthropic.com`; **"Settings apply uniformly to all users in the organization. Per-group configurations are not yet supported."**; cannot deliver `managed-mcp.json`; `policyHelper` and `wslInheritsWindowsSettings` are not honoured; fetch at startup plus hourly; `forceRemoteSettingsRefresh: true` makes startup fail-closed.

### 4.7 Claude apps gateway

Launched **2026-06-29** ([Anthropic blog](https://claude.com/blog/introducing-the-claude-apps-gateway), [docs](https://code.claude.com/docs/en/claude-apps-gateway)).

- A **self-hosted control plane built into the `claude` binary**: one Linux binary plus **PostgreSQL 11+**; Docker Compose and Kubernetes/Cloud Run recipes documented. Boot is fail-closed on config, Postgres, OIDC discovery and upstream construction.
- Upstreams: Amazon Bedrock, Claude Platform on AWS, Google Cloud's Agent Platform, Microsoft Foundry, Anthropic API. "the upstream API key or cloud credential lives only in your infrastructure."
- **OIDC only. "SAML and LDAP aren't supported."** Discovery must be at `/.well-known/openid-configuration`. `groups_claim` defaults to `groups`; "Microsoft Entra emits app roles under `roles`"; "Google's id_token carries no groups claim" so the Workspace Directory API is used; **nested groups are not expanded**.
- **Must resolve to private addresses only** (RFC 1918, link-local, CGNAT, IPv6 ULA, loopback) — "a trusted gateway can push settings that run commands on developer machines."
- Gateway sessions **do not** fetch server-managed settings from claude.ai; policy comes from the gateway's `managed` block.
- **Per-group policy:** `managed.policies` are "evaluated in order; the first match is selected, then merged onto the `match: {}` catch-all base", matching on the JWT `groups` claim or `email_domain`. Allow-lists **replace** the base's; deny-lists and hook arrays **union**; `env`/`modelOverrides`/`skillOverrides` **shallow-merge**. Per-policy `managedMcpServers` is explicitly supported. For plugins: "That policy's `extraKnownMarketplaces` map **doesn't merge** with any other policy's, so list every marketplace the group needs in it."
- **Scaling limit of per-group policy:** first-match-wins plus non-merging `extraKnownMarketplaces` means a developer in two groups receives exactly one policy's marketplace map. Covering overlaps needs a policy per *combination* — 2ᴬ−1 policies, each listing the union, where ordering is correctness. Practical ceiling: **~4 disjoint audiences; it collapses at the first overlapping membership**, and it fails silently.

**Judgement: do not put the gateway on the critical path for shipping.** It is the right rail for per-group policy and for Vertex/Bedrock shops that want central config, but requiring it gates adoption behind another team's roadmap. Rails 1 and 2 of S17 need nobody's permission. Whether OIDC is a realistic ask: usually yes — Okta, Entra, Ping and Keycloak all speak it, and "we use SAML" normally means *our app integrations are SAML*, not *our IdP cannot do OIDC* — but it is a weeks-long identity-team ticket, and SAML-only IdPs (older Shibboleth, SiteMinder) are genuinely blocked.

### 4.8 Plugins and marketplaces

A plugin is one versioned directory of skills, agents, hooks, a hooks module (mod), and MCP servers, with a manifest at `.claude-plugin/plugin.json`. "for each skill, agent, and command that Claude can invoke on its own, the name and description are in Claude's context on every turn… The full text of a skill or agent loads only when it's used."

**Marketplace sources:** `github` (incl. `owner/*` in policy lists), `git` (GitLab/internal), **`url` (a hosted `marketplace.json`, with `headers` and `headersHelper` → an authenticated internal catalogue)**, `file`, `directory`, `settings` (inline catalogue, no hosted file), plus `hostPattern` / `pathPattern` / `skills-dir` in policy lists only.

**Plugin sources:** relative path, `github`, `url`, **`git-subdir`** (sparse/partial checkout of a monorepo subdirectory), `npm` (install scripts never run), **`archive`** (HTTPS zip with a `sha256` pin), and **`command`** (v2.1.229+) — a local command prints a plugin directory path; Claude Code **re-runs it once per session**, so content refreshes without reinstall. `copy` mode (≤256 MiB, ≤20,000 entries, version = file hash) or `link` mode (in place, no size limit, not on Windows). Shown to the user for review; disable org-wide with `disableCommandPluginSources`.

**Org controls:** `extraKnownMarketplaces`, `enabledPlugins` (`true` force-enables, `false` blocks at every scope and hides), `strictKnownMarketplaces` (**include `{"source":"skills-dir"}` or skills-directory plugins stop loading**), `blockedMarketplaces`, `disableSideloadFlags`, `strictPluginOnlyCustomization`, `allowManagedHooksOnly`, `pluginSuggestionMarketplaces`, `pluginTrustMessage`, `syncClaudeAiPlugins`/`syncClaudeAiSkills`.

**Air-gapped:** build at image time with `CLAUDE_CODE_PLUGIN_CACHE_DIR`, mount read-only and set `CLAUDE_CODE_PLUGIN_SEED_DIR` (seed entries override the user's; `autoUpdate` forced off). In CI set `CLAUDE_CODE_SYNC_PLUGIN_INSTALL=1` or the first turn may run without the plugin. Fleet-wide update kill switch: `env.DISABLE_AUTOUPDATER=1` (keep plugin updates with `FORCE_AUTOUPDATE_PLUGINS=1`).

**Observability:** OTel events `claude_code.plugin_installed` and `claude_code.plugin_loaded` (third-party names redacted unless `OTEL_LOG_TOOL_DETAILS=1`); Enterprise-plan `GET /v1/organizations/analytics/plugins` for per-plugin install and invocation counts.

### 4.9 Platform risk, assessed 2026-10-30

| Anthropic capability | Status | Why it does not make this redundant |
|---|---|---|
| Claude apps gateway | Shipped 2026-06-29 | A delivery rail, not content. **Build on it.** |
| Managed plugin / skill / MCP distribution | Shipped | Pipes, no content |
| Org skills via claude.ai admin console (synced plugins/skills) | Shipped | **Cloud-dependent** (`syncClaudeAiSkills`), and synced skills' `` !`cmd` `` does not run locally — unusable for the self-host requirement |
| **Claude Projects** (shared repos + project instructions + project memory across parallel cloud threads) | **Public beta, Pro/Max/Ultra only; "aren't offered on Team or Enterprise plans yet"** | github.com only (not GHES/GitLab/Bitbucket); cloud-only; threads read each repo's `CLAUDE.md` "but not the repositories' skills, `.claude/settings.json`, or `.mcp.json`". Closest thing to this product and currently unavailable to the buyer |
| Self-hosted environments (cloud sessions on customer runners) | Public beta, Team/Enterprise, off by default | **Does not satisfy the hard requirement**: "The conversation itself, including prompts, responses, and tool results, goes to `api.anthropic.com`". Unavailable with Zero Data Retention or HIPAA |
| Native code indexing | None found | Anthropic's approach is agentic search (Glob/Grep/Read). **UNCONFIRMED as a negative** — absence of evidence across primary sources, not a positive confirmation |

**Cursor** (third-party reporting only, **UNCONFIRMED** against cursor.com/docs): Organizations GA 2026-06-03; dashboard-managed Team Rules that admins can mark required, precedence team → project (`.cursor/rules`) → personal; 2026-06-30 a team MCP server configurable once and distributed to IDE/CLI/cloud agents.

**Net read:** every platform is converging on *config distribution* and all of them leave *"what is true about our 35 services"* to the customer. The risk is not redundancy — it is that a product which is only pipes was already commoditised in June. See S22.

### 4.10 Prior art (survey by a separate agent, 2026-10-30)

| Project | Does | Caveat |
|---|---|---|
| **Kubedian** (Apache-2.0) | Service dependency graph from K8s/Kustomize manifests, env vars holding service URLs, Ingress, NetworkPolicy; SQLite + markdown output; MCP server with `service_impact`; confidence tiers | **Closest overlap with S1.** Early, single maintainer |
| **Repowise** (AGPL-3.0) | Multi-repo contract linking, blast radius, keeps CLAUDE.md/AGENTS.md current, flags doc drift | AGPL network clause will likely be blocked in an enterprise |
| **Qodo Software Map** (beta 2026-09-15) | Per-PR cross-repo blast radius | Overlaps S11.1 |
| **Nx Polygraph** (2026) | Cross-repo work via a Claude Code slash command | Nx-centric |
| Backstage, Cortex, OpsLevel, Port, Atlassian Compass, Datadog Software Catalog | All ship MCP servers | Dependency data is **mostly manual YAML**, except Datadog which uses APM telemetry — unavailable here |

**Assessed as unoccupied:** Module Federation remotes aggregated cross-repo; Kafka consumer groups as dependency edges; a PR-reviewed git repo of markdown service cards as the product; a drift engine that opens PRs and puts TTLs on hand-written prose; ticket-to-brief with prior art plus conflicting open PRs.

---

## 5. Architecture

### 5.1 Data model

The canonical layer. Everything above it is source-specific and swappable; everything below it is source-agnostic.

**`Service`** — `id` (canonical), `aliases[]` (**the crux**: chart name, K8s `Service` name, Kafka `client.id`, Module Federation remote name, repo name, Compass/Confluence name — all routinely different), `kind` (`service` | `micro_frontend` | `legacy`), `repo`, `default_branch`, `language`, `owner_id`, `deployed_image_digest`, `fetched_at`, `coverage_state` (`mapped` | `unmapped_no_declared_deps` | `excluded`).

**`Contract`** — `id`, `service_id`, `kind` (`openapi` | `grpc` | `graphql` | `kafka_topic` | `mfe_remote` | `db_table`), `pointer` (`order-api@a3f9c21:/openapi.yaml` — identity, not a copy, per S4), `content_hash`, `version`, `fetched_at`.

**`Edge`** — `from_service`, `to_service` *or* `to_contract`, `kind` (`http_env` | `mfe_remote` | `kafka_produce` | `kafka_consume` | `ingress` | `sdk_import` | `observed_http`), **`evidence[]`** (each: `source`, `pointer`, verbatim `excerpt`), **`confidence`** (`declared` | `derived` | `observed`), `first_seen`, `last_seen`, `extractor_version`. Evidence and confidence are not metadata — they are what makes an answer reviewable and what makes S2 possible.

**`Owner`** — `id`, `team`, `slack`, `codeowners_pattern`, `confluence_space`.

**`Change`** — `id`, `trigger` (`pr` | `ticket` | `deploy` | `deprecation_sweep` | `incident` | `release` | `manual`), `services[]`, `contracts[]`, `raw_ref`, `extracted_at`, `extraction_method` (`structured` | `linked_pr` | `dictionary` | `history` | `llm`), `evidence_quote` (verbatim; the hallucination filter of S12).

**`TicketLink`** — `ticket_key`, `service_id`, `source`, `confidence`, `created_at`. Powers the history-join rung of S12 and the **prior art** feature of S13.

### 5.2 The two boundaries

```
  SOURCES (swappable)                CANONICAL (stable)            RENDERERS (read-only)
  ──────────────────────             ──────────────────            ─────────────────────
  K8s Deployments / env vars  ─┐
  webpack Module Federation   ─┤                                 ┌─ services/*.md cards
  Kafka broker admin          ─┼─► Contract + Edge ──────────────┼─ AGENTS.md + references/
  Ingress rules               ─┤     (evidence, confidence)      ├─ Claude Code plugin
  generated SDK imports       ─┤            │                    ├─ .cursor/rules, .github/instructions
  ingress-nginx logs          ─┘            │                    ├─ PR comment (CI)
                                            │                    └─ MCP server (phase 2)
  PR diff / Jira / deploy     ─┐            │
  deprecation sweep           ─┼─► Change ──┘
  incident / release / manual ─┘
```

**Renderers never read a source.** That is what guarantees a PR comment and an MCP call give the same answer, and it is what makes the §6 experiments meaningful.

### 5.3 Components and where each runs

| Component | Runs where | Notes |
|---|---|---|
| **Extractors** (one per source in S1) | k8s CronJob or CI job, in-cluster, read-only | Needs read on K8s API, git, Kafka admin. Hash-diff; event-driven per S5. **Off the request path.** |
| **Store** | SQLite file committed as a build artifact (or Postgres if the MCP server exists) | Derived and rebuildable from sources, per the recall rule |
| **Card renderer** | CI | Owns `services/*.md`; never touches `overrides/*.md` (S7); merges overrides at build time |
| **Plugin builder** | CI | Emits one skill + `references/` + `SessionStart` hook + `AGENTS.md`; version = content hash |
| **Marketplace** | git repo, or static HTTPS serving `marketplace.json` (+`headers`/`headersHelper`), or `archive` + `sha256`, or seed dir for air-gapped | S17 |
| **PR proposer (bot)** | CI | Two lanes per S6; must not commit to the branch humans PR against |
| **PR-comment renderer** | CI check on every PR | S11.1 — the highest-value renderer, needs no plugin and no adoption |
| **MCP server** (phase 2, optional) | k8s Deployment or Cloud Run in the customer's project | Live queries only; per-user OAuth per S20/§4.4; `managedMcpServers` or repo `.mcp.json` |

**Operational cost estimates** (mine, not vendor-quoted): CI-artifact path **~$0**; small k8s Deployment on an existing cluster **~$10–25/month**; Cloud Run at low QPS **~$0–15/month** with a 1–3 s cold start on `min-instances: 0`. Security review will object to: who can merge to the context repo (answer: CODEOWNERS, and the bot cannot self-merge); public ingress on Cloud Run (answer: internal-only ingress); the blast radius of the extractor's read credentials; and, for a dev cluster, whether it is in scope for anything production-adjacent. **Dev clusters get rebuilt — context that lives only there dies with them**, which is a second reason for S19.

---

## 6. The two experiments

Both must be pre-registered before any card is written, or they measure the cards' authors instead of the cards.

### 6.1 The 20-question test — does the context produce *correct* answers?

**Harvest frame — never ask people for their questions.** Self-reported questions are the ones people knew to ask and remember asking. Harvest from artifacts that record confusion after the fact:

- (a) 90 days of the three busiest engineering Slack channels: messages containing `?` plus an interrogative stem (who / where / which / does / is there / how do I) **that received at least one reply from someone outside the asker's team.** The cross-team-reply filter is the whole trick: it isolates context the asker did not have and could not get locally.
- (b) Every review comment on a cross-repo PR containing a question mark.
- (c) The last 30 incident timelines — every "we didn't know that X" line.

**Stratify** proportionally to each service's commit count, or one noisy service dominates the sample. Harvest n≈60; randomly sample 20; **pre-register those 20**.

This frame still misses questions nobody asked because they gave up and read the code. Proxy for those with §6.2's instrumentation: pre-first-edit exploration calls *are* the unasked-question volume.

**Blind grading by the owner.** Each question is answered twice — once by a session with the artifact, once without — presented to the grader in random order with the condition stripped. The grader is the **CODEOWNERS owner for the service the question concerns**: not the product team (invested), not the original asker (already knows).

| Score | Meaning |
|---|---|
| **2** | Correct and actionable with no further lookup |
| **1** | Right direction; needed one follow-up lookup |
| **0** | Wrong, or hedged into uselessness |
| **−1** | A confident, specific falsehood: dead endpoint, wrong topic, removed dependency |

**The −1 band is the point of the exercise.** Confidently wrong context is worse than absent context, because the developer trusts it — it arrived as authoritative. Averaging it into 0 hides the one failure mode that kills this product.

**Pass bar, fixed in advance: artifact-condition mean ≥ 1.4 AND zero −1 scores.** Any −1 at n=20 on freshly written cards means drift is already real, and S9 moves ahead of everything else.

Cost: about two days for one person with Slack and GitHub read access.

### 6.2 The exploration-token measurement — does the context *save* anything?

**This is the open hole in the evidence.** The injection-cost numbers in §4.1 are measured to ±2 tokens. The baseline they are compared against — the agent exploring repos itself — is currently an **estimate of 15,000–60,000 tokens per task, and it carries the entire ROI case.** It was not measured because it cannot be measured on synthetic fixtures: 35 generated repos have no realistic grep surface, so any number would be an artifact of the generator. It must be run on the real estate.

- **Primary metric: "exploration tokens", not total tokens.** Defined as the sum of tool-result tokens from `Read`, `Grep`, `Glob` and `Bash(cat|rg|find)` calls occurring **before the session's first `Edit` or `Write`**. That boundary is the operational definition of burning tokens on exploration. Do **not** use request totals — they include cache reads and grow with turn count regardless of exploration.
- **Collection.** Either (a) observational: OTel export for two weeks with `OTEL_LOG_TOOL_DETAILS=1` so tool names are present; or (b) controlled: `claude -p --output-format json` over a fixed task list, reading `usage` and the `iterations` array — the same method as §4.1, so the customer can reproduce it.
- **Design.** 20 pre-registered cross-repo tasks from the §6.1 harvest. **Within-task crossover**: each task run in both conditions (artifact present / absent), same model, same effort, order randomised, `/clear` between runs, **3 repetitions per cell** because agent trajectories are high-variance. Report **median and IQR, never a mean** — one runaway exploration spiral would otherwise write the headline.
- **Report three outcomes, not one:** median exploration tokens, wall-clock to first edit, and task success graded by §6.1's rubric. A token win with a success regression is not a win.
- **Stratify on clone state.** A repo already cloned on the developer's disk is cheap to grep; the estate's value is in the 34 repos that are *not* cloned. Record clone state per task and stratify, or the result measures local disk rather than the product.
- **Pre-registered falsifier:** if median exploration tokens fall by **less than ~5,000**, the headline ROI claim is dead. The ~550-token design is still correct, but the pitch must change from "saves tokens" to "gets the right answer" — and then it rests entirely on §6.1, including the zero-−1 condition. **Accept this in advance.**

---

## 7. Build order

### Weeks 1–2 — a team can install it and get value

1. **Context repo** (customer-owned): `services/*.md` (generated), `overrides/*.md` (hand-written, never touched by the generator), `adr/`, `domains/`. CODEOWNERS per service. Front-matter: `owner`, `repo`, `slack`, `runbook`, `fetched_at`, `image_digest`, `reviewed_by`, `reviewed_at`, `ttl: 90d`.
2. **Two extractors only** — K8s Deployment env-var URLs, and **Module Federation `remotes`**. These two are the highest-confidence sources (S1) and the Module Federation one is unoccupied prior art (§4.10). Emit `Service`, `Contract`, `Edge` with evidence and confidence into the store. Report coverage (S2).
3. **Build tool** (`ctxkit build` or equivalent) emitting, from the store:
   - `AGENTS.md` — **≤40 lines**: estate map, naming conventions, and "for service detail, use the `service-context` skill."
   - A Claude Code plugin: **one** skill `service-context` whose body is a one-line-per-service index plus instructions to read `references/<service>.md`; `references/*.md` (35 files, on demand); `hooks/hooks.json` with a `SessionStart` hook printing ≤40 lines — current repo, branch, the 1–3 services in scope derived from the git remote and changed paths.
   - `.claude-plugin/marketplace.json` + `plugin.json`; version = content hash.
   - The six-line `.claude/settings.json` snippet, committed into each of the 35 repos from one template (S17 rail 1).
4. **Install documentation** for all four rails of S17, plus the `archive`+`sha256` and `CLAUDE_CODE_PLUGIN_SEED_DIR` variants.
5. **`ctxkit measure`** — ship §4.1's method as a command so the customer can prove the token numbers on their own estate and so §6.2 is runnable without us.

**Definition of done:** a developer in one service repo starts Claude Code, the session is **~550 tokens heavier than baseline** (verified with `/context` and `claude -p --output-format json`), and "who owns the thing downstream of us that consumes our events, and what breaks if I change the status enum?" is answered from `references/` with no clone and no Confluence tab.

**Staffed in parallel from week 1, not deferred:** the identity-resolution work of S22 — the alias table, and the empirical loop of extract → human rejects wrong edges → fix → repeat. This is the product (S22); the artifact above is its delivery format.

### Explicitly NOT in scope early

- **No MCP server.** Nearly free at startup (+129 tokens measured) but it adds an always-on component, an auth story and a security review for capability the artifact already covers. Phase 2.
- **No OAuth, no token exchange, no IdP integration.** Week 1 content is strictly org-readable, which is why it can be a static artifact at all (S21). Confluence per-user access goes to Rovo MCP (S20).
- **No Confluence or Notion ingestion.** It is a connector; connectors are commodity and Atlassian ships the good one. Revisit only for Data Center (§8).
- **No Helm chart, no Cloud Run deployment.** S19.
- **No Kafka, Ingress, SDK-import or access-log extractors.** Each is a separate confidence tier; land two sources well before adding four badly.
- **No ticket-to-brief, no prior art, no conflict detection.** These need `Change` and `TicketLink` populated, which needs history. Phase 3.
- **No gateway requirement.** §4.7.
- **No agent-abstraction layer.** S18.

### Phase 2 (after §6.1 and §6.2 have run)

Kafka consumer-group edges; ingress and SDK-import extractors; the PR-comment renderer in CI (S11.1 — arguably should be pulled forward if the §6.2 numbers disappoint, because it needs no adoption); the drift engine (S8 + S9) with the two-lane PR proposer (S6); then the MCP server with pre-registered-client OAuth for live queries.

### Phase 3

Deprecation sweep; incident reverse-traversal; prior art and conflict detection (S13); the `observed` tier from ingress-nginx logs; per-audience plugin splitting **for relevance only** (S21).

---

## 8. Open questions and known unknowns

**Open design questions (not settled):**

1. **Store format when the MCP server arrives.** SQLite-as-artifact is right for the artifact path; the server wants Postgres. Does the server read a rebuilt SQLite snapshot, or does the store become Postgres with the SQLite file as an export? Unresolved.
2. **Alias resolution strategy.** The alias table is the compounding asset (S22) but there is no decision on whether aliases are hand-curated, learned from co-occurrence, or both — and no decision on what happens when two services claim the same alias.
3. **Coverage threshold for publishing.** S2 says report coverage; it does not say whether a card for an `unmapped_no_declared_deps` service should be published at all, or suppressed as worse than nothing.
4. **Legacy repos (5–7) with no manifests.** No extraction path exists for them today. Candidates: hand-written override cards only; access-log-derived `observed` edges; or explicit exclusion with that exclusion surfaced in every coverage line.
5. **How the `SessionStart` hook derives "services in scope."** Git remote → service is easy; changed-paths → services in a polyrepo is not obviously well-defined. Unresolved.
6. **Whether micro-frontends and services share one `Service` table** or need separate kinds with different contract shapes. Currently modelled as one table with a `kind` discriminator; untested.
7. **Monetisation / packaging.** S22 concludes the week-1 artifact is an internal tool, not a product. There is no decision on whether this ships as a product at all, as an internal platform capability, or as open source with the identity-resolution work as the paid layer. §10 works the question through to a recommended first motion (migration campaigns) and a wedge (micro-frontend sprawl), but that whole section is unvalidated reasoning, so this stays open.

**UNCONFIRMED capability claims — do not rely on these without checking:**

| Claim | Status |
|---|---|
| Whether Atlassian Rovo MCP supports Confluence/Jira **Data Center** | Every secondary source says Cloud only (LiteLLM integration docs, dev.co). Atlassian's developer page has a Data Center tab whose contents were not read. **This matters: the customer who demands self-hosting is disproportionately likely to run Data Center**, in which case S20's default path is unavailable and the fallback is a community server (`sooperset/mcp-atlassian`, MIT, supports Server/DC with PATs) or our own curated ingestion |
| Whether per-policy gateway `managedMcpServers` **merges with or replaces** the `match: {}` base | The documented merge rules do not list the key |
| Whether gateway per-policy `enabledPlugins` behaves like `extraKnownMarketplaces` (non-merging) | The `cli` block is documented as "a complete Claude Code `managed-settings.json` document", and non-merging is documented for `extraKnownMarketplaces` specifically. Test on one client |
| Which `client_id` URL Claude Code presents for **CIMD**, and whether CIMD works against an authorisation server with no outbound internet | Not documented. **Do not build on CIMD** — use a pre-registered confidential client, which is documented and works offline |
| That Anthropic has **no** native enterprise code-indexing product | Absence of evidence across primary sources, not a positive confirmation |
| Cursor's current admin capabilities (Organizations, Team Rules, team MCP distribution) | Third-party reporting only; not verified against cursor.com/docs |
| Operational cost figures in §5.3 | My estimates, not vendor-quoted |
| Absolute token totals in §4.1 | Measured on one machine with a large pre-existing plugin set. The **bare floor (28,824)** and the **within-bracket deltas** are solid (±2); absolute totals will differ on a clean install |

---

## 9. Where the two researchers disagreed, and how it resolved

Two independent researchers on different models surveyed the same questions on 2026-10-30 and then cross-examined each other. Recorded here so a cold agent does not reopen settled ground.

| Point | Positions | Resolution |
|---|---|---|
| **Delivery rail** | Researcher B recommended managed-settings delivery and did not flag the provider bypass. Researcher A found that server-managed settings are bypassed under `CLAUDE_CODE_USE_VERTEX` / `CLAUDE_CODE_USE_BEDROCK` / non-default `ANTHROPIC_BASE_URL` | **A was right.** Verified verbatim (§4.6) and found to be worse than reported: unrecoverable by policy, and a silent per-developer partial failure. Rail changed to repo-level `.claude/settings.json` primary + `managed-settings.json` backstop (S17). The claude.ai console is unusable for this customer |
| **ACL-restricted content** | B: if any valuable context is restricted, you must be a server from day one. A: split content into per-audience plugins instead | **Split verdict.** A's *mechanism* is real and confirmed (per-group gateway policies can carry `extraKnownMarketplaces`), but A's *conclusion* fails: the tier is explicitly "a client-side control, not a security boundary", with no revocation and no audit, and it collapses at the first overlapping group membership. Settled as S21: splitting is for relevance, a server for authorisation |
| **Confluence** | B: server proxies Confluence as the developer via IdP token exchange. A: don't proxy at all — use each developer's Rovo MCP and mirror only curated summaries | **A was right; B conceded.** Token exchange removed from the roadmap (S20). B's one surviving caveat: Rovo MCP is Cloud-only (UNCONFIRMED for Data Center), so curated mirroring stays as the universal fallback — and is needed anyway as the substrate for TTL and drift |
| **Is it a product?** | Both concluded the moat is thin — B said "~400 lines of CI glue", A said "any platform team assembles it in a week" | **Agreed, and resolved into S22.** The week-1 artifact is an internal tool. The only compounding asset is identity resolution across a specific heterogeneous estate. The *ingestion* half of the roadmap is not defensible (it is a connector); the *drift* half is, but only because it requires the graph. Ordering inverted: the graph is the product, staffed from week 1 |
| **Skills per service** | Considered and rejected | Rejected on a measured platform constraint, not taste: the 1%-of-window skill listing budget evicts descriptions silently (S15, §4.3) |
| **Baseline for the ROI claim** | B measured injection cost precisely but estimated the comparison baseline at 15k–60k tokens | **Conceded as the open hole.** Specified as the §6.2 experiment with a pre-registered falsifier. Until it runs, the ROI claim is an estimate and should be presented as one |

---

## 10. Commercial position

**Status: reasoning, not research.** Nothing here is validated — no customer interview, no design
partner, no pricing test, no market sizing. §9 records that both researchers independently concluded
the moat is thin; this section is the attempt to find where it is not. Every claim is a hypothesis,
and §10.7 names how to kill each one.

### 10.1 Why the system in §7 cannot be sold

Developer tooling is priced against **free**, and free is the honest comparison: §4.1 measures the
injection layer at ~550 tokens always-on, and S17 delivers it in six committed lines of
`.claude/settings.json`. Both researchers put a competent platform team's build time at a sprint or
less, and Anthropic commoditised the distribution rails on 2026-06-29 (§4.7). There is no price for
markdown compilation.

Compliance and reliability tooling is priced against **the cost of the audit finding or the
incident**. Same software, different buyer, two orders of magnitude of price. So the commercial
question is not "which feature do we add" but **"which existing budget line does this attach to"**.

### 10.2 Candidate features, ranked by whether a budget already exists

| Feature | Buyer / budget | Why defensible | Cheap test |
|---|---|---|---|
| **Compliance artifact generation** — dated, provenance-stamped dependency and data-flow graph, regenerated continuously instead of hand-built annually | Risk / internal audit | DORA (in force for EU financial entities) and SOC 2 both want this. Compliance does not care that a platform team *could* build it; it cares that the artifact is continuous and acceptable to an auditor. Least price-sensitive line | Ask one internal auditor whether a generated graph would replace a manual annual exercise |
| **Migration / deprecation campaigns** — burndown to zero consumers of a contract, with a tracked end date | Platform program budget | "Who still consumes `v2`" blocks every migration, and firms already pay consultants to answer it by hand. Clear start, clear finish, trivially procurable | Find one in-flight migration; ask what answering that question costs today |
| **Architecture governance gate** — block a PR that introduces an edge violating a rule (no MFE calling another MFE's backend; no service reading another's database) | Chief architect / platform | Enterprises want this and cannot do it today because nobody holds the edge graph. Requires the asset in S22; cannot be faked | Ask whether *any* architectural rule is currently enforced in CI |
| **Change-failure prevention** — the §7 PR gate, measured against DORA change failure rate | Reliability / SRE | Attaches to a metric executives already track. Needs ~a quarter of attribution data before the ROI is arguable | Count cross-service incidents in the last two quarters whose cause was a contract change |
| **Agent correctness eval, per estate** | The AI budget | Every enterprise rolling out coding agents in 2026 must show whether it works, and no one has a per-estate number. §6.1 is already the harness | Ask who currently owns the number that proves the AI rollout works |
| **Cross-estate benchmarking** — "your fan-out is 90th percentile; coupling worsened 12% this quarter" | CTO | Compounds with every customer and is structurally unavailable to an internal team. The real long-term moat | None until N > 5 customers. **Not a starting feature** |

### 10.3 The wedge: micro-frontend sprawl

Of everything above, this is the one to lead with, because it is narrow, painful, unserved, and the
extraction for it is exact rather than heuristic.

Thirty-plus Module Federation micro-frontends means **shared-dependency version skew**. Two MFEs
ship incompatible versions of a shared library, the host shell breaks at runtime, and nothing in CI
catches it because each repository compiled in isolation. Add bundle-size regressions that no single
team owns and a shell whose load time degrades every sprint.

Two facts make this the wedge rather than a feature:

- **It is revenue-adjacent.** In retail or fintech a broken or slow shell is lost conversion, which
  is a fundable category rather than a developer convenience.
- **Nothing serves it.** The 2026-10-30 survey (§4.7) found no tool that aggregates Module
  Federation `remotes` across repositories; the MF ecosystem tools are per-app or runtime-only. And
  per S1, `remotes` are **declared exactly** — no inference, no heuristics, no tracing. This is the
  only place in the estate where extraction is both exact and uncontested.

### 10.4 What not to build

| | Why |
|---|---|
| The injection layer | ~550 tokens, six lines of settings, free as of Claude Code v2.1.306 (§4.1) |
| A Confluence connector | S20. Atlassian ships the good one; connectors are commodity |
| A bare `ttl:` field marketed as drift detection | One sprint of work and instantly copied. Per §9, drift is defensible *only* because it needs the graph to contradict the prose |
| A dashboard | Nobody buys a second place to look at their services. Answers belong in the PR, the session and the ticket |

### 10.5 Pricing shape

Deployment is in-network (§1), so: **per-estate or per-service annual contract, not per seat.** Seat
pricing invites comparison with Copilot and Cursor at roughly $20/developer/month — a comparison
this product cannot win and is not competing in. Compliance and migration line items are not
benchmarked against developer-tool seat prices.

### 10.6 Sequencing, and the dependency that gates all of it

This section does **not** change the build order in §7. S22 stands: the graph is the product and is
staffed from week 1. What changes is the buyer, and therefore which feature follows the graph.

**First commercial motion: migration campaigns.** That buyer has the shortest sales cycle; the
question they are paying for — "who still consumes this contract" — is precisely the query that
forces extraction correctness; and delivering one migration puts you inside the estate for
everything else in §10.2.

**The gate:** every feature here is downstream of the graph being correct on one real heterogeneous
estate — the alias problem (§8, open question 2) and the manifest-less legacy repositories (§8, open
question 4). Sell nothing before §6.1 passes with **zero −1 scores**. A confidently wrong dependency
claim sold as a compliance artifact is a materially worse outcome than having no product.

### 10.7 How to falsify this section

| If | Then |
|---|---|
| §6.2 shows median exploration tokens falling by **< 5,000** | The token pitch is dead — already pre-registered. Everything here then rests on accuracy and governance alone |
| An internal auditor says a generated dependency graph is **not acceptable evidence** | §10.2 row 1 dies, and with it the highest-margin line |
| The target firm has **no in-flight migration** | §10.2 row 2 has no entry point and the sequencing in §10.6 is wrong |
| Kubedian, Repowise or Qodo ship **Module Federation remote aggregation** | §10.3 closes. Re-run the §4.7 survey quarterly; this is the single most perishable claim in the document |

---

## 11. The one-line version

Extract typed `Contract` and `Edge` records with evidence and confidence from K8s env vars and Module Federation remotes, render them as a PR-reviewed markdown card repo, compile that into an `AGENTS.md` plus a single-skill Claude Code plugin delivered by six lines of committed `.claude/settings.json`, keep nothing in the request path, report coverage honestly, and treat the identity-resolution tail — not the injection mechanism — as the product.
