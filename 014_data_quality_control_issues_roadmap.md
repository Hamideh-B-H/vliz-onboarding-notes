# 014 — Data Quality Control: Open Issues Roadmap

A prioritized plan for working through the open issues on `data-quality-control-action`
(13 open, as of the review). Ordering is based on: resolving risk first, building
confidence with small/well-understood changes before complex ones, and verifying
before building anything that might already be done.

Source: repo issues list (`is:issue state:open`), cross-referenced against the code
walked through in note `013_data_quality_control_phase2.md`.

---

## Phase 0 — Resolve a code-duplication risk (do this before Phase 1)

**Not a tracked issue — a prerequisite investigation.**

`data_model.py` and `extensions.py` both define `read_emobon_csv`, `EMOBONList`,
and `EMOBONRange` — near-identical, slightly out of sync (`data_model.py`'s version
of `read_emobon_csv` has an extra `na_values=[""]` fix, matching its last commit
message: "resolve pandas FutureWarning in read function"). `data_model.py` does
**not** import from `.extensions` — it defines its own local copies. This suggests
`extensions.py`'s versions may be dead code, but this needs confirming (git
history/blame, or asking the team) **before** touching anything these two files
share — otherwise a fix could land in the copy that isn't actually used.

---

## Phase 1 — Quick, low-risk wins

Small, well-understood changes — good first PRs.

- **#22** — cell values `'expected'` should become `'NA'`
- **#21** — cell values `'empty'` should become `'NA'`
  - Both touch `read_emobon_csv()`, which already normalizes `"NA"`, `"ΝΑ"` (Greek),
    and `"nan"` → `""`. Do these two together — same function, same fix shape.
- **#3** — QC for measurements needs to also catch `<` (currently only `>` is checked)
  - Extends `size_frac_low_gt` in `rules.py`, a rule already fully traced in note 013.

---

## Phase 2 — Apply an existing pattern to new columns

- **#23** — normalize `chem_administration` values to full URLs (BaseURI)
- **#20** — normalize ORCID values to full ORCID URLs (BaseURI)
  - Both ask for the exact auto-repair pattern already implemented in
    `organization_edmoid` and `envo()` in `rules.py` (build a `repair` value that's
    a proper URI). Doing these together reinforces the pattern by repetition.

---

## Phase 3 — Verify before building

- **#16** — transformation of ENVO terms to their respective URLs
  - Appears **already implemented** — current `rules.py` has `env_broad_biome`,
    `env_local`, `env_material` all using the `envo()` auto-repair function.
    Before writing any code: comment on the issue to confirm whether it's stale,
    or check with whoever's tracking it. Possibly just closes outright.

---

## Phase 4 — List-handling group (depends on Phase 0)

- **#6** — dealing with lists (1/2 done, `change` label)
- **#17** — QC rule to standardize list-formatted columns in `logsheet_schema_extended`
  - Both center on `EMOBONList`, which currently only checks "no commas" — nothing
    about standardizing format. Since `EMOBONList` exists in both `data_model.py`
    and `extensions.py`, Phase 0's answer directly determines which copy to edit.

---

## Phase 5 — Needs `data_model.py` read in full first

- **#19** — update workflow to normalize boolean values based on `xsd:boolean`
- **#8** — QC for size_frac columns
- **#9** — QC check for depth to change
  - Likely touch `generate_schema()` in `data_model.py` (the `dtype_lookup` table,
    and how habitat/sheet/column rules get assembled) — a part of the file not yet
    fully read through. Finish that file first, then scope these three properly.

---

## Phase 6 — Bigger, structural work (last, or pair with someone senior)

- **#5** — add QC and transformation and TTL rules for ARMS data (`enhancement` label)
  - Matches a gap spotted directly in the code: `HARD_LOGSHEET_URL` is read in
    `__main__.py` but never wired into the habitat-selection `if/elif/else` logic;
    no `HardRuleArray` exists in `rules.py`; unclear whether `data_model.py`
    supports a "hard" habitat at all. Real new-habitat work, touching multiple files.
- **#2** — some changes/transformations to QC pipeline (0/4 checklist, `change/update`)
  - Open and read the actual checklist before scoping — may turn out to be several
    smaller sub-tasks rather than one big one.

---

## Summary table

| Phase | Issues | Why here |
|---|---|---|
| 0 | *(investigation only)* | Prevents fixing the wrong copy of shared code |
| 1 | #22, #21, #3 | Smallest, most understood, safest first PRs |
| 2 | #23, #20 | Reuse a pattern already fully traced |
| 3 | #16 | Might already be done — check before coding |
| 4 | #6, #17 | Same underlying class, depends on Phase 0 |
| 5 | #19, #8, #9 | Needs finishing `data_model.py` first |
| 6 | #5, #2 | Structural/unclear scope — last, or pair with someone senior |

*(Not yet reviewed against the code: any closed issues (8 closed) — worth a quick
skim at some point in case a "closed" issue's fix touched code relevant to one of
the open ones above.)*
