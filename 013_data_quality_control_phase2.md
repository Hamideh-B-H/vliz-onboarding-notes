# 013 — Data Quality Control (Phase 2): Code Walkthrough

Phase 2 of the EMO-BON pipeline. Takes the raw CSVs produced by `populate-action`
(Phase 1) and validates/repairs them before they move on to semantic uplifting
(Phase 3). This note documents the `data-quality-control-action` repo, read file
by file, in the order we studied it.

---

## 1. What Phase 2 is for

- **Input:** the raw logsheet CSVs from Phase 1 (`logsheets/raw`)
- **Output:**
  1. `logsheets/filtered` — raw data with too-recent rows blanked out
  2. `logsheets/transformed` — filtered data with known repairs applied
  3. `data-quality-control/dqc.csv`, `report.csv`, `logfile` — records of what was found/fixed

---

## 2. The three output files, and how they relate

| File | Contains | Written by |
|---|---|---|
| `dqc.csv` | **Every** violation found — both auto-repaired and not | `RuleEngine(...).execute(...)` |
| `report.csv` | Subset of `dqc.csv` — only violations **without** a repair (need a human) | `create_report()` |
| `logfile` | Not about the data at all — errors/warnings from the **script's own execution** | `logging` module |

Relationship:
```
dqc.csv  =  ALL violations (fixed + unfixed)
              ├── has a `repair` value   → also applied in logsheets/transformed
              └── no `repair` value      → this subset = report.csv
```
`logfile` is independent — a script can run with zero errors and still find plenty
of data violations. The two signals don't imply each other.

---

## 3. The threshold date — what it actually controls

`DATA_QUALITY_CONTROL_THRESHOLD_DATE` is **not** "only process new data since last
run." The script has no memory between runs — every run re-checks *everything*
collected before the threshold, including data checked many times before.

**Why:** two reasons.
1. Old Google Sheet entries can be corrected retroactively — need to be re-validated.
2. Validation rules themselves can change — old data needs re-checking against new rules.

**What the threshold is actually for:** a buffer against *incomplete* data. A sample
in `sampling` might be logged immediately, but its lab results in `measured` can lag
by days/weeks. The threshold means "only validate samples old enough that their data
should be complete by now" — avoids false "missing value" violations on data that's
simply not filled in *yet*.

---

## 4. `__main__.py` — the orchestrator, section by section

### 4.1 Imports
Three groups: Python built-ins (`argparse`, `logging`, `os`, `pathlib`, `datetime`),
third-party packages from `requirements.txt` (`pandas`, `numpy`, `python-dotenv`,
`PyGithub`, `py_data_rules`), and local sibling files using relative imports
(`.data_model`, `.pipeline`, `.rules` — the leading dot means "same package as me").

### 4.2 The `--dev` flag
```python
parser = argparse.ArgumentParser()
parser.add_argument('--dev', action='store_true')
args = parser.parse_args()
```
- This script runs in two different places: **GitHub Actions** (automatic, settings
  come from the workflow's `env:` section) and **locally on a laptop** (manual,
  settings must come from somewhere else).
- `--dev` is a simple on/off flag typed after the command. `action='store_true'`
  means "no value needed, just present or absent" → `args.dev` is `True` or `False`.
- **Without `--dev`:** expects env vars already set (GitHub Actions does this).
- **With `--dev`:** loads a local `.env` file instead:
  ```python
  if args.dev:
      assert Path(".env").exists(), ".env file is missing"
      load_dotenv(override=True)
  ```
  `override=True` matters — it makes `.env` win over any leftover `$env:` values
  set manually in a previous PowerShell session (relevant to how Phase 1 was run).

### 4.3 Why `-m action`, not `python action/__main__.py`
- `action` is the Python package's **folder name** — coincidentally the same word
  used for "GitHub Action" (the automated task), but a different concept.
- `-m` tells Python "treat this as a package." This matters because the code uses
  **relative imports** (`from .data_model import ...`), which only work if Python
  knows it's running inside a package. Running the file directly would crash with
  `ImportError: attempted relative import with no known parent package`.

