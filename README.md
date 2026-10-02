# MediaShock APAC — LinkedIn Content Hub

A shared LinkedIn content calendar, planner, and analytics dashboard for MediaShock APAC.
Runs entirely as a single client-side HTML file (no build step), backed by Firebase
(Firestore + Authentication) so the whole team can edit it together with live updates.

**Live:** https://mediashock-apac.github.io/ms-linkedin-hub/

## Access

Sign in with a **@mediashock.com.sg** Google account. Anyone outside that domain is blocked,
both in the UI and independently at the database level via Firestore Security Rules
(`firestore.rules`) — so it's not just a client-side gate.

## What it does

- **Calendar** — plan LinkedIn posts by content bucket (fully editable: add, rename, recolor,
  or remove buckets), attach an image carousel to a post, and insert emoji from a built-in picker.
- **Content Planner** — a kanban board (New Idea → In Review → Needs Changes → Approved) for
  drafting and iterating on post ideas, with a feedback log per idea, before sending an approved
  one straight into the Calendar.
- **Analytics** — a date-range/granularity-aware view of page performance: follower growth,
  impressions, engagement, a top-posts leaderboard, and follower demographics. Opens with
  illustrative sample data; use the Import/Export JSON buttons to bring in real exported data.

All of the above is shared and live: an edit one person makes appears for everyone else with
the page open, no refresh needed. Each person's own analytics date-range/granularity selection
stays local to their browser, so browsing a different time window doesn't change what others see.

## Architecture

- `content-hub-firebase.html` — the canonical source. Edit this file, then regenerate `index.html`:
  ```
  python sync_from_scratchpad.py content-hub-firebase.html
  ```
  (wraps it with the DOCTYPE/head/body structure GitHub Pages needs, plus the favicon link —
  the source file itself starts from a bare `<title>`, matching how it's edited/previewed
  elsewhere).
- **Firestore** — `posts`, `buckets`, and `ideas` collections (one document per item), plus a
  single `analytics/current` document for the shared analytics data. Everything is read via
  live `onSnapshot` listeners.
- **Images** — stored as base64 directly on the post document, not Firebase Storage (new
  Firebase projects require the paid Blaze plan for Storage even within free-tier usage, so we
  skipped it). A size guard blocks saving before Firestore's 1MiB per-document limit is hit,
  with a clear message asking to trim images, rather than a cryptic failure.
- `firebase.json`, `.firebaserc`, `firestore.rules` — standard Firebase CLI project files, for
  the project `mediashock-content-hub-2237f` (Spark/free plan).

## Local development

The Firebase Local Emulator Suite lets you test auth + Firestore fully offline, without
touching the real project:
```
firebase emulators:start --project demo-mediashock-hub
python -m http.server 8765   # serve the file over http:// — Firebase Auth's popup
                              # sign-in flow needs a real http(s) origin, not file://
```
The app auto-detects `localhost`/`127.0.0.1` and points itself at the emulator instead of the
real project in that case — no config changes needed to switch between the two.

## Deploying

Edit `content-hub-firebase.html`, regenerate `index.html` with `sync_from_scratchpad.py`, then
push to `main` — GitHub Pages serves the repo root directly, so the live site updates on its own.
Pushing only `content-hub-firebase.html` changes nothing that anyone can see; `index.html` is the
file Pages actually serves.

`firestore.rules` is **not** covered by that push. It needs a separate
`firebase deploy --only firestore:rules`, or a paste into the Firebase console —
otherwise a new collection or query fails with "Missing or insufficient permissions" even though
the deployed page looks correct.

## Contributing

`CLAUDE.md` in this repo is the deep reference — the data model, each view's design reasoning,
and the gotchas worth knowing before changing anything. `DESIGN.md` covers the icon set and
visual conventions. The patterns shared across all Mediashock internal tools (auth, notifications,
Firestore rules conventions) live in the `CLAUDE.md` one folder up.
