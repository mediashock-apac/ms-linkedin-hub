# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Patterns shared across every Mediashock internal tool (notifications, theme toggle, icons,
> auth, Firestore rules gotchas) live in `CLAUDE.md` in the parent `Claude Projects/` folder —
> check there before building something this project's own architecture below doesn't cover.

## What this is

MediaShock APAC's LinkedIn Content Hub — a shared content calendar, planner (kanban for post
ideas), and analytics dashboard. Single client-side HTML file, no build step, backed by Firebase
(Firestore + Google Auth) for live multi-user editing.

Live at https://mediashock-apac.github.io/ms-linkedin-hub/ (GitHub Pages).

## Critical: canonical source vs. generated file

- **`content-hub-firebase.html` is the canonical source. Always edit this file.**
- **`index.html` is generated — never hand-edit it, never treat a diff in it as the real change.**
  It's `content-hub-firebase.html` wrapped with the DOCTYPE/head/body structure GitHub Pages needs
  (the source file itself starts at a bare `<title>`).
- Regenerate after any edit:
  ```
  python sync_from_scratchpad.py content-hub-firebase.html
  ```
  This also stamps a fresh UTC build version into both `index.html` (`CURRENT_BUILD_VERSION`) and
  `version.txt`, which the deployed page polls to auto-reload clients onto new deploys (never mid-modal
  — see "Live update" in DESIGN.md).
- `<head>`-level additions (meta tags, preload hints, the manifest/favicon links) go in
  `sync_from_scratchpad.py`'s `HEAD_OPEN` template, not in `content-hub-firebase.html` — that file
  has no `<head>` at all.

## Shipping a change

Use the `/ship-content-hub` slash command (`.claude/commands/ship-content-hub.md`) — it drives the
full release loop: sync → `node --check` syntax verification → QA against a real Firebase emulator
via the `content-hub-qa` subagent (`.claude/agents/content-hub-qa.md`) → update `DESIGN.md` if the
change introduced a non-obvious pattern → stage changed files by name. It never commits/pushes
without asking first — that's a standing rule for this project, not a per-change judgment call.

For manual local testing instead of the full flow:
```
firebase emulators:start --project demo-mediashock-hub
python -m http.server 8765   # serve over http:// — Firebase Auth popup needs a real origin, not file://
```
The app auto-detects `localhost`/`127.0.0.1` and points itself at the emulators instead of prod —
no config changes needed.

## Architecture

- **Firestore collections**: `posts`, `buckets`, `ideas`, `notifications`, `suggestions` (one doc
  per item), plus single documents `analytics/current` and `goals/<metric>` (one goal doc per
  metric, capped at 3), plus `settings/*`, `people`, `activity` and `meta/rulesVersion`. Everything
  reads live via `onSnapshot`.
  - **Access control enumerates every collection** in `firestore.rules`, each gated on
    `isMediashock()` (`@mediashock.com.sg` only). This file used to say the opposite — that one
    blanket `match /{document=**}` covered everything, so "a brand-new collection here needs no
    separate rules deploy." **That is no longer true, and following it will cost you an
    afternoon.** An unlisted collection is denied outright, with no error anywhere except
    `permission-denied` at the point of use. That is exactly how `settings/{quickLink,goal}`
    shipped silently broken — see the stale-rules tripwire section below. A new collection now
    needs its own `match` block *and* a rules deploy, same as the sibling Team Project Manager app.
  - `notifications` is queried with `where("recipient", "==", ...)` only (sorted client-side) to
    avoid needing a composite index.
