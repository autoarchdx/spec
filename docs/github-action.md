# GitHub Action

The AutoArchDX GitHub Action scans a repository on push or pull request, compares the generated manifest to the committed one, and opens a pull request when the architecture has changed. It is the primary mechanism for keeping `.autoarch/manifest.json` current without requiring manual scanner runs.

GitHub is the first-class CI target for AutoArchDX. The Action is designed to be zero-config for standard repository layouts, with progressive configuration available for more complex setups.

---

## Quick start

```yaml
# .github/workflows/autoarch.yml
name: AutoArchDX

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # full history for complete provenance

      - uses: autoarchdx/action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

That is a complete, working configuration for most repositories. Everything else in this document is optional.

---

## Lifecycle

### 1. Skip check

Before scanning, the Action checks whether the current event should be skipped entirely.

**Skip conditions — any one of these causes an immediate clean exit (exit 0):**

- The commit message contains `[autoarch-skip]` or `[skip ci]`
- The commit was made by `github-actions[bot]` AND the only changed file is `.autoarch/manifest.json` AND the commit message contains `[autoarch-manifest]` — this is the merge guard that prevents the Action from recursively triggering on its own manifest PRs
- The event is a pull request AND the PR branch is named `autoarch/update-manifest-*` — the manifest PR itself does not re-scan

All three conditions of the merge guard must be true simultaneously. A `github-actions[bot]` commit that touches files other than the manifest is not skipped — it scans normally.

### 2. Scan

The Action runs the scanner against the repository:

```
autoarch scan --root ./ --config .autoarch/config.yml
```

The scanner produces a new manifest in memory. If the scan fails for any reason other than warnings (fatal parse error, missing git context, configuration error), the Action exits with code 1 and surfaces the error in the workflow log. Warnings in the manifest never cause a non-zero exit.

### 3. Manifest comparison

The Action compares the newly generated manifest to the committed `.autoarch/manifest.json`, if one exists.

**No committed manifest:** The manifest is new to this repository. Proceed to step 4 (write and open PR).

**Committed manifest exists, no structural diff:** The architecture has not changed. Exit 0 cleanly. No PR opened.

**Structural diff detected:** Proceed to step 4.

The diff is structural — it compares nodes, edges, groups, and warnings arrays, not the raw JSON bytes. Changes to `generated` timestamp, `scanner_version`, or layout hints that the user has adjusted manually do not constitute a structural diff and do not trigger a PR.

### 4. Open PR

The Action creates a branch named `autoarch/update-manifest-{sha7}` off the current HEAD, commits the new manifest to it, and opens a pull request against the base branch.

**Branch naming:** `autoarch/update-manifest-{sha7}` where `{sha7}` is the first seven characters of the commit that triggered the scan. This makes the PR traceable to its triggering commit.

**PR title:** `[autoarch-manifest] Update architecture manifest ({sha7})`

The `[autoarch-manifest]` prefix in the title is the merge guard signal. When this PR is merged, the resulting merge commit's message will contain `[autoarch-manifest]`, triggering the skip condition in step 1.

**PR body:** A structured diff summary generated from the manifest comparison:

```
## Architecture changes

**Nodes**
+ 2 added: `fork-custom-scorer`, `confidence-signal`
~ 1 modified: `scoring-engine` (status: planned → built)

**Edges**
+ 3 added
- 1 removed: `scoring-engine → old-cache` (declared)

**Warnings**
~ 2 resolved: `MISSING_IMPL` on `scoring-engine`, `run-cache`
+ 1 new: `STALE_ANNOTATION` on `bigquery-warehouse`

---
Triggered by commit abc123d · branch main
Scanner version 0.1.0 · [View full manifest diff](...)
```

The diff summary uses node IDs and human-readable labels. Added items use `+`, removed use `-`, modified use `~`.

**PR labels:** The Action applies the `autoarch` label to the PR. If the label does not exist in the repository, the Action creates it (color: `#1D9E75`).

**Existing open PR:** If a PR named `autoarch/update-manifest-*` is already open against the base branch, the Action updates the existing PR rather than opening a new one — it force-pushes the new manifest to the existing branch and updates the PR body with the latest diff. This prevents accumulating multiple open manifest PRs when the architecture changes rapidly.

---

## Merge guard detail

The merge guard prevents the Action from running recursively when its own manifest PR is merged. It works by requiring all three of these conditions to be true before skipping:

| Condition | How it is checked |
|-----------|------------------|
| Commit author is `github-actions[bot]` | `github.actor == 'github-actions[bot]'` |
| Only changed file is `.autoarch/manifest.json` | Git diff against previous commit |
| Commit message contains `[autoarch-manifest]` | String match on `github.event.head_commit.message` |

The third condition — checking the commit message — is what makes the guard spoofable in principle but not in practice. An engineer would have to intentionally commit a file named `.autoarch/manifest.json` via a bot actor with `[autoarch-manifest]` in the message to trigger a false skip. This is considered an acceptable tradeoff against the complexity of a more robust guard.

---

## Configuration

