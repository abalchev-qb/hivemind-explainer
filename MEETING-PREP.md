# Hivemind — meeting walkthrough & talking points

A script you can talk from. Goal of the meeting: (1) explain what Hivemind is and
how it works, (2) land the pitch for wiring it into our autonomous agents (qbee +
the self-verifying agent in progress). Pair this with the
[interactive explainer](https://abalchev-qb.github.io/hivemind-explainer/) — drive
the slides while you talk.

---

## The one-sentence version (lead with this)

> **Hivemind is a shared, searchable memory of *why the code is the way it is* —
> grown automatically from history and from live corrections, and read back on
> demand by people and by agents.**

If they remember nothing else, they should remember **WHY, not WHAT**.

---

## The hook — the problem (explainer scene 1)

- Open any file at HEAD and you can read **what** it does. What you can't see is the
  **why**: the rejected alternative, the vendor quirk, the reviewer's objection, the
  bug that forced the shape.
- That WHY lived in a PR description, a Jira thread, a review comment, a hallway
  conversation — and then it evaporated. New engineer (or agent) shows up months
  later and re-introduces the exact bug we already paid for.
- **The test that defines the whole system:** *"Could an engineer fluent at HEAD
  re-derive this from the code alone?"* If yes, it's a WHAT — we throw it away. We
  only keep the irreducible WHY. This is what keeps the corpus an expert's lesson
  book instead of a changelog.

> Talking point: "We have great code search. We have zero *reasoning* search. Hivemind
> is reasoning search."

---

## Part 1 — Active extraction: `/learn-from` (explainer scenes 2–5)

Point it at an origin (a folder, a GitHub URL, or a PR) and it runs a four-phase
pipeline. Walk the slides:

1. **Walk** (scene 2) — a *deterministic* breadth-first crawl over the graph of
   related artifacts: commits → the PRs that merged them → the tickets they
   reference → the files they touched. Depth-capped at 2. **No LLM here** — just
   `gh`, `git`, `acli`, so it's cheap and repeatable. Files are breadcrumbs only;
   the signal lives on commits, PRs, and tickets.
2. **Classify** (scene 3) — a cheap **Haiku** classifier votes on each node:
   - `learn` → there's a hidden WHY, write a lesson
   - `skip` → re-derivable from HEAD, drop it
   - `backtrack` → it's a bug-fix/revert, so a past decision was wrong — go find it
   - Obvious skips (empty tickets, file breadcrumbs) are filtered in Python *before*
     spending a token. **Under-emission is the safe failure mode.**
3. **Learn & backtrack** (scene 4) — **Sonnet** (escalating to **Opus** for the hard
   ones) turns each `learn` into one **tiered lesson** (T1 atomic axiom → T2 design
   decision → T3 counterfactual). For each `backtrack`, a **backtracker** walks from
   the fix back to the *introducing* commit and hands the learner a **counterfactual**:
   *"at introduction, what should have been done to prevent this bug?"* Those are the
   highest-value lessons — the rule is **proven** by a real bug, not speculative.
4. **Consolidate & store** (scene 5) — many learners run in parallel, so a
   **consolidator** clusters near-duplicate lessons by shared evidence and
   merges/picks/keeps. Survivors are POSTed to the storage API → **S3** → embedded
   into a **Bedrock Knowledge Base** backed by **Aurora pgvector**. Idempotent:
   re-running upserts instead of piling on duplicates. Every run also uploads a full
   audit log of what it visited, classified, learned, and refused.

> Talking point: "The expensive models only run on the few nodes the cheap model
> flagged. Cost scales with *interesting history*, not repo size."

---

## Part 2 — Retrieval: `hivemind:query` (explainer scene 6)

- A developer or an agent asks a plain-English question + where they're working.
- A **Haiku retriever subagent** extracts keywords, does a semantic search over the
  KB, self-checks, optionally refines once, and returns **up to 5 lessons** as a
  compact JSON envelope (`confidence`, `summary`, `lessons[]`).
- Key ergonomic: the retriever runs as a **subagent**, so the caller's context only
  ever grows by the final answer — not the search work. ~5–15s, ~$0.005 a call.

> Talking point: "It's cheap and contained enough to call *speculatively* — before
> you start, and again as a sanity check before you finish."

---

## Part 3 — The Forager: autonomous capture (explainer scene 7) — *the new bit*

This is the part most people haven't seen, and it's the bridge to the agent pitch.

- **The insight:** the graph walk is *retrospective archaeology* — it only finds WHYs
  that survived in artifacts. But most reasoning never reaches a commit. It's spoken
  in the moment a human corrects an agent: *"no, don't do that — we tried it last
  quarter and it broke X."* **That correction IS a WHY**, and it's the cleanest signal
  we have, because a human explicitly said "no, here's why."
- **How it stays cheap and invisible** — three tiers:
  1. **Regex pre-filter** — runs on *every* prompt as a `UserPromptSubmit` hook.
     No LLM, <50ms, rejects ~90% of prompts. Catches correction-shaped language
     ("no", "actually", "wait", "that's wrong"). **Zero user-visible latency.**
  2. **Haiku → "pollen"** — in the background, a cheap model drafts a loose record of
     the correction (what was corrected, the stated WHY, the agent's prior position).
     Local only, never shipped yet.
  3. **Opus promote** — at session end (`Stop` hook, debounced), Opus reads the full
     transcript + all pollen and decides per item: **promote** (real lesson → hive),
     **demote** (you pushed back then accepted the agent's explanation — that's a
     "teach the agent to explain better" signal, not a lesson), or **discard** (noise).
- **Why two tiers:** catch fast (Haiku, before the moment is gone), judge slow (Opus,
  with the whole conversation in view). Only full-context Opus can tell a real lesson
  from a misunderstanding that resolved cleanly.
- **Cost:** ~$140/month at 50 devs — vs ~$1,800/yr for the naïve "Haiku on every
  prompt." Kill switch: `HIVEMIND_FORAGER=0`.

> Talking point: "Extraction mines what survived. The forager catches what's said out
> loud and would otherwise vanish. Together they cover both halves of our WHY."

---

## The pitch — wiring it into our autonomous agents (explainer scene 8)

Frame: **qbee** and the **self-verifying agent (in progress)** turn a Claude session
into a worker that branches, codes, commits, and opens a PR on its own. Today each run
starts with amnesia. Hivemind gives the worker **memory across runs** at three
touch-points — the flywheel:

| When | Touch-point | What it does |
|---|---|---|
| **Before** it writes code | `hivemind:query` | Pull the hidden constraints, rejected approaches, and past bugs for the area into context — so it doesn't re-walk into them. |
| **During / self-verify** | `hivemind:query` again | Before the agent declares "done," it re-queries: *"does my change violate anything the hive already knows?"* This is a natural hook for the self-verifying agent. Meanwhile the **forager** passively captures any course-corrections. |
| **After** it opens a PR | `/learn-from <that PR>` | Mine durable lessons from the work it just shipped — feeding the *next* agent's "before" step. |

**The flywheel:** `query → work → learn-from → (hive grows) → query`. Every run starts
smarter than the last. That's the headline: *we're not adding a tool, we're giving the
agent fleet a shared, compounding memory.*

> Talking point for the self-verifying agent specifically: "Self-verification today
> checks the code against itself — tests, types, the diff. Hivemind lets it also check
> against *institutional memory*: 'has anyone learned that this exact change is a trap?'
> That's a class of bug tests can't catch."

### Concrete first step to propose

1. **Query-on-entry**: have the agent call `hivemind:query` as part of its task
   kickoff, with the target folder/feature as `working_in`. Cheapest, highest-signal,
   no new infra. Start here.
2. **Learn-on-exit**: after the agent's PR is up, kick `/learn-from <PR-url>`.
3. **Turn the forager on** for supervised/HITL agent sessions (it's hook-based and
   already ships with the plugin — just install and don't set the kill switch).

---

## Likely questions — and answers

**"How is this different from just good docs / a wiki / READMEs?"**
Docs capture what someone bothered to write down, go stale, and aren't retrieved at the
moment of need. Hivemind grows *automatically* (from history and corrections), is
filtered to *only* the non-obvious WHY, and is delivered into context exactly when
you/the agent touch the relevant code.

**"How is it different from RAG over our codebase / code search?"**
Code search answers *what the code is*. Hivemind answers *why it is that way* — the
part that isn't in the code at HEAD by definition. It embeds the lesson's WHY-claim,
not the code.

**"Won't it get flooded with junk / how do you trust it?"**
The WHY-not-WHAT test is enforced at every stage and the system is tuned to
**under-emit**. The classifier defaults to skip; the learner refuses WHAT-lessons; the
consolidator de-dups; the forager's Opus pass demotes/discards non-lessons. Every run
keeps an audit log. We'd rather miss a lesson than pollute the corpus.

**"What does it cost?"**
Extraction: cheap model on most nodes, expensive only on flagged ones. Retrieval:
~$0.005/call. Forager: ~$140/mo at 50 devs. All have kill switches and the heavy work
is async / out of the user's path.

**"What's real today vs. planned?"**
Real: `/learn-from` (folder/URL/PR origins), `hivemind:query`, the forager hooks, S3 +
Bedrock + pgvector storage. Plus the data repo + audit logs. The plugin is marked
**experimental** and is on branch `SOL-1691_hivemind_plugin_v1`.
Planned/deferred: other origin types (Jira epic, library, user, logs), more facets
beyond `code` and `pull_request`, structural retrieval filters, GitHub Pages graph viz.

**"Does the forager add latency to my prompts?"**
Only the <50ms regex is in your path. Everything else (Haiku draft, Opus promote) is a
detached background process. If it ever misbehaves: `HIVEMIND_FORAGER=0`.

**"What about privacy / where does the data go?"**
Pollen is local-only until Opus promotes it. Promoted lessons go to our own AWS (S3 +
Bedrock KB). Worth a slide of its own if the audience is security-minded.

**"Why two separate agents (qbee + the self-verifying one) — same integration?"**
Yes — both are autonomous Claude sessions that branch/code/commit/PR. The three
touch-points are identical; the self-verifying agent just leans harder on the
*during/self-verify* query.

---

## If you have 5 minutes vs 30

- **5 min:** one-sentence version → the WHY-not-WHAT test → the flywheel table. Show
  scenes 1, 7, 8.
- **30 min:** full slide walk 1→9, pause on scene 4 (tiers + counterfactual) and
  scene 7 (forager tiers), then land the flywheel and the "concrete first step."
