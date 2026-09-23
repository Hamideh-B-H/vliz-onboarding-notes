# 015 — Data Quality Control: Revised Issues Roadmap

A revised prioritization of the open issues on `data-quality-control-action`, updated
after a full tour of the repo (Dockerfile, .gitignore, Makefile, README, entrypoint.sh,
requirements.txt) and reading the `py-data-rules` framework README. This supersedes
the ordering in note `014` — the reasoning is more precise now because several
issues turned out to potentially involve a *second* repo, not just this one.

---

## What changed since the first pass (note 014)

1. **`generate_schema()` in `data_model.py` pulls its column rules from an external
   CSV** — `logsheet_schema_extended.csv`, hosted in the separate `observatory-profile`
   repo — not from anything inside `data-quality-control-action` itself. Several
   issues reference `BaseURI` / `DataTypeOut`, which are literally column names in
   that external CSV. Some issues may need changes in **two repos**, not one.

2. **Explicit vs. implicit rules** (from the `py-data-rules` README): explicit rules
   are hand-written functions in `rules.py` (e.g. `depth`, `organization_edmoid`).
   Implicit rules are automatic checks the `RuleEngine` runs based on the `schema`
   alone (data type, nullability, whitespace) — no function needed. The same issue
   title can mean two very different amounts of work depending on which mechanism
   is intended — worth clarifying before coding, not assuming.

3. **The `extensions.py` duplication is now well-evidenced** (though not 100%
   conclusive without git blame): five infrastructure files (`Dockerfile`,
   `.gitignore`, `Makefile`, `README.md`, `entrypoint.sh`) never reference it.

4. **New finding, not yet a tracked issue:** the README's example GitHub Actions
   workflow sets env vars `PAT` / `REPO` / `ASSIGNEE`, but `__main__.py` actually
   reads `GITHUB_TOKEN` / `GITHUB_REPOSITORY` / `DATA_QUALITY_CONTROL_ASSIGNEE`.
   Documentation and code have drifted out of sync — worth its own small issue/PR.

---

## Phase 0 — Verify before touching anything shared or documented

Not tracked issues — investigation/quick-fix tasks first:

- **`extensions.py` duplication** — confirm via `git blame`/history whether it's
  genuinely dead code before editing anything it shares with `data_model.py`
  (`read_emobon_csv`, `EMOBONList`, `EMOBONRange`).
- **README env var mismatch** (`PAT`/`REPO`/`ASSIGNEE` vs. the real variable names)
  — small, self-contained, easy first PR, and a good "found this myself" moment.

---

## Phase 1 — Simple, fully self-contained in this repo

- **#22** — cell values `'expected'` → `'NA'`
- **#21** — cell values `'empty'` → `'NA'`
  - Both touch only `read_emobon_csv()`. No external repo involved, no ambiguity
    about mechanism. Safest starting point.
- **#3** — QC for measurements needs to also catch `<` (currently only `>`)
  - Purely an explicit-rule change to `size_frac_low_gt` in `rules.py`.

---

## Phase 2 — Modify existing explicit rules (still this repo only)

- **#9** — QC check for depth to change
- **#8** — QC for size_frac columns
  - Both modify *existing* explicit rules already fully traced (`depth`,
    `size_frac_low_gt`). Contained to `rules.py`.

---

## Phase 3 — Ambiguous mechanism: clarify before coding

- **#23** — normalize `chem_administration` values to full URLs (BaseURI)
- **#20** — normalize ORCID values to full URLs (BaseURI)
  - Could mean either: (a) add an explicit `repair`-generating rule in `rules.py`,
    following the `organization_edmoid`/`envo` pattern — **or** (b) change
    `DataTypeOut` to `xsd:anyuri` with the right `BaseURI` in the *external*
    schema CSV, letting the implicit type system handle it. Not the same amount
    of work; only (a) stays inside this repo. **Ask on the issue or check with
    the team which approach is intended before starting.**
- **#19** — normalize boolean values based on `xsd:boolean`
  - Looks like it may be entirely a **schema CSV config change** in
    `observatory-profile` — `XSDBoolean()` already exists in `dtype_lookup`.
    Possibly no code change needed in this repo at all. Confirm first.
- **#17** — QC rule to standardize list-formatted columns in `logsheet_schema_extended`
- **#6** — dealing with lists (1/2 done, `change` label)
  - May need both a smarter `EMOBONList` *and* schema CSV changes. Also still
    tangled with the Phase 0 `extensions.py` duplication question.

---

## Phase 4 — Verify before building

- **#16** — transformation of ENVO terms to their respective URLs
  - Looks **already implemented** in `rules.py`'s `envo()` (used for
    `env_broad_biome`, `env_local`, `env_material`). Comment on the issue to
    confirm before writing any code — may just close outright.

---

## Phase 5 — Structural, multi-file

- **#5** — add QC and transformation and TTL rules for ARMS data (`enhancement`)
  - `HARD_LOGSHEET_URL` is read in `__main__.py` but never wired into the
    habitat-selection logic; no `HardRuleArray` exists in `rules.py`; unclear
    whether `data_model.py`/schema support a "hard" habitat at all. Real
    new-habitat work across multiple files. (Note: a `.env.gitkeep` commit
    message — "rename arms to hard" — confirms ARMS and "hard substrate" are
    the same habitat, just renamed at some point.)
- **#2** — some changes/transformations to QC pipeline (0/4 checklist)
  - Open and read the actual checklist before scoping — may be several smaller
    sub-tasks rather than one.

---

## Summary table

| Phase | Issues | Why here |
|---|---|---|
| 0 | *(extensions.py check, README fix)* | Small, self-contained, or a prerequisite |
| 1 | #22, #21, #3 | Simple, this-repo-only, safest first PRs |
| 2 | #9, #8 | Modify existing, already-understood explicit rules |
| 3 | #23, #20, #19, #17, #6 | Mechanism unclear — may span two repos, clarify first |
| 4 | #16 | Might already be done — check before coding |
| 5 | #5, #2 | Structural/unclear scope — last, or pair with someone senior |

*(Still not reviewed: the 8 closed issues, and the `observatory-profile` repo itself
— worth a look before starting anything in Phase 3.)*
