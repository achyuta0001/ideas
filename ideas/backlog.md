# Backlog

Ideas brainstormed but not yet written up. When one gets picked, give it its own note from
[`_template.md`](_template.md), add it to the README index, and remove it from here.

## Brainstorm 2026-10-11

Two directions were picked: rebuilding systems to understand them, and tools for daily use.
recall came out of the same session and gets its own note.

### Rebuild-to-understand

- **quorum** — Raft on top of [ashlar](https://github.com/achyuta0001/ashlar): a 3-node replicated
  log with leader election, log matching and snapshots. The real lesson is a deterministic
  simulator: one thread, a seeded RNG, and a fake clock, network and disk, so any partition or
  crash replays exactly from its seed. Teaches what "committed" means and why Kafka's KRaft
  replaced ZooKeeper. Large.
- **rebalance** — Kafka's consumer-group protocol over ashlar: group coordinator, heartbeats,
  generation IDs, eager vs cooperative-sticky assignment, offset commits. Teaches why rebalances
  stall consumers and what `max.poll.interval.ms` really guards against. Medium.
- **reconcile** — a miniature Kubernetes control plane: a watchable state store with
  `resourceVersion`, an informer cache with a rate-limited work queue, a scheduler that filters
  and scores nodes, and a "kubelet" that runs plain processes on the Mac and the GF75 over
  Tailscale. Teaches level-triggered reconciliation and optimistic concurrency. Medium to large.
- **springlet** — Spring's core in 1-2k lines of Java: component scan, a bean graph with cycle
  detection, constructor injection, scopes, and `@Transactional` through a JDK dynamic proxy.
  Teaches why `this.method()` skips `@Transactional` and what CGLIB proxies change. Small to
  medium.
- **anomaly-lab** — a tiny MVCC store that reproduces dirty reads, lost updates and write skew on
  demand, then fixes them with serializable snapshot isolation. Teaches the
  double-spend-through-write-skew bug class and what REPEATABLE READ vs SERIALIZABLE actually
  buys. Medium.

### Daily-use tools

- **fleet** — one command across every repo under `~/Developer`: unpushed or behind, uncommitted
  changes, files untracked for weeks, duplicate clones of the same remote that drifted apart,
  secret-looking files. A morning report, a shell-prompt badge, and an optional Claude Code hook
  that fetches before branching. Both problems it catches happened in this repo on 2026-10-11.
  Small.
- **glow** — the Syska bulb as a Claude Code status light driven by hooks: working, waiting for
  you, done, failed. Gives [syska-bulb-cli](syska-bulb-cli.md) a reason to finish. Small once the
  bulb is controllable.
- **wake** — summon the GF75: Wake-on-LAN from the Mac, wait for it on the tailnet, run the job,
  sleep it after idle. Needs Ethernet (WoL over Wi-Fi rarely works), WoL enabled in the BIOS, and
  Windows Fast Startup off. Small.

### Pairings and order

- reconcile + wake: the scheduler places real jobs on the box that wake brings up.
- quorum + rebalance: both build on ashlar.
- Suggested order: fleet first (smallest, real pain), rebalance as the main learning project,
  quorum after it.
