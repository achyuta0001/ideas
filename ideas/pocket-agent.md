# Pocket Agent — Drive-Resident CLI Agent (macOS primary, Windows PC as remote node)

Plug in the SanDisk SSD, run `pocket`, get an agent that carries its own models, memory and
vault. Online it escalates to Claude and to the Windows box. Offline it still works, degraded
but useful. Nothing depends on the host Mac having anything installed.

## Verified facts (probed 2026-08-22, not assumed)

```text
  Mac        Apple M5, 10-core GPU, Metal 4, 16 GB unified, arm64, macOS 26.6.2
             go 1.26.5 · claude 2.1.239 · git 2.50.1 · python 3.14.6 · node 24.13.0
             OpenSSH_10.3p1 · tailscale 1.102.2 (CLI, no .app)
             246 GB free on internal disk

  Tailnet    achyutas-macbook-pro  macOS    ONLINE
             achyutasmsi           windows  OFFLINE, last seen 3d ago

  Drive      SanDisk portable SSD 256 GB — NOT PLUGGED IN AT PROBE TIME.
             Filesystem, real throughput and free space unverified.

  Not present: rustc, ollama, llama-server, Tailscale.app
```

## Premise corrections

**1. The Windows PC is not a useful inference accelerator.**
Token generation is memory-bandwidth bound. M5 runs ~153 GB/s with roughly 10 GB usable for
weights after macOS overhead. A GF75 Thin carries a GTX 1650 (4 GB) / 1660 Ti (6 GB) /
RTX 3060 (6 GB) at 128-336 GB/s. It only wins for models that fit under 6 GB, and loses
outright above that because it spills to system RAM over PCIe. The M5 also has per-core
neural accelerators that help prompt processing.

Build the remote node for what it is actually good at: an always-on machine that keeps
running while the laptop is closed or travelling, holds bulk storage, and executes
Windows-only work. Not for GPU offload.

**2. Do not write an agent framework.**
Claude Code is already installed and is better than anything hand-rolled in Go. Online is a
solved problem. The genuinely unsolved problem is the offline loop. So `pocket` is a
portable **context + routing + state** layer, with a minimal offline agent loop as the only
novel component.

**3. The offline tier has a real ceiling. Do not oversell it.**
A 7-14B Q4 model on the M5 is good at summarising, extraction, classification, rewriting,
answering over your own vault, and short code edits. It is weak at long multi-step tool
loops — the failure mode is malformed or hallucinated tool calls. Mitigation is in the
design (GBNF-constrained decoding, step caps), not in wishful thinking.

**4. Tailscale being already configured removes an entire workstream.**
No VPN setup, no router port-forwarding, no dynamic DNS, encrypted by default, works from
any café wifi. The PC being offline for 3 days also proves the remote tier must be strictly
opportunistic and never on a blocking path.

## Architecture

