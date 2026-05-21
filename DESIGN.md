# Terraform state splitter design

This document describes the design of `terraform-state-splitter`, a tool and
library that converts a [Terraform](https://developer.hashicorp.com/terraform)
JSON state file into a tree of per-resource-instance YAML files (and back),
so that state changes produce small, reviewable diffs in a VCS.

It is the Terraform-side counterpart of
[pulumi_state_splitter](https://github.com/house-reliability-engineering/pulumi/tree/main/utilities/pulumi_state_splitter)
and is intended to interoperate with
[gitolize](https://github.com/house-reliability-engineering/gitolize),
while remaining fully usable without it.

The authoritative behavioural contract lives in [REQUIREMENTS.md](REQUIREMENTS.md);
this document only covers *how* those requirements are met.

## Goals

- Round-trip a Terraform V4 JSON state to a split YAML tree with no semantic loss.
- Produce diffs that scope cleanly to a single resource instance.
- Encrypt the same fields that
  [sensitive_annotator](https://github.com/vertexinc/infrastructure-state/blob/master/bin/terraform/sensitive_annotator.py)
  encrypts today, without polluting the state keys with marker suffixes.
- Be usable both as a CLI and as an embeddable Go library.
- Be safe to run from automation that already wraps `terraform` invocations.

## Non-goals

The items listed under
[REQUIREMENTS.md#Non-requirements](REQUIREMENTS.md#non-requirements)
are explicitly out of scope; in particular all atomicity, locking, backup,
checksumming and diffing concerns are delegated to
[gitolize](https://github.com/house-reliability-engineering/gitolize).

## Language choice: Go

We prefer [Go](https://go.dev/) over [Rust](https://www.rust-lang.org/) for this
project for the following reasons:

- [Terraform](https://github.com/hashicorp/terraform) itself is written in Go,
  so we can import its
  [`internal/states/statefile`](https://github.com/hashicorp/terraform/blob/main/internal/states/statefile)
  package (vendored, since it is in `internal/`) instead of re-implementing or
  re-deriving the V4 state schema. This eliminates a whole class of
  schema-drift bugs.
- [SOPS](https://github.com/getsops/sops) — the encryption tool used by
  `sensitive_annotator` and by our wider tooling — is written in Go and
  exposes a stable [Go SDK](https://pkg.go.dev/github.com/getsops/sops/v3),
  letting us encrypt/decrypt specific YAML nodes in-process rather than
  shelling out.
- The SRE team already maintains Go tooling for infrastructure and the
  build/distribution story (single static binary, cross-compilation) is
  simpler than Rust's for our deployment targets.
- Go's standard library covers everything else we need (JSON, filesystem,
  `os/exec` for the `run` wrapper).

The cost — losing Rust's stronger compile-time guarantees — is acceptable
because the data model is small and well-tested round-tripping gives us
strong runtime guarantees regardless.

### Vendoring Terraform's state package

`internal/states/statefile` is not part of Terraform's public API. We vendor
the minimum subset required to parse and serialize V4 state, pinned to a
specific Terraform release tag, and bump it deliberately. The vendored copy
lives under `internal/terraform/` with the upstream `LICENSE` and a
`VERSION` file documenting the source revision. Re-use of the upstream code
is preferred over re-implementation because:

- the V4 schema has subtle invariants (e.g. `dependencies` ordering,
  `sensitive_attributes` paths) that we get for free;
- bug fixes and additions in upstream flow to us through vendor bumps;
- the upstream package already handles forward/backward state version
  migration if we ever read a non-V4 file.

## Dependencies

| Concern               | Choice                                                                                            | Rationale                                                                                                                              |
| --------------------- | ------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| State schema & (de)serialization | Vendored [`terraform/internal/states/statefile`](https://github.com/hashicorp/terraform/tree/main/internal/states/statefile) | Source of truth for the V4 schema.                                                                                                     |
| YAML encoding         | [`sigs.k8s.io/yaml`](https://pkg.go.dev/sigs.k8s.io/yaml)                                         | Wraps [`gopkg.in/yaml.v3`](https://pkg.go.dev/gopkg.in/yaml.v3) but goes through JSON, which keeps the on-disk YAML semantically identical to the JSON state and reuses Terraform's existing JSON struct tags. |
| Field-level encryption | [`github.com/getsops/sops/v3`](https://pkg.go.dev/github.com/getsops/sops/v3) SDK                | Lets us encrypt selected nodes in place, driven by Terraform's own `sensitive_attributes` paths, without inventing key suffixes.       |
| CLI                   | [`github.com/spf13/cobra`](https://github.com/spf13/cobra)                                        | Matches the rest of our Go tooling; subcommand model fits `split`/`unsplit`/`run`.                                                     |
| Filesystem abstraction | `io/fs` + a thin in-package interface                                                            | Enables in-memory testing and avoids temporary files per [AGENTS-CODING.md](../../vertexinc/taxamo-home-filip/AGENTS-CODING.md).        |
| Testing               | Standard `testing` + [`github.com/google/go-cmp`](https://pkg.go.dev/github.com/google/go-cmp)    | Built-in tooling; `cmp` gives readable diffs for round-trip tests.                                                                     |
| Linting               | [`golangci-lint`](https://golangci-lint.run/) with `gofmt`, `govet`, `staticcheck`, `errcheck`    | Standard Go toolchain.                                                                                                                 |
| Pre-commit hooks      | [`pre-commit`](https://pre-commit.com/) running `gofmt`, `golangci-lint`, `go test`, `go vet`     | Enforces the rules from [AGENTS-CODING.md](../../vertexinc/taxamo-home-filip/AGENTS-CODING.md).                                        |

## Repository layout

```
terraform-state-splitter/
├── cmd/terraform-state-splitter/   # main package, thin wrapper around cli
├── pkg/
│   ├── cli/                        # cobra commands, importable for embedding
│   ├── splitter/                   # split / unsplit pure functions over in-memory trees
│   ├── layout/                     # path conventions (project/deployment/...)
│   ├── sensitive/                  # SOPS-backed encrypt/decrypt of YAML nodes
│   ├── stored/                     # discovery and IO of split state on a filesystem
│   └── statefile/                  # facade over the vendored terraform package
├── internal/terraform/             # vendored terraform statefile subset
├── DESIGN.md
├── REQUIREMENTS.md
└── README.md
```

The split between `pkg/` (public, importable as a library) and
`internal/` (vendored or private) follows
[Go's standard project conventions](https://go.dev/doc/modules/layout).
Each requirement in [REQUIREMENTS.md](REQUIREMENTS.md#requirements) about
library reusability maps to one `pkg/` subpackage:

- "loading split state (including decryption)" → `pkg/stored` + `pkg/sensitive`.
- "loading unsplit state" → `pkg/statefile`.
- "splitting" / "unsplitting" → `pkg/splitter`.
- "wrapping commands" → `pkg/cli` (the `run` subcommand).

## On-disk layout

For a Terraform state read from `<project>/<deployment>.json`, the split tree
under `<project>/<deployment>/` mirrors the requirements:

- Singleton resource:
  `<project>/<deployment>/<module-path?>/<type>/<name>.yaml`
- Resource with `for_each` / `count`:
  - `<project>/<deployment>/<module-path?>/<type>/<name>/meta.yaml`
    holds the per-resource metadata (`module`, `mode`, `type`, `name`,
    `provider`) and the ordered list of instance index keys.
  - `<project>/<deployment>/<module-path?>/<type>/<name>/<index>.yaml`
    holds one instance each.

`<module-path>` is the dotted Terraform module address (e.g. `module.foo.module.bar`)
converted to nested directories (`module.foo/module.bar/`) so that diffs and
filesystem navigation align with Terraform's mental model.

The top-level state envelope (`version`, `terraform_version`, `serial`,
`lineage`, `outputs`, `check_results`, …) is written to
`<project>/<deployment>/meta.yaml`. When splitting, child collections in
parent structures are replaced with the path of the file each child was
written to, so that an unsplit can reconstruct the exact JSON by inlining
those files. This applies recursively: resources are children of the state
envelope; instances are children of a `for_each`/`count` resource.

YAML schemas mirror the Terraform V4 Go structs field-for-field; we rely on
`sigs.k8s.io/yaml` going through JSON so the existing
[`json` struct tags](https://github.com/hashicorp/terraform/blob/main/internal/states/statefile/version4.go)
drive both formats and we don't maintain a parallel schema.

## Splitting and unsplitting

Both operations are pure in-memory transforms over a `*statefile.File`
(unsplit, in Terraform's own types) and a `splitter.Tree` (split, a map
from relative path to YAML-ready value). IO is performed by `pkg/stored`
against an `io/fs.FS`-plus-writer abstraction so that:

- production code uses an `os`-backed implementation rooted at the
  backend directory;
- tests use an in-memory implementation, satisfying the
  [AGENTS-CODING.md](../../vertexinc/taxamo-home-filip/AGENTS-CODING.md)
  rule against temporary files.

`split` is deterministic: map iteration is replaced with sorted-key
iteration and instance ordering follows the order in the source state to
keep diffs minimal.

`unsplit` is the inverse: it walks `meta.yaml` files, resolves the
file-reference placeholders, and feeds the reassembled tree back through
`pkg/statefile` so the result is parsed and re-serialized by Terraform's
own code, guaranteeing byte-equivalence with what Terraform would write.

## Handling sensitive values

[`sensitive_annotator`](https://github.com/vertexinc/infrastructure-state/blob/master/bin/terraform/sensitive_annotator.py)
today renames sensitive keys with a marker suffix so that
[SOPS](https://github.com/getsops/sops) can match them by regex. We avoid
that because it (a) mutates state keys and (b) couples encryption policy to
spelling conventions.

Instead, `pkg/sensitive` uses the SOPS Go SDK directly:

- During `split`, for each resource instance we read Terraform's own
  `sensitive_attributes` (a list of attribute paths the provider marked as
  sensitive) and pass those paths to the SOPS SDK as the set of nodes to
  encrypt within the instance's YAML document. The encrypted document
  carries SOPS metadata in a `sops:` key, exactly like a hand-edited
  SOPS-encrypted YAML file, so it round-trips through `sops` CLI as well.
- During `unsplit`, each instance file is decrypted through the same SDK
  before reassembly. Files without a `sops:` block are passed through
  unchanged, which keeps the tool usable on unencrypted backends.

Key material configuration (`.sops.yaml`) is read by the SDK in the usual
way from the working tree, so the encryption policy stays declarative and
shared with the rest of our tooling.

## CLI

`terraform-state-splitter` exposes a [cobra](https://github.com/spf13/cobra)
command tree modelled on `pulumi_state_splitter`:

```
terraform-state-splitter [-d BACKEND_DIRECTORY] <command>
  split    [-s PROJECT/DEPLOYMENT ...]
  unsplit  [-s PROJECT/DEPLOYMENT ...]
  run      [-s PROJECT/DEPLOYMENT ...] -- <command> [args...]
```

- `split` discovers `<project>/<deployment>.json` files under the backend
  directory and writes the corresponding split trees.
- `unsplit` is the inverse.
- `run` `unsplit`s the selected deployments, executes the wrapped command
  (typically `terraform …`), and `split`s the result back. It is the
  primary entry point for day-to-day use and the integration point with
  [gitolize](https://github.com/house-reliability-engineering/gitolize):
  gitolize handles atomicity, locking, backup and diffing around the
  invocation; we only handle the format conversion in between.

The `run` command propagates the wrapped process's exit code so it composes
with shell pipelines and CI.

## Gitolize compatibility

We make no calls into gitolize and have no runtime dependency on it; the
split tree is just files on disk. When gitolize is present, it observes the
file changes produced by `split` and handles VCS interaction; when it is
not, the same files can be committed manually or consumed by any other
tool. The non-requirements list in [REQUIREMENTS.md](REQUIREMENTS.md)
explicitly delegates atomicity, backup, locking and integrity to gitolize.

## Testing strategy

- **Round-trip tests**: for every fixture state, assert
  `unsplit(split(state)) == state` and
  `split(unsplit(tree)) == tree`, using
  [`go-cmp`](https://pkg.go.dev/github.com/google/go-cmp) for diffs.
- **Schema tests**: feed sample V4 states (including ones with `for_each`,
  `count`, nested modules, sensitive attributes, multiple providers)
  through the vendored Terraform parser to ensure our facade exposes
  everything we need.
- **SOPS tests**: use the SDK with a test [age](https://github.com/FiloSottile/age)
  key (committed under `testdata/`) to encrypt/decrypt without external
  dependencies.
- **CLI tests**: exercise cobra commands against an in-memory filesystem.
- Per [AGENTS-CODING.md](../../vertexinc/taxamo-home-filip/AGENTS-CODING.md),
  we target 100% meaningful coverage and enforce it via `go test -cover`
  in pre-commit and CI.

## Open questions

- Whether to also vendor Terraform's `states` package (richer in-memory
  model than `statefile`) for callers that want to manipulate the parsed
  state. Initial implementation will not, to keep the vendored surface
  small; we will revisit if a concrete consumer needs it.
- Whether the `run` subcommand should split only deployments touched by
  the wrapped command. Initial implementation will split all selected
  deployments unconditionally; selectivity is an optimisation, not a
  correctness concern.