```yaml
- uses: autoarchdx/action@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}   # required
    config-path: .autoarch/config.yml           # default
    base-branch: main                           # default: repository default branch
    pr-assignees: ""                            # comma-separated GitHub usernames
    pr-reviewers: ""                            # comma-separated GitHub usernames or teams
    fail-on-warnings: false                     # exit 1 if manifest contains any warn-severity warnings
    min-confidence: 0.2                         # override scanner min_inferred_confidence
    skip-patterns: ""                           # additional skip patterns beyond [autoarch-skip]
    dry-run: false                              # scan and diff but do not open PR
```

**`fail-on-warnings`:** When `true`, the Action exits with code 1 if the generated manifest contains any `warn`-severity warnings. Useful for enforcing architectural hygiene in CI — broken refs, unresolved conflicts, and missing implementations block the build. Defaults to `false` because most teams want to adopt AutoArchDX incrementally without immediately failing CI on pre-existing issues.

**`dry-run`:** Runs the full scan and diff and logs what the PR would contain, but does not create or update any PR. Useful for testing Action configuration changes or previewing manifest updates without triggering GitHub notifications.

---

## Permissions

The Action requires these GitHub token permissions:

```yaml
permissions:
  contents: write       # to create branches and commit the manifest
  pull-requests: write  # to open and update PRs
```

If your repository uses branch protection rules that prevent the `github-actions[bot]` from pushing directly, you will need to either allow bot pushes on the manifest branch pattern (`autoarch/update-manifest-*`) or supply a personal access token with appropriate permissions via the `github-token` input.

---

## Handling the manifest PR

The manifest PR is an architectural review artifact, not just a file change. The recommended review process:

1. **Read the diff summary in the PR body** — it summarizes what changed architecturally in plain language
2. **Review any new `warn`-severity warnings** — `SOT_CONFLICT`, `BROKEN_REF`, `MISSING_IMPL` in the new manifest indicate issues that need resolution
3. **Verify that new nodes and edges reflect actual intent** — the manifest captures both declared and inferred connections; inferred edges in the diff may need `@arch:override deny:` if they are not architecturally meaningful
4. **Merge or request changes** — merging the PR commits the new manifest to the base branch and triggers the skip guard on the resulting commit

The manifest PR should be treated with the same weight as an API contract change. It is a record of how the system's architecture evolved.

---

## Running the scanner locally

The same scanner the Action uses is available as a CLI:

```bash
# Install
go install github.com/autoarchdx/scanner/cmd/autoarch@latest

# Scan and write manifest
autoarch scan

# Scan and print diff against committed manifest
autoarch diff

# Scan without writing — print manifest to stdout
autoarch scan --dry-run

# Check for warnings only — useful in pre-commit hooks
autoarch scan --fail-on-warnings
```

The CLI and Action always produce identical manifests for the same input — they share the same core library. See the [scanner repository](https://github.com/autoarchdx/scanner) for full CLI reference.

---

## Example: full workflow

A typical day in a repository using AutoArchDX:

1. Engineer annotates a new component in `feature/new-cache`:
   ```go
   // @arch:component id:insight-cache type:platform status:built label:"Insight cache"
   // @arch:connects to:api-egress bidir:true
   func NewInsightCache(...) *InsightCache {
   ```

2. Engineer opens a PR from `feature/new-cache` to `main`. The Action runs on the PR, detects that `insight-cache` is new in the manifest, and opens `autoarch/update-manifest-abc123d` as a draft sibling PR showing the architecture change.

3. Reviewer sees both PRs. The manifest PR body shows:
   ```
   + 1 added: `insight-cache` (platform, built)
   + 2 added edges: insight-cache → api-egress (declared, bidir)
   ```

4. Reviewer approves both PRs. The feature PR merges first. The manifest PR merges second, committing the updated manifest to main with the merge guard commit message.

5. The Action runs on the merge commit, detects `[autoarch-manifest]` in the message combined with only `.autoarch/manifest.json` changed by `github-actions[bot]`, and exits 0 without scanning.

---

## Troubleshooting

**Action opens a PR on every push even when nothing changed**

The structural diff is producing false positives. Common causes:
- `layout` coordinates are changing because the renderer is writing back user-adjusted positions to a file that is getting committed. Solution: ensure `.autoarch/manifest.json` is in `.gitignore` for local renderer sessions, or configure the renderer to not auto-commit layout changes.
- `generated` timestamp is being included in the structural diff. This is a scanner bug — open an issue.

**`introduced_at` is missing from all provenance**

The repository was checked out as a shallow clone. Add `fetch-depth: 0` to the checkout step.

**Action cannot push to the manifest branch**

Branch protection rules are blocking the `github-actions[bot]`. Either allow bot pushes on the `autoarch/update-manifest-*` pattern, or supply a PAT with `contents: write` permission via the `github-token` input.

**Manifest PR is opened but immediately conflicts**

Another manifest PR was merged between the triggering commit and the PR creation. The Action will detect the conflict on its next run and force-push the updated manifest to the existing open PR branch.