```text
════════════════════════════════════════════════════════════════════════════
  POCKET AGENT — mac host, drive-resident, opportunistic remote node
════════════════════════════════════════════════════════════════════════════

  SANDISK SSD  /Volumes/POCKET  (APFS, encrypted)
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│  bin/pocket            single Go binary, arm64, ad-hoc signed              │
│  runtime/llama/        llama-server + Metal backend, built for arm64       │
│  models/               GGUF weights, ~25 GB budget                         │
│  data/vault.db         SQLite — sessions, tool calls, embeddings           │
│  data/notes/           plain markdown, git repo, syncs when online          │
│  secrets/vault.age     API keys + SSH key, passphrase-derived encryption   │
│  state/                host fingerprints, last-known tailnet status         │
│                                                                            │
└───────────────────────────────┬──────────────────────────────────────────┘
                                 │  paths resolved from os.Executable()
                                 │  NEVER from a hardcoded /Volumes path
                                 ▼
                  ┌──────────────────────────────┐
                  │   LAUNCH + PROBE              │  local only, no network,
                  │   drive root · free space     │  hard 300 ms budget
                  │   tailnet peer state          │
                  └──────────────┬───────────────┘
                                 ▼
                  ┌──────────────────────────────┐
                  │        MODEL ROUTER           │
                  └──┬──────────┬──────────┬─────┘
                     ▼          ▼          ▼
       ┌───────────────┐ ┌────────────┐ ┌──────────────────┐
       │ T0 DETERMINISTIC│ │ T1 LOCAL   │ │ T2 CLAUDE        │
       │ no model at all │ │ llama.cpp  │ │ shells out to    │
       │                 │ │ + Metal    │ │ `claude` CLI, or │
       │ file ops, grep, │ │ on drive   │ │ API for headless │
       │ git, vault FTS  │ │            │ │                  │
       │ ALWAYS works    │ │ ALWAYS     │ │ needs internet   │
       │ $0              │ │ works, $0  │ │ metered          │
       └───────────────┘ └────────────┘ └──────────────────┘
                     │          │          │
                     └──────────┼──────────┘
                                ▼
                  ┌──────────────────────────────┐
                  │       AGENT LOOP              │
                  │  step cap · token budget      │
                  │  GBNF-constrained tool JSON   │
                  │  checkpoint every step        │
                  └──────────────┬───────────────┘
                                 ▼
      ┌──────────────────────────┴──────────────────────────┐
      ▼                                                       ▼
┌──────────────────┐                            ┌──────────────────────────┐
│ LOCAL TOOLS       │                            │ REMOTE NODE (optional)    │
│ fs · shell · git  │                            │ achyutasmsi over tailnet  │
│ vault search      │                            │ ssh · pc.run · pc.get     │
│ web (when online) │                            │                            │
│                   │                            │ health read from local     │
│                   │                            │ tailscaled — no round trip │
│                   │                            │ OFFLINE = tools hidden,    │
│                   │                            │ never a blocking wait      │
└──────────────────┘                            └──────────────────────────┘
                                 │
                                 ▼
                       ┌──────────────────┐
                       │   CLI (only UI)   │
                       │  pocket           │  REPL
                       │  pocket run       │  one-shot
                       │  pocket ask       │  vault Q&A
                       │  pocket note      │  capture
                       │  pocket pc        │  remote box
                       │  pocket doctor    │  what tier am I on
                       │  pocket sync      │  git push notes
                       └──────────────────┘
```

## The router, concretely

Escalation is explicit and logged, so cost and capability are always visible.

```text
  T0  deterministic     no model. file ops, grep, git, SQLite FTS5 lookups.
                        always available, instant, free.

  T1  local GGUF        llama-server with Metal, loaded from drive.
                        default brain. always available.
                        summarise · extract · classify · rewrite · vault Q&A

  T2  Claude            only when internet is up AND the task is tagged hard
                        or T1 self-reports low confidence.
                        shells out to the installed `claude` CLI with the
                        drive as workspace; API path for headless/scripted use.
```

There is no tier for "send it to the Windows PC to think". That is deliberate — see
premise correction 1.

### Local model shortlist (16 GB unified, ~10 GB practical ceiling)

```text
  default    Qwen3-8B-Instruct Q4_K_M        ~5 GB   good tool-calling
  stronger   Gemma-3-12B-it Q4_K_M           ~7 GB   better reasoning, slower
  code       Qwen2.5-Coder-7B Q4_K_M         ~4.7 GB
  embed      nomic-embed-text-v1.5 Q8        ~140 MB vault search
  ─────────────────────────────────────────────────────────────
  ~17 GB of the 25 GB model budget. Room for one more.
```

Tool-calling reliability is the binding constraint on small models, not raw quality. Pick on
function-calling benchmarks, not on chat leaderboards.

### Making a small model survive an agent loop

The reason local agents usually fail is malformed tool calls. Three mitigations, all cheap:

- **GBNF grammar** in llama.cpp forces syntactically valid tool-call JSON at decode time.
  The model cannot emit a broken call; it can only emit a wrong one.
- **Hard step cap and token budget** per task. Fail loudly at the cap, never spin.
- **Checkpoint every step** to SQLite, so a bad run is resumable and inspectable rather
  than lost.

## Remote node

`achyutasmsi` is already on the tailnet, reachable from any network.

**Transport: SSH over Tailscale.** Zero new protocol, zero new daemon to write. Requires
enabling OpenSSH Server on Windows once (Optional Feature, needs admin that one time).
Tailscale SSH can carry the auth so no key material has to live on the drive at all — worth
preferring over a drive-stored private key, since a lost drive then leaks nothing.

**Health check reads local tailscaled, not the network.** `tailscale status --json` already
knows whether the peer is online. No round trip, no timeout, no hang. If the peer is down,
the `pc.*` tools are simply not registered for that session and the model never sees them —
which is far more reliable than letting it try and fail.