### 4.4 Reading the 8 settings
```python
GITHUB_WORKSPACE, GITHUB_TOKEN, GITHUB_REPOSITORY,
WATER_LOGSHEET_URL, SEDIMENT_LOGSHEET_URL, HARD_LOGSHEET_URL,
DATA_QUALITY_CONTROL_THRESHOLD_DATE, DATA_QUALITY_CONTROL_ASSIGNEE
```
All read via `os.getenv("NAME")` — returns the value if set, `None` if not.
- `GITHUB_WORKSPACE` is **required** — `Path(None)` crashes, so this must be set.
- The three `*_LOGSHEET_URL` vars aren't used to download anything here — only
  checked for truthy/falsy, to decide which habitat(s) to process.
- `HARD_LOGSHEET_URL` is read but **never actually used** later in the habitat logic.

### 4.5 Folder setup + validation + logging
```python
LOGSHEETS_PATH = GITHUB_WORKSPACE / "logsheets/raw"              # input, no mkdir
LOGSHEETS_FILTERED_PATH = GITHUB_WORKSPACE / "logsheets/filtered"
LOGSHEETS_FILTERED_PATH.mkdir(parents=True, exist_ok=True)
LOGSHEETS_TRANSFORMED_PATH = GITHUB_WORKSPACE / "logsheets/transformed"
LOGSHEETS_TRANSFORMED_PATH.mkdir(parents=True, exist_ok=True)
DQC_PATH = GITHUB_WORKSPACE / "data-quality-control"
DQC_PATH.mkdir(parents=True, exist_ok=True)
```
- `/` on a `Path` object joins paths (not division) — OS-independent.
- `.mkdir(parents=True, exist_ok=True)`: creates missing parent folders too, and
  doesn't crash if the folder already exists (important — script gets run more than once).
- `DQC_PATH` is a **folder**, not a file. `logfile`, `dqc.csv`, `report.csv` all
  live *inside* it (`DQC_PATH / "logfile"`, `DQC_PATH / "dqc.csv"`, etc).

```python
assert XSDDate().match(DATA_QUALITY_CONTROL_THRESHOLD_DATE), msg
```
`assert condition, message` = "check this; if false, crash immediately with this
message." Fail-fast pattern — catches a malformed date here, rather than letting it
cause a confusing bug deep inside the filtering logic later.

```python
logging.basicConfig(filename=DQC_PATH / "logfile", filemode="w", level=logging.INFO)
```
Sets up the logging channel **before** any work happens — like turning on a camera
before opening the shop. Must exist before the first possible `logging.info(...)`
call anywhere later in the script. `filemode="w"` = fresh file each run.
This is different from `dqc.csv`, which isn't written incrementally — it's built
entirely in memory and written out in one shot, much later, by `RuleEngine.execute()`.

### 4.6 `filter_logsheets(habitat)`
Three sub-steps, one per logsheet type:

1. **`sampling`:** read the CSV → blank out any row whose `collection_date` is on/after
   the threshold → save to `filtered`.
   ```python
   df_sampling.loc[pd.to_datetime(df_sampling["collection_date"]) >= THRESHOLD] = ""
   ```
   `.loc[condition] = value` selects rows matching a condition and overwrites them —
   here, the whole row becomes empty strings (row stays, just emptied — keeps row
   positions aligned with the original file).

2. **`measured`:** blank out any row whose `source_mat_id` no longer appears in the
   *already-filtered* `sampling` table:
   ```python
   df_measured.loc[~df_measured["source_mat_id"].isin(df_sampling["source_mat_id"])] = ""
   ```
   `.isin(...)` = "is this value found in that other list?" `~` = NOT (flips
   True/False). This is a **consistency check**: if a sample got filtered out of
   `sampling`, its measurements shouldn't survive in `measured` either — avoids
   orphaned measurements pointing at a "non-existent" sample.

3. **`observatory`:** read and save through **unchanged** — no date/ID to filter by,
   since it describes the fixed station, not a sample or event.

**Data model reminder:** `sampling` = when/where a sample was collected;
`measured` = lab results for that sample; `observatory` = the fixed station itself.
`source_mat_id` is the shared key (like a foreign key) linking `sampling` ↔ `measured`.

