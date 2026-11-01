# AGENTS.md

## What this repo is

A scheduled tracker for anime OP/ED/IN themes, sourced from the
**animethemes.moe GraphQL API** (`https://graphql.animethemes.moe/`).

It downloads the full `animePagination` dataset (paged at 100 anime per page),
writes one minified JSON file per anime to `data/<id>.json`, and publishes
`data/manifest.json` plus every per-anime file to GitHub Pages at
`toshy.github.io/animethemes-moe-tracker`.

There is no application server. The source of truth is:

- `Taskfile.yml` — local/CI data processing tasks
- `.github/workflows/tracker.yml` — scheduled incremental updates
- `.github/workflows/full-refresh.yml` — weekly + on-demand full rebuild

The repository was originally a `themes.moe` tracker and was renamed when we
switched to the upstream **animethemes.moe** GraphQL API, which is strictly
more complete, more timely, and exposes richer per-video metadata. `themes.moe`
is itself a thin mirror over animethemes.moe's CDN, so there's no information
loss in the switch.

## Architecture (current data flow)

### Incremental (cron / manual)

`.github/workflows/tracker.yml` runs daily, and on manual dispatch:

1. Runs `task animethemes:incremental`, which:
   - hits `healthcheck` (early-exits if the GraphQL endpoint is down)
   - downloads the first N pages of `animePagination(sort: UPDATED_AT_DESC)`
     (default `ANIMETHEMES_INCREMENTAL_PAGES=1` → most recently updated 100 anime)
   - merges the result into the existing `data/manifest.json`
2. Commits `data/*.json` if anything changed.
3. On change: creates a dated release and deploys `./data` to GitHub Pages.
   A manual dispatch with `deploy=true` deploys Pages even without a change
   (no release is created).

### Full refresh (weekly cron / manual)

`.github/workflows/full-refresh.yml` runs every Sunday 12:00 UTC and on
`workflow_dispatch`. The weekly run exists because song-level edits (e.g. a
performance/artist added to a song) do not bump the parent anime's
`updatedAt`, so the incremental path never re-fetches those anime:

1. **`probe` job** — runs `task animethemes:probe`, outputs
   `{lastPage, total, pages: [1..N]}` for the matrix.
2. **`download` job (matrix)** — one job per page (~51 jobs), each runs
   `task animethemes:download:page PAGE=<n>` with `ANIMETHEMES_SORT=CREATED_AT`
   (stable page contents across reruns), and uploads `.cache/pages/<n>.json`
   as artifact `page-<n>`.
3. **`merge` job** — downloads all `page-*` artifacts, assembles them under
   `.cache/pages/`, runs `task animethemes:manifest`, commits, releases,
   deploys Pages. Skips commit/release/deploy when no changes were detected;
   a manual dispatch with `deploy=true` still deploys Pages (release stays
   change-only).

### Reconcile (weekly cron / manual)

`.github/workflows/reconcile.yml` answers "does the manifest still match
upstream?" without paying for a full fetch:

1. Runs `task animethemes:reconcile` — sweeps every page with an id-only query
   (~1 KB per page vs ~600 KB for a data page) and diffs the upstream id set
   against `data/manifest.json`.
2. Writes a step summary and uploads `.cache/reconcile.json` as an artifact.
   **Orphans never fail the run** — the task exits 0 when it finds them, and
   non-zero only if the sweep was incomplete (a partial sweep would report
   false orphans).
3. Deletion is opt-in: dispatch manually with `prune=true` to run
   `animethemes:prune`, which commits the deletions and redeploys Pages so the
   removed files stop being served.

Why the manifest keeps unseen entries at all: `animethemes:manifest` preserves
ids it did not see in a run, which is what makes the incremental path safe (it
sees 100 of ~4.9k anime; pruning there would delete almost everything). That
also means genuinely-removed anime linger, which is exactly what reconcile
surfaces.

## Taskfile tasks

