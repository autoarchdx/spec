# Provenance model

Every node, edge, and group in an AutoArchDX manifest carries provenance — a record of where it came from, who last touched it, and when. Provenance is not metadata appended after the fact; it is a first-class field in the manifest schema, present on every element regardless of source.

Provenance serves three purposes:

1. **Navigation** — clicking a node in the renderer opens the exact file and line (and symbol) where each contributing annotation lives
2. **Drift detection** — comparing `introduced_at` to `commit` reveals how much the code has changed since architectural intent was documented
3. **Accountability** — `author`, `branch`, and `symbol` surface who made a decision, in what context, and on what symbol — supporting code review and architectural governance

---

## Provenance by source

Every manifest element has a `source` field identifying its origin. The provenance shape differs by source.

### Annotation provenance

For elements declared via `@arch:*` annotations in source code. When a single component is declared by multiple annotations (same `id`, different files or symbols), `provenance` becomes an array of annotation provenance objects — one per contributing annotation:

```jsonc
// Single annotation — provenance is an object
"provenance": {
  "source": "annotation",
  "file": "internal/scoring/engine.go",
  "line": 14,
  "symbol": "NewScoringEngine",
  "symbol_kind": "function",
  "annotation": "@arch:component id:scoring-engine type:platform status:built",
  "commit": "abc123def456",
  "introduced_at": "9f1e2a3b",
  "committed_at": "2026-03-10T14:22:00Z",
  "branch": "main",
  "author": "john.bear"
}

// Multiple annotations — provenance is an array
"provenance": [
  {
    "source": "annotation",
    "file": "internal/scoring/engine.go",
    "line": 14,
    "symbol": "NewScoringEngine",
    "symbol_kind": "function",
    "annotation": "@arch:component id:scoring-engine type:platform status:built",
    "commit": "abc123def456",
    "introduced_at": "9f1e2a3b",
    "committed_at": "2026-03-10T14:22:00Z",
    "branch": "main",
    "author": "john.bear"
  },
  {
    "source": "annotation",
    "file": "internal/scoring/score.go",
    "line": 8,
    "symbol": "Score",
    "symbol_kind": "method",
    "annotation": "@arch:component id:scoring-engine",
    "commit": "def456abc789",
    "introduced_at": "1a2b3c4d",
    "committed_at": "2026-03-12T09:15:00Z",
    "branch": "main",
    "author": "john.bear"
  }
]
```

### YAML provenance

For elements declared in `.autoarch/definitions.yml` or other definition files:

```jsonc
"provenance": {
  "source": "yaml",
  "file": ".autoarch/definitions.yml",
  "yaml_key": "components[2]",
  "commit": "abc123def456",
  "introduced_at": "9f1e2a3b",
  "committed_at": "2026-03-10T14:22:00Z",
  "branch": "main",
  "author": "jane.architect"
}
```

YAML provenance uses `yaml_key` instead of `line` — a stable pointer into the definition file in the format `section[index]` (e.g. `components[0]`, `connections[3]`). This is more stable than a line number as the file grows.

### Inferred provenance

For edges produced by the scanner's import graph analysis:

```jsonc
"provenance": {
  "source": "inferred",
  "import_path": "github.com/wfai/platform/internal/cache",
  "confidence": 0.9,
  "commit": "abc123def456",
  "branch": "main"
}
```

Inferred edges do not carry `author`, `symbol`, or `introduced_at` — they are structural observations, not human decisions.

### Dual provenance

When `source` is `both` (a component declared in both annotation and YAML), provenance carries both shapes:

```jsonc
"provenance": {
  "annotation": { ... },   // annotation provenance object or array
  "yaml": { ... }          // yaml provenance object
}
```

---

## Field reference

| Field | Type | Present on | Description |
|-------|------|-----------|-------------|
| `source` | enum | all | `annotation` `yaml` `inferred` `both` |
| `file` | string | annotation, yaml | Path relative to repo root |
| `line` | number | annotation | Line number of the `@arch:` tag |
| `yaml_key` | string | yaml | Pointer into definition file: `section[index]` |
| `symbol` | string | annotation | Name of the symbol immediately following the annotation |
| `symbol_kind` | string | annotation | `function` `method` `struct` `class` `interface` `type` `variable` `constant` `module` `package` `unknown` |
| `annotation` | string | annotation | Raw annotation string as it appears in source |
| `commit` | string | all | Full SHA of the commit where this element was last modified |
| `introduced_at` | string | annotation, yaml | Full SHA of the commit where this element first appeared |
| `committed_at` | ISO 8601 | annotation, yaml | Timestamp of `commit` |
| `branch` | string | all | Branch name at time of scan |
| `author` | string | annotation, yaml | From `git blame` at the element's location |
| `import_path` | string | inferred | The import statement that produced the edge |
| `confidence` | number | inferred | Scanner certainty, 0.0–1.0 |

All fields are optional in the JSON Schema sense — the scanner emits what it can determine from available git context. In environments without git (e.g. shallow clones), some fields may be absent. The scanner never fails due to missing git context.

---

## Symbol tracking

The scanner infers the symbol following each annotation from the next non-comment, non-blank line, without requiring language-specific AST parsing:

```go
// @arch:component id:scoring-engine type:platform status:built
func NewScoringEngine(cfg Config) *ScoringEngine {
// scanner records: symbol="NewScoringEngine", symbol_kind="function"
```

```python
# @arch:component id:inference-api type:api status:planned
class InferenceAPI:
# scanner records: symbol="InferenceAPI", symbol_kind="class"
```

```go
// @arch:component id:postgres-main type:db status:built
var db *pgxpool.Pool
// scanner records: symbol="db", symbol_kind="variable"
```

