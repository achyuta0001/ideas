# Intel

A unified project intelligence CLI. Scans dependencies, config, and logs — surfaces findings in one prioritized report.

No cloud required. No LLM required. Optional LLM synthesis when you want it.

---

## The Problem

Modern projects have three blind spots:

1. **Dependencies** — you added a package six months ago and haven't checked it since. CVEs accumulate silently.
2. **Config** — prod drifted from what's in version control. No one noticed until something broke.
3. **Logs** — errors are happening but buried in noise. You only look when something pages you.

Three separate tools exist for each. None of them talk to each other.

---

## What Intel Does

```
intel scan          # runs all three scanners
intel deps          # dependency risk only
intel config        # config drift only
intel logs          # log patterns only
intel report        # last scan results
```

Single prioritized output:

```
CRITICAL  [deps]    lodash 4.17.20 — 3 CVEs (2 high, 1 medium)
WARNING   [config]  DB_POOL_SIZE drifted in prod (expected: 10, observed: 3)
WARNING   [deps]    react-scripts license is UNLICENSED
INFO      [logs]    error spike in auth-service — 47 errors between 03:00–03:15 UTC
INFO      [config]  CACHE_TTL missing in staging, present in prod
```

---

## Architecture

```
Source Adapters → Canonical Layer → Rules Engine → Finding → Report Layer
                                                                   ↑
                                                        Synthesizer interface
                                                        ├── TemplateReporter  (free, default)
                                                        └── LLMReporter       (opt-in, paid)
```

### Canonical Finding

Every scanner emits the same structure:

```python
@dataclass
class Finding:
    severity: Literal["critical", "warning", "info"]
    scanner:  Literal["deps", "config", "logs"]
    title:    str
    detail:   dict        # scanner-specific structured data
    context:  str | None  # raw excerpt for LLM context
```

The `detail` field is structured so `TemplateReporter` can render it without LLM. `LLMReporter` uses `context` to narrate.

### Source Adapters

Each adapter implements:

```python
class Adapter(Protocol):
    def collect(self) -> list[RawRecord]: ...
```

**Deps adapters:** `npm` (package-lock.json), `pip` (requirements.txt / poetry.lock), `cargo` (Cargo.lock), `go` (go.sum)

**Config adapters:** `.env` files, TOML/YAML config, terraform `.tfvars`, dotfiles

**Log adapters:** plaintext log files, JSON structured logs, nginx access logs

### Rules Engine

Rules are pure functions: `RawRecord → Finding | None`. No LLM cost, no network calls.

**Deps rules:**
- CVE lookup against OSV.dev (free, no auth)
- License compatibility check against allowlist
- Staleness flag (no publish in N days)
- Maintainer count below threshold

**Config rules:**
- Key present in expected, missing in observed → `WARNING`
- Value mismatch → `WARNING` (redacts secrets before logging)
- Type mismatch (string vs int) → `WARNING`
- Unexpected key in observed → `INFO`

**Log rules:**
- Error rate spike (rolling window, configurable threshold)
- Recurring error clustering (edit-distance grouping, no embeddings needed at small scale)
- Slow request detection (parse duration fields)
- Pattern matching against known bad signatures

### Synthesizer Interface

```python
class Synthesizer(Protocol):
    def summarize(self, findings: list[Finding]) -> str: ...

class TemplateReporter:
    """Default. Renders structured markdown from finding fields."""

class LLMReporter:
    """Opt-in. Sends structured findings as prompt context to your LLM of choice."""
    def __init__(self, client, model: str): ...
```

The LLM never receives raw logs or raw config. It receives the already-processed `Finding` list. This keeps token counts low and cost predictable.

---

## The Cross-Scanner Advantage

When LLM synthesis is enabled, findings from all three scanners are sent together. This enables correlations no single tool surfaces:

> "Your DB_POOL_SIZE config drifted to 3 at the same time the auth-service error spike started at 03:00 UTC. The two are likely related."

Without LLM, you still see both findings — you just connect the dots yourself.

---

## Cost

| Component         | Free tier                        | Paid                          |
|-------------------|----------------------------------|-------------------------------|
| Dep scanning      | OSV.dev API (free, no auth)      | —                             |
| Config diffing    | local only                       | —                             |
| Log processing    | local only                       | —                             |
| CVE database      | OSV.dev                          | —                             |
| LLM synthesis     | not enabled                      | your API key, your model      |
| Hosting           | runs locally or in CI            | —                             |

**Total free tier: $0.** LLM synthesis cost scales with how often you run it and which model you choose. Haiku-class models on batch findings are cents per month.

---

## CI Integration

```yaml
# .github/workflows/intel.yml
- name: Intel scan
  run: intel scan --fail-on=critical
```

Exits non-zero on any `CRITICAL` finding. Integrates with any CI that reads exit codes.

---

## Configuration

```toml
# intel.toml

[deps]
ecosystems = ["npm", "pip"]
license_allowlist = ["MIT", "Apache-2.0", "BSD-3-Clause"]
staleness_days = 365

[config]
sources = [".env", "config/prod.toml"]
expected = "config/expected.toml"
redact_patterns = [".*SECRET.*", ".*KEY.*", ".*PASSWORD.*"]

[logs]
sources = ["logs/app.log"]
error_spike_threshold = 20   # errors per 15min window
cluster_min_size = 5

[reporter]
backend = "template"          # "template" (free) or "llm"
# model   = "claude-haiku-4-5-20251001"
# api_key_env = "ANTHROPIC_API_KEY"
```

---

## Build Sequence

1. Canonical `Finding` dataclass + `TemplateReporter`
2. Deps scanner — npm adapter + OSV.dev CVE lookup + license rules
3. Config scanner — `.env` adapter + diff rules
4. `intel scan` CLI + unified report output
5. Log scanner — plaintext adapter + spike detection + clustering
6. CI exit-code integration (`--fail-on`)
7. Additional adapters (pip, cargo, YAML config, JSON logs)
8. `LLMReporter` + cross-scanner synthesis prompt

---

## Core Design Principle

> Everything above the canonical `Finding` is scanner-specific and swappable. Everything below it — the report format, the Synthesizer interface, the CI integration — never changes when a new scanner or adapter is added.

Rules handle the deterministic 90%. LLM handles the last 10%. You pay for neither until you choose to.