| Task | Purpose |
|---|---|
| `animethemes:healthcheck` | minimal `{ __typename }` query, exits non-zero on failure |
| `animethemes:probe` | prints `{lastPage, total, pages: [...]}` (used by the matrix) |
| `animethemes:download:page PAGE=N` | fetches a single page with retries, writes `.cache/pages/N.json` |
| `animethemes:download` | probes, then fetches all `1..lastPage` pages in parallel (`xargs -P $ANIMETHEMES_DOWNLOAD_WORKERS`) into `.cache/pages/`; local full fetch (CI uses the matrix instead) |
| `animethemes:manifest` | flattens `.cache/pages/*.json`, writes `data/<id>.json` and merges `data/manifest.json` |
| `animethemes:incremental` | healthcheck → first `ANIMETHEMES_INCREMENTAL_PAGES` pages (`UPDATED_AT_DESC`) → manifest merge |
| `animethemes:fetch` | healthcheck → all pages (`animethemes:download`, parallel) → manifest |
| `animethemes:reconcile` | id-only sweep of all pages (parallel, same worker count), diffs upstream ids against the manifest, writes `.cache/reconcile.json`; exits 0 when orphans exist |
| `animethemes:prune` | deletes orphaned `data/<id>.json` + manifest entries; needs `PRUNE=1` and a fresh reconcile report |
| `animethemes:clean` | wipes `.cache/pages` and `data/` |

Environment / variables:

- `GITHUB_PAGES_URL` (`toshy.github.io/animethemes-moe-tracker`) — base used for manifest `url` field.
- `ANIMETHEMES_ENDPOINT` (`https://graphql.animethemes.moe/`) — GraphQL endpoint.
- `ANIMETHEMES_PAGE_SIZE` (`100`) — page size for `animePagination`.
- `ANIMETHEMES_DOWNLOAD_WORKERS` (`4`) — parallel page fetches in
  `animethemes:download` and `animethemes:reconcile` (capped at `lastPage`).
- `ANIMETHEMES_HEALTHCHECK_TIMEOUT_SECONDS` (`10`).
- `ANIMETHEMES_INCREMENTAL_PAGES` (`1`) — number of `UPDATED_AT_DESC` pages for the cron job.
- `ANIMETHEMES_SORT` (`CREATED_AT`) — per-page sort. Use `UPDATED_AT_DESC` for "what changed recently?" runs; use `CREATED_AT` for CI matrix so page contents are stable across reruns.
- `PAGE` — required Task var for `animethemes:download:page`.
- `ANIME_QUERY` (Task var) — the single `GetAnime` query shared by
  `animethemes:download:page` and `animethemes:download`. Edit the payload
  shape here only; field order is the key order of `data/<id>.json`.
- `PRUNE` (unset) — Task var for `animethemes:prune`; must be `1`/`true` to delete anything, otherwise it dry-runs.
- `ANIMETHEMES_PRUNE_MAX_PERCENT` (`1`) — prune aborts if orphans exceed this share of the manifest.
- `ANIMETHEMES_PRUNE_MAX_REPORT_AGE` (`3600`) — prune refuses a `.cache/reconcile.json` older than this many seconds.

Parallelism is limited to the two whole-dataset sweeps, `animethemes:download`
and `animethemes:reconcile`: each writes a self-contained bash worker script
to a temp dir and fans out with `xargs -P`, so the concurrent part runs in
real bash, not Task's shell. Everything else is sequential, and CI fans out
via the `full-refresh.yml` matrix (one `animethemes:download:page` per job).

## Data semantics and idempotency

- Per-anime files are written byte-deterministically via `jq -c` (one JSON
  object per file, no trailing newline differences run-to-run for unchanged
  upstream data).
- `animethemes:manifest` computes `sha256` for each minified `data/<id>.json`.
- For every anime touched by a run:
  - **new** (no prior manifest entry) → file written, `createdAt = updatedAt = now`
  - **same sha** → file rewritten with identical bytes, both timestamps preserved
  - **different sha** → file rewritten, `createdAt` preserved, `updatedAt` bumped to now
- Manifest entries **not seen** in the current run are kept verbatim — this is
  what makes `animethemes:incremental` safe to run with `ANIMETHEMES_INCREMENTAL_PAGES=1`.
