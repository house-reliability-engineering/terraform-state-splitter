# Terraform state splitter requirements

## Requirements


### Basic functionality

- This is to be a functional equivalent of
  [pulumi_state_splitter](https://github.com/house-reliability-engineering/pulumi/tree/main/utilities/pulumi_state_splitter),
  but for terraform.

- it should be compatible with [gitolize](https://github.com/house-reliability-engineering/gitolize),
  but should correctly work without it.

- terraform JSON state file should be read from and written to the following path:
  `<project-name>/<workspace-name>.json`.

- each resource instance should be stored in a separate file.
  The resource file path in the state repository tree should be of the form:
  - singletons: `<project-name>/<workspace-name>/<optional module-path>/<resource-type>/<resource-name>.yaml`
  - resources with `for_each` or `count`:
    - `<project-name>/<workspace-name>/<optional module-path>/<resource-type>/<resource-name>/meta.yaml`
      for `module`, `mode`, `type`, `name`, `provider` and instances index keys list.
    - `<project-name>/<workspace-name>/<optional module-path>/<resource-type>/<resource-name>/<resource-index>.yaml`
      for particular instances

- lay out the code so that it can be reused as a library, for example for:
  - loading split state (including decryption)
  - loading unsplit state
  - splitting
  - unsplitting
  - wrapping commands

- the `run` command should support unsplitting multiple states at once (optionally all) so that the
  [`terraform_remote_state`](https://developer.hashicorp.com/terraform/language/state/remote-state-data)
  data source works properly.

- keep YAML schemas as close as possible to corresponding
  [Terraform V4 state](https://github.com/hashicorp/terraform/blob/main/internal/states/statefile/version4.go)
  schema components.

- when splitting the state structure to separate files,
  the child structures should be removed from the parent structure,
  and the parent structure should get an extra attribute containing
  a list of paths of files containing the definitions of the child resources,
  for example:
  - [`stateV4`](https://github.com/hashicorp/terraform/blob/42fe4265/internal/states/statefile/version4.go#L666-L674):
    - [`Resources`](https://github.com/hashicorp/terraform/blob/42fe4265/internal/states/statefile/version4.go#L672) is removed,
    - `ResourcesPaths []string` is added.
  - [`resourceStateV4`](https://github.com/hashicorp/terraform/blob/42fe4265/internal/states/statefile/version4.go#L692-L700)
    - [`Instances`](https://github.com/hashicorp/terraform/blob/42fe4265/internal/states/statefile/version4.go#L699) is removed,
    - `InstancesPaths []string` is added.

### Encryption

- Include the functionality provided by
  [sensitive annotator](https://github.com/vertexinc/infrastructure-state/blob/master/bin/terraform/sensitive_annotator.py),
  without modifying the state keys if possible (like using SOPS SDK to point it at values to encrypt instead
  of relying on an artificial suffix for example).
  We should also encrypt the
  [`private`](https://github.com/hashicorp/terraform/blob/42fe426/internal/states/statefile/version4.go#L715)
  field unless we can prove that providers never store sensitive data there.

- The SOPS mac should be calculated for encrypted values, similar to the effect of the
  [`--mac-only-encrypted`](https://github.com/getsops/sops/blob/42c158d/cmd/sops/main.go#L1817-L1820)
  SOPS flag.

- If a sensitive value does not change between subsequent splits, then its ciphertext should stay the same.

### Other

- if Go is a preferred language over Rust, then
  - provide rationale for this preference.
  - use terraform code for state schema, possibly including loading and saving

## Non-requirements

- Handling file name length limits.
- `validate` CLI command.
- handling preexisting output files
- describing gitolize functionality
- staging of file changes
- gitolize functionality:
  - ensuring atomicity
  - performing backups
  - calculating and verifying checksums
  - diffing
  - locking
  - integrity checks

