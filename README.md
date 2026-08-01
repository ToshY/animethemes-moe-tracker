<h1 align="center">🎵 AnimeThemes.moe Tracker</h1>

<div align="center">
    <img src="https://img.shields.io/github/v/release/toshy/animethemes-moe-tracker?logo=github&label=Release" alt="GitHub Release">
    <img src="https://img.shields.io/github/actions/workflow/status/toshy/animethemes-moe-tracker/tracker.yml?branch=main&label=Tracker" alt="Tracker">
    <br /><br />
    <div>An auto-updated repository for anime opening, ending and insert themes, sourced from the <a href="https://animethemes.moe">animethemes.moe</a> GraphQL API.</div>
</div>

## 🎶 Data & Manifest

All data is retrieved from the [animethemes.moe GraphQL API](https://graphql.animethemes.moe/) and normalized into per-anime JSON files with [jq](https://github.com/jqlang/jq).

### Per-anime files

- One file per anime: [`./data/<id>.json`](./data), where `<id>` is the upstream `Anime.id` from animethemes.moe.
- Each file is the raw GraphQL `Anime` node, minified and byte-deterministic. It contains:
  - core fields: `id`, `title` (`romaji`, `english`, `native`), `slug`, `synopsis`, `format`, `season`, `year`, `createdAt`, `updatedAt`
  - `resources` (external IDs incl. MAL, AniList, Kitsu, …)
  - `images`, `studios`, `series`
  - `animethemes[]` with `song`, `animethemeentries[]`, and full `videos[]` metadata (`resolution`, `source`, `nc`, `subbed`, `lyrics`, `overlap`, …) plus per-video `audio`

### Manifest

A [`./data/manifest.json`](./data/manifest.json) index is rebuilt on every run and sorted by `id`. Each entry contains:

- `id`: upstream `Anime.id`
- `malID`: MyAnimeList ID, or `null` if the anime has no MAL resource
- `url`: the GitHub Pages URL of the per-anime JSON
- `sha256`: checksum of the per-anime JSON
- `createdAt`: when the entry first appeared in the manifest
- `updatedAt`: when its payload last changed (i.e. its `sha256` differs)

Per-anime files are only overwritten when their `sha256` changes; otherwise `updatedAt` is preserved so unchanged entries don't churn git history.

### Update cadence

| Workflow | Trigger | What it does |
|---|---|---|
| `tracker.yml` | cron `0 0 * * *` + `workflow_dispatch` | Pulls the first N pages of `animePagination(sort: UPDATED_AT_DESC)` (default 1 page = 100 anime) and merges them into the manifest. Cheap, fast, idempotent. |
| `full-refresh.yml` | `workflow_dispatch` only | Probes pagination, fans out a matrix job per page (~49 jobs × 100 anime), merges all artifacts and rebuilds the manifest. |
| `reconcile.yml` | cron `0 3 * * 1` + `workflow_dispatch` | Sweeps every upstream anime id with a minimal `{ id }` query (~1 KB per page) and diffs it against the manifest, reporting anime that were **removed upstream** (orphans) or are not yet tracked. Report-only by default — orphans never fail the run. Deleting them is opt-in via `workflow_dispatch` with `prune` enabled. |

Scheduled cron times may be [delayed](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#schedule) by GitHub Actions.

### 📦 Publishing

- The full `./data` directory is published as a static site at [`toshy.github.io/animethemes-moe-tracker`](https://toshy.github.io/animethemes-moe-tracker).
- `manifest.json` and every `<id>.json` are available at the root.
- Pages deployment only runs when a commit was actually produced.

### 📅 Releases

- [Releases](https://github.com/ToshY/animethemes-moe-tracker/releases) are created on every run that produces a change, tagged as `YYYY.MM.DD-<run_id>` (`-full` suffix for full refreshes).

## ℹ️ Disclaimer

- The source is [animethemes.moe](https://animethemes.moe) via their public [GraphQL API](https://graphql.animethemes.moe/).
- This repository is _not_ affiliated with animethemes.moe.

## 📚 Scope & responsible use

This repository is a **metadata index**, not a media archive:

- The published JSON files contain anime, theme, song, entry and video **metadata** (titles, IDs, slugs, resolutions, source flags, etc.) and **URLs** that point back to animethemes.moe's own CDN.
- No video or audio files are downloaded, re-hosted or otherwise redistributed by this project. Playback always happens on animethemes.moe's infrastructure.
- The tracker is a small, non-commercial, read-only consumer of the public GraphQL API. It is not a search, streaming or download service and is not intended to compete with animethemes.moe or themes.moe.
- API usage is intentionally light: a single page of recently updated anime daily (incremental), with full refreshes only on manual request and paginated at the API's standard page size.

If you are building something that needs **bulk media** rather than metadata, please use the official [AnimeThemes.moe OP/ED Collection Full Backup](https://animethemes.moe/faq) instead of scraping the CDN.

> [!NOTE]
> If you're part of the AnimeThemes team and have concerns about this project, please open an issue and it will be addressed.

## ❕ License

This repository comes with a [BSD 3-Clause License](./LICENSE).
