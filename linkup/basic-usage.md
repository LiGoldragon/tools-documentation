---
source: https://github.com/shauryajain21/linkup-cli (README) + https://pypi.org/project/linkup-cli/
fetched: 2026-04-23
trimmed: dropped pipx/brew install notes, `linkup setup` browser flow (our wrapper provides the key from gopass), SDK-level details
---

# linkup basic usage

## What it is

Linkup is a web-search API aimed at grounding LLMs — you send a query, you get back either a list of sources or a concise AI-written answer with citations. `linkup-cli` wraps the API for quick interactive queries from a terminal. Built on the `linkup-sdk` Python package.

## Setup (in our workspace)

`linkup` is on PATH via the mentci-tools bundle. The wrapper shell script sources `LINKUP_API_KEY` from gopass at `linkup.so/api-key` before exec'ing the real CLI, so you don't run `linkup setup` — the key is already there.

```
linkup config      # verify the key is being picked up
```

If you need the raw API key for a non-wrapped caller: `gopass show -o linkup.so/api-key`.

## Core commands

```
linkup search "<query>"         # AI-grounded answer with sources (default)
linkup fetch "<url>"            # extract page content as cleaned markdown
linkup config                   # show current config
```

Short aliases: `linkup s` for `search`.

## Search — depth and output

Two dimensions matter:

- **`--depth` / `-d`** — how hard the search works.
  - `fast` — quickest, cheapest, for simple lookups.
  - `standard` (default) — normal web search, good for most queries.
  - `deep` — agentic workflow, slower but handles multi-hop research questions.
- **`--output` / `-o`** — what you get back.
  - `sourcedAnswer` (default) — a written answer with citations.
  - `searchResults` — a list of source documents (title, url, snippet), no synthesis.

```
linkup search "population of tokyo"                             # standard + sourcedAnswer
linkup search "compare Rust vs Zig async models" --depth deep
linkup search "rust async runtimes" --output searchResults
linkup s "latest AI research" -d deep -o searchResults          # short forms
```

## Fetch

```
linkup fetch "https://example.com/article"
```

Returns the page as cleaned markdown. Useful when a URL is paywalled-adjacent or heavy on JS and a plain `curl` + html2text would miss the content. The underlying API supports JS rendering and raw HTML inclusion; the CLI exposes the simple form.

## Response format

Pretty-printed to the terminal (the CLI depends on `rich`). For `sourcedAnswer` you get an answer paragraph followed by a list of source URLs with snippets. For `searchResults` you get a list of sources only. No JSON mode flag in the current CLI — if you need structured output, use the SDK directly.

## When to reach for it

- **`linkup search` vs WebSearch tool** — linkup gives cited, synthesized answers from a search API tuned for LLM grounding; WebSearch gives raw SERP-style results. For "what's the current state of X" questions, linkup is usually the better first cut.
- **`linkup fetch` vs WebFetch** — WebFetch sends page content through a summarizer; `linkup fetch` returns raw cleaned markdown. Use linkup fetch when you need the actual bytes, not a summary.
- **`linkup` vs raw `curl` to the API** — the CLI is faster to type and handles auth + formatting. Use raw curl only when you need parameters the CLI doesn't expose (structured output schemas, async batching).

## Gotchas

- Every call costs credits against your Linkup plan. `--depth deep` costs noticeably more than `standard`. Don't loop over `deep` queries in a script.
- No JSON flag on the CLI — output is styled text. Scrape-parsing it is fragile; drop to the SDK if you need machine-readable results.
- `linkup setup` opens a browser to fetch an API key. Don't run it — our wrapper already supplies `LINKUP_API_KEY` from gopass and running setup will try to overwrite config.

## Want more

- CLI repo + README: https://github.com/shauryajain21/linkup-cli
- API docs: https://docs.linkup.so
- Python SDK (more parameters, async, structured output): https://github.com/LinkupPlatform/linkup-python-sdk
