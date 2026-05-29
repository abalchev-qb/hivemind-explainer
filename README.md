# Hivemind — interactive explainer

A single-page, click-through animation that walks through how **Hivemind** works:
it walks a project's code graph (commits → PRs → tickets → files), extracts the
non-obvious **WHY** behind changes as small tiered lessons, stores them in a
vector knowledge base, and serves them back at retrieval time.

**[▶ View the explainer](https://abalchev-qb.github.io/hivemind-explainer/)**

- Self-contained single `index.html` — no build, no dependencies, works offline.
- Navigate with **→ / space** (advance), **←** (back), or the dots.
- Six scenes: the problem · the graph walk · classify · learn & backtrack ·
  consolidate & store · retrieve.

The worked example (a fictional `billing-dashboard` feature with a plan-usage
limit bug) is illustrative — it mirrors the real extraction flow without naming
any internal systems.

Built with Claude Code, inspired by Simon Willison's
[Interactive Explanations](https://simonwillison.net/guides/agentic-engineering-patterns/interactive-explanations/)
pattern.
