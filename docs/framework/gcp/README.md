# Google Cloud

Google Cloud is not part of our managed estate today. There are no `terraform-gcp-*` building blocks published and no root modules or workspaces targeting it.

This page exists so the expectation is written down rather than improvised the day it is needed.

## When we adopt it

The [Cloud Framework](../terraform/README.md) applies unchanged; only the primitives are substituted.

| Layer | What it looks like on Google Cloud |
| :-- | :-- |
| [Building block](../terraform/BUILDING-BLOCKS.md) | `terraform-gcp-<resource>`, one resource and its children, same `metadata.tf` construct |
| [Root module](../terraform/ROOT-MODULES.md) | `terraform-wanted-<domain>`, composing blocks per domain |
| [Workspace](../terraform/WORKSPACES.md) | `production-<domain>`, `terraform.tfvars` plus a pipeline |

The landing zone maps as follows: the organisation and folder hierarchy takes the place of [Azure management groups](../azure/LANDING-ZONE.md), projects take the place of subscriptions, organisation policy constraints take the place of Azure policy, and a Cloud Storage bucket with object versioning takes the place of the state storage account. The [naming convention](../azure/NAMING-CONVENTION.md) carries over, with the `gl-` prefix retained and project ids using the compressed form.

> _Nothing here is provisioned yet - treat this as intent, not as documentation of a running estate._
---
<sup><sub>_2024 &copy; All rights reserved - WANTED.solutions s.r.o. [<@wanted-solutions>](https://github.com/wanted-solutions)_</sub></sup>
