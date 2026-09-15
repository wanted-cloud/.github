# Azure

Azure is our primary cloud. This section covers what is specific to it - the landing zone it is organised into and the convention every resource is named by. The delivery model itself is provider independent and described in the [Cloud Framework](../terraform/README.md).

## Table of Contents

- [Guidelines](#guidelines)
- [Domains](#domains)
- [Provider baseline](#provider-baseline)
- [Authentication](#authentication)

## Guidelines

- [Azure Landing Zone](./LANDING-ZONE.md) - management groups, subscriptions, state and cost
- [Naming Convention](./NAMING-CONVENTION.md) - how every Azure object is named
- [Building Blocks](../terraform/BUILDING-BLOCKS.md) - the `terraform-azure-*` module set
- [Root Modules](../terraform/ROOT-MODULES.md) and [Workspaces](../terraform/WORKSPACES.md) - how those blocks get deployed

## Domains

The estate is split into domains, each one a root module and a workspace per environment:

| Domain | Owns |
| :-- | :-- |
| governance | management groups, subscriptions, state backends, cost export |
| connectivity | virtual networks, gateways, private DNS, private endpoints |
| identity | app registrations, managed identities, access groups |
| corporate | shared corporate services and managed databases |
| contributors | the contributor and test estate |
| online | the production runtime estate |
| ai-hub | AI Foundry hubs, projects and search |

A resource belongs to exactly one domain. When two domains need the same thing, one owns it and the other consumes its id through a variable.

## Provider baseline

- Modules declare the **minimum** `azurerm` version they need; workspaces pin the exact version through the lock file committed with their root module.
- A provider major release is a **deliberate upgrade**, planned per domain. An open upper bound plus an unpinned module reference means the next major release reaches production unannounced - that has broken an estate before, and it is the reason building block references are pinned.

## Authentication

- Pipelines authenticate with a service principal; no human credentials are used for apply.
- Data planes prefer **Entra authentication over keys** wherever the service supports it - Terraform state (`use_azuread_auth`), PostgreSQL and MySQL group roles, storage. A key that exists is a key that leaks.
- Database access for people is granted by **Entra group membership**, not by handing out passwords; the group to role binding is Terraform, the membership is an approval.

> _Azure specifics change; the layering does not._
---
<sup><sub>_2024 &copy; All rights reserved - WANTED.solutions s.r.o. [<@wanted-solutions>](https://github.com/wanted-solutions)_</sub></sup>
