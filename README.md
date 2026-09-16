# Job Radar

A personal remote-job aggregator: scrapes ~15 job boards and per-company
ATS APIs every few hours, filters out noise, scores each new listing
against a private candidate profile with an LLM, and publishes the
results as a static, filterable site via GitHub Pages — no server, no
database, the whole pipeline runs on a GitHub Actions cron schedule.

## How it works

Each run (`run.py`) does, in order:

1. **Re-validate stored links** — re-checks a rotating batch of already-seen
   listings and hides any that now 404/410 (closed postings).
2. **Fetch every source** — see [Sources](#sources) below. A per-source
   "rate limited" flag skips sources with a published quota outside a
   small allowlisted set of hours.
3. **Filter** — a title-keyword allowlist/blocklist (editable from the
   site itself, see below) narrows results before anything expensive
   happens; a second pass drops non-remote and non-Europe/Ukraine
   listings once the full description is available.
4. **Score with AI** — new listings are batched (5 at a time) and sent to
   Groq (`llama-3.1-8b-instant`) along with the candidate profile, which
   returns a 1–10 fit score, region/remote classification, salary,
   pros/cons, and a few company/domain tags.
5. **Persist** — results are stored in `data/jobs.json` (keyed by URL, so
   re-runs can't duplicate a listing) and pruned after `RETENTION_DAYS`.
6. **Render & publish** — a static `site/index.html` is generated and
   deployed to GitHub Pages; `data/jobs.json` and live progress
   (`data/status.json`) are committed back to `main`.

A capped `MAX_NEW_JOBS_PER_RUN` and a bounded per-batch retry/backoff on
Groq 429s keep any single run inside GitHub Actions' job time limit even
during a listings spike — anything over the cap is simply picked up as
"new" again on the next scheduled run instead of being lost.

## Sources

Aggregator/board APIs and RSS feeds: **Djinni, DOU, Work.ua** (Ukraine,
HTML-scraped), **Remotive, Arbeitnow, Jobicy, Adzuna, Working Nomads,
Himalayas, Jooble**.

Curated per-company ATS boards (no auth needed, each company verified live
to actually return postings): **Greenhouse, Lever, Ashby, SmartRecruiters,
Workable** — dozens of companies per ATS, hand-picked for a track record of
hiring in the target roles.

Every source funnels through the same title-keyword filter, remote check,
and region check before a listing is worth spending an AI call on.

## The site

The published page isn't just a list — it's a small write-capable app
layered on a static host:

- **Filters** — score threshold, region, and a multi-select source picker,
  all client-side, no reload.
- **Keywords panel** — add/remove title keywords that gate what gets
  scraped, directly from the page.
- **🚫 "Блокувати схожі"** on any card — blacklists that title pattern
  going forward.
- **Live run status** — while a scheduled run is still in flight, the page
  shows its current stage (fetching / scoring N/M) by polling
  `data/status.json`, which the pipeline commits and pushes immediately at
  each stage rather than waiting for the run to finish.

All three write actions above go through the GitHub Contents/Actions API
directly from the browser, authenticated with a personal access token the
user pastes in once (kept in `localStorage`, never sent anywhere but
GitHub) — there's no backend to hold it.

## Project layout

```
run.py                  Pipeline entry point (see "How it works")
job_search/
  sources.py             One fetch_X() per job source
  config.py               Keyword/exclusion gates, source registry, tunables
  scoring.py               Groq batch-scoring calls
  profile.py                Loads the private candidate profile
  keywords.py                 Title-keyword allowlist persistence
  exclusions.py                 Title-keyword blocklist persistence
  storage.py                     data/jobs.json read/write + retention pruning
  status.py                       Live progress reporting (data/status.json)
  render.py                        Generates site/index.html
data/
  jobs.json               Scored/tracked listings (committed by the workflow)
  keywords.json             Title-keyword allowlist (editable from the site)
  exclude_keywords.json       Title-keyword blocklist (editable from the site)
  status.json                   Current run's live progress
.github/workflows/
  scrape-and-publish.yml  Cron schedule + build + commit + Pages deploy
```

## Configuration

Nothing that identifies the candidate lives in this repo — `config.py`,
the keyword/exclusion lists, and every source fetcher are safe to be
public. The one piece of private data is the candidate profile, supplied
via a `CANDIDATE_PROFILE` secret (JSON):

```json
{
  "target": ["Technical Account Manager", "Solutions Consultant", "..."],
  "not_interested": ["pure sales roles", "on-site only", "..."],
  "key_experience": "Free-text summary of relevant background.",
  "fit_criteria": "What makes a listing a good vs. bad match."
}
```

### GitHub Actions secrets

| Secret               | Required | Purpose                                   |
|-----------------------|:--------:|--------------------------------------------|
| `GROQ_API_KEY`        | yes      | AI scoring (Groq, free tier)               |
| `CANDIDATE_PROFILE`   | yes      | The JSON profile above                     |
| `ADZUNA_APP_ID`       | optional | Enables the Adzuna source                  |
| `ADZUNA_APP_KEY`      | optional | Enables the Adzuna source                  |
| `JOOBLE`              | optional | Enables the Jooble source                  |

Without the optional secrets, those specific sources are skipped — every
other source still runs.

## Running locally

```bash
pip install -r requirements.txt

# instead of the CANDIDATE_PROFILE secret, drop the same JSON shape at:
# data/candidate_profile.local.json (gitignored, never committed)

export GROQ_API_KEY=...
python3 run.py
```

Output lands in `site/` (gitignored locally) and `data/jobs.json`.

## Schedule

Runs four times a night via `.github/workflows/scrape-and-publish.yml`
(00/02/04/06 UTC) so a backlog has multiple chances to fully drain before
a morning check, rather than trickling in during the day. Can also be
triggered manually via `workflow_dispatch`.
