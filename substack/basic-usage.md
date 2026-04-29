---
source: (our own — derived from /home/li/git/substack-cli src/application.rs + src/cli.rs + src/client.rs + src/local_post_manifest.rs + src/prosemirror.rs + tests/)
fetched: 2026-04-29
trimmed: no trimming; command surface and markdown/manifest behavior verified against the implementation
---

# substack basic usage

## What it is

Rust CLI (binary name `substack`) for creating, publishing, and managing
Substack posts and publication settings via Substack's private web API.
Authentication is just the `substack.sid` session cookie plus the publication
hostname.

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

## Command surface

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
                     [--title <s>] [--subtitle <s>] [--cover-image <path>]
                     [--link-manifest <path>] [--publish-linked-files] [--draft]
substack post update <post-id> (--body <markdown> | --file-path <path>)
                     [--title <s>] [--subtitle <s>] [--cover-image <path>]
                     [--link-manifest <path>] [--publish-linked-files]
substack post list   [--limit <n>]            # default 10
substack post get    <post-id> [--full] [--save-html <path>] [--save-json <path>]
substack post delete <post-id>
```

`--body` and `--file-path` conflict on `post create`/`post update`.
Clap does not enforce presence; the app errors if neither is provided.
`--community-enabled` and `--community-disabled` conflict on `publication update`.

## Daily workflows

Publish a Markdown file:

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

## Post pipeline

For `post create` / `post update`, the implementation does this:

1. Read Markdown from `--body` or `--file-path`.
2. Split leading frontmatter delimited by `---`.
3. Choose the title in this order:
   `--title` > frontmatter `title:` > first `# Heading` in the body > `Untitled`.
4. Choose the subtitle in this order:
   `--subtitle` > frontmatter `subtitle:`.
5. Remove the leading `# Heading` from the body if it exactly matches the chosen title.
6. Render GFM pipe tables to PNG, upload them, and replace the table block with a Markdown image.
7. Upload local Markdown images `![alt](./file.png)` and rewrite them to Substack CDN URLs.
8. Rewrite local Markdown article links to canonical Substack post URLs through the manifest.
9. Convert the resulting Markdown to ProseMirror JSON and send it to Substack.

`post create --draft` stops after draft creation and draft update. It does not
publish the post and does not record the source file in the local manifest.

`post update <post-id>` updates the existing draft/post id in place and then
hits Substack's publish endpoint again.

## Markdown behavior

Supported body conversion after the preprocessing steps above:

- `##` and `###` headings
- paragraphs
- blockquotes
- hard line breaks via trailing `\`
- inline links: `[label](href)`
- `*italic*`, `**bold**`, `***bold italic***`
- standalone image blocks: `![alt](url)` become Substack `captionedImage` nodes

Not supported as native rich-text constructs:

- lists
- code blocks
- inline image syntax inside ordinary paragraph text

The last point matters: only a block that is exactly `![alt](src)` becomes an
image node. If image syntax appears inline with other text, the local file may
still be uploaded and the URL rewritten, but the body renderer will leave the
Markdown image syntax as paragraph text.

## Local links and manifest

Repo-local `.md` links are special:

- The CLI looks up `.substack-posts.json` at `--link-manifest <path>` if given.
- Otherwise it discovers the nearest ancestor containing `.jj` or `.git`,
  starting from the source file's directory or `cwd`.
- If a linked Markdown file already has `post_id` + `slug` in the manifest, the
  link is rewritten to `https://<hostname>/p/<slug>`.
- Link fragments are preserved:
  `../diet/Ambrosian_Diet.md#closing` → `https://<hostname>/p/<slug>#closing`.
- If a local Markdown target is missing from the manifest, the command fails by
  default.
- `--publish-linked-files` recursively publishes missing linked Markdown files,
  records them in the manifest, and then rewrites the links.
- Cycles in local Markdown links are detected and rejected.
- Linked Markdown files must live under the manifest root; links outside it are errors.

Manifest entries can also carry `banner_image`. If `--cover-image` is omitted
and the source file has a manifest `banner_image`, that image is uploaded and
used as the Substack cover automatically.

## Images and tables

- `substack image upload --file ./image.png` uploads a local image and prints JSON with the CDN URL.
- `publication set-logo|set-wide-logo|set-cover-photo|set-email-banner --file <path>` uploads the file first, then writes the returned URL into publication settings.
- Local Markdown images resolve relative paths against the source file's directory.
- Remote image sources beginning with `http://`, `https://`, or `data:` pass through unchanged.
- GFM pipe tables are handled before Markdown-to-ProseMirror conversion:
  each table block is rendered to a PNG and uploaded, so published posts show a table image rather than literal pipes or row-paragraph fallback.

## Read commands

- `post list --limit N` returns JSON summaries only:
  `id`, `title`, `slug`, `post_date`, `audience`, `wordcount`.
- `post get <id>` returns the same summary shape by default.
- `post get <id> --full` prints the full JSON returned by `/api/v1/drafts/<id>`.
- `--save-html` writes `body_html` to a file.
- `--save-json` writes pretty-printed `body_json` to a file.
- If `body_html` / `body_json` are absent from the API response, the save flags error with
  `post response did not include body_html` / `body_json`.
- If you pass save flags without `--full`, stdout becomes a JSON array describing the saved files, not the post summary.

## Notes

- This tool talks to Substack's private web API endpoints directly:
  `/api/v1/publication`, `/api/v1/publication_user`, `/api/v1/drafts/...`,
  `/api/v1/archive`, `/api/v1/image`.
- API behavior is unversioned and can break without notice.
- Logs go to stderr via `tracing`; use `RUST_LOG=debug` or `RUST_LOG=info`.

## Want more

- Repo: https://github.com/LiGoldragon/substack-cli
- Command contract: `/home/li/git/substack-cli/CLAUDE.md`
- Implementation: `/home/li/git/substack-cli/src/cli.rs`, `/home/li/git/substack-cli/src/application.rs`, `/home/li/git/substack-cli/src/prosemirror.rs`, `/home/li/git/substack-cli/src/local_post_manifest.rs`
