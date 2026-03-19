# Fork model

AutoArchDX supports forked repositories as first-class citizens. A fork can extend, override, or annotate an upstream architecture without modifying the upstream source. The manifest produced by a fork scan is a complete, self-contained graph that makes the fork's additions and overrides visible and traceable.

---

## Concepts

**Upstream manifest** — the `.autoarch/manifest.json` from the repository being forked. It is the base layer. The fork does not need to re-declare anything already in the upstream manifest — all upstream nodes, edges, and groups are available by reference.

**Fork layer** — the fork's own `.autoarch/definitions.yml` and `@arch:*` annotations. These sit on top of the upstream manifest. They can add new components, declare new edges to upstream components, assign upstream components to new groups, and override upstream field values.

**Merged manifest** — the final manifest produced by the scanner on the fork. It is the union of the upstream manifest and the fork layer, with conflicts resolved per the standard [conflict resolution](./annotation-grammar.md#conflict-resolution) rules.

---

## Configuration

Forks declare their upstream in `.autoarch/config.yml`:

```yaml
# .autoarch/config.yml (fork repo)
project: wfai-platform-fork
repo_url: https://github.com/my-org/wfai-platform-fork

upstream:
  repo_url: https://github.com/autoarchdx/wfai-platform
  manifest_ref: abc123def456    # commit SHA of upstream manifest to use as base
```

`manifest_ref` pins the upstream manifest to a specific commit. The scanner fetches the upstream manifest at that ref during scanning. This makes fork scans reproducible — the upstream base does not shift until `manifest_ref` is explicitly updated.

The `upstream` block in the generated manifest mirrors this configuration:

```jsonc
{
  "version": "1.0",
  "project": "wfai-platform-fork",
  "upstream": {
    "repo_url": "https://github.com/autoarchdx/wfai-platform",
    "manifest_ref": "abc123def456",
    "merged_at": "2026-03-19T05:00:00Z"
  },
  "nodes": [...],
  "edges": [...],
  "groups": [...],
  "warnings": [...]
}
```

---

## The `origin` field

Every node, edge, and group in a fork manifest carries an `origin` field that records where it came from in the layered merge:

| Value | Meaning |
|-------|---------|
| `local` | Declared entirely in the fork — not present in the upstream manifest |
| `upstream` | Declared in the upstream manifest — fork contributed nothing |
| `fork-override` | Declared in the upstream manifest — fork overrode one or more fields |
| `fork-extended` | Declared in the upstream manifest — fork added connections or group membership |

This field is fork-specific. It does not appear in non-fork manifests.

```jsonc
{
  "id": "scoring-engine",
  "type": "platform",
  "status": "built",
  "label": "Scoring engine",
  "origin": "upstream",
  ...
}

{
  "id": "fork-custom-scorer",
  "type": "platform",
  "status": "built",
  "label": "Custom domain scorer",
  "origin": "local",
  ...
}

{
  "id": "bigquery-warehouse",
  "type": "db",
  "status": "built",
  "label": "BigQuery — data warehouse",
  "origin": "fork-override",    // fork changed the description field
  ...
}
```

The renderer uses `origin` to visually distinguish upstream nodes from fork-local ones, and to highlight nodes the fork has modified. This gives the fork's maintainers — and anyone reviewing a PR between fork and upstream — an immediate view of what the fork has changed architecturally.

---

## What forks can do

### Add new components

Declare new nodes that exist only in the fork:

```yaml
# fork's .autoarch/definitions.yml
components:
  - id: fork-custom-scorer
    type: platform
    status: built
    label: "Custom domain scorer"
    description: "Fork-specific scoring logic for healthcare vertical"
    tags: [fork, healthcare]
```

Or via annotation in the fork's source:

```go
// @arch:component id:fork-custom-scorer type:platform status:built label:"Custom domain scorer"
func NewCustomScorer(cfg Config) *CustomScorer {
```

These produce nodes with `origin: local` in the merged manifest.

---

### Connect fork components to upstream components

Reference upstream node IDs directly — no redeclaration needed:

```yaml
connections:
  - from: fork-custom-scorer
    to: bigquery-warehouse      # upstream node — available by ID
    label: ingress

  - from: fork-custom-scorer
    to: postgres-main           # upstream node
    bidir: true
```

Or via annotation:

```go
// @arch:component id:fork-custom-scorer type:platform status:built
// @arch:connects to:bigquery-warehouse label:ingress
// @arch:connects to:postgres-main bidir:true
func NewCustomScorer(cfg Config) *CustomScorer {
```

These edges carry `origin: local` in the merged manifest — they are the fork's additions.

---

### Override upstream component fields

Use the `!` prefix to assert that the fork's values should replace the upstream's on specific components:

```yaml
components:
  - !id: scoring-engine         # upstream component
    status: partial             # fork says it's only partial in this context
    description: "Adapted for healthcare vertical — weight schema differs"
```

Or via annotation on a reimplemented symbol:

```go
// @arch:component !id:scoring-engine status:partial description:"Adapted for healthcare vertical"
func NewCustomScorer(cfg Config) *CustomScorer {
```

Without `!`, a fork-layer declaration of an upstream component `id` that differs in field values produces a `SOT_CONFLICT` warning and resolves toward the fork layer (fork is treated as the annotation side in the merge). The `!` suppresses the warning and makes the override intentional and explicit.

Overridden components appear with `origin: fork-override` in the merged manifest.

---

### Extend upstream connections

Override individual fields on upstream edges using field-level `!`:

```yaml
connections:
  - from: run-lifecycle         # upstream node
    to: postgres-main           # upstream node
    !bidir: true                # fork asserts this field value
    !label: "extended write path"
```

Non-overridden fields on the upstream edge are preserved. The edge appears with `origin: fork-override` in the merged manifest.

---

### Add upstream components to fork groups

```yaml
groups:
  - id: healthcare-layer
    label: "Healthcare layer"
    members:
      - fork-custom-scorer      # fork-local node
      - scoring-engine          # upstream node — valid member reference
      - api-egress              # upstream node
```

Upstream nodes referenced in fork group membership appear with `origin: fork-extended` — they were not overridden, but the fork added context around them.

---

## What forks cannot do

**Forks cannot remove upstream components.** The upstream manifest is the base layer — all its nodes, edges, and groups are present in the merged manifest. A fork can override fields and add context, but cannot delete upstream architectural elements. This preserves the integrity of the upstream graph when viewed through the fork lens.

**Forks cannot change upstream node IDs.** IDs are the anchor of the entire merge model. Renaming an upstream node in a fork would break all references to it.

If a fork genuinely needs to suppress an upstream component from its rendered view, it can set `status: undef` via override and note the reason in `description`. The node remains in the manifest but renders with a gray pip indicating it is not relevant in this fork's context.

---

## Merge order in forks

```
Upstream manifest (pinned at manifest_ref)
  ↓  base layer
Fork YAML definitions
  ↓  merges into
Fork code annotations
  ↓  merges into
Fork inferred edges (import graph — fork codebase only)
  ↓  filtered by
Fork @arch:override deny: rules
  ↓
Merged fork manifest
```

Conflict resolution follows the standard [matrix](./annotation-grammar.md#conflict-resolution) with one addition: the upstream manifest is treated as the YAML side of any conflict. Fork-layer annotations and YAML definitions are treated as the annotation side. This means:

- Fork annotations without `!` win on `status` (fork code is ground truth for what's built in the fork)
- Fork YAML without `!` wins on `type` and `label` (unless upstream has `!id:`)
- `!` on either side follows the standard override rules

---

## Updating the upstream pin

When the upstream repository releases new architecture, the fork updates `manifest_ref` in `.autoarch/config.yml` to the new upstream manifest commit. The next scan merges the new upstream base with the fork layer and surfaces any new conflicts as warnings.

This is an intentional manual step — the fork controls when it absorbs upstream changes, rather than silently tracking `HEAD`. The GitHub Action can be configured to open a PR when a new upstream manifest is detected, similar to how Dependabot handles dependency updates.

---

## Fork manifest example

Given an upstream manifest with `scoring-engine`, `bigquery-warehouse`, and `postgres-main`, and a fork that adds `fork-custom-scorer` and overrides `scoring-engine`'s description:

```jsonc
{
  "version": "1.0",
  "project": "wfai-platform-healthcare",
  "upstream": {
    "repo_url": "https://github.com/autoarchdx/wfai-platform",
    "manifest_ref": "abc123def456",
    "merged_at": "2026-03-19T05:00:00Z"
  },
  "nodes": [
    {
      "id": "scoring-engine",
      "type": "platform",
      "status": "built",
      "label": "Scoring engine",
      "description": "Adapted for healthcare vertical — weight schema differs",
      "origin": "fork-override",
      "source": "both",
      "provenance": {
        "annotation": {
          "source": "annotation",
          "file": "internal/scoring/engine.go",
          "line": 14,
          "symbol": "NewScoringEngine",
          "symbol_kind": "function",
          "branch": "main"
        },
        "yaml": {
          "source": "yaml",
          "file": ".autoarch/definitions.yml",
          "yaml_key": "components[0]",
          "branch": "main"
        }
      }
    },
    {
      "id": "fork-custom-scorer",
      "type": "platform",
      "status": "built",
      "label": "Custom domain scorer",
      "origin": "local",
      "source": "annotation",
      "provenance": {
        "source": "annotation",
        "file": "internal/scoring/custom.go",
        "line": 8,
        "symbol": "NewCustomScorer",
        "symbol_kind": "function",
        "branch": "main"
      }
    },
    {
      "id": "bigquery-warehouse",
      "type": "db",
      "status": "built",
      "label": "BigQuery — data warehouse",
      "origin": "upstream",
      "source": "yaml",
      "provenance": {
        "source": "yaml",
        "file": ".autoarch/manifest.json",
        "yaml_key": "nodes[1]"
      }
    }
  ],
  "edges": [
    {
      "id": "e:fork-custom-scorer->bigquery-warehouse",
      "from": "fork-custom-scorer",
      "to": "bigquery-warehouse",
      "kind": "declared",
      "bidir": false,
      "label": "ingress",
      "origin": "local",
      "provenance": {
        "source": "annotation",
        "file": "internal/scoring/custom.go",
        "line": 9,
        "branch": "main"
      }
    }
  ],
  "groups": [],
  "warnings": []
}
```