- **Notifications**: `notifyRecipients` writes a `notifications` doc per recipient.
  `notifyWithMentions` (used by both the feedback-add and feedback-reply handlers on Posts and
  Ideas) wraps it to merge two recipient sets into one notification each: anyone `@Name`-mentioned
  in the text (matched via `parseMentions` against `getKnownNames()` — every name that's ever
  appeared as a post/idea Owner, since there's no fixed team roster) gets type `"mention"`, and
  everyone else in the primary recipient list (the item's Owners for a new feedback item, or the
  original author for a reply) gets `"feedback"`/`"feedback_reply"`. `wireMentionAutocomplete`
  drives the `@`-triggered suggestion menu on both feedback textareas (mirrors Flowboard's
  task-comment mention menu). Assigning/tagging is feedback-only for now — the checklist/assignee
  UI in the Post/Idea modals is dead code (`createTaskListInput` is defined but never wired to any
  DOM elements — see the "Tasks UI is removed for now" comments near `currentPostTasks`/
  `currentIdeaTasks`), so there's no live per-task assignment to notify on.
  - **Desktop popups**: an opt-in toggle in the user menu (`desktopNotifToggleBtn`, `localStorage`
    key `mscontenthub_desktop_notif`) fires a native `Notification` from the `notifications`
    `onSnapshot` listener in `startListeners` for anything added *after* the listener's first
    snapshot (`notifListenerReady` flips true once that first snapshot resolves). Deliberately not
    a timestamp comparison: an earlier version compared each notification's client-generated `at`
    field against this tab's own `Date.now()` at attach time, which silently ate popups whenever
    the notifying user's and the recipient's machine clocks disagreed (the in-app bell has no such
    check, so it kept working — only desktop popups went quiet). Tab-open-somewhere only — no
    service-worker push, no server. Requires the browser's Notification permission to be granted;
    `notifyRecipients` never notifies the person who triggered it, so testing needs a second
    account/tab, not self-feedback/self-mention.
- **Images** are pasted URLs, not uploads — no Firebase Storage (would require the paid Blaze plan)
  and no base64-on-document (hits Firestore's 1MiB limit and looked pixelated). A Drive folder link
  pasted into the same Images field is auto-detected and rendered as a distinct folder chip. Image
  thumbnails are wrapped in a link so they open full-size in a new tab; a thumbnail/preview whose
  URL 404s gets a `.broken`/`.broken-single` placeholder instead of the browser's native broken-
  image icon (see "Icons" in DESIGN.md — this placeholder is drawn with the same masked-SVG
  convention as every other icon, not an emoji).
- **Four tabs**: Calendar (post scheduling by content bucket — shows a trailing-4-week posting
  cadence badge), Content Planner (kanban: New Idea → In Review → Needs Changes → Approved, sending
  an approved idea into the Calendar), Analytics (date-range/granularity-aware performance view —
  opens with sample data until real exported data is imported), Suggestions (comment/reply feature
  board, same embedded-array reply model as Post/Idea feedback). A notification bell (top-right of
  the topbar) surfaces new feedback/replies/suggestions per-user.
  - **Analytics import** (`#importAnalyticsBackdrop`) is JSON-only — a Python script
    (`agent/import_linkedin_xls.py`/`export_analytics.py`, outside this HTML file) converts the
    raw LinkedIn XLS export first; the browser side just parses/validates/writes whatever JSON it's
    given. The file input sits in a `.file-drop-zone` that also accepts a dragged-and-dropped file
    (`loadAnalyticsFileIntoTextarea`, shared by both the `change` handler and the drop handler) —
    this was the first real OS-file drag-and-drop in the app; every other drag-and-drop here moves
    an existing DOM element (image reorder, kanban cards), not a file, so there was no precedent to
    extend. A `window`-level `dragover`/`drop` `preventDefault()` guard stops a file dropped just
    outside the zone from triggering the browser's default "navigate to this file" behavior. The
    zone's own `dragenter`/`dragleave` pair is depth-counted (`importAnalyticsDropDepth`), not a
    bare toggle — the zone has child elements (the label, the file input), and the browser fires
    enter/leave on those too as the cursor crosses their boundaries, which flickered the highlight
    on and off during a single drag if the toggle was bare. Counting nets to zero only once the
    cursor has left every element inside the zone.
    Numeric `DAY_FIELDS` are coerced with `Number()` (invalid/missing values default to `0`, each
    counted separately) rather than passed through as-is, so a stray non-numeric cell doesn't
    silently corrupt a later sum in the analytics view — the import summary reports how many
    values needed defaulting instead of pretending the file was clean. Import feedback stays the
    single inline `#importAnalyticsError` paragraph (red for errors, green for success) — it sits
    next to the field it's about and persists while you fix the file, which a toast wouldn't.
  - **Stale-data reminder**: `updateStaleAnalyticsBanner`, called from `renderAnalytics()`, shows a
    dismissible banner (reusing `.onboarding-banner`'s visual recipe) when the newest day in
    `state.days` is more than `STALE_ANALYTICS_DAYS` (14) old — never for sample data. Unlike the
    onboarding banner's dismiss-forever flag, dismissal here is keyed to the specific stale date
    (`mscontenthub_stale_analytics_dismissed` in `localStorage`), so importing fresher data
    automatically re-arms it the next time *that* data goes stale, rather than staying silently
    dismissed forever after the first click.
- **Mobile (≤720px)**: view-only for Post/Idea editing (creation entry points hidden, existing
  items open read-only with a "View only" badge); everything else stays fully functional, including
  adding (not removing) idea/post feedback comments. Month/week grid becomes an agenda list; the
  Top Posts table collapses to stacked cards.
- **PWA-installable**: `manifest.json`/`sw.js`/icons are hand-maintained, separate from the sync
  pipeline (not derived from `content-hub-firebase.html`). `sw.js` deliberately does no caching.

## Who can edit what

`posts` and `ideas` are `allow read, write: if isMediashock()` — **any signed-in teammate can
edit or delete any post or idea.** That is the original model and the current one.

**Ownership-scoped editing shipped on 2026-08-31 and was reverted on 2026-09-02.** Worth reading
before proposing it again:

- It was ported from Flowboard, where restricting edits to the assignee works. It does not
  transfer. **A content calendar is not a task board**: a post is picked up by whoever is free,
  several people touch the same post, and many posts have no owner set at all.
- **`owners` was never a permission field.** It was added as a soft "who's looking after this"
  label, years before anything read it for access. Nobody had curated it for the job, so turning
  it into a hard gate locked people out of work they were actively doing.
- The lockout was also **silent in a second way**: owner names are free-typed, so "Harin" vs
  "Harin Thiran" failed the match. `canonicalOwnerName()` was written to fix that and is still
  in the client — it's now purely cosmetic consistency (same person, same spelling, same avatar)
  rather than the difference between editing and not.
- **The rules helpers are gone, not commented out** — `canEditItem()` and `changedKeys()` were
  deleted. A dead permission helper sitting in a rules file reads as active protection. If this
  is ever revisited, `git show 50db604:firestore.rules` has the full working version.
- **If protecting deletion is worth revisiting on its own**, the narrow version is to leave
  `update` open and gate only `delete` on ownership. That was offered and not taken. **Ask before
  re-tightening either.**

Still in place from that pass, and unaffected by the revert:

- **`admins()` / `isAdmin()`** — a hardcoded email list (`aiuser@`, `deane@`), kept identical to
  Flowboard's. Now used only by the `people` rule. Hardcoded rather than a `role` field on a
  document, because a role in a document is only as safe as the rule guarding that document.
- **`suggestions` delete is author-or-admin** (update stays open — replies are an embedded array,
  so replying *is* an update to someone else's doc). This is not editing access to content and
  was left tightened; say so if you want it reverted too.
- **`buckets`/`goals`** stay team-writable: shared configuration, no `owners`, no per-person work.
- **`writeErrorMessage(err, item, movePhrase)`** in the client still translates a Firestore
  `permission-denied` into a readable sentence. With posts and ideas reopened, nothing routine
  should reach it any more — it is now a safety net rather than an everyday path.

**A separate cause that is not permissions at all, and was mistaken for them:** the Hub blocks
Post/Idea editing on any viewport under 720px (`isMobileView()`, `setPostReadOnly`, the
"View only" badge — see DESIGN.md "Mobile / view-only mode"). That is deliberate and predates all
of the above. The tell is that **the Save button is absent entirely**, rather than present and
erroring.

## Stale-rules tripwire (`meta/rulesVersion`)

**Incident, 2026-09-08:** a teammate given full access via the normal sign-in link still got
`permission-denied` on ordinary post edits. Root cause was exactly the gap the parent
`Claude Projects/CLAUDE.md` warns about: rules are deployed separately from the site, that step
was missed after the 2026-09-02 ownership-scoped-editing revert, and the *live* Firestore rules
still enforced the old (reverted-in-code) restriction — silently, with no signal anywhere that
the deployed rules and this repo's `firestore.rules` had drifted apart.

Same investigation also turned up that `settings/{quickLink,goal}` had **no rule at all** —
enumerating collections deliberately means an omitted one is denied outright, and this one was
just missed when the quick-link feature shipped. Fixed alongside the tripwire below; same root
category of bug (a collection silently unprotected/misconfigured with no visible symptom until
someone hits it).

The fix isn't "redeploy once" — it's a live tripwire so the *next* drift is visible instead of
silent:

- `firestore.rules` gets a `meta/{docId}` match: team-readable, admin-write-only (so a
  compromised/buggy session can't rewrite it to mask a real drift).
- The client (`content-hub-firebase.html`) holds `EXPECTED_RULES_VERSION`, subscribes to
  `meta/rulesVersion` via `onSnapshot` in `startListeners()`, and banners
  (`#rulesVersionBanner`, red `.banner-danger` variant of `.onboarding-banner`) whenever the two
  disagree. No dismiss button — it's meant to clear itself live, for everyone, the moment an
  admin fixes the drift, so hiding it would hide a real still-open problem.
- Rules can't report their own content to a client, so there is **no way to derive this
  automatically** — three copies are kept in sync **by hand**, every time `firestore.rules`
  changes and gets deployed:
  1. The `RULES_VERSION: <date>` comment directly above the `meta/{docId}` block in
     `firestore.rules`.
  2. `EXPECTED_RULES_VERSION` in `content-hub-firebase.html` (then re-run
     `sync_from_scratchpad.py` so `index.html` picks it up).
  3. The actual `meta/rulesVersion` document's `version` field, set by an admin via the Firebase
     Console's Firestore **Data** tab (a console edit is a project-owner action and bypasses
     rules, same as any other manual doc edit there) — **this is the step that actually matters**;
     the other two are just what the client compares against.
- A missing `meta/rulesVersion` doc (freshly deployed, nobody's created it yet) is treated as
  "unknown," not "stale" — otherwise every fresh install would banner permanently for a mismatch
  nobody's actually observed. A `permission-denied` reading it (meaning the live rules predate
  the `meta/` match entirely) banners with "unknown (pre-dates this check)" rather than failing
  silently, since that too means the live rules are behind.

## Team roster (`people`)

One doc per teammate, **doc ID = their Firebase uid**, `{name, email, lastSeen}`, upserted by
`registerPresence(user)` on every sign-in (before `startListeners`, so a first-time signer-in is
in their own session's snapshot and pickable without a reload). Ported from Flowboard.

- **`teamRoster()`** — roster names UNION `getKnownNames()` (names already on posts/ideas). It
  replaced `getKnownNames()` as the source for the owner datalist, `parseMentions`,
  `enrichFeedbackText` and the @mention autocomplete. `getKnownNames()` still exists and is still
  the item-derived half; keeping it means an owner on a pre-roster post stays suggestable and
  mentionable even though no uid backs them.
- **`canonicalOwnerName(name)` is the load-bearing part in this app.** The owners field is a
  free-typed chip input, not a picker — a `<select>` doesn't fit a multi-value field — so instead
  of constraining input, the typed name is snapped to the roster's spelling when the two differ
  only by case or padding. It runs at `createOwnerChipInput`'s single `add()` commit point, so
  the chip shown is exactly the string that gets stored and later compared by `firestore.rules`.
  A name matching nothing falls through unchanged, so someone who hasn't signed in yet can still
  be added.
- The module-level list is **`teamPeople`, not `people`** — same shadowing trap as Flowboard.
- The `people` listener calls `renderOwnerNamesList()` on change (Flowboard's equivalent
  deliberately doesn't re-render, because its picker rebuilds on modal open; here the datalist is
  a persistent DOM node that nothing else refreshes when a new person signs in).

## Where to look for conventions

**`DESIGN.md` is the authoritative, actively-maintained reference** for this app's established
patterns — color tokens, spacing rhythm, icon conventions, reusable input widgets (chip input,
checklist, emoji picker, avatar coloring, status-picker dropdown), and a long list of specific
gotchas with their history (why they exist, what bug they fixed). Read it before adding new
fields/panels/icons, and read the relevant section before touching an area it documents — don't
duplicate that content here. `/ship-content-hub` updates it automatically when a change introduces
something non-obvious.

## Syntax check (guards against a one-typo blackout)

The app is one `<script type="module">` with no build step, so a single syntax error anywhere kills
the *entire* page, not just the feature that introduced it. The sibling Flowboard repo shipped
exactly that and its live board was dead for days -- nothing about the deployed HTML looks wrong, so
only an actual parse catches it. This repo is the more exposed of the two, because
`sync_from_scratchpad.py` does blind `str.replace()` into the HTML, and DESIGN.md records that
mechanism silently disabling the build-version guard once already.

```
node scripts/check-syntax.mjs
```

Checks **both** files by default -- `content-hub-firebase.html` (the source you edit) and
`index.html` (the file that actually deploys) -- and reports failures against each file's own line
numbers. Wired in two places, because neither alone is enough:

- **`.githooks/pre-push`** actually *prevents* the bad deploy, but git doesn't distribute hooks, so
  each clone needs `git config core.hooksPath .githooks` once. `--no-verify` bypasses it.
- **`.github/workflows/syntax-check.yml`** is the backstop for pushes where the hook wasn't enabled
  or was bypassed. It runs *after* the push, so it makes breakage loud rather than preventing it.

`.gitattributes` pins `.githooks/*` to LF: a CRLF checkout on Windows turns the shebang into
`/bin/sh` and silently disables the hook.

`/ship-content-hub` already ran `node --check` as part of its flow; this is the same check made
unskippable for anyone pushing directly.

## Testing

No unit tests. QA is a real Firebase emulator + Playwright run against the live app, done via the
`content-hub-qa` subagent — see that agent file for emulator setup (Java 21 + Playwright cached
under `.qa-tools/`, gitignored), sign-in flow, and this app's specific selector-scoping gotchas
(near-duplicate Post/Idea modal structures). Don't hand-write a test harness from scratch; delegate
to that agent, briefed with specifics of what changed.

## Automated UI/UX optimization reviews

A scheduled cloud routine (`https://claude.ai/code/routines/trig_01JanYSJWSBVFMmJp9TAekmt`, cron
`16 */5 * * *`, ~5 runs/day) reviews this repo unattended and opens a GitHub issue titled
"UI/UX Optimization Report — <date>" when it finds something concrete — categorized 🔴 Critical /
🟡 Refinement / 🔵 Feature Optimization, citing specific file/function. It skips creating a new
issue if one from the last 24h already exists, and opens nothing at all if it found nothing real.

**Push-notification fallback.** Every run of the sibling Flowboard repo's identical routine found
real things but silently failed to file them — `403 Resource not accessible by integration` on
every issue-creation attempt — worth knowing if this repo's issue tracker looks suspiciously empty
too. The prompt now tries issue creation first and, if it fails for any reason, pushes a mobile
notification with the same categorized report instead (prefixed with the error). **The root cause
isn't fixed by this** — it's a GitHub App permissions gap (`issues:write` missing on the
integration), fixable only from GitHub's UI. See "Scheduled cloud routines" in the parent
`Claude Projects/CLAUDE.md` for the exact fix — same gap, same GitHub App installation, for both
this repo's routine and Flowboard's.

**This is report-only by design, not the original "propose then wait for a reply" spec it was
adapted from.** A cron-fired cloud session runs once, unattended, and finishes — it cannot pause
mid-run and wait days for a human to reply "1, 3, 4" the way an interactive chat can. So the
automated run's tools are deliberately restricted to `Bash`/`Read`/`Grep`/`Glob` (no `Edit`/
`Write`), and its prompt forbids touching code entirely. **Implementing anything from a report is
always a separate, manually-triggered step**: open the GitHub issue, then ask Claude in a normal
interactive session (e.g. "implement items 1 and 3 from issue #N") — that request, in a real
conversation, is the actual approval gate. Manage/disable the schedule at
`https://claude.ai/code/routines`.
