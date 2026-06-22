# Hivemind — architecture brief (one page)

**What it is:** a shared, searchable memory of *why the code is the way it is*. It
grows automatically — from git/PR/ticket history and from live corrections — and is
read back on demand by people and autonomous agents. Ships as a Claude Code plugin
(`hivemind`), data lives in a separate repo + AWS store. Status: **experimental**,
branch `SOL-1691_hivemind_plugin_v1`.

**Core principle — WHY, not WHAT.** Every lesson must fail this test:
*"could an engineer fluent at HEAD re-derive it from the code alone?"* If it passes
(it's re-derivable), it's a WHAT and is discarded. We keep only the irreducible WHY —
rejected alternatives, vendor quirks, reviewer objections, the bug that forced a shape.
The system is tuned to **under-emit**.

---

## Three subsystems

### 1. Active extraction — `/learn-from <folder | github-url | PR>`

```
origin ─▶ WALK ─────▶ CLASSIFY ─────▶ LEARN / BACKTRACK ─────▶ CONSOLIDATE ─▶ STORE
         (Python)     (Haiku)         (Sonnet→Opus)            (Sonnet→Opus)
         gh·git·acli  learn/skip/     tiered lesson +          cluster by
         BFS depth 2  backtrack       counterfactual           shared evidence
```

- **Walk** — deterministic BFS over commits → PRs → tickets → files. No LLM. Files are
  breadcrumbs (visited, not classified). Two bundles per content-bearing node: a
  *direct* bundle and a *lens* bundle (parent's content as context).
- **Classify** — Haiku votes `learn` / `skip` / `backtrack` per bundle. Obvious skips
  pre-filtered in Python (no token spend).
- **Learn** — emits one tiered lesson per `learn` (T1 atomic axiom → T2 design decision
  → T3 counterfactual). **Backtrack** — walks bug-fix → introducing commit (blame /
  pickaxe for local, GraphQL blame for GitHub), then the learner writes a *proven*
  counterfactual. Model escalates to Opus for counterfactuals and contested bundles.
- **Consolidate** — clusters near-duplicate lessons (shared source / lens / scope /
  provenance), merges or picks. Operates only within the run's staging dir (replay-safe).
- **Store** — POST `/v1/lessons` (SigV4) → S3 → Bedrock KB ingest → Aurora pgvector.
  Idempotent upsert. Run-event audit log uploaded to S3.

Architecture notes: agents are **facet-agnostic** — domain rules live in
`facets/<name>/` (today: `code`, `pull_request`); adding a facet needs no core edits.
The graph is **not** persisted as a file — it's a projection over each lesson's
`provenance` / `context_chain` / `backtrack_chain` frontmatter.

### 2. Retrieval — `hivemind:query`

A `hivemind-retriever` **subagent** (Haiku) does: keyword extraction → semantic KB
retrieve → self-check → optional 1-shot refine → package. Returns a compact JSON
envelope (`confidence`, `summary`, up to 5 `lessons[]` with `why_claim`,
`trigger_condition`, `source`, `score`). Runs as a subagent so the caller's context
only grows by the final answer. ~5–15s, ~$0.005/call, capped at 2 retrieve passes.
Embeds the YAML WHY-claim (not the body); ranks by `source_date` for freshness; MMR for
diversity.

### 3. The Forager — passive capture from live corrections

```
every prompt ─▶ REGEX pre-filter ─▶ (match) ─▶ HAIKU draft ─▶ POLLEN (local)
               UserPromptSubmit hook            background       │
               no LLM · <50ms · ~90% rejected                    ▼
                                              session Stop hook ─▶ OPUS review
                                                                   promote / demote / discard
```

- **Why:** most WHY is spoken, never committed. The moment a human corrects an agent is
  the cleanest WHY signal there is. The graph walk can't see it.
- **Tier 1 — regex** (`hooks/forager-prefilter.py`): correction-shaped patterns, in the
  prompt path, zero LLM, <50ms. Enqueues a window + dispatches a detached sweeper.
- **Tier 2 — Haiku pollen** (`scripts/forager_sweep.py`): loose, lossy draft of the
  correction. Local only (`~/.hivemind-forager/<session>/`).
- **Tier 3 — Opus promote** (`hooks/forager-session-end.py` → `forager_promote.py`):
  full-transcript review, debounced 10 min. Promote → hive; demote → "agent should
  explain better" meta-signal; discard → noise.
- **Cost:** ~$140/mo @ 50 devs. **Kill switch:** `HIVEMIND_FORAGER=0`. Never blocks the
  user — all hook errors caught, exit 0.

---

## Autonomous-agent integration (qbee + the self-verifying agent)

Both turn a Claude session into an agent that branches/codes/commits/opens a PR. Wire
in Hivemind at three points to give the fleet a **compounding shared memory**:

| Phase | Call | Purpose |
|---|---|---|
| **Before** coding | `hivemind:query` (folder as `working_in`) | Load known constraints / scars before writing. |
| **During / self-verify** | `hivemind:query` + forager | Check the change against institutional memory before declaring done; capture corrections passively. |
| **After** PR opened | `/learn-from <PR-url>` | Mine lessons from what was just shipped → feeds the next run. |

**Flywheel:** `query → work → learn-from → hive grows → query`. Every run starts smarter
than the last. Recommended first step: **query-on-entry** (cheapest, highest signal, no
new infra), then **learn-on-exit**, then enable the forager for supervised sessions.

---

## Config & ops

| Item | Value |
|---|---|
| Required env | `HIVEMIND_API_URL`, `HIVEMIND_RUNS_BUCKET`, `AWS_PROFILE` (e.g. `dev.qb-poweruser`) |
| Local home | `~/.hivemind` (audit logs, staging), `~/.hivemind-forager` (pollen) |
| Storage | S3 + Bedrock Knowledge Base + Aurora pgvector |
| Lesson tiers | T1 atomic axiom · T2 design decision · T3 counterfactual |
| Lesson kinds | `direct` · `lens-anchored` · `counterfactual` |
| Kill switch | `HIVEMIND_FORAGER=0` |
| Plugin repo | `QuickBase/claude-code-plugins` → `plugins/hivemind/` |
| Data repo | `QuickBase/hivemind` (data + audit logs only; no extraction logic) |
| Ticket | SOL-1691 (under SOL-1463) |

**In scope today:** `code` + `pull_request` facets, folder/URL/PR origins, retrieval,
forager, direct API publish. **Deferred:** other origins (Jira epic, library, user,
logs), more facets, structural retrieval filters, GitHub Pages graph viz.