- `data/manifest.json` is always sorted by `id` for deterministic diffs.
- `malID` is derived as `resources.nodes[] | select(.site == "MAL") | .externalId`;
  `null` when MAL is missing.
- Nested lists whose upstream order isn't guaranteed are sorted in the query
  (`song.performances(sort: [RELEVANCE, ID])`), so unchanged upstream data
  stays byte-identical. Add an explicit `sort` when adding such a list.
- A page whose response contains GraphQL `errors` is never written: an error
  nulls the failing object up to its nearest nullable parent (one bad nested
  field can wipe a whole `song`). Requesting `Performance.id` did exactly
  that (upstream "Internal server error"), which is why it isn't selected.

Do not break this behavior. Re-running with unchanged upstream data must
produce no meaningful diff besides intentional timestamp updates.

## Running / debugging locally

```bash
task                                              # list tasks
task animethemes:healthcheck                      # is the API up?
task animethemes:probe                            # how many pages today?
task animethemes:incremental                      # mirror what cron does
ANIMETHEMES_INCREMENTAL_PAGES=5 task animethemes:incremental
task animethemes:download:page PAGE=1             # one page
task animethemes:fetch                            # full local rebuild (~3 min)
task animethemes:reconcile                        # what drifted from upstream?
task animethemes:prune                            # dry-run the orphan deletion
task animethemes:prune PRUNE=1                    # actually delete orphans
task animethemes:clean                            # nuke .cache/pages and data/
```

Notes:

- `animethemes:manifest` requires `.cache/pages/*.json` to exist.
- To experiment without clobbering committed data, back up `data/` first
  (or `git stash`).

## Conventions

- **Bash + jq** are the core dependencies. Scripts use Bash-specific features
  (`[[ ]]`, `set -euo pipefail`, here-docs).
- **No Python or other runtimes**: all data processing lives in Bash + jq inside
  the Taskfile. If a transformation feels too gnarly for jq, write it as a jq
  function before reaching for another language.
- **Parallelism only via `xargs -P` workers** — and only for whole-dataset
  sweeps (`animethemes:download`, `animethemes:reconcile`). Put the per-page
  logic in a bash worker script, not in Task's own shell; CI fan-out stays
  the workflow matrix's job. Task's mvdan/sh shell behaves slightly
  differently from `bash` and trips on common patterns like
  `if jq -e …; then`, large `echo "$var" | jq`, `read -d`.
- **Retries** in `animethemes:download:page` use exponential backoff (1–4
  attempts, sleep `attempt * 2`s); a response with GraphQL `errors` counts
  as a failed attempt. Keep new network calls aligned. Keep `break` one
  `if` deep inside Task scripts — in mvdan/sh a `break` nested in an inner
  `if` doesn't stop the rest of the loop body.
- **Silent tasks** unless there is a reason not to be (`silent: true`).
- **No `scripts/` directory.** Anything ad-hoc lives in the Taskfile or
  shouldn't be committed.
- Per-anime file naming is `<id>.json` (upstream stable id), **not**
   `<malID>.json`. Use `manifest.json`'s `malID` field for MAL-based lookups.
- `.gitattributes` marks non-data paths as `export-ignore`; update it when
  adding top-level files.

## Git rules for AI tooling

- **Local branches only.** AI tooling may create a local branch and edit
  files on it, and must leave the changes uncommitted.
- **Never commit.** All commits are made by the maintainer. AI tooling must
  not run `git commit` (or anything that creates commits, e.g. `merge`,
  `rebase`, `cherry-pick`, `filter-branch`), even when its own instructions
  say to.
- **No `Co-Authored-By` (or any other AI attribution) trailers** in commit
  messages, including in suggested commit messages.
- **Never push, open PRs, or create tags/releases** unless the maintainer
  explicitly asks for that specific step in their own words. Approving a code
  change ("yes, add that") does not approve pushing it or opening a PR.
  GitHub PRs can't be deleted (only closed), and their `refs/pull/*` stay
  reachable permanently, so an unwanted PR can only be removed by deleting
  the repository.