Symbol tracking enables:
- **Renderer navigation** — click a node, see all contributing symbols with direct links to source
- **Drift signal enrichment** — the scanner can observe which contributing symbols changed since `introduced_at`, not just whether the file changed
- **Coverage reporting** — which functions in a package contribute to which architectural components

When the scanner cannot determine a symbol (e.g. the annotation is at end of file, or the next line is another annotation), `symbol` is omitted and `symbol_kind` is set to `unknown`.

---

## The `introduced_at` / `commit` distinction

These two fields together are the primary drift signal.

`introduced_at` is the commit where this annotation or YAML definition first appeared. It answers: "when was this architectural decision documented?"

`commit` is the commit where this element was last modified. It answers: "when was this documentation last touched?"

The code surrounding an annotation changes independently of the annotation itself. A node whose `introduced_at` is eighteen months old and whose contributing symbols have been modified forty times since then is a candidate for review — the annotation may no longer accurately describe what the code does.

The scanner surfaces this as a `STALE_ANNOTATION` warning (at `info` level) when the number of commits touching any contributing symbol's file since `introduced_at` exceeds a configurable threshold (default: 20 commits).

```jsonc
{
  "code": "STALE_ANNOTATION",
  "severity": "info",
  "message": "Component 'scoring-engine': annotation in engine.go has not been updated in 34 commits. Consider reviewing for accuracy.",
  "node_id": "scoring-engine",
  "provenance": {
    "source": "annotation",
    "file": "internal/scoring/engine.go",
    "line": 14,
    "symbol": "NewScoringEngine",
    "commit": "abc123def456",
    "introduced_at": "9f1e2a3b",
    "branch": "main"
  }
}
```

Configurable threshold in `.autoarch/config.yml`:

```yaml
scanner:
  stale_annotation_threshold: 20   # commits since introduced_at; 0 to disable
```

---

## Branch provenance

The `branch` field records the branch name at time of scan — the branch the scanner was running on, not necessarily where the annotation was introduced.

Branch provenance is most useful in two contexts:

**Feature branch nodes:** A node annotated on `feature/experimental-llm-gateway` that hasn't merged to `main` is a different signal from a stale annotation on `main`. The renderer surfaces branch in the node detail panel.

**GitHub Action scans:** The action records `GITHUB_REF_NAME` as the branch for every element. The manifest always knows whether it was generated from a PR branch, release branch, or main.

Branch provenance on warnings is particularly valuable. A `SOT_CONFLICT` pointing to `branch:feature/new-scoring` is expected churn during development. The same warning pointing to `branch:main` may need immediate attention.

---

## Confidence scoring

Inferred edges carry a `confidence` value between 0.0 and 1.0:

| Signal | Confidence range |
|--------|-----------------|
| Direct import of a file containing `@arch:component` | 0.85–1.0 |
| Import of a package containing annotated components | 0.60–0.85 |
| Transitive import (A imports B which imports C) | 0.30–0.60 |
| Import of a file with no `@arch:component` annotation | 0.10–0.30 |
| Import path matches a known utility pattern (`/utils`, `/logger`, `/config`) | 0.00–0.15 |

The renderer scales inferred edge visual weight by confidence. The scanner's `--min-confidence` flag filters edges below a threshold from the manifest entirely. Default minimum: `0.2`.

```yaml
scanner:
  min_inferred_confidence: 0.2
```

---

## Provenance in warnings

Every `warnings[]` entry includes a `provenance` block. For conflicts between annotations and YAML definitions, provenance always points to the annotation — never the YAML file.

This is deliberate: the annotation is the human decision that needs to be examined. When a conflict needs resolving, the engineer opens the file at the line the warning points to, reads the annotation in context, and decides whether to update it, assert `!id:`, or add `@arch:override allow:*`.

```jsonc
{
  "code": "SOT_CONFLICT",
  "severity": "warn",
  "message": "Field 'status' conflict on component 'bls-oews': yaml=built, annotation=planned",
  "node_id": "bls-oews",
  "fields": ["status"],
  "resolution": "annotation",
  "provenance": {
    "source": "annotation",
    "file": "internal/data/bls.go",
    "line": 8,
    "symbol": "blsOEWSClient",
    "symbol_kind": "variable",
    "commit": "abc123def456",
    "branch": "main",
    "author": "john.bear"
  }
}
```

---

## Git context requirements

The scanner requires:

- A git repository at or above the configured root
- At least one commit in history
- `git` available on `PATH`

For shallow clones, `introduced_at` may not be resolvable if the commit is outside the cloned depth. The field is omitted and a `SHALLOW_CLONE` info entry appears in the manifest:

```jsonc
{
  "code": "SHALLOW_CLONE",
  "severity": "info",
  "message": "Repository is a shallow clone. 'introduced_at' fields may be incomplete. Use fetch-depth: 0 in your GitHub Action checkout step."
}
```

Recommended Action configuration:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0    # full history — enables complete provenance
```

---

## Provenance in the GitHub Action

When the Action generates a manifest, it supplements provenance with CI context:

```jsonc
"provenance": {
  "source": "annotation",
  "file": "internal/scoring/engine.go",
  "line": 14,
  "symbol": "NewScoringEngine",
  "symbol_kind": "function",
  "commit": "abc123def456",
  "introduced_at": "9f1e2a3b",
  "committed_at": "2026-03-10T14:22:00Z",
  "branch": "main",
  "author": "john.bear",
  "ci": {
    "run_id": "12345678",
    "workflow": "autoarch-scan.yml",
    "triggered_by": "push",
    "pr_number": null
  }
}
```

The `ci` block is only present when the manifest is generated by the Action.
