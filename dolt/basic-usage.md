---
source: https://docs.dolthub.com/ (introduction/getting-started/{database,git-for-data}, sql-reference/version-control/{remotes,dolt-sql-procedures,dolt-system-tables,querying-history})
fetched: 2026-04-22
trimmed: dropped replication/clustering, DoltLab, enterprise specifics, internal storage format, exhaustive list of dolt_* procedures; distilled to daily-dev CLI usage
---

# Dolt basic usage

## Mental model

Dolt is **git for SQL tables**. Same commands, same semantics — but the versioned unit is a row/cell rather than a line. `dolt commit` snapshots the whole database; `dolt diff` shows per-row changes; `dolt merge` does row-level three-way merges. If you know git, you mostly know Dolt.

One DB lives in one directory (`.dolt/` under it, same role as `.git/`). Branches, commits, remotes, reflog — all the git furniture is there.

## Setup

```
dolt config --global --add user.name "Your Name"
dolt config --global --add user.email "you@example.com"
dolt init                             # in an empty dir: creates .dolt/ and an initial DB
```

## Two interfaces

Everything versioning-related has two equivalent forms:

- **CLI**: `dolt status`, `dolt commit -m ...`, `dolt merge feature`. Good for scripts and one-shots.
- **SQL**: `call dolt_commit('-m', '...')`, `select * from dolt_log`, `call dolt_merge('feature')`. Good from inside a sql-server session or an app.

Pick by context; semantics are identical.

## Schema + data

```
dolt sql -q "create table t (id int primary key, name varchar(64));"
dolt sql -q "insert into t values (1, 'alice'), (2, 'bob');"
dolt sql -q "select * from t"
```

Or drop into an interactive shell with `dolt sql`.

## Git-like commands

```
dolt status                           # staged / unstaged tables
dolt diff                             # row-level diff of working vs HEAD
dolt add t                            # stage table t  (or `dolt add .`)
dolt commit -m "add t with seed rows" # commit staged changes
dolt log                              # commit history
```

SQL equivalents: `select * from dolt_status`, `select * from dolt_diff`, `call dolt_add('t')`, `call dolt_commit('-m', 'msg')`, `select * from dolt_log`.

Tip: `call dolt_commit('-am', 'msg')` is stage-all-modified + commit in one shot.

## Branching + merging

```
dolt branch                           # list
dolt checkout -b feature              # create + switch
dolt checkout main
dolt merge feature                    # merge feature into current
dolt branch -d feature                # delete
```

SQL: `call dolt_checkout('-b', 'feature')`, `call dolt_merge('feature')`, `select * from dolt_branches`.

Conflicts surface as rows in `dolt_conflicts_<table>`; resolve by editing, then commit.

## Time travel

```
dolt sql -q "select * from t as of 'HEAD~3'"
dolt sql -q "select * from t as of 'feature'"
dolt sql -q "select * from t as of 'qv3k2...'"     -- commit hash
```

`AS OF` accepts any ref: commit hash, branch name, or relative (`HEAD^2`, `HEAD~N`).

## Remotes + DoltHub

```
dolt remote add origin ligoldragon/mydb            # DoltHub shorthand
dolt remote -v                                     # see full URL
dolt push origin main
dolt pull origin main
dolt clone ligoldragon/mydb
```

DoltHub shorthand expands to `https://doltremoteapi.dolthub.com/<owner>/<db>`. For a **git-remote mode** (store dolt data under a ref in a regular git repo), dolt accepts exactly two URL shapes:

```
dolt remote add origin git+https://github.com/ORG/REPO.git   # HTTPS
dolt remote add origin git@github.com:ORG/REPO.git           # SSH
```

The `git+` prefix only pairs with `https://`; SSH is the bare `git@host:ORG/REPO.git` form.

## Auth

Two parallel mechanisms — don't confuse them:

- **Credential keypair** for `dolt push`/`pull` (dolt-protocol on DoltHub): `dolt login` opens a browser, generates a keypair, associates it with your DoltHub account. Used by the CLI transparently thereafter.
- **API token** for the DoltHub REST API at `https://www.dolthub.com/api/v1alpha1/...`. Separate from the keypair; issued from DoltHub settings. Our token lives at gopass `dolthub.com/api-token`. See `reference_dolthub_api.md` in memory.

## Running a sql-server

```
dolt sql-server                       # serves current DB on 127.0.0.1:3306
```

Default: MySQL-compatible, user `root`, no password, 8h timeout. Connect with any MySQL client:

```
mysql --host 127.0.0.1 --port 3306 -u root
```

Flags: `--host`, `--port`, `--user`, `--password`, `-l debug`, or a YAML config via `--config`.

## Want more

- Full docs: https://docs.dolthub.com/
- CLI reference: https://docs.dolthub.com/cli-reference/cli
- Stored procedures: https://docs.dolthub.com/sql-reference/version-control/dolt-sql-procedures
- System tables: https://docs.dolthub.com/sql-reference/version-control/dolt-system-tables
- Git-for-data walkthrough: https://docs.dolthub.com/introduction/getting-started/git-for-data
