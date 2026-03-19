# Annotation grammar

AutoArchDX annotations are structured comments placed directly in source code. They are the primary mechanism for declaring architectural intent — what a component _is_, what it _connects to_, and what its current _status_ is.

Annotations are intentionally co-located with the code they describe. This is the core design principle: architectural documentation lives in the same file, subject to the same review, the same diff, the same version history as the implementation.

---

## Syntax overview

All annotations follow this pattern:

```
@arch:<directive> [key:value ...]
```

The `@arch:` prefix is the scanner's entry point. The directive follows immediately after the colon. Key-value pairs use colon separators and are space-delimited. Values containing spaces or colons must be quoted with double quotes.

```
key:value               — unquoted; no spaces or colons in value
key:"some value"        — quoted; spaces and colons allowed inside quotes
!key:value              — override assertion; see conflict resolution
```

Annotations appear inside comments using whatever comment syntax the host language uses:

```go
// @arch:component id:scoring-engine type:platform status:built label:"Scoring engine"
func NewScoringEngine(...) { ... }
```

```python
# @arch:component id:inference-api type:api status:planned label:"Inference API"
class InferenceAPI:
```

```typescript
// @arch:component id:results-map type:client status:built label:"Results map"
export function ResultsMap() {
```

```yaml
# @arch:component id:bls-oews type:datasrc status:planned label:"Gov labor data"
bls_oews:
```

The scanner recognizes `//`, `#`, `--`, `/* */`, and `<!-- -->` comment styles. The annotation must appear on its own line within the comment. Inline annotations (code on the same line as an `@arch:` tag) are not supported.

---

## The `id` field

Every `@arch:component` annotation should declare an explicit `id`. The ID is the stable identity of the component across all sources — annotations, YAML definitions, and the manifest. It is how the scanner knows that multiple annotations, or an annotation and a YAML definition, are all talking about the same architectural component.

**IDs are not required to be unique across annotations.** A component is a logical unit that may span many files, many functions, many structs, and many classes. Multiple annotations sharing the same `id` are additive declarations of the same component — the scanner merges them into one node. This is the intended pattern for any non-trivial component.

```go
// @arch:component id:scoring-engine type:platform status:built label:"Scoring engine"
func NewScoringEngine(cfg Config) *ScoringEngine {

// ... elsewhere in the same or different file ...

// @arch:component id:scoring-engine
func (e *ScoringEngine) Score(ctx context.Context, input Input) (*Result, error) {

// @arch:component id:scoring-engine
func (e *ScoringEngine) ScoreWithExplanation(ctx context.Context, input Input) (*ExplainedResult, error) {
```

All three annotations contribute to the same `scoring-engine` node in the manifest. The node's `provenance` field becomes an array, with one entry per contributing annotation. Each entry records the file, line, symbol name, and git context of that specific annotation.

**What `id` uniqueness actually means:** an `id` must refer to exactly one logical component. Two different components must not share an `id`. Two annotations sharing an `id` must genuinely be describing the same component — if they conflict on `type`, `status`, or other fields without an override assertion, the scanner warns.

**ID conventions:**

- Lowercase kebab-case: `scoring-engine`, `bigquery-warehouse`, `api-egress`
- Stable across refactors — do not encode file paths, version numbers, or implementation details
- Descriptive enough to be readable in a `@arch:connects to:scoring-engine` annotation

When `id` is omitted, the scanner generates one as a hash of `file + line number`. Generated IDs are stable only as long as the annotation doesn't move. For any component that will be referenced from other annotations or from YAML definitions, always set an explicit `id`.

---

## Symbol tracking

The scanner records the symbol immediately following each annotation — the function, method, struct, class, or variable that the annotation describes. This is inferred automatically from the next non-comment, non-blank line without requiring language-specific parsing.

```go
// @arch:component id:scoring-engine type:platform status:built
func NewScoringEngine(cfg Config) *ScoringEngine {
//   ↑ scanner records symbol:"NewScoringEngine" symbol_kind:"function"
```

Symbol information appears in each annotation's provenance entry:

```jsonc
{
  "file": "internal/scoring/engine.go",
  "line": 14,
  "symbol": "NewScoringEngine",
  "symbol_kind": "function",
  ...
}
```

