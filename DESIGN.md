# Terraform state splitter design

## Purpose

`terraform-state-splitter` makes [Terraform](https://developer.hashicorp.com/terraform)
state reviewable in Git by converting one Terraform JSON state file into a tree
of YAML files with one file per resource instance. It reverses that conversion
before Terraform reads the state.

The tool is the Terraform equivalent of
[pulumi_state_splitter](https://github.com/house-reliability-engineering/pulumi/tree/main/utilities/pulumi_state_splitter).
It should integrate with
[gitolize](https://github.com/house-reliability-engineering/gitolize), while
remaining useful as a standalone command and reusable Go library.

## Scope

The implementation handles Terraform state format version 4, split and unsplit
operations, command wrapping, and secret encryption support equivalent to the
[sensitive annotator](https://github.com/vertexinc/infrastructure-state/blob/master/bin/terraform/sensitive_annotator.py)
without modifying state keys.

The implementation intentionally does not handle output file conflict
resolution, filename length limits, staging changes, validation-only commands,
or repository-management concerns such as atomicity, locking, checksums,
backups, integrity checks, or diffs.

## Language and dependency choices

Go is the preferred implementation language.

Terraform is implemented in Go and its source tree contains the most accurate
statefile parsing, state representation, and state writing code. Reusing that
code where it is available through importable packages avoids maintaining a
parallel schema model for Terraform state version 4 and reduces the risk of
producing JSON that Terraform can parse but interprets differently.

The Go implementation should use these primary dependencies:

| Dependency | Use | Rationale |
| --- | --- | --- |
| [Terraform statefile package](https://github.com/hashicorp/terraform/tree/main/internal/states/statefile) | Read and write Terraform state JSON when practical | Keeps the unsplit state representation aligned with Terraform |
| [Terraform states package](https://github.com/hashicorp/terraform/tree/main/internal/states) | In-memory state representation | Preserves Terraform semantics for modules, resources, instances, outputs, and dependencies |
| [SOPS Go SDK](https://github.com/getsops/sops) | Encrypt and decrypt sensitive values in split YAML files | Encrypts values directly instead of renaming keys for suffix-based encryption |
| [yaml.v3](https://pkg.go.dev/gopkg.in/yaml.v3) | YAML encoding and decoding | Preserves maps and scalar values well enough for stable human review |
| [cobra](https://github.com/spf13/cobra) | CLI command structure | Provides conventional subcommands and help output with minimal custom code |

Terraform packages under `internal/` cannot be imported directly from another
module. The project should first verify whether the required statefile behavior
is available through importable Terraform packages. If not, vendor a narrow copy
of Terraform statefile version 4 loading and saving code with attribution and
tests that compare behavior against Terraform CLI state round trips. That copied
code should remain isolated under an `internal/terraformstate` package so it can
be removed if Terraform exposes a stable API later.

## State locations

The tool treats the backend directory as its root. A stack is identified by
`project-name/deployment-name`.

The unsplit Terraform state JSON path is:

```text
<project-name>/<deployment-name>.json
```

The split state tree root is:

```text
<project-name>/<deployment-name>/
```

Commands should support selecting one or more stacks explicitly and should find
all applicable states when no stack is selected.

## Split file layout

Each Terraform resource is represented by metadata plus one or more instance
files.

Singleton resources use one file:

```text
<project-name>/<deployment-name>/<optional module-path>/<resource-type>/<resource-name>.yaml
```

Resources with `for_each` or `count` use a resource directory:

```text
<project-name>/<deployment-name>/<optional module-path>/<resource-type>/<resource-name>/meta.yaml
<project-name>/<deployment-name>/<optional module-path>/<resource-type>/<resource-name>/<resource-index>.yaml
```

`meta.yaml` contains the resource-level Terraform state fields:

```yaml
module: module.example
mode: managed
type: aws_instance
name: web
provider: provider["registry.terraform.io/hashicorp/aws"]
instances:
  - blue
  - green
```

Each instance file contains one Terraform state instance using YAML fields that
closely match Terraform state version 4:

```yaml
index_key: blue
schema_version: 1
attributes:
  id: i-0123456789abcdef0
sensitive_attributes:
  - - type: get_attr
      value: password
private: eyJzY2hlbWFfdmVyc2lvbiI6IjEifQ==
dependencies:
  - aws_security_group.web
```

The split root should also contain the state-level fields from the Terraform
state file with large child collections replaced by file paths. For example,
`resources` should become a list of resource metadata or instance file paths
instead of embedding every resource instance inline. This mirrors the Pulumi
splitter approach and keeps parent YAML files small while preserving a reversible
mapping.

## Module path mapping

Terraform resource addresses can include module paths such as:

```text
module.network.module.subnet.aws_subnet.private
```

The split tree should map module path segments to directories in their original
order:

```text
module.network/module.subnet/aws_subnet/private.yaml
```

Module path parsing must use Terraform address parsing rather than string
splitting where possible. This avoids ambiguous behavior for resource names or
keys that contain characters with Terraform address meaning.

## Resource and instance identity

The splitter must preserve enough identity to reconstruct the original
`resources` list exactly enough for Terraform to accept it.

Resource identity is the tuple:

```text
module, mode, type, name, provider
```

Instance identity is the Terraform `index_key`. Singleton resources have exactly
one instance without an index key and therefore use the singleton resource file
layout. Resources with any indexed instances use the directory layout even if
there is only one current instance.

File names for instance indexes should be deterministic and reversible:

| Terraform index key | File name |
| --- | --- |
| Number from `count` | Decimal number, such as `0.yaml` |
| String from `for_each` | URL-path escaped string, such as `blue.yaml` or `us-east-1%2Fa.yaml` |

Unsplit must decode the file name back to the original index key type. The
`meta.yaml` instance list is the source of truth for order and type.

## Sensitive value handling

Terraform state version 4 records sensitive attribute paths in
`sensitive_attributes`. The tool should use those paths to locate values in
`attributes` and pass only those YAML nodes to the SOPS encryption layer.

The implementation must not rename Terraform state keys to trigger encryption.
That avoids producing split files whose cleartext structure differs from the
actual Terraform state schema.

During split:

- load Terraform state JSON;
- convert each resource instance to a YAML node tree;
- walk `sensitive_attributes` paths and mark the corresponding YAML nodes for
  SOPS encryption;
- write encrypted YAML files when SOPS configuration is available;
- fail when sensitive values are present but encryption is neither configured nor
  explicitly disabled.

During unsplit:

- load split YAML files through the SOPS decryption path;
- reconstruct Terraform state structures from decrypted YAML values;
- write Terraform JSON state without SOPS metadata.

Outputs should be treated similarly: an output with `sensitive: true` should
have its `value` encrypted in split form and restored as normal JSON during
unsplit.

## Library layout

The code should be structured for reuse by other tools.

```text
cmd/terraform-state-splitter/
  main.go
internal/terraformstate/
  load.go
  save.go
  model.go
pkg/state/
  stack.go
  paths.go
pkg/split/
  split.go
  unsplit.go
pkg/secrets/
  sops.go
pkg/runner/
  run.go
```

`pkg/state` owns stack discovery and path mapping.

`pkg/split` owns deterministic conversion between unsplit Terraform JSON state
and split YAML trees.

`pkg/secrets` owns encryption and decryption of sensitive YAML nodes.

`pkg/runner` owns temporary unsplitting around a wrapped command and
re-splitting after the command exits.

`internal/terraformstate` isolates Terraform-specific loading, saving, and any
vendored compatibility code.

## CLI design

The command should provide the same operational shape as the Pulumi splitter:

```console
terraform-state-splitter [--backend-directory DIR] split [--stack PROJECT/DEPLOYMENT]
terraform-state-splitter [--backend-directory DIR] unsplit [--stack PROJECT/DEPLOYMENT]
terraform-state-splitter [--backend-directory DIR] run [--stack PROJECT/DEPLOYMENT] -- COMMAND [ARGS...]
```

`split` converts `<project-name>/<deployment-name>.json` files into split YAML
trees and removes the original JSON files only after the split succeeds.

`unsplit` converts split YAML trees into JSON state files and removes the split
tree only after the JSON file is written successfully.

`run` unsplits selected states, runs the requested command, then splits the
states again. It should return the wrapped command exit code after best-effort
re-splitting. If re-splitting fails, the command should report that failure and
return a non-zero exit code even when the wrapped command succeeded.

## Determinism and ordering

Stable output is essential because the project exists to make state diffs
reviewable.

The implementation should:

- sort discovered stacks by path;
- sort resources by Terraform address before writing split files;
- preserve instance ordering from Terraform state in `meta.yaml`;
- sort map keys when Terraform semantics do not depend on insertion order;
- emit YAML with a stable indentation and scalar style;
- emit Terraform JSON in Terraform-compatible ordering where Terraform requires
  ordering for dependency or provider references.

## Error handling

Failures must be explicit and actionable.

The tool should fail if:

- the Terraform state version is unsupported;
- a resource path maps outside the backend directory;
- two resources map to the same split file path;
- a `sensitive_attributes` path cannot be resolved;
- sensitive values are present but encryption is not configured or explicitly
  disabled;
- SOPS decryption fails;
- unsplit cannot reconstruct the state without losing fields.

The tool should not silently skip invalid resources or missing instance files.

## Testing strategy

The implementation should have meaningful coverage for:

- Terraform state version 4 load and save round trips;
- singleton resource split and unsplit;
- `count` and `for_each` resource split and unsplit;
- nested module path mapping;
- sensitive resource attributes and sensitive outputs;
- SOPS encryption and decryption behavior;
- deterministic output ordering;
- `run` command exit-code handling and re-splitting behavior;
- error cases for unsupported versions, duplicate paths, invalid sensitive
  paths, and malformed split trees.

Golden tests should compare split file trees against expected YAML files.
Round-trip tests should verify that `unsplit(split(state))` produces Terraform
state accepted by Terraform and semantically equivalent to the input.

## Migration and compatibility

The first implementation should support only Terraform state version 4. If a
future Terraform state version appears, the loader should fail with a clear
unsupported-version error until compatibility is designed and tested.

Split YAML schemas should stay close to Terraform state version 4 structures.
Additional splitter metadata should be minimal, documented, and placed in parent
metadata files rather than injected into Terraform resource or instance
structures.
