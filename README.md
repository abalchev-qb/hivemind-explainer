# Hivemind — interactive explainer

A single-page, click-through animation that walks through how **Hivemind** works:
it walks a project's code graph (commits → PRs → tickets → files), extracts the
non-obvious **WHY** behind changes as small tiered lessons, **also forages WHYs
from live corrections**, stores them in a vector knowledge base, and serves them
back at retrieval time — to people *and* to autonomous agents.

**[▶ View the explainer](https://abalchev-qb.github.io/hivemind-explainer/)**

- Self-contained single `index.html` — no build, no dependencies, works offline.
- Navigate with **→ / space** (advance), **←** (back), or the dots.
- Nine scenes: the problem · the graph walk · classify · learn & backtrack ·
  consolidate & store · retrieve · **the forager (autonomous capture)** ·
  **agents plug in (the flywheel)** · the full loop.

Each technical scene opens with an **"In plain terms"** line so a newcomer can
follow the idea before the details.

## The three parts of Hivemind

1. **Active extraction** — `/learn-from <folder|PR>`: a deterministic graph walk
   feeds a Haiku classifier (learn / skip / backtrack), then Sonnet/Opus learners
   and backtrackers write tiered lessons, which a consolidator de-dups before
   they land in the store.
2. **Retrieval** — `hivemind:query`: a Haiku retriever does semantic search over
   the Bedrock KB and returns up to 5 lessons as a compact JSON envelope.
3. **The forager** — passive, hook-driven capture of WHYs from the moments a human
   corrects an agent (regex pre-filter → Haiku "pollen" → Opus promote at
   session end). Zero user effort.

## Meeting prep

- [`MEETING-PREP.md`](./MEETING-PREP.md) — a talking-points / walkthrough script:
  the narrative arc, key points per topic, the integration pitch, and likely Q&A.
- [`ARCHITECTURE-BRIEF.md`](./ARCHITECTURE-BRIEF.md) — a one-page architecture
  reference covering all three parts plus autonomous-agent integration; shareable.

The worked example (a fictional `billing-dashboard` feature with a plan-usage
limit bug) is illustrative — it mirrors the real extraction flow without naming
any internal systems.

Built with Claude Code, inspired by Simon Willison's
[Interactive Explanations](https://simonwillison.net/guides/agentic-engineering-patterns/interactive-explanations/)
pattern.
