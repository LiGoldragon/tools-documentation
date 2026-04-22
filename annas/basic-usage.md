---
source: https://github.com/iosifache/annas-mcp (README + cmd/annas-mcp + internal/modes/cli.go)
fetched: 2026-04-23
trimmed: dropped MCP JSON configuration, deep protocol details, and political/legal preamble; kept CLI surface + our wrapper specifics
---

# annas basic usage

## What it is

A Go CLI (plus optional MCP server) that searches and downloads books and academic articles from Anna's Archive. Upstream binary is `annas-mcp`; our mentci-tools wrapper exposes it as `annas`. Most operations (all downloads) require an Anna's Archive API key, which in turn requires a paid donation/membership.

## Setup (in our workspace)

Nothing to install — `annas` is on `PATH` via the mentci-tools CLI bundle. The wrapper transparently populates three env vars each invocation:

- `ANNAS_SECRET_KEY` — pulled from gopass at `annas-archive.gl/secret-key`
- `ANNAS_DOWNLOAD_PATH` — defaults to `$PWD` (so files land where you ran the command)
- `ANNAS_BASE_URL` — defaults to `annas-archive.gd`

Override by exporting any of these before calling `annas`.

## Daily commands

```
annas book-search "<query>"               # search books by title, author, topic
annas book-download <md5-hash> <filename> # download by MD5; filename MUST include extension (.pdf, .epub)
annas article-search "<query-or-DOI>"     # keyword search OR DOI lookup (auto-detected if starts with "10.")
annas article-download <doi>              # download paper by DOI
annas mcp                                 # run as an MCP server over stdio
annas --help                              # list commands
annas --version                           # print version
```

Notes verified from `cli.go`:

- `book-download` takes **exactly two** positional args: hash and filename. The filename must have an extension — the tool derives the format from it (`.pdf` → `pdf`, `.epub` → `epub`).
- `article-search` sniffs for a DOI prefix (`10.`) and does a single-paper lookup instead of a keyword search. Useful shortcut: `annas article-search "10.1038/nature12345"` prints the paper metadata without downloading.
- `article-download` first tries a fast download via the book-style hash path (needs `ANNAS_SECRET_KEY`), then falls back to SciDB.

## Search result format

Book results print one block per hit:

```
Title: ...
Authors: ...
Publisher: ...
Language: ...
Format: ...
Size: ...
URL: ...
Hash: ...        # <-- feed this to `annas book-download`
```

Article (paper) results:

```
DOI: ...
Title: ...
Authors: ...
Journal: ...
Size: ...
Hash: ...
Download URL: ...
Sci-Hub: ...
Page: ...
```

## Gotchas

- **`.env` warning on startup** is harmless. The binary always tries `godotenv.Load()` from the cwd first; our wrapper sets the env vars directly, so the "Error loading .env file" message can be ignored.
- **Secret key required** for any download (`book-download`, `article-download`). Searches work without it but downloads 401. Key comes from an Anna's Archive paid membership.
- **Mirror outages.** We default to `annas-archive.gd`. If it's down, export `ANNAS_BASE_URL=annas-archive.se` (or `.li`, `.pm`, `.in`). Check https://open-slum.org for current mirror status.
- **Download path is `$PWD`** by our wrapper's default — so `cd` to where you want the file first, or set `ANNAS_DOWNLOAD_PATH` explicitly.

## MCP mode

`annas mcp` starts a stdio MCP server exposing four tools: `book_search`, `book_download`, `article_search`, `article_download`. Use this only if you want Claude Desktop / Cursor to invoke Anna's Archive directly as a tool. In our setup we prefer the CLI wrapper route — agents just shell out to `annas` — so MCP mode is rarely needed.

## Want more

- Upstream README: https://github.com/iosifache/annas-mcp/blob/main/README.md
- Releases: https://github.com/iosifache/annas-mcp/releases
