# YAML definition spec

YAML definition files are the companion to [code annotations](./annotation-grammar.md). Where annotations live inside source files and describe components that _exist in the codebase_, YAML definitions describe components that exist _outside_ it: third-party APIs, managed cloud services, external data providers, infrastructure that isn't annotatable.

They are also the preferred mechanism for declaring groups, for bulk-assigning group membership, and for any architectural intent that doesn't have a natural home in a specific source file.

Anything expressible as an annotation is expressible as a YAML definition, and vice versa. The two surfaces are intentionally equivalent in expressive power — the choice of where to declare something is a workflow decision, not a capability decision.

---

## File location and discovery

Definition files must be placed in the `.autoarch/` directory at the repository root:

```
your-repo/
  .autoarch/
    definitions.yml        # primary definition file
    definitions.*.yml      # additional definition files (all are merged)
    manifest.json          # generated — do not edit by hand
  src/
    ...
```

The scanner merges all `.autoarch/definitions.*.yml` files in lexicographic order, then merges the result with code annotations. Duplicate IDs across definition files produce a `DUPLICATE_ID` warning; the first definition (lexicographically) wins.

A single `definitions.yml` is sufficient for most projects. Splitting into multiple files (`definitions.external.yml`, `definitions.groups.yml`) is a style choice for larger architectures.

---

## Top-level structure

```yaml
# .autoarch/definitions.yml
version: "1.0"

components:
  - ...

connections:
  - ...

groups:
  - ...
```

All three top-level keys are optional. A valid definition file can contain any combination of them.

---

## Components

Each entry in `components` declares a node in the architecture graph, or contributes to an existing node already declared by an annotation.

**Required fields:**

| Field | Required on first declaration | Description |
|-------|------------------------------|-------------|
| `id` | yes | Stable human slug. Lowercase kebab-case. The anchor for conflict detection and cross-source merging. |
| `type` | yes | `platform` `api` `db` `client` `datasrc` `partner` `group` |
| `label` | yes | Display name shown in the rendered node. |

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| `status` | enum | `built` `partial` `planned` `undef`. Defaults to `undef` if omitted. |
| `description` | string | Short subtitle. ~60 chars max for clean rendering. |
| `tags` | list of strings | Merged union across all sources. |
| `group` | string | ID of a group this component belongs to. |
| `url` | string | External URL (docs, dashboard, API reference). Surfaced in the renderer's node detail panel. |
| `owner` | string | Team or person responsible. Surfaced in provenance. |
| `layout` | object | Renderer position hint: `{x: number, y: number}`. Overridable by the user in the renderer. |

**Example:**

```yaml
components:
  - id: bigquery-warehouse
    type: db
    status: built
    label: "BigQuery — data warehouse"
    description: "All datasets converge here · single query surface"
    tags: [data, gcp]
    layout:
      x: 200
      y: 200

  - id: bls-oews
    type: datasrc
    status: planned
    label: "Gov labor data"
    description: "BLS, OEWS"
    tags: [external, labor-data]

  - id: lightcast
    type: partner
    status: planned
    label: Lightcast
    description: "Labor market enrichment"
    url: https://lightcast.io/docs
    owner: data-team
```

---

## The `!` override prefix

