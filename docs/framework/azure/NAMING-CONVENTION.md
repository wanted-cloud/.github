# Naming Convention

Every object we provision on Azure is named by the same rule. Names are the only metadata that shows up in *every* tool - the portal, a bill, an alert, a `kubectl` context - so they carry the information needed to answer "what is this and whose is it" without a lookup.

## Table of Contents

- [The pattern](#the-pattern)
- [Segments](#segments)
- [Resource type tokens](#resource-type-tokens)
- [Names that cannot take hyphens](#names-that-cannot-take-hyphens)
- [Identity objects](#identity-objects)
- [Rules](#rules)

## The pattern

```
gl-<type>-<scope>-<workload>-<environment>-<instance>
```

```
gl-rg-swc-contributors-persistence-prod-001
│  │   │   │                       │    └── instance, always three digits
│  │   │   │                       └─────── environment
│  │   │   └─────────────────────────────── workload, one or more words
│  │   └─────────────────────────────────── scope - region for regional resources
│  └─────────────────────────────────────── resource type
└────────────────────────────────────────── organization prefix
```

Lowercase, hyphen separated, no abbreviations beyond the type token. A name is read left to right from the most general to the most specific.

## Segments

| Segment | Values | Notes |
| :-- | :-- | :-- |
| organization | `gl` | fixed |
| type | see [below](#resource-type-tokens) | the Azure resource kind |
| scope | region code (`swc` = Sweden Central), `mca` for subscriptions, tenant or product token for identity objects | regional resources carry their region |
| workload | free text, hyphenated | what it serves: `contrib-cluster`, `private-endpoints`, `aihub-search` |
| environment | `prod`, `test`, `dev` | omitted on a few long lived shared resources; include it in new names |
| instance | `001`, `002`, ... | always present on Azure resources, always three digits |

Subscriptions use the billing scope rather than a region, because a subscription is not regional:

```
gl-sub-mca-online-prod-001
gl-sub-mca-contributors-dev-001
gl-sub-mca-online-prod-001-budget      # a budget appends its purpose to its parent
```

## Resource type tokens

| Token | Resource |
| :-- | :-- |
| `rg` | Resource group |
| `sub` | Subscription |
| `vnet` | Virtual network |
| `snet` | Subnet |
| `pip` | Public IP address |
| `vpngw` | Virtual network gateway |
| `aks` | Kubernetes service |
| `psql` | PostgreSQL flexible server |
| `mysql` | MySQL flexible server |
| `kv` | Key Vault |
| `srch` | AI Search service |
| `hub` | AI Foundry hub |
| `proj` | AI Foundry project |
| `uami` | User assigned managed identity |
| `st` | Storage account |
| `app` | Entra application registration |
| `sga` | Entra security group |

Extend the table rather than inventing a second token for a type that already has one.

## Names that cannot take hyphens

Storage accounts (and anything else limited to lowercase alphanumerics, 24 characters) use the same segments with the separators removed:

```
gl st swc contrib blb prod 001   ->   glstswccontribblbprod001
```

Drop words from the workload before dropping segments, and never drop the instance. Older accounts in the estate use `sa` as the type token or omit it entirely - they stay as they are, new ones use `st`.

## Identity objects

Entra objects are not regional, so the scope segment names the tenant or the product they belong to:

```
gl-sga-wanted-database-administrators            # security group, tenant scoped
gl-sga-wanted-psql-online-readwrite              # group granting a database role
gl-app-outlays-identity-ledger-prod-001          # app registration for a product service
```

Security groups end in the access they grant (`-owners`, `-administrators`, `-readonly`, `-readwrite`), because their name is what an approver sees when granting membership.

## Rules

1. **The name is not the documentation**, but it must be enough to route an alert to an owner.
2. **Never encode a secret, a customer name or a person** in a resource name.
3. **Never renumber.** `002` follows `001` even if `001` is gone; reusing an instance number makes history lie.
4. **The environment segment is part of the identity**, not a suffix to strip - `-prod` and `-test` resources never share a name.
5. **Names live in `terraform.tfvars`**, never in module code.

> _If you cannot tell what a resource is from its name, the name is wrong._
---
<sup><sub>_2024 &copy; All rights reserved - WANTED.solutions s.r.o. [<@wanted-solutions>](https://github.com/wanted-solutions)_</sub></sup>
