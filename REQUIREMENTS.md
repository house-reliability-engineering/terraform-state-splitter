# Terraform state splitter requirements

## Requirements

- This is to be a functional equivalent of
  [pulumi_state_splitter](https://github.com/house-reliability-engineering/pulumi/tree/main/utilities/pulumi_state_splitter),
  but for terraform.

- Include the functionality provided by
  [sensitive annotator](https://github.com/vertexinc/infrastructure-state/blob/master/bin/terraform/sensitive_annotator.py),
  without modifying the state keys if possible (like using SOPS SDK to point it at values to encrypt instead
  of relying on an artificial suffix for example).

- it should be compatible with [gitolize](https://github.com/house-reliability-engineering/gitolize),
  but should correctly work without it.

- terraform JSON state file should be read from and written to the following path:
  `<project-name>/<deployment-name>.json` .

- each resource instance should be stored in a separate file.
  The resource file path in the state repository tree should be of the form:
  - singletons: `<project-name>/<deployment-name>/<optional module-path>/<resource-type>/<resource-name>.yaml`
  - resources with `for_each` or `count`:
    - `<project-name>/<deployment-name>/<optional module-path>/<resource-type>/<resource-name>/meta.yaml`
      for `module`, `mode`, `type`, `name`, `provider` and instances index keys list.
    - `<project-name>/<deployment-name>/<optional module-path>/<resource-type>/<resource-name>/<resource-index>.yaml`
      for particular instances

- when writing children of a structure to separate files (e.g. resources are children of state) during splitting,
  replace them in the parent structure with their file paths.

- keep YAML schemas as close as possible to corresponding
  [Terraform V4 state](https://github.com/hashicorp/terraform/blob/main/internal/states/statefile/version4.go)
  schema components.

- lay out the code so that it can be reused as a library, for example for:
  - loading split state (including decryption)
  - loading unsplit state
  - splitting
  - unsplitting
  - wrapping commands

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

