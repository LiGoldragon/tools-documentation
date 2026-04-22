# tools-documentation

Curated docs from upstream tools we use daily, plus our own discoveries that
don't live anywhere else. Small files, one topic per file, so agents can do
targeted partial reads instead of loading giant references.

## Layout

```
jj/        — Jujutsu (jj)
bd/        — beads (bd)
dolt/      — Dolt + DoltHub
nix/       — Nix, flakes, home-manager
```

Each file has a frontmatter with the source URL and date fetched, so we can
tell when to refresh against upstream.

## Conventions

- One topic per file. Keep each under ~100 lines.
- Header format:
  ```
  ---
  source: <canonical-url>
  fetched: <YYYY-MM-DD>
  trimmed: <what-we-stripped>
  ---
  ```
- Prefer verbatim excerpts from upstream over paraphrase, so the fidelity
  that would've been lost through a WebFetch summarizer is preserved.
- For our own discoveries (things not in upstream docs), mark with
  `source: (our own — <context>)`.

## Why this exists

- WebFetch loses structure via its summarizer; local Read returns exact bytes.
- Curated subset beats full upstream doc for the sections we actually use.
- Splittable per-topic means `Read --offset --limit` targets the relevant section only.
- Git history of edits = audit of what we learned when.

Pointer memories in bd (e.g. `bd-global memories`) should reference files
here by path rather than inline the content.
