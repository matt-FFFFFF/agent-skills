---
name: terraform-schema
description: Look up Terraform/OpenTofu provider and module schemas from the registry with the `tfpluginschema` and `tfmoduleschema` CLIs instead of documentation websites. Covers resource, data source, ephemeral resource, provider-config and provider-function schemas, available provider/module versions, module input variables, outputs, required_providers and submodules. Use whenever writing, editing, reviewing or upgrading Terraform configuration that uses a provider resource or calls a registry/local module.
---

# Terraform Schema

When writing Terraform, get argument names, types, required/optional status and nesting from the **real schema** instead of from memory or registry documentation pages.

- `tfpluginschema` downloads the provider binary and returns its schema. This is exactly what `terraform validate` checks against.
- `tfmoduleschema` downloads a module and parses its HCL. It returns variables, outputs, required providers and resources.

Both return compact JSON or plain-text lists, which costs far fewer tokens than scraping registry.terraform.io or provider docs. They are also correct for the exact version you pin.

**Rule:** before writing a `resource`, `data`, `ephemeral`, `provider` or `module` block, query its schema with these tools. Do not guess attribute names. Do not fetch registry documentation pages unless the schema leaves a semantic question unanswered (for example, what a value means). Even then, query the schema first.

## Install / check

```bash
command -v tfpluginschema tfmoduleschema
# if missing:
brew install matt-ffffff/tap/tfpluginschema matt-ffffff/tap/tfmoduleschema
# or: go install github.com/matt-FFFFFF/tfpluginschema/cmd/tfpluginschema@latest
#     go install github.com/matt-FFFFFF/tfmoduleschema/cmd/tfmoduleschema@latest
```

`jq` is strongly recommended for trimming output.

## Common conventions (both tools)

- Global flags can go **before or after** the subcommand.
- `--registry` / `-r` defaults to `opentofu`. Its registry mirrors nearly every public provider and module. Pass `-r terraform` if something is missing there, or to match the Terraform registry exactly.
- `--version-constraint` / `--vc` accepts a concrete version (`5.2.0`) or a constraint (`~> 5.0`). If you omit it, you get the latest version. **Match the constraint used (or about to be used) in the configuration**, because schemas change between major versions.
- The first call downloads into a local cache, which can take seconds for large providers like `aws` or `azurerm`. Later calls are instant. Add `--quiet` to suppress the `downloading:` / `cache hit:` lines on stderr. `--force-fetch` bypasses the cache.
- A missing name exits with code `1` and prints `error: ... not found` on stderr. When that happens, run the matching `list` command to find the right name.
- `list` subcommands print one name per line. `schema` subcommands print JSON.

## tfpluginschema - providers

Identify the provider with `--ns <namespace> -n <name>` (the two halves of the provider `source`, e.g. `hashicorp/aws` becomes `--ns hashicorp -n aws`, and `Azure/azapi` becomes `--ns Azure -n azapi`).

| Command | Output |
|---|---|
| `version list` | All versions, **oldest first** (use `tail -1` for latest) |
| `resource list` | Resource type names |
| `resource schema <type>` | Schema for one resource |
| `datasource list` / `datasource schema <type>` | Data sources |
| `ephemeral list` / `ephemeral schema <type>` | Ephemeral resources |
| `function list` / `function schema <name>` | Provider-defined functions (`provider::<name>::<fn>`) |
| `provider schema` | The `provider "<name>" {}` configuration block |

**Always pass a name to `schema`.** Without one it dumps every schema in the provider (about 94 MB for `aws`).

```bash
tfpluginschema --quiet --ns hashicorp -n aws --vc '~> 6.0' resource list | grep s3_bucket
tfpluginschema --quiet --ns hashicorp -n aws --vc '~> 6.0' resource schema aws_s3_bucket
tfpluginschema --quiet --ns hashicorp -n aws function schema arn_parse
tfpluginschema --quiet --ns hashicorp -n azurerm version list | tail -1
```

### Reading a provider schema

The output is Terraform's standard JSON schema format:

```
{ "version": N,
  "block": {
    "attributes":  { "<attr>":  { "type", "required"|"optional"|"computed", "sensitive", "deprecated", "description" } },
    "block_types": { "<block>": { "nesting_mode": "single|list|set|map|group", "min_items", "max_items", "block": { ...recursive... } } }
  } }
```

- `required: true` means you must set it. `optional` + `computed` means you can set it, and the provider fills it in if you don't. `computed` alone means read-only, so only reference it (e.g. `aws_s3_bucket.x.arn`) and never set it.
- Entries under `attributes` are written as `name = value`. Entries under `block_types` are written as nested blocks (`name { ... }`). Repeat a nested block for `list`/`set` nesting. A block with `min_items >= 1` is required.
- An attribute with `nested_type` (protocol-6 providers) takes an object or list-of-objects **value**, not a nested block.
- `type` uses cty notation: `"string"`, `["list","string"]`, `["map","string"]`, `["object",{...}]`.
- Avoid any attribute or block marked `deprecated: true`. Look for its replacement, which is often a separate resource in newer major versions.

