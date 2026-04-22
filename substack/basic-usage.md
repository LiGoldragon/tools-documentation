---
source: (our own — derived from github.com/LiGoldragon/substack-cli src/cli.rs + README.md + CLAUDE.md)
fetched: 2026-04-23
trimmed: no trimming; full subcommand surface verified against clap definitions
---

# substack basic usage

## What it is

Rust CLI (binary name `substack`) for creating, publishing, and managing
Substack posts and publication settings via Substack's private web API. It
authenticates with the same `substack.sid` session cookie the browser uses.

Features: create/update/list/get/delete posts, upload images, set the four
publication image slots (logo, wide-logo, cover-photo, email-banner), read and
update publication settings.

## Setup (in our workspace)

On PATH through the mentci-tools bundle as `substack`. The packaged binary is
the flake's wrapped default: it shells out to `gopass` before `exec`ing the
unwrapped CLI, injecting the two required env vars:

- `SUBSTACK_API_KEY` ← `gopass show -o substack.com/api-key`
- `SUBSTACK_HOSTNAME` ← `gopass show -o substack.com/api-key publication-url`
  (with any `https://` / trailing `/` stripped)

If either env var is already set in the environment, the wrapper skips the
corresponding gopass call. The unwrapped CLI itself only reads those two env
vars — no config file, no keyring lookup in Rust.

Missing either → exits with `SUBSTACK_API_KEY must be set` /
`SUBSTACK_HOSTNAME must be set`.

## Core commands

Three top-level groups: `publication`, `image`, `post`.

```
substack publication get
substack publication update [--name <s>] [--hero-text <s>] [--language <s>]
                            [--copyright <s>] [--logo-url <s>] [--logo-url-wide <s>]
                            [--cover-photo-url <s>] [--email-banner-url <s>]
                            [--theme-var-background-pop <s>]
                            [--community-enabled | --community-disabled]
substack publication set-logo         --file <path>
substack publication set-wide-logo    --file <path>
substack publication set-cover-photo  --file <path>
substack publication set-email-banner --file <path>

substack image upload --file <path>

substack post create (--body <markdown> | --file-path <path>)
                     [--title <s>] [--subtitle <s>] [--cover-image <path>] [--draft]
substack post update <post-id> (--body <markdown> | --file-path <path>)
                     [--title <s>] [--subtitle <s>] [--cover-image <path>]
substack post list   [--limit <n>]            # default 10
substack post get    <post-id> [--full] [--save-html <path>] [--save-json <path>]
substack post delete <post-id>
```

`--body` and `--file-path` conflict on `post create`/`post update` (clap enforced).
`--community-enabled` and `--community-disabled` conflict on `publication update`.

## Common workflows

Publish a Markdown article (frontmatter `title:` / `subtitle:` honored; first
`# Heading` becomes title and is stripped from the body):

```
substack post create --file-path ./article.md
substack post create --file-path ./article.md --draft            # draft, don't publish
substack post create --file-path ./article.md --cover-image ./cover.jpg
```

Edit an existing post and republish:

```
substack post update 123456789 --file-path ./article.md
```

Inspect / archive a post:

```
substack post list --limit 20
substack post get  123456789 --full --save-html post.html --save-json post.json
```

## Gotchas

- Markdown → ProseMirror conversion is intentionally tiny: `##`/`###`
  headings, paragraphs, blockquotes, `\`-trailing hard breaks, and
  `*italic*` / `**bold**` / `***bold italic***`. No lists, no code blocks, no
  links — anything else is not converted.
- The CLAUDE.md rule "do not describe HTML as Markdown" — when you save a
  post with `--save-html`, that's HTML from the API, not round-trippable
  Markdown.
- Substack's private web API is unversioned and can break without notice.
- Logs go to stderr via `tracing`; bump with `RUST_LOG=debug` (or `info`).

## Want more

- Repo: https://github.com/LiGoldragon/substack-cli
- Full command contract: `/home/li/git/substack-cli/CLAUDE.md` (§ CLI Contract)
- Markdown / image behavior detail: `/home/li/git/substack-cli/README.md`
