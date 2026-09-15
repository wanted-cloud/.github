# WANTED.solutions Cloud Framework

Our infrastructure is delivered in **three layers**. Each layer has one job, and the boundary between them is what keeps the estate reviewable: code that knows *how* never knows *where*, and data that knows *where* never contains logic.

## Table of Contents

- [The three layers](#the-three-layers)
- [Why the split](#why-the-split)
- [Repository map](#repository-map)
- [Shared tooling](#shared-tooling)
- [Guidelines](#guidelines)

## The three layers

| Layer | Repository | Contains | Answers |
| :-- | :-- | :-- | :-- |
| [Building block](./BUILDING-BLOCKS.md) | `terraform-<provider>-<resource>` | one resource and its children | *how* is a resource built |
| [Root module](./ROOT-MODULES.md) | `terraform-wanted-<domain>` | composition of building blocks | *what* a domain consists of |
| [Workspace](./WORKSPACES.md) | `production-<domain>` | `terraform.tfvars` and pipelines | *where* and *with which values* |

```
terraform-azure-postgresql-server   <- building block, published to the registry
          ^
          | source = git::https://github.com/wanted-cloud/...?ref=main
          |
terraform-wanted-corporate          <- root module, for_each over var.postgresql_servers
          ^
          | TF_CONTEXT_DIRECTORY + TF_VARS_FILE
          |
production-corporate                <- workspace, terraform.tfvars + pipeline
```

## Why the split

- **A building block is reusable because it is ignorant.** It never knows which subscription, environment or tenant it lands in, which is why the same block serves production, test and a customer estate.
- **A root module is reviewable because it is declarative.** Adding a database is a new object in a list, not new Terraform code.
- **A workspace is auditable because it is only data.** A change to production is a diff of values, and the pipeline that applies it is pinned to an approval gate.

The practical test: if a change requires editing Terraform *code* to add another instance of something that already exists, the layering is wrong.

## Repository map

| Domain | Root module | Workspace |
| :-- | :-- | :-- |
| Corporate services and databases | `terraform-wanted-corporate` | `production-corporate` |
| Governance, management groups, subscriptions | `terraform-wanted-governance` | `production-governance` |
| Networking and connectivity | `terraform-wanted-connectivity` | `production-connectivity` |
| Identity and app registrations | `terraform-wanted-identity` | `production-identity` |
| Contributor (test) estate | `terraform-wanted-contributors` | `production-contributors` |
| Production runtime estate | `terraform-wanted-online` | `production-online` |
| AI hub | `terraform-wanted-ai-hub` | `production-ai-hub` |

Pipeline templates shared by every workspace live in `vsts-pipeline-templates`.

## Shared tooling

Every module repository - block or root - carries the same `Makefile`:

```shell
make format     # terraform fmt
make docs       # terraform-docs, injected into README.md
make package    # format + docs, run this before every commit
make valid      # terraform validate
make plan       # terraform plan
```

## Guidelines

- [Building blocks](./BUILDING-BLOCKS.md) - authoring the reusable units
- [Root modules](./ROOT-MODULES.md) - composing a domain
- [Workspaces](./WORKSPACES.md) - environments, state and delivery
- [Azure framework](../azure/README.md) - landing zone and naming convention

> _Predictable infrastructure comes from predictable structure._
---
<sup><sub>_2024 &copy; All rights reserved - WANTED.solutions s.r.o. [<@wanted-solutions>](https://github.com/wanted-solutions)_</sub></sup>