### Trimming output with jq

```bash
S='tfpluginschema --quiet --ns hashicorp -n aws resource schema aws_s3_bucket'

# Required top-level arguments
$S | jq -r '.block.attributes | to_entries[] | select(.value.required) | .key'

# Settable arguments with types (drop computed-only and descriptions)
$S | jq -c '.block.attributes | to_entries[] | select(.value.required or .value.optional) | {k:.key, t:.value.type, req:(.value.required//false)}'

# Nested blocks and their nesting constraints
$S | jq -c '.block.block_types | to_entries[] | {k:.key, mode:.value.nesting_mode, min:.value.min_items, max:.value.max_items}'

# Drill into one nested block
$S | jq '.block.block_types.lifecycle_rule.block'

# Deprecated attributes
$S | jq -r '.block.attributes | to_entries[] | select(.value.deprecated) | .key'
```

Read `description` only when the meaning of an argument is unclear. Fetch the description of that one attribute (`jq '.block.attributes.<name>.description'`) rather than the whole schema.

## tfmoduleschema - modules

Identify a registry module with `--ns <namespace> -n <name> -s <system>`, the three parts of a registry source. For example, `terraform-aws-modules/vpc/aws` becomes `--ns terraform-aws-modules -n vpc -s aws`, and `Azure/avm-res-resources-resourcegroup/azurerm` becomes `--ns Azure -n avm-res-resources-resourcegroup -s azurerm`.

You can use `--source` instead for a local path or a go-getter URL (`./modules/net`, `git::https://...`). It cannot be combined with `--ns`/`-n`/`-s`. With `--source`, `--vc` must be a concrete version if it is set at all.

| Command | Output |
|---|---|
| `version list` | All versions, **newest first** (use `head -1` for latest) |
| `variable list` / `variable schema [name]` | Inputs: `name`, `type`, `default`, `required`, `description`, `sensitive` (fields omitted when unset/false) |
| `output list` / `output schema [name]` | Outputs: `name`, `description`, `sensitive` (omitted when false) |
| `provider list` / `provider schema [name]` | `required_providers`: `source` + `version_constraints` |
| `submodule list` | Paths under `modules/` |
| `module schema` | Everything: variables, outputs, `required_core`, `required_providers`, `managed_resources`, `data_resources` |

Add `--submodule modules/<path>` (`--sm`) to target a submodule instead of the root module. The module address then becomes `<ns>/<name>/<system>//modules/<path>`.

```bash
M='--quiet --ns terraform-aws-modules -n vpc -s aws --vc ~>6.0'

tfmoduleschema $M version list | head -1
tfmoduleschema $M variable list
tfmoduleschema $M variable schema | jq -c '.[] | select(.required) | {name, type}'   # mandatory inputs
tfmoduleschema $M variable schema azs                                               # one input
tfmoduleschema $M output list
tfmoduleschema $M provider schema                                                   # provider constraints to satisfy
tfmoduleschema $M submodule list
tfmoduleschema $M --sm modules/vpc-endpoints variable list
tfmoduleschema --quiet --source ./modules/network variable schema                  # local module
```

Module inputs can number in the hundreds (`vpc` has more than 230). Start with `variable list`, or filter `variable schema` to `required` inputs plus the specific ones you need. Never paste the whole schema into context. Ignore the `pos` field (cache file location) unless you need to read the module's source.

## Workflow when authoring configuration

1. **Pin versions.** Run `version list`, then write `required_providers` / module `version` constraints. Use the same `--vc` for every later query.
2. **Find the right type.** Run `resource list` / `datasource list` / `ephemeral list` and grep, rather than guessing names (e.g. `azurerm_linux_virtual_machine` vs `azurerm_virtual_machine`).
3. **Get the shape.** Query the required arguments, the settable arguments and the nested blocks with the jq filters above. Write the block from that output.
4. **For modules:** set every `required` variable and match each variable's `type`. Reference outputs only by names that `output list` returns. Make sure the root's provider constraints satisfy what `provider schema` reports.
5. **Wire references.** Only reference attributes that exist in the schema. Computed attributes like `id` and `arn` are valid references.
6. **Validate:** `terraform init -backend=false && terraform validate` (or `tofu`). If validation reports an unknown argument, query the schema again instead of guessing.

When upgrading a provider or module major version, query the schema at both the old and the new `--vc` and diff the results (attribute keys, `required` flags, `deprecated`) to find breaking changes.
