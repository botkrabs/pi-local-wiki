---
name: local-search
description: Use when searching, looking up facts, or fact-checking world knowledge — routes between local_wiki (offline Wikipedia) and web_search (SearXNG).
---

# Local Search

Two local sources — `local_wiki_get` / `local_wiki_search` (offline Wikipedia ZIM, ~35 ms, frozen at the ZIM snapshot — dates: `pi-learning/memory.md` → local_wiki line) and `web_search` (local SearXNG; the only source with recency). The tool metadata covers calling mechanics (lead mode, section/full, similar-title retry, `lang: 'en'|'zh'`, archives) — this skill is routing + policy + gotchas.

## Fetch surgically (context tokens are the budget)
- Narrow question → answer from the lead; one `section=` before `full=True`; never whole-article for a narrow fact.
- Big MCP results spill to `/tmp/pi-mcp-output-*/` — grep the file, `read` with offset/limit.
- SearXNG: ≤ 8 results; `news` category for time-bound (expect some noise — filter it).

## Patterns (route by question type)
- **Title known** → `local_wiki_get` directly.
- **Phrase/concept, no title** → `local_wiki_search` → `get` the top hit (add `section=` for depth).
- **Title unknown but guessable** → guess via `get`; on "not found", retry with a suggested related title (mechanics in the tool description; with `lang="zh"` the candidate list can be noisy — pick the semantically right one).
- **Unknown transliteration / variant spelling** (e.g. 哈拉瑪莉) → `local_wiki_search` with the user's exact spelling (`lang="zh"`); if not a top hit, ONE more local search with a guessed variant or a related entity (actor/director in a snippet) before considering `web_search`. Two ~35 ms local calls beat escalation.
- **Short 2–3 word searches** → occurrence-ranked noise is possible (known libzim fulltext weakness) — treat low-confidence hit lists as candidates; verify with `get` before citing.
- **Discovery** ("what exists around X", news, buying, community) → `web_search` first, then `local_wiki_get` the Wikipedia hit for the deep read.
- **Fact-check** → `local_wiki_get` first; mismatch or no article → `web_search` for a second independent source; load-bearing claims → primary source, and state the snapshot date.
- **Chinese-language topic** (zh names, zh sources) → `local_wiki_search/get` with `lang="zh"` (en wiki often stub/transliterated; zh usually fuller). Titles come back in zh — pass them straight back. First zh query is slow (cold 13.7 GB index over 9p); repeats fast.

## Existence checks
- "Does X exist as an article?" → `local_wiki_search(X)`: hits = exists, empty = strong evidence it doesn't (absence of article ≠ absence of fact).

## Memoize
- After verifying facts, write a few-line distilled note (fact + ZIM snapshot date) to working memory — next sessions reuse verified claims instead of re-guessing from weights.

## Guardrails
- local_wiki facts are "as of the ZIM snapshot" — say so whenever a fact is load-bearing.
- Post-snapshot topics, disputed topics, and current numbers → `web_search` (local_wiki cannot have fixed what Wikipedia fixed online after the snapshot).
- Tool errors (404/timeout): check service status once (`docker ps`, `ss -tlnp | grep -E '8080|3211'`), report the result, don't loop retries.
