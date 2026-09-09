# Patient & Family Resource — Feedback Dashboard

A single-page dashboard for the MGBfC care team to review in-app feedback from
the caregiver app, plus a scheduled job that keeps the free-tier Supabase
backend from pausing.

## Dashboard

`index.html` is a self-contained page (no build step, no dependencies) served
via GitHub Pages. It reads anonymous Likert ratings from Supabase and shows:

- response count, latest submission, and active question-set version
- average rating per question
- responses over time
- a table of recent responses
- filters by environment (production vs. dev/test) and by hospital unit

**Live URL:** _(GitHub Pages → Settings → Pages; enabled on the `main` branch root)_

### Signing in

The page requires the shared team login. The dashboard's embedded Supabase key
is **public by design** — row-level security lets it only *insert* feedback,
never read it. Reading requires the authenticated team account, so the data is
safe even though the page and key are public.

### Where the questions come from

Question labels are read live from the published `feedback_questions.json` on
the app's content mirror, so the dashboard always matches whatever survey is
live. The `question_set_version` column separates answers to different survey
versions.

## Backend monitor (`keep-warm.yml`)

This job was written as a keep-alive — the free Supabase tier pauses a project
after ~7 days idle, and the theory was that a scheduled REST read would count as
activity and reset that clock.

**It does not.** Three complete cycles, with the ping green throughout:

| | last human dashboard activity | pings green | paused by | days down |
|---|---|---|---|---|
| cycle 1 | 2026-07-23 | 07-27, 07-30 | 2026-08-03 | ~3 (restored 08-06) |
| cycle 2 | 2026-08-06 (restore) | 08-10, 08-13 | 2026-08-17 | ~5 (restored ~08-22) |
| cycle 3 | ~2026-08-22 (restore) | 08-24 | 2026-08-27 | ~12 (restored ~09-08) |

Every time, the project paused ~7 days after the last *human* dashboard session,
with successful pings in between changing nothing. An anonymous read that
row-level security blocks does not register with Supabase's inactivity tracker.

Note the last column. Nobody is watching the Actions tab, so each outage has
lasted longer than the one before — cycle 3 ran roughly twelve days, four
consecutive failed runs, entirely unnoticed. Detection, not prevention, is what
this workflow can actually improve.

So the workflow is now a **monitor, not a fix**: it runs daily, retries three
times before alerting (a pause lasts days and fails all three; a transient
runner DNS blip does not), and fails loudly with a diagnosis — paused vs. key
rotated vs. network. That caps time-to-detection at about a day. It does not
keep the backend up.

A pause shows up two different ways depending on who is asking. Every failed run
from a GitHub runner so far has been NXDOMAIN (`curl` exit 6); a probe from a
laptop minutes later saw a Cloudflare **521** instead, because the DNS record can
outlive the origin. The workflow handles both — don't assume either one is *the*
signature.

### Fixing it for real

Two options, neither of them this workflow:

1. **Upgrade to a paid plan.** Pausing is a free-tier behavior; a paid project
   does not pause. This is the only approach Supabase actually supports.
2. **A heartbeat that writes.** Add a throwaway `keepalive` table with an
   anon-INSERT policy and have this job insert a row. A real write does real
   database work, which a blocked read may not. Unproven — if you take this
   route, keep the monitor running to catch it failing anyway, and do **not**
   write heartbeat rows into `feedback_responses`, which would corrupt the
   pilot's response counts.

### When it fires

A failing run means caregiver submissions are not landing. The app queues them
on-device, but only retries when someone reopens the feedback screen — and the
UI tells a user who already submitted that they're done, so most queued
submissions will never retry. Treat a pause as active data loss and restore
promptly.

To restore: sign in at supabase.com and un-pause the project. **The owning
account is a personal login, separate from the `mgbfcapp@mgb.harvard.edu` team
account used to read this dashboard** — that team login has no project
administration access and cannot restore anything. The owning address is
recorded in the feedback plan doc in the CMS repo; it is deliberately not
published here.

Preserve the project ref (`qaqqpnjerjjyuygjqwpf`) across any restore — that
hostname is compiled into shipped app builds, so a new ref requires a new
app release.

**Caveat:** GitHub disables scheduled workflows after **60 days with no commits
to this repo**. If that happens, re-enable it from the **Actions** tab.

## Related

- Feedback feature plan & contracts: `PLAN-2026-07-22-in-app-feedback.md` in the
  CMS repo (`patient-family-resource-cms-nextjs`).
- The app submits feedback from Settings → Send App Feedback.
