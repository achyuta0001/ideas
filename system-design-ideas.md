# System / Infra Design Ideas — Job Search Prep

Running list of system-design and infra-design project ideas for interview and portfolio conversations with banks, fintech companies, and FANG-tier companies. Companion to `hive-mind.md` and `intel.md`. Each idea is tagged **Buildable** (realistically shippable solo, good portfolio piece) or **Talking-point** (interview-caliber design discussion, not meant to be built).

## Fintech / banking-flavored, buildable

- **Ledger reconciliation engine** — double-entry ledger core with adapters for multiple upstream feeds (bank statement CSV/OFX, internal event log), a canonical "Discrepancy" record, and a rules engine that surfaces drift. Direct sibling to `intel.md`'s shape.
- **Idempotent payment gateway simulator** — small service demonstrating idempotency keys, the outbox pattern, and exactly-once effect under at-least-once delivery, with a duplicate-charge detection dashboard.
- **Compliance-aware config drift detector** — extension of `intel.md`'s config scanner: diff live config against a compliance baseline (PCI/SOC2-relevant keys) and flag risk level instead of just mismatch.
- **Event-sourced audit trail library** — append-only event log with snapshotting and replay, framed for regulated-industry audit requirements (immutability, tamper-evidence via hash chaining).

## Fintech / banking-flavored, talking-point only

- **Sharded payment ledger at scale** — the classic "design Stripe's ledger" interview problem: horizontal sharding, strong consistency for money movement, idempotency at the API boundary, cross-region replication, and async reconciliation jobs.
- **Real-time fraud detection pipeline** — streaming architecture (Kafka/Flink-style) blending a rule engine with ML scoring and a feature store, sub-100ms SLA, feature-freshness vs. cost trade-offs.
- **Distributed exchange matching engine** — low-latency order book, sequencer design, deterministic replay for auditability; centralized-sequencer vs. sharded-by-symbol trade-off discussion.
- **Multi-region DR for a core banking system** — RPO/RTO trade-offs, active-active vs. active-passive, consistency across regions under data-residency regulation.

## General infra, buildable

- **IaC/runtime drift detector** — sibling tool to `intel.md`: diff what's actually deployed (Terraform state / kubectl) against declared IaC, producing a structured drift report with an optional dry-run remediation plan.
- **SLO burn-rate alerting toolkit** — CLI/library that ingests Prometheus-style metrics, computes error budgets and multi-window burn-rate alerts (Google SRE workbook method), and emits alert rule configs.
- **Local chaos-testing harness** — proxy-based fault injection (latency, errors, timeouts) for local dev that verifies retry/circuit-breaker behavior and produces a resilience "report card."

## General infra, talking-point only

- **Global cache invalidation / CDN coherence design** — consistency models, TTL vs. event-driven invalidation, thundering-herd mitigation at edge scale.
- **Distributed job scheduler at scale** (Airflow/Temporal-style) — leader election, exactly-once task execution, backpressure and retry storms.

## Shared design signature

`hive-mind.md` and `intel.md` both center on a canonical data model as a strict boundary (`Finding`, the canonical document), pluggable source adapters above that boundary, and cost-consciousness by default — free/local/deterministic first, with LLM or paid services strictly opt-in behind an interface. The buildable ideas above intentionally reuse this shape; it's a consistent, well-rehearsed story to tell across interviews.
