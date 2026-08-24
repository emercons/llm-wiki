# Proposal: validate Core Principles numbering in docs-consistency tests

This document is intended to be split into a GitHub issue and a small pull
request for current upstream `nvk/llm-wiki` (observed against the v0.24.1
source imported by the Knowmade downstream).

## Copy-ready issue

### Title

`test-docs-consistency.sh` does not detect duplicate or regressing Core Principles numbers

### Problem

The numbered Core Principles in `AGENTS.md` and
`claude-plugin/skills/wiki-manager/SKILL.md` are behavioral contract anchors,
but the deterministic documentation tests do not validate their sequence.

This is easy to break when a downstream or an upstream feature inserts new
principles in the middle of the list. The Markdown still renders, plugin sync
tests still pass after mirrors are regenerated, and the existing docs
consistency test remains green even when numbers are duplicated or regress.

The Knowmade downstream reproduced this after inserting private-goal and hub
inbox principles before later upstream principles. Its canonical skill ended
with `... 15, 16, 14, 15`, while its portable `AGENTS.md` ended with
`... 16, 17, 18, 16, 17, 18`. All existing deterministic tests passed.

Current upstream v0.24.1 itself has a contiguous sequence. This issue is about
closing the missing regression guard, not claiming that the current upstream
documents are already misnumbered.

### Expected behavior

`tests/test-docs-consistency.sh` should fail when the numbered entries inside
the `## Core Principles` section of either canonical contract are not exactly:

```text
1, 2, 3, ..., N
```

The check should:

- inspect `AGENTS.md` and `claude-plugin/skills/wiki-manager/SKILL.md`;
- limit parsing to the `## Core Principles` section;
- reject duplicates, gaps, and regressions;
- ignore numbered lists in unrelated sections;
- produce a readable failure containing the file and observed sequence.

### Why this belongs upstream

The failure mode is generic to any feature insertion and to any downstream
that carries the portable/canonical contracts. Upstream already centralizes
cross-document invariants in `tests/test-docs-consistency.sh`; this is the
natural place for the guard.

### Acceptance criteria

- A deliberately duplicated principle number makes
  `tests/test-docs-consistency.sh` fail.
- A deliberately skipped number makes it fail.
- The unmodified upstream documents pass.
- Existing plugin mirror and structural tests continue to pass.

## Copy-ready pull request

### Title

Test Core Principles numbering for documentation drift

### Summary

Add a deterministic docs-consistency check that requires contiguous numbering
inside the Core Principles sections of the portable `AGENTS.md` contract and
the canonical Claude skill. This prevents feature insertions from silently
leaving duplicate, skipped, or regressing behavioral reference numbers.

### Suggested implementation

Add the following logic to the Python block in
`tests/test-docs-consistency.sh`:

```python
for relative in ["AGENTS.md", "claude-plugin/skills/wiki-manager/SKILL.md"]:
    text = (root / relative).read_text(encoding="utf-8")
    section = re.search(
        r"^## Core Principles\n(?P<body>.*?)(?:\n## |\Z)",
        text,
        re.M | re.S,
    )
    if not section:
        fail(f"{relative} is missing the ## Core Principles section")
        continue
    numbers = [
        int(value)
        for value in re.findall(r"^(\d+)\. \*\*", section.group("body"), re.M)
    ]
    expected = list(range(1, len(numbers) + 1))
    if numbers != expected:
        fail(f"{relative} Core Principles numbering is not contiguous: {numbers}")
```

No runtime behavior, schemas, commands, or package versions need to change.

### Test plan

```bash
./tests/test-docs-consistency.sh
./tests/test-plugin-validate.sh
./tests/test-codex-sync.sh
./tests/test-opencode-sync.sh
```

For a negative test, temporarily duplicate or skip a number in each target
file and verify that `test-docs-consistency.sh` fails with the expected path
and observed sequence. Restore the fixture change before committing.

### Files expected in the PR

- `tests/test-docs-consistency.sh`

Optionally add a dedicated negative fixture if maintainers prefer fixture-based
coverage over the compact invariant check.
