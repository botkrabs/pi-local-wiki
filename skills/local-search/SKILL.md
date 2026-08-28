---
name: local-search
description: Use when searching, looking up facts, or fact-checking world knowledge — routes between local_wiki (offline Wikipedia) and web_search (SearXNG).
---

# Local Search

Two local sources; route by question type (patterns below):
- `local_wiki_get(title, section=..., full=..., lang=...)` — offline Wikipedia ZIM, ~35 ms, frozen at the ZIM's download date (stated in the local_wiki README)
- `local_wiki_search(query, lang=...)` — phrase search over article text; ranked titles + snippets, no exact title needed
- `lang="zh"` (either tool) → Chinese Wikipedia ZIM (same API, full zh text + FTS). Useful for zh entities, cross-checking, and Chinese-language sources; first zh query is slow (cold 13.7 GB index over 9p), repeats are fast.
- `web_search` — local SearXNG; the only source with recency

## Fetch surgically (context tokens are the budget)
- Never pull a whole article into context for a narrow question.
- Long articles (text > ~8 KB) return **lead mode by default**: the intro (up to the first `##`) + a section list + a pointer — answer from the lead when it suffices; pass `section="Name"` for one section (~1–7 KB) or `full=True` for the whole article. Short articles come back whole, as before. A bad section name returns the full section list to pick from.
- Large MCP results still spill to `/tmp/pi-mcp-output-*/` — grep that file for the part you need, then `read` with offset/limit.
- SearXNG: keep result counts low (≤ 8); use the `news` category for time-bound questions and expect some false-positive noise (filter it).

## Patterns
- **Title known** → `local_wiki_get` directly.
- **No title, just a phrase/concept** → `local_wiki_search(phrase)` → `get` the top hit (add `section=` for depth).
- **Title unknown but guessable** → guess via `get`; on "not found", retry with one of the suggested related titles (one round-trip, no web needed). Works in `lang="zh"` too (per-gram merge since 2026-08-28; the candidate list can be noisy — pick the semantically right one).
- **Unknown transliteration / variant spelling** (e.g. a Chinese phonetic film title like 哈拉瑪莉) → `local_wiki_search` with the user's exact spelling (`lang="zh"` for zh text); if the article itself isn't a top hit, do ONE more local search with the guessed variant or a related entity (actor/director in a snippet) before considering `web_search`. Two ~35 ms local calls beat escalation; only go to the web after the local follow-up misses.
- Short 2–3 word searches can return occurrence-ranked noise (a known libzim fulltext weakness) — treat low-confidence hit lists as candidates, verify with `get` before citing.
- **Discovery** ("what exists around X", news, buying, community) → `web_search` first, then `local_wiki_get` the Wikipedia result for the deep read.
- **Fact-check** → `local_wiki_get` first; mismatch or no article → `web_search` for a second independent source; load-bearing claims → primary source, and state the snapshot date.
- **Chinese-language topic** (zh place/person/work names, zh sources) → `local_wiki_search/get` with `lang="zh"`; the en wiki often has a stub or a transliterated title — zh is usually the fuller article. Titles come back in zh; pass them straight back to `get(lang="zh")`.

## Existence checks
- "Does X exist as an article?" → `local_wiki_search(X)`: hits = exists, empty = strong evidence it doesn't (absence of article ≠ absence of fact).

## Memoize
- After verifying facts, write a few-line distilled note (fact + ZIM snapshot date) to working memory — next sessions reuse verified claims instead of re-guessing from weights.

## Guardrails
- local_wiki facts are "as of the ZIM snapshot" — say so whenever a fact is load-bearing.
- Post-snapshot topics, disputed topics, and current numbers → `web_search` (local_wiki cannot have fixed what Wikipedia fixed online after the snapshot).
- Tool errors (404/timeout): check service status once (`docker ps`, `ss -tlnp | grep -E '8080|3211'`), report the result, don't loop retries.
