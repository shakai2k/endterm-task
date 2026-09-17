# SIT742 2026T2 Assignment 2 — Group Forecasting Project

This repository contains the reproducible group submission for the SIT742
Assignment 2 tourism-demand forecasting project. It uses the public TULIP Lab
`ISF-TDF2023` dataset to forecast 20 destination series for `2023M08` through
`2024M07`.

The working notebook is
[`SIT742-2026T2-A2-G3.ipynb`](SIT742-2026T2-A2-G3.ipynb). Assignment material,
including the authoritative file-naming pattern, is under [`docs/`](docs/).

## Repository layout

```text
SIT742-2026T2-A2-G3.ipynb          Working graded notebook
README.md                           Developer and handoff guide
pyproject.toml                      Python 3.11 project dependencies
uv.lock                             Locked dependency graph
docs/
  SIT742-2026T2-A2-Specification.md Assignment specification
  SIT742-2026T2-A2-Public-Rubric.pdf Public rubric
  SIT742-2026T2-A2-Starter.ipynb    Untouched reference notebook
  SIT742-2026T2-A2-Function-Examples.md
  SIT742-2026T2-A2-Forecast-Measures-Guide.md
  Video-Trascript.md
```

The forecast CSV is a generated submission artifact and may not exist until the
Q5 export cell has run successfully.

## Required submission names

The assignment specification defines these names, where `<GroupID>` is the
identifier supplied through Olympus or by the teaching team:

```text
SIT742-2026T2-A2-<GroupID>.ipynb
SIT742-2026T2-A2-<GroupID>.pdf
SIT742-2026T2-A2-<GroupID>-Forecast.csv
SIT742-2026T2-A2-<GroupID>-Video.<approved format or link>
```

Use the `SIT742` prefix from the specification. If an older notebook prompt,
cell, or saved output uses `SIG742`, treat it as a stale typo and correct it
before the final export. Do not guess `<GroupID>` from the repository name;
confirm it against Olympus and keep the notebook filename, `GROUP_INFO`, PDF,
CSV, and video filename consistent.

## Environment setup

The project targets Python 3.11 and uses [uv](https://docs.astral.sh/uv/) for a
reproducible environment:

```bash
uv sync --frozen
uv run jupyter lab
```

Open the working notebook and run it from top to bottom. The first complete run
downloads the public dataset and may download the TimesFM checkpoint, so it can
take substantially longer than later runs. These downloads are local caches and
must not be committed.

For a non-interactive clean run:

```bash
uv run jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.timeout=-1 \
  SIT742-2026T2-A2-G3.ipynb
```

## Notebook contracts

Preserve the required question headings, function names, object names, and all
`A2:ANSWER` markers. There must be exactly one `START` and one `END` marker for
each of Q1–Q7, in question order. Added analysis and code cells may change raw
cell numbers; markers must remain in their corresponding question section.

The main workflow is:

1. Load the public dataset and construct cutoff-safe training/validation tables.
2. Q1–Q3 generate baselines, audit wide tables, and calculate MAE/MASE/MAPE.
3. Q4 compares seven candidate models and records exactly one selected model in
   `model_summary`.
4. Q5 creates `forecast_submission_wide`, validates it into
   `forecast_submission_audit`, and exports the CSV.
5. Q6 interprets aggregate and selected-market results.
6. Q7 records the group video and contribution evidence.

Important final objects:

- `model_summary`: one row per model and exactly one `selected=True` row.
- `forecast_submission_wide`: `Date` plus exactly 20 destination columns and 12
  rows from `2023M08` to `2024M07`.
- `forecast_submission_audit`: all required Q2 audit fields,
  `can_align is None`, and `is_valid is True`.

## Regenerating the forecast CSV

Do not edit the CSV manually. Run the notebook through the Q5 forecast cell,
check the audit, and then run the export **code** cell:

```python
group_id = (GROUP_INFO.get("group_id") or "").strip()
if not group_id:
    raise ValueError("Set GROUP_INFO['group_id'] before export")

export_path = f"SIT742-2026T2-A2-{group_id}-Forecast.csv"
forecast_submission_wide.to_csv(export_path, index=False)
print(f"Exported {export_path}")
```

Before accepting the generated file, confirm:

- the audit reports `is_valid=True` and `can_align=None`;
- there are 12 unique dates from `2023M08` through `2024M07`;
- there are exactly 20 destination columns in public-dataset order;
- all forecast values are numeric and finite; and
- there is no index, `model_label`, diagnostic, note, or actual/result column.

## Developer workflow

### Before editing

```bash
git switch main
git pull
git switch -c initials/short-task-name
```

Agree on notebook ownership before editing. Jupyter notebooks are JSON files,
and simultaneous edits often create hard-to-review merge conflicts. Prefer one
notebook editor at a time; otherwise use separate branches and non-overlapping
sections.

### While editing

- Edit the working notebook, not the starter notebook under `docs/`.
- Keep written explanations next to the relevant output or figure.
- Use relative paths only; never commit credentials or private data.
- Record any new public external source and cutoff evidence in
  `external_feature_log`.
- Do not use assessment-period actuals, manual forecast overrides, or
  uncontrolled random output.
- Avoid unrelated cell reformatting, cell splitting/merging, or clearing all
  outputs because these create noisy diffs.
- Ensure Python content is stored in a code cell, especially the Q5 CSV export.

### Validation before handoff

Run the affected cells for documentation-only changes. For changes to data,
features, models, evaluation, dependencies, or export logic, restart the kernel
and run the complete notebook.

Useful structural checks:

```bash
# Notebook must remain valid JSON.
jq empty SIT742-2026T2-A2-G3.ipynb

# Inspect all answer markers; expect Q1–Q7 START and END exactly once each.
rg -o 'A2:ANSWER:Q[1-7]:(START|END)' SIT742-2026T2-A2-G3.ipynb \
  | sort | uniq -c

# Review only the files intentionally changed.
git status --short
git diff --stat
```

Also verify that the Table of Contents links still target unique HTML anchors
and that Q1–Q7 anchors land at their question headings rather than inside answer
blocks.

### Commit and handoff

```bash
git add SIT742-2026T2-A2-G3.ipynb README.md
git commit -m "Describe the focused change"
git push -u origin initials/short-task-name
```

The handoff or pull request should state:

- which sections or cells changed;
- which checks were run;
- whether dependencies, source data, model selection, or forecasts changed;
- whether the generated CSV changed; and
- any assumptions, limitations, or remaining work.

Review the rendered notebook as well as the raw diff. When resolving notebook
conflicts, do not accept an entire file from one side without checking individual
cells; that can silently delete another contributor's work.

## Current status and release checks

- Q1–Q3, integrated baseline evidence, EDA, and Q4–Q6 are present.
- Seven forecasting candidates are compared; `blended_recovery` is the recorded
  selected model.
- Q5 contains final forecast construction, audit, and export logic.
- Q7 still requires the final video filename/link, participation summary, and
  contribution details.
- The working notebook name contains `G3`, while its current `GROUP_INFO` has
  used `Team16`. The team must resolve this against the official group ID and
  rename all deliverables consistently.
- Check that the Q5 prompt and export cell use the specification's `SIT742`
  prefix before the final run.

The Olympus Assignment page and current unit-site instructions remain the source
of truth if they differ from this repository.

## Final review

One member should run the notebook from a clean environment and regenerate the
CSV. A second member should compare the rendered notebook/PDF and all generated
files against the public rubric and specification. Confirm filenames, group
details, answer markers, visible outputs, citations, Q7 evidence, forecast schema,
and reproducibility before submission.
