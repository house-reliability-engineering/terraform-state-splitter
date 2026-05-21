# terraform-state-splitter design

## Overview

`terraform-state-splitter` is a library and CLI for converting a Terraform state file between two representations:

- the canonical Terraform JSON state file stored at `<project-name>/<deployment-name>.json`
- a split repository tree where each resource instance is stored as a separate YAML document

The tool is intended to be the Terraform equivalent of [pulumi_state_splitter](https://github.com/house-reliability-engineering/pulumi/tree/main/utilities/pulumi_state_splitter), while also incorporating the behavior of the [sensitive annotator](https://github.com/vertexinc/infrastructure-state/blob/master/bin/terraform/sensitive_annotator.py). It must work on its own and remain compatible with [gitolize](https://github.com/house-reliability-engineering/gitolize).

## Goals

- Preserve Terraform state semantics while expressing resource instances as stable, reviewable YAML files.
- Keep the serialized schemas as close as practical to [Terraform state v4](https://github.com/hashicorp/terraform/blob/main/internal/states/statefile/version4.go) structures.
- Support round-tripping between split and unsplit forms without lossy transformations.
- Preserve sensitive field names and encrypt values in place instead of relying on synthetic key suffixes.
- Expose the implementation as reusable Go packages, not only as a CLI.

## Non-goals

- Handling file name length limits.
- A `validate` CLI command.
- Reconciling or merging with pre-existing output files.
- Re-implementing gitolize features such as locking, checksums, backups, diffing, or atomic multi-file staging.

## Language choice

Go is the preferred implementation language.

The main reason is direct alignment with Terraform's own implementation. Terraform's state v4 schema is defined in Go, and a Go implementation can either reuse upstream types directly where appropriate or mirror them with minimal translation risk. That lowers the chance of schema drift, makes future upgrades easier, and reduces maintenance compared with introducing a Rust-specific schema layer for a format whose reference implementation already lives in Go.

Go is also a better fit for the expected operational surface:

- Terraform-adjacent tooling in this area is predominantly Go, which improves interoperability and contributor familiarity.
- [SOPS](https://github.com/getsops/sops) has Go-native libraries that can encrypt selected YAML nodes while preserving document structure.
- Building both a small CLI and reusable packages is straightforward without adding FFI or code-generation complexity.

Rust would also be viable, but it would require more custom schema modeling for no clear functional advantage in this project.

## External interfaces

## Repository layout

The repository root is the logical state store root. A deployment uses these paths:

- unsplit state: `<project-name>/<deployment-name>.json`
- split resources root: `<project-name>/<deployment-name>/`

Resource paths in the split tree are:

- singleton resources: `<project-name>/<deployment-name>/<optional module-path>/<resource-type>/<resource-name>.yaml`
- `count` or `for_each` resources:
  - metadata: `<project-name>/<deployment-name>/<optional module-path>/<resource-type>/<resource-name>/meta.yaml`
  - instances: `<project-name>/<deployment-name>/<optional module-path>/<resource-type>/<resource-name>/<resource-index>.yaml`

`<optional module-path>` is derived from the Terraform module address and expanded into path segments such as `module.foo/module.bar`.

## Reference encoding

When a structure is split and one of its children moves into a separate file, the parent stores the child as a path string instead of embedding the child object.

Paths are stored relative to the file that contains the reference:

- the root JSON file points to resource YAML files using relative paths from `<deployment-name>.json`
- a `meta.yaml` file points to instance YAML files such as `0.yaml` or `example-key.yaml`

Relative references keep the tree relocatable and avoid embedding repository-root-specific absolute paths.

## CLI surface

The initial CLI should expose these commands:

- `split --root <dir> --project <name> --deployment <name>`
- `unsplit --root <dir> --project <name> --deployment <name>`
- `run --root <dir> --project <name> --deployment <name> -- <command ...>`

`run` is a thin wrapper that:

1. materializes the unsplit JSON state
2. runs the wrapped command
3. writes the split form back if the wrapped command exits successfully

The core logic lives in the library so the same orchestration can be reused by other callers.

## Internal architecture

## Package layout

The codebase should be organized into public packages that map to the required reusable capabilities.

- `pkg/statefile`
  - load and save Terraform v4 state in JSON form
  - define compatibility adapters around upstream Terraform state structures
- `pkg/layout`
  - map Terraform addresses to repository paths
  - normalize module segments and instance keys into file names
- `pkg/splitter`
  - convert unsplit state structures into split in-memory documents plus file write plans
- `pkg/unsplitter`
  - resolve path references and rebuild canonical Terraform state structures
- `pkg/sensitive`
  - identify sensitive values and apply or remove [SOPS](https://github.com/getsops/sops) encryption
- `pkg/store`
  - abstract filesystem access, path resolution, and document reads and writes
- `pkg/wrapcmd`
  - run external commands around split and unsplit transitions
- `cmd/terraform-state-splitter`
  - CLI entrypoint

This layout keeps the domain logic reusable from tests, other programs, and future higher-level automation.

## Data model

## Root state document

The root JSON document remains a Terraform state JSON document. Fields unrelated to resource placement remain inline. The `resources` collection is transformed from an array of embedded resource objects into an array of path strings, each path resolving to either:

- a singleton resource YAML file
- a resource directory `meta.yaml` file for multi-instance resources

This keeps the root file close to the upstream state schema while making the split boundary explicit.

## Resource documents

Singleton resource YAML files contain a single Terraform resource object with the same field names Terraform uses, except that nested children moved to separate files are represented by relative path strings.

For resources with multiple instances:

- `meta.yaml` contains the resource-level fields:
  - `module`
  - `mode`
  - `type`
  - `name`
  - `provider`
  - ordered list of instance keys
- each instance file contains one entry from the resource's `instances` array

This keeps the common metadata stable while isolating per-instance churn.

## Sensitive data handling

Sensitive value handling follows the intent of the existing sensitive annotator while avoiding synthetic key mutations.

The design is:

- discover sensitive locations from Terraform state semantics, especially `sensitive_attributes`
- convert those locations into document-local field paths during split
- encrypt the referenced YAML values in place with SOPS
- keep original Terraform keys unchanged

Each encrypted YAML document carries normal SOPS metadata at the document root. During load, decryption happens before schema unmarshalling, and the `sops` metadata block is excluded from Terraform object reconstruction.

This approach preserves reviewable YAML, avoids suffix-based key rewriting, and keeps encryption concerns orthogonal to Terraform field naming.

## Split flow

## Input

`split` reads `<project-name>/<deployment-name>.json` from the selected root directory and decodes it as a Terraform state v4 document.

## Transformation

The splitter performs these steps:

- parse the state into structured Go types
- iterate through `resources`
- derive the target repository path for each resource from module path, resource type, name, and instance key shape
- replace each resource entry in the root document with the child file path
- for multi-instance resources, emit `meta.yaml` plus one file per instance and replace `instances` in `meta.yaml` with instance file paths
- apply sensitive-value encryption to each YAML document before serialization

## Output

The splitter writes:

- the rewritten root JSON file
- one YAML file per singleton resource
- one metadata YAML file plus one YAML file per instance for multi-instance resources

If any target output path already exists, the command fails explicitly instead of attempting merge behavior.

## Unsplit flow

## Input

`unsplit` reads the root JSON file, then recursively resolves path references found in the root resource list and in any resource metadata documents.

## Transformation

The unsplitter performs these steps:

- decode the root JSON document
- replace each resource path with the referenced YAML document
- for `meta.yaml`, resolve each instance path and rebuild the `instances` array in the original order
- decrypt SOPS-managed documents before unmarshalling
- reconstruct a Terraform-compatible state structure with no path placeholders or SOPS metadata

## Output

The output is a canonical Terraform JSON state file written back to `<project-name>/<deployment-name>.json`.

The tool does not delete split files automatically unless the calling workflow explicitly asks for cleanup. That keeps the transformation behavior simple and predictable.

## Command wrapping

The wrapper package exists so callers can safely perform workflows that expect the unsplit JSON file but store state in split form at rest.

The wrapper sequence is:

- unsplit to materialize the canonical JSON state
- invoke the child command with inherited stdio and environment
- if the command succeeds, split the resulting state back into the repository tree
- if the command fails, leave the current files untouched after the failure point and return the child exit code

This behavior keeps failure semantics explicit and avoids hiding command errors behind automatic cleanup logic.

## Path and naming rules

Module addresses map directly to `module.<name>` path segments. Resource names and types are preserved verbatim. Instance keys are rendered as file names using these rules:

- numeric `count` indexes use decimal file names such as `0.yaml`
- string `for_each` keys use the exact key text when it is filesystem-safe
- otherwise they are encoded with a reversible escaping scheme based on percent-encoding

The same encoding must be used by both splitter and unsplitter so round-tripping is deterministic.

## Error handling

Errors should be explicit and typed where practical:

- malformed Terraform state
- unsupported schema version
- missing referenced file
- duplicate resolved output path
- ambiguous or non-reversible instance key encoding
- SOPS encryption or decryption failure
- wrapped command execution failure

The library returns structured errors; the CLI renders concise operator-facing messages and exits non-zero.

## Compatibility with gitolize

The splitter writes normal files and directories without requiring gitolize-specific behavior. That is sufficient for standalone use and lets gitolize layer its own guarantees on top.

To remain compatible, the tool should:

- use deterministic file ordering and serialization
- avoid writing transient bookkeeping files
- avoid assuming exclusive control over higher-level staging or locking

## Serialization rules

JSON output should remain close to Terraform's canonical form. YAML output should preserve stable ordering for human review:

- scalar and collection field names remain identical to Terraform state field names
- map key ordering is deterministic
- empty collections are preserved when Terraform semantics distinguish them from null
- YAML documents use standard UTF-8 text and Unix newlines

Deterministic serialization matters more than preserving incidental formatting from prior files.

## Testing strategy

The implementation should be exercised with:

- unit tests for path mapping, reversible instance key encoding, and sensitive-path discovery
- round-trip tests that split and unsplit representative Terraform states
- fixtures covering singleton resources, `count`, `for_each`, nested modules, and sensitive values
- wrapper tests that verify exit-code propagation and no silent success on partial failures

The highest-value acceptance test is round-tripping real Terraform v4 state fixtures and confirming semantic equivalence before and after split plus unsplit.

## Open decisions

The following items should be settled during implementation:

- whether upstream Terraform state types can be imported directly without pulling in unnecessary CLI dependencies
- the exact YAML library to use alongside SOPS so field ordering and node-path encryption remain predictable
- whether split-file cleanup after `unsplit` should remain library-only, caller-controlled behavior or gain an explicit CLI flag later

## Summary

This design keeps the canonical Terraform state format intact at the system boundary, moves resource instances into reviewable YAML files, preserves compatibility with gitolize, and treats sensitive values as an encryption concern rather than a schema-renaming concern. The proposed Go package structure is intended to make splitting, unsplitting, loading, and command wrapping reusable from both the CLI and other programs.