### 4.7 `create_report(input_path, output_path)`
Builds `report.csv` from `dqc.csv`, column by column, into a new empty DataFrame:
- **Direct copies:** `Diagnosis`, `Column`, `Row`, `FilePath`, `DataType`
- **Derived via `np.select([conditions], [choices], default=...)`** (like a
  vectorized if/elif/else applied to a whole column at once):
  - `LogsheetType`: first letter of the `table` code (`s`→sediment, `w`→water)
  - `LogsheetTab`: second letter (`m`→measured, `o`→observatory, `s`→sampling)
  - `Requirement`: `nullable` True/False → `"optional"`/`"mandatory"`
- **Derived via `np.where(condition, if_true, if_false)`** (simpler, two-outcome
  version of `np.select`):
  - `Value`: missing → literal text `"<empty>"` (so a blank isn't mistaken for a
    rendering glitch in the CSV)
  - `ExtendedDiagnosis`: missing → literal `"\\"` — note the code actually assigns
    this column *twice* in a row; the first assignment is immediately overwritten
    and has zero effect (dead code — harmless but redundant, worth knowing how to spot)
- **The actual fixed→unfixed filter:**
  ```python
  df_report = df_report[df_report["Repair"].isna()]   # keep only unrepaired rows
  df_report = df_report.drop(columns=["Repair"])       # then drop the now-unneeded column
  ```
  This is the exact line where `dqc.csv`'s full list becomes `report.csv`'s
  "still needs a human" subset.

### 4.8 `create_issue()`
```python
repo = Github(GITHUB_TOKEN).get_repo(GITHUB_REPOSITORY)
repo.create_issue(title=..., body=..., assignee=...)
```
- Logs into GitHub via `PyGithub`, gets a handle on the repo, opens a real issue.
- Title uses today's date (`date.today()`); body uses Markdown links `[text](url)`
  to `logfile` and `report.csv`, plus guidance on changing the threshold date.
- **This is the only line in the whole script that reaches the internet / has a
  real-world side effect.** Everything else only touches local folders.
- Only called when **not** `--dev`:
  ```python
  if not args.dev:
      create_issue()
  ```
  → running locally with `--dev` is guaranteed safe — no real issue ever gets created.

### 4.9 Habitat selection (`if __name__ == "__main__":` block)
- `if __name__ == "__main__":` — Python auto-sets `__name__` to `"__main__"` only
  when a file is run directly (not when imported). Keeps function *definitions*
  reusable/importable while the "do the actual work" code only fires when the file
  is deliberately executed. Everything above this line in the file is definitions
  only — nothing runs yet.
- Two dictionaries map short codes to real filenames, e.g.
  `{"sm": "sediment_measured", "so": "sediment_observatory", "ss": "sediment_sampling"}`
  (naming convention: `alias2basename` = "alias **to** basename", `2` = "to").
- `if/elif/elif/else` picks sediment / water / both, based on which URL env vars
  are truthy (non-`None`, non-empty):
  - Both set → `habitat = "all"`, merge both dictionaries with `{**a, **b}`
    (unpacks and combines dict contents), call `filter_logsheets()` for each.
  - Only one set → that habitat only.
  - Neither set → `raise AssertionError("invalid logsheet_url configuration")`.
- **`HARD_LOGSHEET_URL` is never checked here** — hard-substrate habitat isn't
  wired into this branching logic at all (consistent with the missing
  `hard_measured.csv` seen in Phase 1).

### 4.10 Running the actual checks
```python
data_model = generate_data_model(logsheets_path=LOGSHEETS_FILTERED_PATH, alias2basename=alias2basename)
rules = generate_rules(habitat=habitat)
RuleEngine(data_model=data_model, rules=rules).execute(report_path=DQC_PATH / "dqc.csv")
```
- Note: builds the data model from `LOGSHEETS_FILTERED_PATH`, **not raw** — filtering
  (4.6) always happens before any quality checking.
- `RuleEngine` is a **class**, not a function (capitalized name is the convention
  signal). `RuleEngine(...)` creates a configured **object**; `.execute(...)` is a
  **method** call on that object that actually does the work. Class = blueprint,
  object = one instance built from it, method = an action the object can perform,
  holding onto its own data (`data_model`, `rules`) internally.
- Composes three separate pieces — data model / rules / engine — a common pattern
  for separating "what you're checking" from "what to check" from "how checking works."
