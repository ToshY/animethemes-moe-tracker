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
- `.github/workflows/full-refresh.yml` — on-demand full rebuild

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

### Full refresh (manual only)

`.github/workflows/full-refresh.yml` is `workflow_dispatch` only:

1. **`probe` job** — runs `task animethemes:probe`, outputs
   `{lastPage, total, pages: [1..N]}` for the matrix.
2. **`download` job (matrix)** — one job per page (~49 jobs), each runs
   `task animethemes:download:page PAGE=<n>` with `ANIMETHEMES_SORT=CREATED_AT`
   (stable page contents across reruns), and uploads `.cache/pages/<n>.json`
   as artifact `page-<n>`.
3. **`merge` job** — downloads all `page-*` artifacts, assembles them under
   `.cache/pages/`, runs `task animethemes:manifest`, commits, releases,
   deploys Pages. Skips commit/release/deploy when no changes were detected.

## Taskfile tasks

| Task | Purpose |
|---|---|
| `animethemes:healthcheck` | minimal `{ __typename }` query, exits non-zero on failure |
| `animethemes:probe` | prints `{lastPage, total, pages: [...]}` (used by the matrix) |
| `animethemes:download:page PAGE=N` | fetches a single page with retries, writes `.cache/pages/N.json` |
| `animethemes:download` | sequential `while` loop over `1..lastPage`; local-only convenience |
| `animethemes:manifest` | flattens `.cache/pages/*.json`, writes `data/<id>.json` and merges `data/manifest.json` |
| `animethemes:incremental` | healthcheck → first `ANIMETHEMES_INCREMENTAL_PAGES` pages (`UPDATED_AT_DESC`) → manifest merge |
| `animethemes:fetch` | healthcheck → all pages (sequential) → manifest |
| `animethemes:clean` | wipes `.cache/pages` and `data/` |

Environment / variables:

- `GITHUB_PAGES_URL` (`toshy.github.io/animethemes-moe-tracker`) — base used for manifest `url` field.
- `ANIMETHEMES_ENDPOINT` (`https://graphql.animethemes.moe/`) — GraphQL endpoint.
- `ANIMETHEMES_PAGE_SIZE` (`100`) — page size for `animePagination`.
- `ANIMETHEMES_HEALTHCHECK_TIMEOUT_SECONDS` (`10`).
- `ANIMETHEMES_INCREMENTAL_PAGES` (`1`) — number of `UPDATED_AT_DESC` pages for the cron job.
- `ANIMETHEMES_SORT` (`CREATED_AT`) — per-page sort. Use `UPDATED_AT_DESC` for "what changed recently?" runs; use `CREATED_AT` for CI matrix so page contents are stable across reruns.
- `PAGE` — required Task var for `animethemes:download:page`.

No parallelism inside the Taskfile by design: parallelism lives in the CI
matrix (`full-refresh.yml`), so the local tasks stay simple, predictable
and easy to debug.

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
- **No parallelism in Tasks** — fan-out is the workflow matrix's job. This keeps
  local runs reproducible and avoids subtle ordering bugs (Task's mvdan/sh
  shell behaves slightly differently from `bash` and trips on common patterns
  like `if jq -e …; then`, large `echo "$var" | jq`, `read -d`).
- **Retries** in `animethemes:download:page` use exponential backoff (1–4
  attempts, sleep `attempt * 2`s). Keep new network calls aligned.
- **Silent tasks** unless there is a reason not to be (`silent: true`).
- **No `scripts/` directory.** Anything ad-hoc lives in the Taskfile or
  shouldn't be committed.
- Per-anime file naming is `<id>.json` (upstream stable id), **not**
   `<malID>.json`. Use `manifest.json`'s `malID` field for MAL-based lookups.
- `.gitattributes` marks non-data paths as `export-ignore`; update it when
  adding top-level files.