**Tools:** `pc.run <cmd>`, `pc.get <path>`, `pc.put <path>`, `pc.status`.

**Use it for:** long jobs that outlive a closed laptop, bulk storage, Windows-only software,
scheduled work. Not for inference.

## Storage and drive discipline

**Filesystem: APFS, encrypted.** Windows portability is no longer a requirement — the PC is
reached over the tailnet, not by plugging the drive into it. That buys journaling and
native FileVault-grade encryption with no third-party driver. If direct Windows mounting is
ever wanted, this decision has to be revisited; exFAT is the fallback and costs journaling.

**SQLite on removable media.** A yanked drive during a write corrupts a database.
`journal_mode=WAL` is the wrong default here — use `TRUNCATE` with `synchronous=FULL`, batch
writes, and never fsync per token. Slower writes, but it survives an unplug.

**Path resolution.** Everything derives from `os.Executable()`. No `/Volumes/POCKET` string
anywhere, including inside SQLite rows — macOS silently mounts a second same-named volume as
`/Volumes/POCKET 1` and every stored absolute path breaks.

**Clean eject.** Handle SIGTERM/SIGINT, flush and close the database, release every file
handle. Otherwise the drive refuses to eject and macOS nags — the most common daily
annoyance in tools like this.

**Pure-Go SQLite** (`modernc.org/sqlite`), not cgo. Keeps builds trivial and reproducible.

**Launch shim.** A five-line script in `~/.local/bin/pocket` that locates the mounted drive
and execs the real binary. One-time host setup, keeps everything else drive-resident.
Self-built binaries carry no quarantine attribute; ad-hoc sign with `codesign -s -` anyway.

## Cost

```text
  drive .................. owned (SanDisk 256 GB)
  Go, llama.cpp, GGUF .... $0
  Tailscale .............. $0 (personal plan)
  local inference ........ $0 marginal
  remote node ............ $0 (hardware owned)
  Claude tier ............ existing subscription; API metered, router-gated
  ──────────────────────────────────────────────────────────────────
  TOTAL: $0 additional. Only variable cost is T2 escalation.
```

## Build order

Each step ends in something runnable. No step depends on a later one.

1. **Skeleton + probe + `pocket doctor`.** Go binary, drive-relative path resolution, tailnet
   peer status, clean-eject signal handling. Proves the portability rules before any AI
   exists. Ship this first — it is the part that quietly breaks everything if wrong.
2. **T1 local inference.** Bundle llama.cpp Metal build and one GGUF, supervise
   llama-server on loopback, streaming REPL. First moment it is useful offline.
3. **Storage.** SQLite with yank-safe pragmas, session persistence, markdown vault directory.
4. **Agent loop + local tools.** fs, shell, git. GBNF-constrained tool calls, step cap,
   per-step checkpointing. The novel component; expect the most iteration here.
5. **Secrets vault + T2 Claude escalation.** Passphrase-derived encryption, explicit
   escalation logging, network detection.
6. **Remote node.** Tailscale SSH, `pc.*` tools, peer-state gating so they vanish when the
   box is offline.
7. **Vault search.** Local embeddings + SQLite FTS5, `pocket ask`. This is where it converges
   with [hive-mind](hive-mind.md) — same compile/serve idea, personal scale.
8. **`pocket sync`.** Git push/pull of notes when online.

## Key design principle

**The drive is the source of truth; every host is disposable.** Nothing is written outside
the drive except one launch shim. Nothing on a blocking path may require network, the remote
node, or a Claude API key. Every tier above T1 is an optimisation that can vanish
mid-session without breaking the tool — because the Windows box was offline for three days
and the design has to assume that is normal.

## Open questions

- Which GF75 SKU (GTX 1650 4 GB / 1660 Ti 6 GB / RTX 3060 6 GB)? Only matters if the
  no-GPU-offload conclusion gets challenged.
- Drive was unplugged at probe time — real sequential throughput and current filesystem
  still unverified. Reformatting to APFS destroys existing contents.
- Does the Windows box need to reach *back* into the Mac, or is the relationship one-way?
- Is `pocket` meant to be usable on someone else's Mac, or only yours? Only yours means the
  launch shim and Gatekeeper story get simpler still.
