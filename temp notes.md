

The story so far, in one paragraph: Load all the tools this script needs — Python's own utilities, some outside packages, and its own sibling files. Set up a listener that checks whether you typed --dev when running the script. If you did, that means you're a human testing this locally — so make sure a .env file exists, and if it does, load every setting from it into the environment (overwriting any leftovers from before).

The story so far, in one paragraph: Load all the tools this script needs. Check whether you're running it as a human tester (--dev) — if so, load settings from a local .env file. Then, regardless of where those settings came from (.env locally, or GitHub Actions' own environment), read each individual setting by name: where the working files live, credentials for GitHub, the three logsheet URLs (used only to detect which habitats to process), the cutoff date, and who to assign the results-issue to.

The story of create_report(), in one paragraph: Take the raw dqc.csv violation log, and build a new table column by column: some columns copied directly (Diagnosis, Column, Row, FilePath, DataType), some translated from compact internal codes into human-readable labels (LogsheetType, LogsheetTab, Requirement), and one cleaned up so missing values are shown explicitly (Value, ExtendedDiagnosis). Then, keep only the rows that were never auto-repaired — those are the ones a human actually needs to act on — drop the now-unneeded Repair column, and save the result as report.csv.

The story of the habitat-selection block, in one paragraph: Define two lookup dictionaries translating short two-letter codes into real filenames, one for sediment, one for water. Then, based on which logsheet URLs were actually provided as settings, decide whether to process sediment only, water only, or both — calling filter_logsheets(...) for each relevant habitat, and building the appropriate alias2basename dictionary to match. If neither URL was provided, crash immediately with a clear error, since there's nothing to do.






#Mapping its concepts onto what we've already read

DataModel — an abstract representation of the data files on disk, consisting of a list of table aliases, each referring to a file path, schema, and read method (defaulting to pd.read_csv). This is exactly the structure generate_data_model() builds in data_model.py — remember the dictionary it returns, keyed by alias ("wm", "ss", etc.), each with a path, reader (that's read_emobon_csv!), and schema. 
github

Schema — schemas list the label, data type, nullability and whitespace policy for each column in the table. This matches generate_schema() exactly — remember schema.add_column(label=lct, data_type=data_type, nullable=(req == "optional"), trim="both"). 
github

Table objects — after applying the read method on the file path, tables are internally represented by Table objects, a subclass of the pandas DataFrame, accessed via the table_dict attribute on a DataModel. This is genuinely new and useful context: a Table isn't a plain DataFrame, it's pandas' DataFrame extended with something extra (we'd need to check py_data_rules itself to know exactly what) — worth remembering for later, since rules.py uses things like data_model.isna(...) and data_model[alias], which are custom behaviors layered on top of normal pandas. 
github

Rules — explicit vs. implicit — this is a genuinely new distinction, worth sitting with:
Some rules can be defined as functions in an external python file — these are explicit rules. Other rules are implicit, defined at runtime by the RuleEngine, e.g. datatype checks, missing values, whitespace trimming. 
github

This is an important clarification! Everything we read in rules.py (depth, source_mat_id, biomass, etc.) is an explicit rule — a rule someone deliberately wrote as a Python function. But things like "is this cell actually a valid date/number/boolean" (which we saw configured via the schema — XSDDate(), XSDInteger(), etc.) are implicit rules — the RuleEngine automatically checks these against the schema, without anyone writing a dedicated function for them. This explains something we might have glossed over: dqc.csv likely contains violations from both sources — explicit rule functions and automatic schema/type mismatches — even though rules.py only shows us the explicit half.

RuleEngine — the object that effectively performs the data validation and generates a violation report; a DataModel and a list of Rules are required to instantiate an engine. Exactly matches how we saw it constructed and called. 
github

Why this matters for your work going forward

This confirms something we speculated on earlier: data-quality-control-action really is a thin, domain-specific layer on top of a general-purpose library. If you ever need to understand how a rule actually gets evaluated cell-by-cell, or how schema-based implicit checks work, or what a Violation object really contains under the hood — that logic lives in this repo (py-data-rules), not in data-quality-control-action. Good to know which repo to go look in, depending on what kind of problem you're debugging.

Want me to add this PyDataRules framework mapping (especially the explicit/implicit rules distinction) to your notes, or would it be worth briefly opening py_data_rules' own source folder next time to see Violation, Rule, and RuleEngine for real?

Mapping its concepts onto what we've already read please keepthis an dremind me late

Confirming plans to keep and recall the mapping later.

Got it — I'll keep the PyDataRules mapping (DataModel/Table/Schema/explicit vs. implicit rules/RuleEngine) in mind, and bring it back in when it's relevant — especially once we open py_data_rules' own source, or when we get back to finishing data_model.py's generate_schema(), since that's where the schema/implicit-checks connection matters most.