Supported symbol kinds: `function`, `method`, `struct`, `class`, `interface`, `type`, `variable`, `constant`, `module`, `package`. The scanner uses a best-effort heuristic — if it cannot determine the kind, it records `unknown`.

Symbol tracking enables renderer navigation (click a node, see all contributing symbols with links to source), and improves drift detection (the scanner can observe which contributing symbols have changed since `introduced_at`).

---

## Directives

### `@arch:component`

Declares a node in the architecture graph, or contributes additional symbols to an existing node. This is the foundational annotation.

**Syntax:**

```
@arch:component id:<slug> [type:<type>] [status:<status>] [key:value ...]
```

**Fields:**

| Field | Required on first declaration | Description |
|-------|------------------------------|-------------|
| `id` | recommended | Stable identity slug. Required for conflict detection and cross-file references. |
| `type` | yes, on first declaration | See [node types](#node-types). Subsequent declarations may omit it. |
| `status` | yes, on first declaration | See [status values](#status-values). Subsequent declarations may omit it. |
| `label` | no | Display name. Defaults to the symbol name following the annotation. |
| `description` | no | Short subtitle shown in the rendered node. ~60 chars max. |
| `tags` | no | Comma-separated strings. Merged across all declarations. |
| `group` | no | ID of a group this component belongs to. |

**First declaration** establishes the node's `type`, `status`, and `label`. Subsequent declarations of the same `id` contribute symbols and may update `tags` and `group`. If a subsequent declaration sets `type`, `status`, or `label` to a different value without an override assertion, the scanner emits `SOT_CONFLICT`.

**Examples:**

```go
// @arch:component id:scoring-engine type:platform status:built label:"Scoring engine" description:"Weighted fields, explanation factors" tags:core,go
func NewScoringEngine(cfg Config) *ScoringEngine {
```

```go
// @arch:component id:scoring-engine
func (e *ScoringEngine) Score(ctx context.Context, input Input) (*Result, error) {
```

```go
// @arch:component id:postgres-main type:db status:built label:PostgreSQL description:"Runs, recs, snapshots, ZCTA cache"
var db *pgxpool.Pool
```

---

### `@arch:connects`

Declares an edge between two components. Must appear in the same file as one of the two components it references. Declares _architectural intent_ — this component is designed to communicate with that one.

This is distinct from an import or call that the scanner _infers_ from the code. A declared connection says "this is intentional and known." An inferred connection says "the scanner observed this." Both appear in the manifest; they are visually differentiated in the renderer.

**Syntax:**

```
@arch:connects to:<id-or-label> [from:<id-or-label>] [key:value ...]
```

**Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `to` | yes | Target component ID or label. |
| `from` | no | Source component. Defaults to the nearest preceding `@arch:component` in the same file. |
| `label` | no | Edge label shown in the renderer. |
| `bidir` | no | `true` or `false`. Bidirectional connection. Defaults to `false`. |
| `description` | no | Explanation of the connection. Present in manifest, not shown in renderer. |

**Examples:**

```go
// @arch:component id:scoring-engine type:platform status:built label:"Scoring engine"
// @arch:connects to:bigquery-warehouse label:ingress
// @arch:connects to:postgres-main bidir:true
func NewScoringEngine(cfg Config) *ScoringEngine {
```

Multiple `@arch:connects` annotations can follow a single `@arch:component`. They all attach to the most recently declared component in the file.

**Connection field overrides:**

Individual fields on a connection can be asserted as authoritative using the `!` prefix on the key. Use this when an annotation and a YAML definition describe the same edge with conflicting field values:

```go
// @arch:connects !to:bigquery-warehouse !bidir:true label:ingress
```

Only `!`-prefixed fields are treated as override assertions. Unprefixed fields follow normal conflict resolution. The longhand equivalent:

```go
// @arch:override type:connection from:scoring-engine to:bigquery-warehouse bidir:true
```

---

### `@arch:group`

Declares a logical grouping of components. Groups render as dashed containers in the diagram. A group is not itself a node in the graph — it has no edges.

**Syntax:**

```
@arch:group id:<slug> label:<label> [key:value ...]
```

**Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `id` | yes | Stable identifier. |
| `label` | yes | Display name shown on the dashed container. |
| `description` | no | Optional subtitle. |

**Example:**

```go
// @arch:group id:orchestration-layer label:"Orchestration layer"
package orchestration
```

Members are added to a group via their `@arch:component` annotation:

```go
// @arch:component id:scoring-engine type:platform status:built group:orchestration-layer
```

---

### `@arch:override`

Handles two distinct use cases: suppressing inferred edges, and declaring deference to YAML definitions.

---

#### Suppressing inferred edges

Explicitly denies a connection that the scanner would otherwise infer from the import or call graph:

```go
// @arch:component id:scoring-engine type:platform status:built
// @arch:override deny:utils-logger reason:"Logging dependency, not architectural"
// @arch:override deny:config-loader reason:"Shared config, not a meaningful edge"
func NewScoringEngine(cfg Config) *ScoringEngine {
```

| Field | Required | Description |
|-------|----------|-------------|
| `deny` | yes | ID or label of the inferred connection to suppress. |
| `from` | no | Source node. Defaults to nearest preceding `@arch:component`. |
| `reason` | no | Human explanation. Surfaced in the manifest for audit purposes. |

---

#### Deferring to YAML definitions

An engineer implementing a component fully specified in YAML can declare that conflicts resolve toward the YAML definition:

```go
// @arch:override allow:* type:component
// @arch:component id:scoring-engine type:platform status:built
func NewScoringEngine(cfg Config) *ScoringEngine {
```

| Field | Required | Description |
|-------|----------|-------------|
| `allow` | yes | Currently only `*` (all fields). |
| `type` | no | `component` `connection` `group`. Defaults to `component`. |
| `scope` | no | `file` — applies deference to all `@arch:component` annotations in the current file. |

The scanner still records `SOT_DEFERRED` at `info` severity pointing to this annotation. The annotation's values are preserved in the manifest even when YAML wins — nothing is silently discarded.

**File-scoped deference:**

```go
// @arch:override allow:* type:component scope:file

// @arch:component id:scoring-engine type:platform status:built
func NewScoringEngine(...) { ... }

// @arch:component id:run-cache type:platform status:built
func NewRunCache(...) { ... }
```

---

## Override assertion: the `!` prefix

When an annotation and a YAML definition describe the same component and conflict, the `!` prefix on the `id` key asserts that this annotation is authoritative for the whole component:

```go
// @arch:component !id:bls-oews type:datasrc status:planned label:"Gov labor data"
```

`!id:` means: annotation values win on all conflicting fields. Scanner emits `SOT_OVERRIDE` at `info` level.

Without `!`:

```go
// @arch:component id:bls-oews type:datasrc status:planned
```

On conflict with a YAML definition: scanner emits `SOT_CONFLICT` at `warn` level, resolves toward the annotation, warning points to this file and line.

The `!` prefix on connections applies per-field:

```go
// @arch:connects !to:bigquery-warehouse !bidir:true label:ingress
```

Only `!to` and `!bidir` are asserted as authoritative. `label` follows normal resolution.

---

## Conflict resolution

When the same component (matched by `id`) is declared in both an annotation and a YAML definition:

| Annotation | YAML definition | Resolution | Warning |
|------------|-----------------|------------|---------|
| `id:slug` | `id: slug` | Annotation wins on conflicting fields | `SOT_CONFLICT` · `warn` · points to annotation |
| `!id:slug` | `id: slug` | Annotation wins on all fields | `SOT_OVERRIDE` · `info` · points to annotation |
| `id:slug` | `!id: slug` | YAML wins on all fields | `SOT_OVERRIDE` · `info` · points to annotation |
| `!id:slug` | `!id: slug` | Annotation wins on conflicting fields | `SOT_DUAL_OVERRIDE` · `warn` · points to annotation |
| `allow:*` + `id:slug` | any | YAML wins on conflicting fields | `SOT_DEFERRED` · `info` · points to annotation |

When the same component is declared by multiple annotations (same `id`, different files or symbols):

| Situation | Resolution | Warning |
|-----------|------------|---------|
| Non-conflicting fields | Merged | none |
| Conflicting fields, no `!` on either | First declaration wins | `SOT_CONFLICT` · `warn` |
| Conflicting fields, one has `!id:` | `!` annotation wins | `SOT_OVERRIDE` · `info` |
| Conflicting fields, both have `!id:` | First `!` declaration wins | `SOT_DUAL_OVERRIDE` · `warn` |

**Rules that hold in all cases:**

- Warning provenance always points to the annotation declaration — file, line, symbol, commit, branch
- Non-conflicting fields are always merged regardless of override state
- All annotation values are preserved in the manifest under `provenance` even when another source wins — nothing is silently discarded
- `SOT_CONFLICT` and `SOT_DUAL_OVERRIDE` at `warn` signal that human reconciliation is needed
- `SOT_OVERRIDE` and `SOT_DEFERRED` at `info` confirm the system resolved as intended

---

## Multi-line annotations

Continuation lines (comment lines with no `@arch:` tag immediately following an `@arch:` tag) are read as key-value extensions of the preceding directive:

```go
// @arch:component id:run-lifecycle type:platform status:built
//   label:"Run lifecycle"
//   description:"Idempotency, multi-tenant auth, provenance, audit trail"
//   tags:core,go,orchestration
//   group:orchestration-layer
// @arch:connects to:postgres-main bidir:true
// @arch:connects to:api-egress
// @arch:connects to:bigquery-warehouse label:ingress
func NewRunLifecycle(...) *RunLifecycle {
```

---

## ID resolution order

When a `@arch:connects` annotation references a target by `id` or label:

1. Exact match on `id` within the same file
2. Exact match on `id` in YAML definition files
3. Exact match on `id` across all scanned annotation files
4. Case-insensitive match on `label` across all sources
5. Unresolved → `BROKEN_REF` warning, edge not created

If a label match is ambiguous (multiple components share a label), the scanner emits `AMBIGUOUS_REF` and does not create the edge. Use `id` references to resolve ambiguity.

---

## Node types

| Type | Meaning | Typical examples |
|------|---------|-----------------|
| `platform` | Core internal service or processing component | Scoring engine, orchestration layer, run lifecycle |
| `api` | HTTP/gRPC/GraphQL interface layer | REST egress, internal service API |
| `db` | Persistent data store | PostgreSQL, BigQuery, Redis |
| `client` | Consumer-facing UI or external client | Web frontend, mobile app, embedded widget |
| `datasrc` | External data source feeding the system | BLS OEWS, pre-scored model outputs |
| `partner` | Third-party integration | Lightcast, Geode Health |
| `group` | Logical container — not a node, only for visual grouping | Orchestration layer, client layer |

---

## Status values

| Value | Meaning | Renderer |
|-------|---------|----------|
| `built` | Fully implemented and in production or active use | Glowing green pip |
| `partial` | Implementation exists but is incomplete | Dimmed green pip |
| `planned` | Designed but not yet built | Blue pip |
| `undef` | Acknowledged but not yet designed | Gray pip |

---

## Warning reference

| Code | Severity | Cause |
|------|----------|-------|
| `SOT_CONFLICT` | `warn` | Conflicting field values, no `!` on either side |
| `SOT_OVERRIDE` | `info` | One side declared `!`, resolved cleanly |
| `SOT_DUAL_OVERRIDE` | `warn` | Both sides declared `!` simultaneously |
| `SOT_DEFERRED` | `info` | Annotation declared `allow:*`, YAML values used |
| `MISSING_ID` | `warn` | `@arch:component` has no explicit `id` and a YAML definition exists |
| `BROKEN_REF` | `warn` | A connection references an unknown ID |
| `AMBIGUOUS_REF` | `warn` | A label reference matches more than one component |
| `ORPHANED_NODE` | `info` | A declared component has no edges of any kind |
| `MISSING_IMPL` | `warn` | YAML defines a component with no annotation and status is not `undef` |
| `STALE_ANNOTATION` | `info` | Annotated file modified more than threshold commits since `introduced_at` |

---

## Language support

| Language | Comment styles |
|----------|---------------|
| Go | `//` |
| Python | `#` |
| TypeScript / JavaScript | `//` `/* */` |
| Rust | `//` |
| Java / Kotlin | `//` `/* */` |
| Ruby | `#` |
| YAML | `#` |
| TOML | `#` |
| HTML / templates | `<!-- -->` |
| SQL | `--` |

---

## Versioning

The annotation grammar is versioned alongside the [manifest schema](../schema/manifest.schema.json). Breaking changes increment the major version. Additive changes increment the minor version. The scanner emits both `version` and `scanner_version` in every manifest.
