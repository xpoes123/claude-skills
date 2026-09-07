Publish a Claude session brief to your own domain so you can read it in a browser.

**Template — fill in `$VPS_HOST`, `$SHARE_REPO` (e.g. `you/your-share`), `$SHARE_DOMAIN`.**

## How this works (app + manifest, pull-to-deploy)

A small app (FastAPI or similar) runs on your VPS behind a reverse proxy, serving a
"now" feed at `/`, per-project indexes at `/{project}/`, and pages at
`/{project}/{slug}.html`, enforcing access control.

The single source of truth is a **`manifest.json`** in the repo `$SHARE_REPO` (local
clone `~/code/<your-share-repo>`, VPS working tree e.g. `/opt/share`). Each page has an
entry: project, date, title, tag, visibility (`public`/`unlisted`/`password`). The app
renders ALL indexes from the manifest — never hand-build index HTML.

You publish by writing the page HTML + a manifest entry, committing, pushing, and
pulling on the VPS. Never SCP loose files. The app re-reads the manifest per request,
so no restart is needed.

```
write ~/code/<share-repo>/{project}/{slug}.html + upsert manifest.json
  → git add/commit/push → ssh $VPS_HOST "cd /opt/share && git pull --ff-only"
```

Visibility control (password-gating, unlisted, moving/renaming pages) belongs in an
admin dashboard, not here.

## When to use

At the end of a big working session — overnight work, anything to read later. Also
mid-session for checkpoints.

## Arguments

`$ARGUMENTS` can be:
- Empty — auto-detect project from current directory, auto-generate title from date
- A title
- A project override

## Step 0 — Sync the local clone

```bash
git -C ~/code/<share-repo> pull --ff-only
```

## Step 1 — Gather session context

```bash
git log --oneline --since="24 hours ago" 2>/dev/null || git log --oneline -20
git diff HEAD~5..HEAD --stat 2>/dev/null | head -40
git status --short
git branch --show-current
```

## Step 2 — Write the brief

Write a thorough markdown brief. Be honest and specific — this is a diagnostic, not a
PR description.

```
# [Title] — [Date]

## TL;DR
2-4 sentences: state of the project now vs when the session started, the single most
important thing to know.

## What was asked
## What was attempted (narrative, don't skip failed attempts)
## What succeeded
## What failed or is incomplete
## Current state
## Known issues / untested paths
## Recommendations
## Commit log
```

## Step 3 — Render to HTML into the local clone

Enforce naming: `{project}/YYYY-MM-DD-<kebab-slug>.html`. A minimal markdown→HTML
renderer + your own manifest upsert helper does the rest (see the repo's own
`manifest.py` for the upsert function if you built one, or write a small helper that
appends `{file, project, date, title, tag, visibility}` to the JSON).

## Step 4 — Deploy (commit → push → pull)

```bash
cd ~/code/<share-repo> && git add -A && \
  git commit -m "share: <project> — <title>" && \
  git push && \
  ssh $VPS_HOST "cd /opt/share && git pull --ff-only"
```

No service restart needed if the app re-reads the manifest per request.

## Step 5 — Notify (optional)

Ping whatever notification channel you use (Discord bot, etc.) with the title + TL;DR
+ URL — see the `notify` skill in this repo.

## Step 6 — Report to the user

- The URL
- TL;DR in 2-3 sentences
- The single most important action item

## Setup notes (fill in for your own infra)

- App runs behind your reverse proxy on a private port; venv + `.env` for secrets
  (never commit `.env` — gitignore it).
- If your admin dashboard writes back to the repo, the VPS deploy key needs write
  access.
