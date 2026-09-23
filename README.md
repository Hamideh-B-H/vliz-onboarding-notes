
# vliz-onboarding-notes

Personal reference notes from onboarding at VLIZ / EMO-BON. Organized by theme, not by date, so anything is easy to find later.

## Files

| File | Covers |
| --- | --- |
| `002.rdf-rdfs-sosa.md` | RDF triples, RDFS, SOSA ontology, Turtle syntax |
| `003.semantic-mapping.md` | Why/how tabular data becomes RDF |
| `004.data-pipeline-stages.md` | Raw → Filtered → Transformed, mapped to the 4-phase pipeline |
| `005.vocabularies.md` | Darwin Core, Dublin Core, SOSA — who covers what |
| `006.emo-bon-repo-catalog.md` | All 55 repos in the emo-bon GitHub org |
| `007.emo-bon-actions-used.md` | The 6 GitHub Actions used in the observatory workflow |
| `008.vliz-github-conventions.md` | Branching, commits, PRs |
| `009.vliz-python-conventions.md` | Poetry, linting, testing, style |
| `010.git-and-environment-basics.md` | Git commands, env vars, dependencies, venv |
| `011.observations.md` | My own notes/feedback on repo documentation quality |
| `012.useful-links.md` | *(placeholder — fill in: curated links referenced across notes)* |
| `013_data_quality_control_phase2.md` | data-quality-control-action, full code walkthrough: __main__.py, pipeline.py, rules.py, the py-data-rules framework |
| `014_data_quality_control_issues_roadmap.md` | First-pass prioritisation of the 13 open issues on data-quality-control-action |
| `015_data_quality_control_issues_roadmap_revised.md` | Revised roadmap after discovering the observatory-profile schema dependency and the explicit/implicit rules distinction |

## Related repos

- [bpns-workflow-local](https://github.com/Hamideh-B-H/bpns-workflow-local) — the hands-on exercise this learning is tied to (Phase 1 complete, Phase 2 code study complete)
- [bpns-workflow-practice](https://github.com/Hamideh-B-H/bpns-workflow-practice) — running the same pipeline as an automated GitHub Action, for comparison