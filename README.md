# Production Scheduler

A production scheduling tool for 4Wall live events, film shoots and installations. You build each job as a project, add tasks hour by hour, and tag them with the people, departments, locations and equipment involved. The schedule can be viewed as a Gantt chart, a calendar or a list, alongside a Documents tab for ground plans, gear lists and links. Crews get a read-only page per project.

The app is a single HTML file (`index.html`) that runs as a published **claude.ai artifact**. It has no build step and no server of its own.

## What's in this repo

| Path | What it is |
| --- | --- |
| `index.html` | The whole app: markup, CSS and JavaScript in one file. |
| `scripts/build_snapshot.py` | Builds a read-only, single-project crew page from the app plus an export of its data. |
| `docs/notes.md` | Design decisions and history. |

There is no schedule data in this repo. Tasks, projects, crew names and uploaded files live in claude.ai's storage for the published app. Exported data and built crew pages are git-ignored because they contain crew names.

## How it runs

`index.html` is published to claude.ai with the Artifact tool, either by Claude or from a Claude Code / Cowork session. claude.ai wraps the file in its own page skeleton at publish time, which is why the page content starts with `<title>` and `<style>` inside `<body>`.

The page reaches claude.ai's runtime through `await window.claude.use(name)` and declares four capabilities when published:

| Capability | Used for |
| --- | --- |
| `db` | Shared document store for projects, tasks, views and documents. It updates live through `onSnapshot`. |
| `user` | Deciding whether the viewer can edit. View-only people and outside guests get a read-only page. |
| `assets` | Uploading PDFs and images for the Documents tab, served at `/_blob/<assetId>`. |
| `downloads` | Saving the calendar CSV and document files. |

Publish declaration: `capabilities: {"db": {}, "user": {}, "downloads": true, "assets": {}}`

**Opened outside claude.ai** (GitHub Pages, a local file), `window.claude` doesn't exist. The page loads and shows "Schedule unavailable". To run it on its own, swap the code in the `data` and `boot` sections for another backend, for example Firebase or Supabase. The rest of the app only touches `S.projects`, `S.tasks`, `S.views`, `S.docs` and `saveTask()` / `S.db.doc(...)`.

External libraries:

- Google Fonts: Bebas Neue and Roboto.
- pdf.js 3.11.174 from cdnjs, loaded only when a PDF is opened.

## Data model

All four collections hold JSON documents keyed by a short random id.

**`projects/{id}`**
```
name, type ("Live event" | "Film shoot" | "Installation"), venue,
start, end ("YYYY-MM-DD"), color (0-7, index into --c1..--c8)
```

**`tasks/{id}`**
```
projectId, title, status ("Planned" | "Confirmed" | "In progress" | "Done" | "On hold"),
start, end ("YYYY-MM-DDTHH:MM", local venue time with no time zone),
people[], departments[], locations[], equipment[], tags[], notes, updatedAt (ISO)
```
Older tasks may have a single `location` string. It's converted to `locations[]` when read.

**`views/{id}`** (saved, shareable views)
```
name, mode ("gantt" | "calendar" | "list" | "docs"), zoom, groupBy, colorBy, calMode, q,
filters { project[], people[], departments[], locations[], equipment[], tags[], status[] }, createdAt
```
A view's link is `<artifact URL>#view-<id>`. Set `ARTIFACT_URL` near the top of the script to the published URL so "Copy link" builds full links.

**`docs/{id}`**
```
projectId, title, kind ("pdf" | "image" | "link"), note, departments[], createdAt,
assetId, contentType, sizeBytes   (uploaded files)
url                               (links)
```

## Code map

`index.html` is organized into sections marked `/* ---------- name ---------- */`:

| Section | Responsibility |
| --- | --- |
| colors | Bar colors by department, project, person, location or status. |
| filtering | `matches()` and the filter rail's values and counts. |
| render loop | `render()` batches redraws; draws banner, filter rail, toolbar, chips and the current view. |
| Gantt | Gantt chart at hour, day or week zoom, grouped by project, department, location, person or equipment. |
| Calendar | Month grid, plus week and day hour grids (click an empty slot to add a task). |
| Documents | Document tabs, PDF and image viewer, upload and link editor. |
| List | Table grouped by day. |
| Gantt drag | Drag to move or resize, on Gantt bars and on calendar blocks (15-minute snap). |
| events | Clicks, tabs, filters, keyboard. |
| data | `saveTask()` and write-error handling. |
| task editor | Side drawer with tag inputs, length presets and "Add to Outlook/Google" links. |
| projects | Projects dialog: create, edit, delete. |
| share & export | Saved views, links, calendar CSV and iCalendar text. |
| boot | Gets capabilities and subscribes to the four collections. |

All state lives in the global object `S`. Every color is a CSS token on `:root`, with dark-mode overrides.

## Crew snapshots

Anyone with access to the live app can read every project. To give a crew only their own job, build a snapshot: a separate read-only page with one project's data built into the page. Other projects, and any crew or departments used only on them, never reach the page.

```
python3 scripts/build_snapshot.py index.html data/ snapshots/my-show.html \
    "<snapshot URL or empty>" "23 Nov 2026" --project "My Show"
```

- `data/` is an export of the live store, one JSON file per document: `data/projects/*.json`, `data/tasks/*.json`, `data/views/*.json`, `data/docs/*.json`. Claude can produce it with the ArtifactData tool (`list` with `out_dir`).
- For a new project, build once with an empty URL and publish (capabilities: `{"downloads": true}`). Then rebuild with the new URL and republish, so view links point at the snapshot.
- Snapshots don't update on their own. Rebuild and republish after schedule changes.
- Uploaded documents aren't copied into snapshots yet. The script accepts a `FILES_JSON` map of `assetId` to a published file path for when that's added.

## Branding

The app follows the 4Wall Brand Guidelines (2021):

- 4Wall Blue `#005EB8`, rich black and white.
- Bebas Neue for headings and Helvetica for body text (Arial fallback).
- The reverse logo at 48px, embedded as a data URI and taken from the official guidelines PDF.

Check with 4Wall Marketing before making this repository public.

## Sharing limits (as of Nov 2026)

4Wall's claude.ai organization settings turn off public links for artifacts. People outside 4Wall can be invited by email and need a claude.ai account.