- **This line is where `dqc.csv` is actually created.**

### 4.11 Final steps
```python
create_report(input_path=DQC_PATH / "dqc.csv", output_path=DQC_PATH / "report.csv")
if not args.dev:
    create_issue()
Pipeline(
    input_path=LOGSHEETS_FILTERED_PATH,
    output_path=LOGSHEETS_TRANSFORMED_PATH,
    dqc_path=DQC_PATH / "dqc.csv",
    alias2basename=alias2basename,
).run()
```
Turn `dqc.csv` into `report.csv` → open a GitHub issue if in production → run the
`Pipeline`, which applies whatever automatic repairs are possible and produces the
final `logsheets/transformed` output.

---

## 5. `pipeline.py` — the `Pipeline` class

### 5.1 Class basics (new concept today)
- **Class** = a blueprint for creating "things" (objects) that hold their own data
  and have their own actions (methods). Analogy: a car blueprint → many actual cars,
  each with its own independent fuel level.
- **`self`** = "this particular object" — how a method refers to the specific
  object it was called on, so two different objects don't share/overwrite each
  other's data.
- **`__init__`** = special method that runs automatically once, the moment an
  object is created — the "factory setup" step.

### 5.2 `__init__` — configuration only, no work yet
```python
class Pipeline:
    def __init__(self, input_path, output_path, dqc_path, alias2basename):
        self.input_path = input_path
        self.output_path = output_path
        self.dqc_path = dqc_path
        self.alias2basename = alias2basename
```
Just stores the four settings onto the object. Nothing executes yet — same as
filling in a car's color/starting fuel before it's driven.

### 5.3 `.run()` — reading input
```python
def run(self):
    self.dqc = pd.read_csv(self.dqc_path)
    self.dfs = {}
    for alias, base_name in self.alias2basename.items():
        self.dfs[alias] = read_emobon_csv(self.input_path / f"{base_name}.csv")
```
Loads `dqc.csv` and **every filtered logsheet** into memory at once, storing them
in a dictionary keyed by short code (`self.dfs["wm"]`, `self.dfs["ss"]`, etc.) — all
attached to `self` so other methods on this object (like `quick_fix`) can use them.

### 5.4 `quick_fix()` — applying repairs, cell by cell
```python
def quick_fix(self):
    df_repair = self.dqc[~self.dqc["repair"].isna()]
    for _, row in df_repair.iterrows():
        self.dfs[row["table"]].at[row["row"] - 1, row["column"]] = row["repair"]
```
- Same `~...isna()` pattern as `create_report` — keeps only violations that **do**
  have a `repair` value.
