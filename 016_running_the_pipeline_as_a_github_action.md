# 016 — Exercise 1: Running the observatory-bpns-crate Pipeline as a GitHub Action

How the EMO-BON pipeline was run automatically, end to end, using GitHub's own
infrastructure instead of a local machine.

---

## Step 1 — Create your own GitHub repository

Created a new personal repo (`Hamideh-B-H/bpns-workflow-practice`) to practice
in, rather than touching the real `observatory-bpns-crate` repo directly.
**Why:** a safe sandbox — anything that breaks only breaks your own copy.

## Step 2 — Add the observatory's workflow file

Placed `observatory-bpns-crate`'s own `workflow.yml` into the repo, at the
exact path GitHub requires: `.github/workflows/workflow.yml`. **Why this
specific path:** GitHub Actions only looks for workflow definitions in that
folder — a YAML file anywhere else is simply ignored.

## Step 3 — Understand what the file actually contains

Before changing anything, read through the file's structure: a **workflow**
(the whole recipe) is **triggered** by an event (`schedule` and
`workflow_dispatch` — meaning it runs on a timer and can also be started
manually), and runs one or more **jobs**, each made of **steps**, some of
which call reusable **actions** — the same `populate-action` →
`data-quality-control-action` → `semantic-uplifting-action` →
`rocrate-to-pages` chain studied in depth later.

## Step 4 — Make the three required changes

The template is shared across every observatory repo, but a few values are
specific to any one copy of it:

- Changed `DATA_QUALITY_CONTROL_ASSIGNEE` from `isanti` (the real BPNS
  assignee) to `Hamideh-B-H` — so any auto-generated issue would notify the
  right person, not someone else's.
- Enabled **read/write permissions** for GitHub Actions in the repo's
  settings — by default, Actions only get read access, and this workflow
  needs to write files back (commit results, push to `gh-pages`).
- Set up **GitHub Pages** in the repo's settings — this is what turns the
  final published output into an actual live web page.

## Step 5 — Commit the file properly

A snag worth remembering: the file didn't actually commit on the first
attempt. It had to be recreated, choosing **"Commit directly to main"**
rather than opening a pull request — since this is a solo practice repo, a PR
would just sit unreviewed.

## Step 6 — Trigger the workflow manually

Using `workflow_dispatch` — the trigger that adds a "Run workflow" button on
GitHub's Actions tab — the pipeline was started by hand, rather than waiting
for its scheduled time.

## Step 7 — Verify the full pipeline actually ran

Three concrete signs of success:

- **Populated data folders** appeared in the repo (the raw/filtered/
  transformed logsheets)
- A new **`gh-pages` branch** was created automatically
- A **live published page** appeared at
  `https://hamideh-b-h.github.io/bpns-workflow-practice` — the final
  linked-data output, generated entirely by GitHub's own servers

---

## The core idea

Every observatory repo (BPNS, RFormosa, and the rest) runs the *exact same*
`workflow.yml` template — only a handful of values differ (the Google Sheets
URLs, the assignee). This exercise proved the shared template could be taken,
adapted for a few observatory-specific values, and get the entire pipeline —
download, validate, convert to RDF, publish — running unattended on GitHub's
infrastructure.
