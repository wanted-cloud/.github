# Azure Landing Zone

The landing zone is the part of the estate that exists before any workload does: the management group hierarchy, the subscriptions, who may act on them, where state is kept and how spend is observed. It is owned by a single domain - the `terraform-wanted-governance` root module with its `production-governance` workspace - and everything else is deployed *into* it.

## Table of Contents

- [Hierarchy](#hierarchy)
- [Subscriptions](#subscriptions)
- [What governance owns](#what-governance-owns)
- [State storage](#state-storage)
- [Cost visibility](#cost-visibility)
- [Adopting what already exists](#adopting-what-already-exists)

## Hierarchy

```
Tenant Root Group
└── WANTED.solutions                      (wanted-solutions)
    ├── WANTED.solutions sandbox          (wanted-solutions-sandbox)   <- default for new subscriptions
    ├── WANTED.solutions decommissioned                                <- parking, nothing runs here
    └── domain subscriptions
```

- The **root management group is looked up, not created** - a data source resolves it by display name, so the code never assumes it owns the tenant.
- New subscriptions land in a **default management group** (`management_group_default`) rather than at the root, so an unclassified subscription is never implicitly privileged.
- **Decommissioning is a move, not a delete.** A subscription that is being retired is parked under the decommissioned group where it keeps its history and loses its parent's policy.

## Subscriptions

One subscription per domain and environment, named on the [naming convention](./NAMING-CONVENTION.md):

| Subscription | Purpose |
| :-- | :-- |
| `gl-sub-mca-platform-prod-001` | platform and shared services |
| `gl-sub-mca-management-prod-001` | management tooling |
| `gl-sub-mca-online-prod-001` | production runtime estate |
| `gl-sub-mca-online-dev-001` | non production runtime |
| `gl-sub-mca-contributors-prod-001` | contributor estate |
| `gl-sub-mca-contributors-dev-001` | contributor estate, non production |

Each subscription carries a **budget** named after it with a `-budget` suffix. A subscription without a budget is a subscription nobody is watching.

## What governance owns

| File | Owns |
| :-- | :-- |
| `management-group-root.tf` | lookup of the tenant root group |
| `management-group.tf` | the group hierarchy, with `import` blocks for pre-existing groups |
| `management-group-default.tf` | where new subscriptions land |
| `management-group-rbac.tf` | who may act at group scope |
| `subscription.tf`, `subscription-group.tf` | subscriptions, their budgets and access groups |
| `state-storage-accounts.tf` | the Terraform state backends |
| `finops-cost-export.tf` | the billing export feeding cost reporting |
| `resource-groups.tf` | the resource groups the above live in |

Nothing that belongs to a workload belongs here. If a change is about an application, it belongs to that application's domain.

## State storage

The state backends are themselves provisioned by governance, which makes the ordering explicit: governance is bootstrapped first, everything else initialises against what it created.

```
gl-rg-swc-terraform-state-prod-001
└── glswctfsp001                 # one container, one blob key per workspace
```

State is accessed with `use_azuread_auth=true` - the pipeline identity, never an account key. See [Workspaces](../terraform/WORKSPACES.md) for how a workspace selects its key.

## Cost visibility

A FinOps export is configured at **billing account scope** and lands in a dedicated storage account, read by a named group (`gl-sga-wanted-finops-readers`). Cost data is therefore available to people who have no access to the resources that generated it, which is the point - reviewing spend must not require production access.

## Adopting what already exists

A landing zone is usually older than the code that describes it. Existing management groups and subscriptions are brought under management with `import` blocks keyed by the same `for_each` as the module call, so adoption is a plan a human can read:

```hcl
import {
  for_each = { for mg in var.management_groups : mg.name => mg }
  id       = format("/providers/Microsoft.Management/managementGroups/%s", each.value.name)
  to       = module.management_group[each.key].azurerm_management_group.this
}
```

> A role assignment or group that Terraform did not create is imported, never deleted. Deleting to "clean up a conflict" at this layer removes real people's access.

> _The landing zone is the one place where a mistake is expensive for everyone._
---
<sup><sub>_2024 &copy; All rights reserved - WANTED.solutions s.r.o. [<@wanted-solutions>](https://github.com/wanted-solutions)_</sub></sup>