- Loops through each repairable violation, and for each one: looks up the right
  table (via `row["table"]`, the short code), pinpoints the exact cell
  (`.at[row_number, column_name]`), and overwrites it with the repair value.
  (`- 1` adjusts from spreadsheet-style row counting to pandas' 0-indexed rows.)
- In plain English: *"for every violation with a known fix, go find that exact
  cell and correct it."*

### 5.5 Still unimplemented
```python
# create missing columns
...
# rename existing columns
...
# drop excess columns
...
```
Literal placeholders — not yet written. Real signal this part of the codebase is
still under active development; likely relevant to work I'll be doing.

### 5.6 Writing output
```python
for alias, df in self.dfs.items():
    base_name = self.alias2basename[alias]
    df.to_csv(self.output_path / f"{base_name}.csv", index=False)
```
Saves every (now quick-fixed) table out to `logsheets/transformed`.

---

## 6. `rules.py` — the actual quality checks (habitat = "water", per our example)

| Rule | What it checks | Auto-repairs? |
|---|---|---|
| `biomass` | Text matches a specific numeric/scientific-notation pattern | No |
| `chem_administration` | Matches `CHEBI:##### YYYY-MM-DD` pattern | No |
| `ship_date_after_samp_store_date` | Ship date must come after storage date | No |
| `ship_date_seq_after_ship_date` | Sequencing ship date after general ship date | No |
| `arr_date_hq_after_ship_date` | HQ arrival after ship date | No |
| `arr_date_seq_after_arr_date_hq` | Sequencing arrival after HQ arrival | No |
| `arr_date_seq_after_ship_date_seq` | Sequencing arrival after sequencing ship date | No |
| `depth` | Sample depth ≤ observatory's total water column depth | No |
| `source_mat_id` | Sample ID must match a strict naming pattern (built from observatory ID + date + sample type + replicate) | No |
| `tax_id_versus_scientific_name` | Cross-checks `scientific_name` against **NCBI Taxonomy** (live web lookup) | No |
| `contact_orcid` / `other_person_orcid` / `sampl_person_orcid` / `store_person_orcid` | Cross-checks name against the **ORCID registry** (live web lookup), for 4 different person/role columns | No |
| `organization_edmoid` | Must be a list of integers → repaired into proper EDMO URIs | **Yes** |
| `env_broad_biome` / `env_local` / `env_material` | ENVO ontology terms must be valid → repaired into proper URIs | **Yes** |
| `size_frac_low_gt` | Must not contain a `>` character | No |
| `comm_samp` (sediment only) | Must be one of `micro`/`meio`/`macro`/`blank` | No |

**Water-specific rules (`WaterRuleArray`):** not yet written — placeholder `...`.

**Important operational note:** `tax_id_versus_scientific_name` and the four
`*_orcid` checks make **live calls to external services** (NCBI, ORCID) while the
script runs — meaning full quality-control isn't purely offline/local; these checks
can be slow, or fail, if those services are unreachable or rate-limit requests.

**Not yet studied in code detail:** the internals of `rules.py` (nested functions,
closures, the `rf.` rule factory helpers, `getmembers`/reflection at the bottom
that auto-collects all the rule functions into a list) — for a future session.
Also not yet opened: `data_model.py`.

---

## 7. New Python/programming concepts learned today

- **`argparse` flags** (`--dev`) — reading optional command-line switches
- **`-m` + relative imports** — why packages need `python -m package_name`
- **`.env` files + `python-dotenv`** — loading local settings, `override=True`
- **`assert condition, message`** — fail-fast validation
- **f-strings** (`f"text {variable}"`) — string interpolation
- **`pathlib.Path`** and the `/` join operator — OS-independent paths
- **`.mkdir(parents=True, exist_ok=True)`** — safe folder creation
- **pandas `dtype=object`, `keep_default_na=False`** — reading CSVs as raw text, no auto-conversion
- **`.loc[condition] = value`** — bulk row selection + overwrite
- **`.isin(...)` and `~`** — membership check and logical NOT
- **`np.select([...], [...], default=...)`** and **`np.where(cond, a, b)`** — vectorized if/elif/else and if/else on whole columns
- **Dead code** — an assignment immediately overwritten before use, no effect
- **`{**dict1, **dict2}`** — merging dictionaries via unpacking
- **Classes, `self`, `__init__`, methods** — blueprints vs. objects vs. actions
- **`if __name__ == "__main__":`** — separating definitions from "actually run this"

---

## 8. Still to cover

- `rules.py` internals: nested functions/closures, `rf.` rule factory, `getmembers`
- `data_model.py` — never opened yet
- Planning and executing the actual local run (`.env` values, which threshold date to pick for testing)

# my final sum
 the whole cycle, step by step:

__main__.py reads settings, sets up folders, and figures out which habitat(s) to process.
filter_logsheets() (inside __main__.py) blanks out rows collected too recently (past the threshold date), producing the filtered logsheets.
data_model.py builds a DataModel from those filtered logsheets — this tells the system what each column is supposed to contain (its type, whether it's required).
rules.py builds the list of Rules to check — both data_model.py and rules.py rely on classes (DataModel, Rule) borrowed from the external py-data-rules library.
RuleEngine (also from py-data-rules) takes that data model and those rules, checks every cell, finds errors, and writes every violation it finds into dqc.csv — but it doesn't fix anything itself, only diagnoses.
create_report() (inside __main__.py) reads dqc.csv and filters it down to report.csv — just the violations that have no known fix, for a human to look at.
Pipeline (defined locally in this repo, not from the external library) comes in next — it reads dqc.csv (the diagnosis) and the filtered data files again.
For every violation that does have a known repair value, Pipeline overwrites that exact cell with the corrected value — this is the quick_fix() method.
The result gets saved to logsheets/transformed — the final, repaired data, ready for the next pipeline phase (semantic uplifting).
