# Recall — local search over everything Claude and I have written down

One command finds what was said, decided or committed, across Claude Code transcripts, Claude's
memory files, the Obsidian vault and git commit messages: `recall "tuya local key"` returns
dated, ranked snippets with a way back to the source. It keeps its own archive, so nothing is lost
when Claude Code deletes old sessions. Local only: no network, no LLM, no daemon. A CLI comes
first; an MCP server so Claude can search mid-session wraps the same core later.

## Verified facts (probed 2026-10-11, not assumed)

```text
  Transcripts  ~/.claude/projects/<project>/<session>.jsonl
               314 files, 370 MB, 18 project folders; 193 files are
               subagent transcripts. Oldest surviving file: 2026-09-11.

  Retention    cleanupPeriodDays was unset, so the 30-day default applied.
               Everything before 2026-09-11 is gone, and there is no Time
               Machine backup. Set to 3650 on 2026-10-11.

  Format       JSON lines. user / assistant records carry message.content
               as a string or a list of parts: text, thinking, tool_use,
               tool_result. Also ai-title (session title), pr-link, and
               bookkeeping types. The field set changes between versions.
               Thinking is the largest part: one sampled session had 215k
               of ~370k content characters in thinking.

  Memory       41 files under ~/.claude/projects/*/memory/
  Obsidian     ~/Vaults/main: 11 notes, 264 KB, not a git repo
  Git          20 repos under ~/Developer, 1,691 commits
  Tooling      go 1.26.5 · sqlite3 3.54.0 CLI with FTS5

  Not present: any `recall` binary on PATH (the name is free)
```

## Premise corrections

