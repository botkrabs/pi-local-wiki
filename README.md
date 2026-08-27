# pi-local-wiki

The pi-side companion to [`local-wiki`](https://github.com/botkrabs/local-wiki)
— the offline-Wikipedia MCP server. This package bundles the `local-search`
routing skill: how a pi agent should use `local_wiki` (title-first `get`,
phrase `search`, surgical fetches) instead of hitting the web by default. It
does not install the server itself.

## Prerequisite

A running `local_wiki` MCP server (see
[botkrabs/local-wiki](https://github.com/botkrabs/local-wiki): install →
download a ZIM from <https://download.kiwix.org/zim/wikipedia/> → run),
registered with your MCP client so the tools
`local_wiki_get` / `local_wiki_search` (or however your client names them)
exist. The skill also assumes a `web_search` tool for the routing rules.

## Install

```
pi install git:github.com/botkrabs/pi-local-wiki@v0.1.0   # pinned
pi install git:github.com/botkrabs/pi-local-wiki          # track main
```

## What you get

**`local-search` skill** — the routing policy between `local_wiki` and
`web_search`: title-first `get`, phrase `search` as title-finder, surgical
fetches (lead / `section=` before `full=`), existence checks, recency
fallback, and the anti-hallucination guardrails (facts are "as of the ZIM
snapshot").

## Development

A single skill — no build step, no dependencies. Deliberately no extensions
in this package: footer/stat lines are generic pi tools, not local-wiki
specific, and belong in their own package if ever shared.
