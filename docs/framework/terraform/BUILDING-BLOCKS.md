# Building Blocks

A **building block** is the smallest reusable unit of the [WANTED.solutions Cloud Framework](./README.md). It wraps exactly one cloud resource together with the children that resource owns, and nothing else. Building blocks are opinionated on structure and deliberately unopinionated on composition - wiring blocks together is the job of a [root module](./ROOT-MODULES.md).

## Table of Contents

- [What belongs in a building block](#what-belongs-in-a-building-block)
- [Naming](#naming)
- [Repository layout](#repository-layout)
- [The metadata construct](#the-metadata-construct)
- [Variables](#variables)
- [Outputs](#outputs)
- [Examples](#examples)
- [Documentation](#documentation)
- [Versioning and publishing](#versioning-and-publishing)
- [Release checklist](#release-checklist)

## What belongs in a building block

A building block owns:

- **one main resource**, always named `this`,
- the **child resources that cannot exist without it** - a Key Vault's secrets, a Storage Account's containers, a Bot Service's channels,
- a **data source lookup** for the scope it is created in, typically `data "azurerm_resource_group" "this"`.

A building block does **not** own:

- resources that have their own lifecycle and their own building block - an app registration, an Application Insights instance, a role assignment. Take these as inputs (an id, a name, a secret) and return what others need as outputs,
- any composition of several top level resources - that is a root module,
- provider configuration. Blocks declare `required_providers`, never `provider` blocks.

> If you find yourself creating a second unrelated top level resource, you are writing a root module, not a building block.

## Naming

| Item | Convention | Example |
| :-- | :-- | :-- |
| Repository | `terraform-<provider>-<resource>` | `terraform-azure-bot-service` |
| Registry address | `wanted-cloud/<resource>/<provider>` | `wanted-cloud/bot-service/azure` |
| Main resource | `this` | `azurerm_bot_service_azure_bot.this` |
| Child resources | `this`, iterated with `count` or `for_each` | `azurerm_bot_connection.this` |
| Source files | lowercase, hyphenated, named after the child | `resource-group.tf`, `channel-ms-teams.tf` |

## Repository layout

Every block starts as a clone of [`terraform-module-template`](https://github.com/wanted-cloud/terraform-module-template) and keeps its shape:

```
terraform-<provider>-<resource>/
├── main.tf                 # the "this" resource, and the module header comment
├── variables.tf            # every input variable
├── outputs.tf              # every output value
├── versions.tf             # terraform and provider version constraints
├── locals.tf               # local.definitions - the module's own metadata defaults
├── metadata.tf             # shared construct - DO NOT MODIFY
├── <child>.tf              # one file per child resource
├── Makefile                # format, docs, valid, package targets
├── README.md               # generated, never hand edited between the TF_DOCS markers
├── .terraform-docs.yaml    # documentation generator configuration
├── LICENSE
└── examples/
    ├── base/main.tf        # the minimal working call
    └── with-<feature>/     # one example per optional feature
```

The header comment at the top of `main.tf` becomes the README title and description, so write it for the reader of the registry page:

```hcl
/*
 * # wanted-cloud/terraform-azure-bot-service
 * 
 * Simple Terraform building block wrapping Azure Bot Service together with its messaging channels.
 */
```

## The metadata construct

`metadata.tf` is identical in every module and **must not be edited**. It merges three layers - module defaults from `local.definitions`, caller overrides from `var.metadata`, and built in fallbacks - into `local.metadata`. This is what lets a caller retune a module from the outside without the module exposing a variable for every knob.

Declare the module's own defaults in `locals.tf`:

```hcl
locals {
  definitions = {
    tags = { ManagedBy = "Terraform" }

    validator_expressions    = { sku = "^(F0|S1)$" }
    validator_error_messages = { sku = "The sku must be either F0 (free) or S1 (standard)." }
  }
}
```

**Timeouts.** Every resource declares a `timeouts` block resolved per resource type with a fallback to the default, so a caller can extend a slow resource without touching the module:

```hcl
timeouts {
  create = try(
    local.metadata.resource_timeouts["azurerm_bot_service_azure_bot"]["create"],
    local.metadata.resource_timeouts["default"]["create"]
  )
  # read, update and delete follow the same shape
}
```

Only declare the operations the resource actually supports.

**Tags.** Taggable resources merge the metadata tags under the caller's tags: `tags = merge(local.metadata.tags, var.tags)`.

**Validators.** Enumerations are validated through the metadata expressions so a caller can widen them when the provider gains a value before the module does:

```hcl
validation {
  condition     = can(regex(local.metadata.validator_expressions["sku"], var.sku))
  error_message = local.metadata.validator_error_messages["sku"]
}
```

> **Note:** a `validation` block that references a local is evaluated at **plan** time, not by `terraform validate` - Terraform treats cross object conditions as non static. Requires `terraform >= 1.9`.

## Variables

- **One variable per child collection**, flat at the top level - `secrets`, `queues`, `connections` - typed as `list(object({...}))` with `default = []`.
- **Optional singletons are objects defaulting to `null`**, created with `count = var.x != null ? 1 : 0`. An empty object then means "create it with provider defaults".
- **Optional strings default to `""`** and are passed through as `var.x != "" ? var.x : null`, so an unset input never sends an empty string to the API.
- **Never mark a variable `sensitive` if it is used in `count` or `for_each`** - Terraform rejects sensitive values there. Protect the secret at the output boundary instead.
- Descriptions are full sentences; they are the registry documentation.

## Outputs

- Expose the **id** of every resource the module creates - callers need them as scopes for role assignments and diagnostics.
- Collections are returned as maps keyed by the same key used in `for_each`.
- Generated credentials are exposed but marked `sensitive = true`.
- Prefer named outputs over dumping the whole resource object; an object containing a sensitive attribute forces the entire output to be sensitive.

## Examples

Each example is a directory under `examples/` containing a `main.tf` that calls the module with `source = "../.."`. `examples/base` is the minimal working call and is embedded in the README; every optional feature gets its own `with-<feature>` example. Use obviously fake values (`00000000-0000-0000-0000-000000000000`, `example-rg`) - examples are documentation, not deployable configuration. A `provider.tf` may be added when the example needs to be validated locally; it is not published as part of the usage documentation.

## Documentation

`README.md` is generated by [terraform-docs](https://terraform-docs.io) between the `<!-- BEGIN_TF_DOCS -->` markers and is never hand edited there. Always run:

```shell
make package    # terraform fmt + terraform-docs
```

before committing, and run it **without a local `.terraform.lock.hcl` present**. With a lock file terraform-docs prints the resolved provider version; without it, it prints the constraint from `versions.tf`, which is what a reusable module has to advertise.

## Versioning and publishing

- Modules follow **SemVer**, tagged as plain `x.y.z` without a `v` prefix.
- **A tag is the release.** The Terraform Registry publishes from tags, so an untagged repository is not consumable from the registry, no matter how complete it is.
- The **first** publication of a module is a manual step in the registry; every later tag is picked up automatically.
- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org).

## Release checklist

1. `terraform validate` passes for the module and for every example.
2. `make package` produced no diff you did not intend.
3. Every resource has a `timeouts` block wired through `local.metadata`.
4. Every taggable resource merges `local.metadata.tags`.
5. `metadata.tf` is unchanged from the template.
6. `examples/base` is the minimal call and actually works.
7. The repository description is set on GitHub - it is what the registry shows in search results.
8. Tag `x.y.z` and push the tag.

> _Thank you for keeping our building blocks predictable!_
---
<sup><sub>_2024 &copy; All rights reserved - WANTED.solutions s.r.o. [<@wanted-solutions>](https://github.com/wanted-solutions)_</sub></sup>