**1. Claude Code transcripts are not a permanent record.**
Claude Code deletes transcripts older than `cleanupPeriodDays` (default 30), silently, in a
background sweep after a session starts
([docs](https://code.claude.com/docs/en/claude-directory)). The 2026-08-29 session that diagnosed
the [Syska bulb](syska-bulb-cli.md) is already gone; only its memory note survived. Raising the
setting stops further loss, but recall must still keep its own copy of every record it indexes.
It is an archive, not an index of pointers.

**2. The transcript format is not an API.**
It is undocumented and grows new fields between versions: this session's records carry fields
older ones lack. The parser reads a minimal field set and ignores everything else.

**3. A second copy of transcripts is a second copy of their secrets.**
Tool output holds pasted tokens and environment dumps. recall duplicates them, and the MCP phase
would feed hits back into new sessions. Secrets are redacted before anything is stored.

**4. No embeddings needed.**
The corpus is small: roughly 70 MB of text, extrapolated from one sampled session. SQLite FTS5
with BM25 ranking covers it. Semantic search can be a later layer if keyword search falls short.

## Architecture

```text
 sources (adapters)              canonical record            store (one SQLite file)       front ends
 ──────────────────              ────────────────            ───────────────────────       ──────────
 transcripts  ~/.claude/projects ─┐
 memory       */memory/*.md      ─┼─ Scan(cursor) ─▶ Record ─▶ records     archive   ─▶ recall "q"   v1
 obsidian     ~/Vaults/main      ─┤                           records_fts derived   ─▶ recall show  v1
 git          repos ~/Developer  ─┘                           cursors               ─▶ recall mcp   phase 2
```

- **`Record` is the boundary**, the same pattern as hive-mind's canonical document and intel's
  `Finding`. Fields: `id, source, kind, project, session, title, ts, text, ref, sidechain`.
  Kinds: `user`, `assistant`, `tool_call`, `tool_result`, `thinking`, `memory`, `note`,
  `commit`. Adapter code is per source; everything below the boundary is source-agnostic.
- **Indexing runs at the start of every query.** No background process. Each adapter keeps a
  cursor:

  ```text
    transcripts   byte offset + inode per file. Reads only new, complete lines;
                  a half-written last line waits for the next run, so live
                  sessions are safe. File replaced or shorter: re-read it
                  from the start. Records are keyed by session, message uuid
                  and part index, so a re-read only upserts; nothing already
                  archived is deleted.
    memory        content hash per file
    obsidian      content hash per file; notes split into chunks at headings
    git           commit hash; stored once
  ```

- **Two deletion rules.**
  - *Archive*: transcripts and git. When the source disappears, its records stay. This is the
    point of recall.
  - *Mirror*: memory and Obsidian. When a file is edited or deleted, its old records go. A
    deleted memory usually means it was wrong, and keeping it would resurface wrong facts.
- **`records` is the source of truth.** The full-text index is derived from it and can be rebuilt
  at any time without touching the sources.
- **The transcript parser reads only** `type`, `message.content`, `timestamp`, `cwd`, `sessionId`
  and `isSidechain`, plus `ai-title` and `pr-link` records.

## Storage and search

- One file: `~/Library/Application Support/recall/recall.db`, permissions 0600, directory 0700.
  WAL journaling is fine on the internal SSD.
- Tables:

  ```text
    records       id PK, source, kind, project, session, title, ts, text, ref, sidechain
    records_fts   FTS5 over records(title, text), external content
                  tokenizer: porter unicode61
    cursors       (source, key, value)
  ```

  `porter` matches word variants ("deleted" finds "deleting"). `snake_case` splits into words, so
  phrase queries match. `camelCase` stays one word; prefix queries (`cleanup*`) find it.
- Ranking: BM25 plus a small recency boost (half-life about 90 days).
- Default results leave out `thinking`, `tool_result` and subagent records, which are noisy.
  `--all` brings them back.
- Each hit shows date · source · project · session title, a highlighted snippet, and a way back
  to the source: `claude --resume <id>` while the transcript exists (`archived` once it is gone),
  a file path, or `git show <hash>`.

## Privacy

- Redaction runs at ingest, before storage. Patterns: GitHub tokens (`ghp_…`, `github_pat_…`),
  Anthropic (`sk-ant-…`), OpenAI (`sk-…`), AWS access keys (`AKIA…`), Slack (`xox…`),
  private-key blocks, JWTs, and `Bearer` headers. Each becomes `[REDACTED:kind]`.
- Redaction is irreversible by design: the archive never holds the secret.
- It is best-effort, so the 0600 database and having no network code at all are the backstop.
- The MCP phase leaves out thinking and tool output by default.

## Failure handling

- **Malformed line:** skip it and count it; `recall stats` shows the counts. One bad line never
  stops a file.
- **Unknown record type or field:** ignored on purpose.
- **Missing source folder:** skip that source. Never read a missing folder as "every file was
  deleted": under the mirror rule that would wipe the whole vault from the index.
- **Concurrent runs:** one writer, enforced with a file lock. A search that finds the lock held
  answers from the last committed state instead of waiting.
- **Schema change:** migrations via `PRAGMA user_version`. `recall reindex` rebuilds only the FTS
  index from `records` and never re-reads sources, so the archive is never at risk.

## CLI

```text
recall <query>            update the index, then search (default kinds, top 10)
  -n 20  --all  --source s  --project p  --kind k  --since 30d  --json
recall show <id> [-C 3]   the hit plus 3 turns either side from the same session
recall index              update the index only (for a hook at session end)
recall stats              counts by source and kind, skipped lines, db size, last update
recall reindex            rebuild the FTS index from records
```

No config file in v1. Defaults: `~/.claude`, `~/Vaults/main` (or `OBSIDIAN_VAULT_PATH`), repos up
to four levels deep under `~/Developer`. Flags and environment variables override them.

## Testing

- **Synthetic fixtures only.** They cover every record shape seen in real files, including an
  unknown type, a malformed line and a half-written last line, plus markdown files and a
  temporary git repo built inside the test. Real transcripts never enter the repo.
- **Behaviour tests:**
  - appending stores only the new lines
  - a replaced or shortened file is re-read once
  - deleting a transcript keeps its records; deleting a memory file removes them
  - a missing vault folder deletes nothing
  - sample secrets are redacted and look-alike non-secrets are untouched
  - indexing and searching at the same moment works
- **Real-data smoke test**, local only. Targets to measure, not promises: first full index under
  60 s, an update with no changes under 200 ms, a search under 100 ms.
- **Acceptance searches:** "AP mode power cycles" finds the bulb memory; "cleanupPeriodDays"
  finds the 2026-10-11 session; "tripwyre" finds commits.

## Cost

```text
  runtime ................. $0 (local binary, no network, no LLM)
  recall database ......... estimated 100-200 MB; measure on the first index
  transcripts kept longer . ~4-5 GB/yr at the current rate, now that
                            cleanupPeriodDays is 3650
  ──────────────────────────────────────────────────────────
  TOTAL: $0
```

## Build order

Each step ends in something that runs. Code lives in its own repo, `recall`, under
`~/Developer/Projects`.

1. **Database, transcripts adapter (message text only), redaction, `recall <query>`.** Useful on
   day one. Redaction is in step 1 so no secret ever reaches the database.
2. **Incremental indexing.** Cursors, replaced-file handling, the archive rule, the file lock,
   `recall index` and `recall stats`.
3. **Memory and Obsidian adapters.** The mirror rule and the missing-folder guard.
4. **Git adapter.**
5. **`recall show`, filters, `--json`, links back to sources,** and the optional session-end hook.
6. **Phase 2: `recall mcp`.** `search` and `show` tools over the same core, with thinking and
   tool output left out by default.

## Key design principle

**Keep what Claude Code deletes; drop what you delete.** Transcripts and commits are archived
because their sources vanish on someone else's schedule. Memory and notes are mirrored because
their sources change on yours. Everything else follows from the `Record` boundary: a new source is
a new adapter, and nothing below the boundary changes.

## Open questions

- **Repo visibility.** Public is safe, since the code never holds data. Your call.
- **Thinking in default results.** It is stored but hidden behind `--all`. Revisit after a few
  weeks of real use.
- **Substring search.** A trigram index would find fragments inside paths and `camelCase`, at
  roughly three times the index size. Add it only if prefix queries fall short.
- **MCP result limits.** How many hits and how much text per `search` call, so one query can't
  flood a session's context.
- **Subagent transcripts.** Indexed but hidden by default. Worth indexing at all?
- **Git depth.** Commit messages only in v1. Diffs later, if ever?

## Related

- [hive-mind](hive-mind.md) — the same "compile, then serve to agents" family at team scale, with
  LLM synthesis. recall is raw personal recall: no synthesis, no LLM.
- [pocket-agent](pocket-agent.md) — `pocket ask` (its build step 7) can reuse recall's index. Both
  chose pure-Go SQLite.
- [syska-bulb-cli](syska-bulb-cli.md) — the loss that prompted recall: its diagnosis session was
  deleted after 30 days; only the memory note survived.
- [backlog](backlog.md) — fleet also walks every repo under `~/Developer`; the two can share repo
  discovery.
