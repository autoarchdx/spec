# autoarchdx/spec

This repository is the authoritative specification for AutoArchDX — a tool for keeping architectural documentation alive inside the codebase that it describes.

If the scanner and this spec disagree, the spec is right.

---

## What is AutoArchDX?

Architecture diagrams lie. Not because people intend them to, but because they live in wikis and slide decks and Confluence pages — separate from the code, outside the review process, untethered from the commit history. They drift. Nobody notices until someone makes a decision based on a diagram that stopped being true six months ago.

AutoArchDX puts architectural intent back into the source files where it belongs. Engineers annotate their code with lightweight `@arch:*` comments. Architects stub out external systems and groupings in a small YAML definition file. A scanner reads both, merges them with an import graph, and produces a single manifest — a versioned, diffable, reviewable record of the architecture as it actually exists.

The manifest feeds a renderer. The renderer produces a live diagram. The GitHub Action keeps the manifest current automatically, opening a pull request whenever the architecture changes. The diagram is always in sync because the diagram is generated from the code.

---

## The two personas

AutoArchDX is designed for two people who need to work together without getting in each other's way:

**The architect** thinks in systems. They stub out the full architecture in a YAML definition file before implementation begins — here are the twelve components, here are the edges between them, here are the groupings. The YAML file is their medium. They can declare everything upfront, and the scanner's warning list becomes an implementation progress tracker: `MISSING_IMPL` warnings appear for every unimplemented component and disappear as engineers annotate their work.

**The engineer** lives in the code. They implement a component, drop an `@arch:component` annotation above the function, and move on. They don't want to edit a separate file. They want ownership to flow naturally from writing the implementation.

When both touch the same component, the scanner detects the conflict and surfaces it as a warning — pointing at the annotation's file and line so it's easy to find and resolve. A `!` prefix on either side asserts authority and resolves the conflict cleanly. Either party can defer to the other explicitly. Nothing is silently discarded.

---

## What's in this repository

```
spec/
  README.md                  — you are here
  schema/
    manifest.schema.json     — formal JSON Schema for the generated manifest
  docs/
    annotation-grammar.md    — @arch:* syntax reference
    yaml-definition-spec.md  — .autoarch/definitions.yml format
    manifest-spec.md         — full manifest format, fields, and warning reference
    provenance-model.md      — file, line, symbol, commit, branch, and drift detection
    fork-model.md            — upstream layering for forked repositories
    github-action.md         — action lifecycle, skip signals, PR format
```

---

## How to read this spec

**Start with [annotation-grammar.md](./docs/annotation-grammar.md).** It covers the `@arch:*` annotations that go directly in source code — the primary input to the scanner. Five directives: `@arch:component`, `@arch:connects`, `@arch:group`, `@arch:override`, and the override assertion syntax using `!`. If you only read one document, read this one.

**Then read [yaml-definition-spec.md](./docs/yaml-definition-spec.md).** It covers the `.autoarch/definitions.yml` format for external components, groups, and connections that don't belong in a specific source file. Anything expressible as an annotation is also expressible in YAML — the two surfaces are intentionally equivalent.

**The [manifest spec](./docs/manifest-spec.md) is the output contract.** It describes the shape of `.autoarch/manifest.json` — the file the scanner produces and the renderer consumes. Read this when building tooling that consumes manifests.

**The [provenance model](./docs/provenance-model.md)** explains how every manifest element carries a traceable record back to its source: file, line, symbol, commit hash, `introduced_at`, branch, and author. This is the foundation for drift detection and renderer navigation.

**The [fork model](./docs/fork-model.md)** covers how forked repositories extend an upstream architecture — referencing upstream node IDs without redeclaring them, overriding fields with `!`, and tracking the `origin` of each element (`local`, `upstream`, `fork-override`, `fork-extended`).

**The [GitHub Action spec](./docs/github-action.md)** describes the full CI lifecycle: skip signals, the diff-and-PR workflow, the merge guard that prevents recursive triggering, and configuration options.

---

## Key design decisions

**Annotations are co-located with code.** The documentation lives in the same file as the implementation, subject to the same review and the same diff. There is no separate documentation repository to keep in sync.

**One component, many annotations.** An `id` slug identifies a logical component, not a single file or symbol. Multiple `@arch:component` annotations sharing the same `id` are additive — they contribute symbols to the same node. This is the intended pattern for any non-trivial component that spans multiple functions, structs, or files.

**Declared vs inferred edges are distinct.** An `@arch:connects` annotation declares *intent* — this component is designed to talk to that one. An edge inferred from the import graph declares *structural reality* — the scanner observed this dependency. They are both in the manifest and visually differentiated in the renderer. Architects can suppress inferred edges that are not architecturally meaningful with `@arch:override deny:`.

**The `!` prefix asserts authority.** When an annotation and a YAML definition conflict, `!id:slug` on the annotation means "annotation wins." `!id: slug` on the YAML definition means "YAML wins." Without `!` on either side, the annotation wins on `status` (code is ground truth for what's built) and YAML wins on `type` and `label` (architect owns the taxonomy). All conflicts are surfaced as warnings pointing at the annotation's file and line.

**The manifest is diffable by design.** Arrays are sorted deterministically. IDs are stable slugs. Running the scanner twice on the same codebase produces identical manifests (modulo the `generated` timestamp). A manifest diff means something architectural actually changed — which is what makes the GitHub Action's PR workflow meaningful.

---

## Versioning

The spec is versioned with semver. The current version is **0.1.0** (pre-release — breaking changes expected while the scanner and renderer are being built).

| Change type | Version bump |
|-------------|-------------|
| Removed directives, changed required fields, renamed warning codes | Major |
| New optional fields, new directives, new warning codes | Minor |
| Clarifications, examples, wording | Patch |

All AutoArchDX tools embed the spec version they implement. The scanner emits `version` and `scanner_version` in every manifest. Downstream tools can detect version mismatches and warn accordingly.

---

## Related repositories

| Repository | Description |
|------------|-------------|
| [autoarchdx/scanner](https://github.com/autoarchdx/scanner) | Go CLI and library — scans repos, produces manifests |
| [autoarchdx/renderer](https://github.com/autoarchdx/renderer) | Standalone HTML renderer — takes a manifest, produces a diagram |
| [autoarchdx/action](https://github.com/autoarchdx/action) | GitHub Action — diff-and-PR workflow for keeping manifests current |
| [autoarchdx/docs](https://github.com/autoarchdx/docs) | Public documentation site |

---

## Contributing

This spec is intentionally small and precise. Changes should be minimal and deliberate — every addition to the grammar or manifest shape has to be implemented by the scanner and handled by the renderer.

If you are proposing a change:

1. Open an issue describing the use case that motivates it
2. Changes to the annotation grammar or manifest shape require corresponding updates to `schema/manifest.schema.json`
3. All breaking changes require a major version bump and a `CHANGELOG.md` entry
4. Examples in the spec should be real — taken from or directly applicable to actual codebases

The spec documents decisions that have already been made through design discussion. If you disagree with a fundamental decision, open an issue — don't send a PR that silently changes it.