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

## Keep-warm job

`.github/workflows/keep-warm.yml` pings Supabase twice a week so the free tier
never hits its ~7-day inactivity pause (which would make the next caregiver's
submission fail silently).

**Caveat:** GitHub disables scheduled workflows after **60 days with no commits
to this repo**. During pre-launch that's the window this matters most, so it
runs now. Once caregivers are actively submitting feedback, that organic traffic
keeps Supabase awake on its own. If the repo goes idle for 60 days, re-enable
the workflow from the **Actions** tab (or push any commit).

## Related

- Feedback feature plan & contracts: `PLAN-2026-07-22-in-app-feedback.md` in the
  CMS repo (`patient-family-resource-cms-nextjs`).
- The app submits feedback from Settings → Send App Feedback.
