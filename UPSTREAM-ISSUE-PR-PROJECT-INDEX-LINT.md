# Proposal: align output-index lint with v0.2 Project membership

This document is intended to be split into a GitHub issue and a small pull
request for current upstream `nvk/llm-wiki` (observed against the v0.24.1
source imported by the Knowmade downstream).

## Copy-ready issue

### Title

`lint` requires every Project member in `output/_index.md` despite the v0.2 `WHY.md` membership model

### Problem

The current Project contract says that:

- an active Project is a folder under `output/projects/<slug>/` containing
  `WHY.md`;
- `WHY.md` is the only required control file;
- membership is derived by scanning the Project folder;
- Project-local `_index.md` files were removed by the v0.2 simplification.

However, recursive index-completeness lint currently treats every Markdown
member below `output/projects/<slug>/` as a row that must appear in the global
`output/_index.md`. A valid Project containing `WHY.md` and ordinary member
artifacts therefore fails unless the global output index duplicates its entire
membership list.

This makes `output/_index.md` both a Project catalog and an accidental flat
member manifest, contradicting the scan-derived membership model. It also
creates high-churn index rows for large Projects.

### Minimal reproduction

```text
.wiki/output/
├── _index.md
└── projects/
    └── trainer-center-launch/
        ├── WHY.md
        └── launch-plan.md
```

Let `output/_index.md` contain one Project row linking
`projects/trainer-center-launch/WHY.md`, but no row for `launch-plan.md`.
Current lint reports the member as missing from the output index.

### Expected behavior

For recursive completeness of `output/_index.md`:

- loose output artifacts remain indexed normally;
- each active Project is represented by
  `projects/<slug>/WHY.md` in the Projects section;
- other files below `projects/<slug>/` are not required as global index rows;
- hidden and archived structures retain their existing behavior.

The member count in the human-readable Projects table can remain advisory; it
is not a second membership authority unless the Project spec explicitly makes
it one.

### Why this belongs upstream

This is a generic mismatch between the documented Project data model and the
shared deterministic linter. It affects every sufficiently useful Project,
not a downstream-specific extension.

### Acceptance criteria

- A Project with `WHY.md` and one or more member artifacts passes when only
  its `WHY.md` is represented in `output/_index.md`.
- Removing the Project's `WHY.md` row still produces an index-completeness
  finding.
- Missing loose output rows are still detected.
- Existing index, archive and lint fixtures continue to pass.

## Copy-ready pull request

### Title

Align output-index lint with scan-derived Project membership

### Summary

Teach recursive output-index discovery to represent a Project by its active
`WHY.md` control file and to skip ordinary Project member artifacts. Add a
regression fixture containing a valid Project with an unlisted member file.

This implements the existing v0.2 Project contract; it does not add a schema
or a project-local manifest.

### Suggested implementation

In `index_content_files`, after hidden-path filtering and before appending a
discovered file:

```python
rel_parts = md_file.relative_to(directory).parts

if directory.name == "output" and rel_parts[:1] == ("projects",):
    if len(rel_parts) != 3 or rel_parts[2] != "WHY.md":
        continue
```

This keeps exactly `projects/<slug>/WHY.md` as the global-index representative
and excludes nested Project members. If the implementation supports archived
Projects in the same traversal, preserve the existing archive policy around
this condition.

### Regression test

Extend `tests/test-local-cli-lint.sh` with a golden-wiki copy that adds:

```text
output/projects/trainer-center-launch/WHY.md
output/projects/trainer-center-launch/launch-plan.md
```

Replace the fixture's `output/_index.md` with both:

- a Projects row linking the new `WHY.md`; and
- the existing loose-output row.

Assert that lint succeeds without a row for `launch-plan.md`. Existing
negative fixtures already cover ordinary missing index entries; optionally add
a second assertion that deleting the `WHY.md` link fails.

### Test plan

```bash
./tests/test-local-cli-lint.sh
./tests/test-plugin-validate.sh
./tests/test-codex-sync.sh
./tests/test-opencode-sync.sh
```

### Files expected in the PR

- `scripts/llm-wiki`
- `tests/test-local-cli-lint.sh`
- regenerated CLI mirrors required by the repository's sync policy