When a YAML definition and a code annotation both declare the same component (matched by `id`), the scanner applies [conflict resolution](./annotation-grammar.md#conflict-resolution). The `!` prefix on a YAML component's `id` asserts that the YAML definition is authoritative — its values win over the annotation on all conflicting fields.

**Asserting override from YAML:**

```yaml
components:
  - !id: bls-oews
    type: datasrc
    status: built
    label: "Gov labor data"
    description: "BLS, OEWS"
    tags: [external, labor-data]
```

`!id:` means: YAML values win on all conflicting fields. Scanner emits `SOT_OVERRIDE` at `info` level, pointing to the annotation for traceability.

**Without override assertion:**

```yaml
components:
  - id: bls-oews
    type: datasrc
    status: built
```

If an annotation also declares `id:bls-oews` with a different `status`, the scanner emits `SOT_CONFLICT` at `warn` level, resolves toward the annotation, and points the warning at the annotation's file and line.

The full conflict resolution matrix is defined in the [annotation grammar](./annotation-grammar.md#conflict-resolution). The YAML and annotation surfaces are symmetric — `!id:` in an annotation and `!id:` in YAML are equivalent assertions from opposite sides.

---

## Connections

Each entry in `connections` declares an edge between two components. YAML-defined connections follow the same semantics as `@arch:connects` annotations — they produce `kind: declared` edges in the manifest.

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `from` | node id | Source component. Must resolve to a known ID. |
| `to` | node id | Target component. Must resolve to a known ID. |

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| `label` | string | Edge label shown in the renderer. |
| `bidir` | boolean | Bidirectional connection. Defaults to `false`. |
| `description` | string | Longer explanation. Present in manifest, not shown in renderer. |

**Example:**

```yaml
connections:
  - from: bls-oews
    to: bigquery-warehouse
    label: ingress

  - from: lightcast
    to: bigquery-warehouse
    label: enrichment

  - from: run-lifecycle
    to: postgres-main
    bidir: true
```

**Connection field overrides:**

Individual fields can be asserted as authoritative using the `!` prefix on the field name — mirroring `!to:` and `!bidir:` in annotations:

```yaml
connections:
  - from: scoring-engine
    !to: bigquery-warehouse
    !bidir: true
    label: ingress
```

Only `!`-prefixed fields are treated as override assertions. Unprefixed fields follow normal conflict resolution.

---

## Groups

Each entry in `groups` declares a logical container. Groups render as dashed bounding boxes in the diagram. They have no edges and are not themselves nodes in the graph.

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Stable identifier. Referenced by component `group` fields. |
| `label` | string | Display name shown on the dashed container. |

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| `description` | string | Optional subtitle rendered inside the container header. |
| `members` | list of node ids | Explicit membership. Components can also self-assign via their `group` field — both are merged. |
| `layout` | object | `{x, y, width, height}` for the renderer. |

**Example:**

```yaml
groups:
  - id: orchestration-layer
    label: "Orchestration layer"
    members:
      - run-lifecycle
      - schema-config
      - run-cache
      - confidence-signal
    layout:
      x: 40
      y: 310
      width: 700
      height: 400

  - id: client-layer
    label: "Client layer"
    members:
      - results-map
      - visualizations
      - narrative-insights
      - weight-adjustment
```

---

## Merge semantics

The scanner merges sources in this order:

```
YAML definitions
  ↓  merges into
Code annotations
  ↓  merges into
Inferred edges (import graph)
  ↓  filtered by
@arch:override deny: rules
  ↓
Manifest
```

**Field-level merge rules (no `!` on either side):**

| Field | Default winner | Rationale |
|-------|---------------|-----------|
| `id` | First defined (YAML for external nodes) | Identity must be stable |
| `type` | YAML | Architect owns the taxonomy |
| `label` | YAML | Architect owns the display name |
| `status` | Annotation | Code is ground truth for what's built |
| `description` | YAML unless annotation explicitly sets it | Architect owns the narrative |
| `tags` | Merged union | Both sources contribute |
| `group` | Merged union | A component may belong to multiple groups |
| `connections` | Additive | All declared connections appear in the manifest |
| `layout` | Annotation (user-positioned) | User-adjusted positions persist |

When `!` is present on either side, see the [conflict resolution matrix](./annotation-grammar.md#conflict-resolution).

---

## ID conventions

IDs must:
- Be lowercase kebab-case
- Refer to exactly one logical component — multiple YAML entries with the same `id` are an error
- Be stable — changing an `id` changes the node's identity in the manifest and breaks all references

Multiple annotations sharing an `id` is valid and expected — they are additive declarations of the same component. Multiple YAML entries sharing an `id` is always an error (`DUPLICATE_ID` warning, first definition wins).

---

## Stub-driven development

YAML definitions support an architect-first workflow. Stub out the full architecture in YAML before implementation begins, then claim each component with annotations as it gets built.

A component stubbed in YAML with no corresponding annotation and status other than `undef` produces a `MISSING_IMPL` warning. This warning disappears when an engineer annotates the implementation. The warning list becomes the implementation progress tracker.

```yaml
components:
  - id: scoring-engine
    type: platform
    status: planned        # MISSING_IMPL until annotated
    label: "Scoring engine"
    description: "Weighted fields, explanation factors"

  - id: run-cache
    type: platform
    status: planned        # MISSING_IMPL until annotated
    label: "Run cache"
    description: "Idempotency key lookup"
```

When the engineer implements and annotates:

```go
// @arch:component id:scoring-engine type:platform status:built label:"Scoring engine"
func NewScoringEngine(...) *ScoringEngine {
```

The `MISSING_IMPL` warning for `scoring-engine` resolves. The `status` conflict (`planned` vs `built`) resolves toward the annotation per default merge rules — no override needed. This is the intended workflow.

---

## Fork extensions

In a forked repository, YAML definitions can extend the upstream architecture by referencing upstream node IDs. The fork does not need to redeclare upstream components — they are available from the upstream manifest referenced in the envelope.

```yaml
# Fork-local definitions.yml

components:
  - id: fork-custom-scorer
    type: platform
    status: built
    label: "Custom domain scorer"
    description: "Fork-specific scoring logic"

connections:
  - from: fork-custom-scorer
    to: bigquery-warehouse      # declared in upstream manifest
    label: ingress

  - from: run-lifecycle         # upstream node
    to: postgres-main           # upstream node
    !bidir: true                # fork asserts this field
```

See [fork model](./fork-model.md) for the full layering semantics.

---

## Validation warning reference

| Code | Severity | Cause |
|------|----------|-------|
| `DUPLICATE_ID` | `warn` | Two YAML entries declare the same `id` |
| `BROKEN_REF` | `warn` | A connection references an ID not found in any source |
| `AMBIGUOUS_LABEL` | `warn` | A label used in a connection matches more than one component |
| `INVALID_TYPE` | `warn` | `type` value not in the allowed enum |
| `INVALID_STATUS` | `warn` | `status` value not in the allowed enum |
| `MISSING_REQUIRED` | `warn` | A required field (`id`, `type`, `label`) is absent |
| `MISSING_IMPL` | `warn` | Component has no annotation and status is not `undef` |
| `SOT_CONFLICT` | `warn` | Field conflict with annotation, no `!` on either side |
| `SOT_OVERRIDE` | `info` | One side declared `!`, resolved cleanly |
| `SOT_DUAL_OVERRIDE` | `warn` | Both sides declared `!` simultaneously |

---

## Full example

```yaml
# .autoarch/definitions.yml
version: "1.0"

components:
  - id: bls-oews
    type: datasrc
    status: planned
    label: "Gov labor data"
    description: "BLS, OEWS"
    tags: [external, labor-data]

  - id: lightcast
    type: partner
    status: planned
    label: Lightcast
    description: "Labor market enrichment"
    url: https://lightcast.io/docs
    owner: data-team

  - id: client-hr-data
    type: datasrc
    status: planned
    label: "Client HR data"
    description: "Phase 3"
    tags: [external, phase-3]

  - id: bigquery-warehouse
    type: db
    status: planned
    label: "BigQuery — data warehouse"
    description: "All datasets converge here"
    tags: [data, gcp]
    layout:
      x: 200
      y: 200

connections:
  - from: bls-oews
    to: bigquery-warehouse
    label: ingress

  - from: lightcast
    to: bigquery-warehouse
    label: enrichment

  - from: client-hr-data
    to: bigquery-warehouse

groups:
  - id: orchestration-layer
    label: "Orchestration layer"
    members:
      - run-lifecycle
      - schema-config
      - run-cache
      - confidence-signal
    layout:
      x: 40
      y: 310
      width: 700
      height: 400

  - id: data-sources
    label: "Data sources"
    layout:
      x: 40
      y: 20
      width: 720
      height: 120
```

---

## Versioning

Definition files declare their schema version in the top-level `version` field. The scanner warns if the definition file version does not match its supported version. Breaking changes to the definition format increment the major version alongside the annotation grammar.
