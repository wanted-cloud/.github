# Workspaces

A **workspace** is one environment of one domain: the values it is deployed with, the state it owns, and the pipeline that applies it. Workspaces live in `production-<domain>` repositories and contain **no Terraform code at all** - the code is the paired [root module](./ROOT-MODULES.md).

## Table of Contents

- [Repository layout](#repository-layout)
- [Pairing a workspace with its root module](#pairing-a-workspace-with-its-root-module)
- [State](#state)
- [Variables and secrets](#variables-and-secrets)
- [The tfvars file](#the-tfvars-file)
- [Approvals](#approvals)
- [The README is the runbook](#the-readme-is-the-runbook)
- [Rules](#rules)

## Repository layout

```
production-<domain>/
├── terraform.tfvars                    # the environment's values - the whole point of the repo
├── pipelines/
│   └── <purpose>-provisioning.yaml     # Azure DevOps pipeline extending the shared templates
└── README.md                           # operational notes for this environment
```

## Pairing a workspace with its root module

The pipeline checks out **both** repositories side by side and points Terraform at the code in one and the values in the other:

```yaml
resources:
  repositories:
    - repository: vsts-pipeline-templates
      type: git
      name: infrastructure/vsts-pipeline-templates
      ref: refs/heads/main
    - repository: module
      type: git
      name: infrastructure/terraform-wanted-corporate
      ref: refs/heads/main

extends:
  template: templates/terraform/validate-plan-and-apply.yaml@vsts-pipeline-templates
  parameters:
    RUNTIME_REPOSITORIES:
      - self        # the workspace
      - module      # the root module
    TF_BACKEND_TYPE: azurerm
    TF_CONTEXT_DIRECTORY: terraform-wanted-corporate
    TF_VARS_FILE: ./../production-corporate/terraform.tfvars
```

`TF_CONTEXT_DIRECTORY` is the code, `TF_VARS_FILE` is the data. The `ref` on the module repository is what pins which revision of the domain a given environment runs.

## State

The root module declares only `backend "azurerm" {}`; the workspace supplies the rest as partial configuration at init:

```shell
terraform init \
  --backend-config="use_azuread_auth=true" \
  --backend-config="storage_account_name=$(TF_BACKEND_STORAGE_ACCOUNT_NAME)" \
  --backend-config="container_name=$(TF_BACKEND_CONTAINER_NAME)" \
  --backend-config="key=$(TF_BACKEND_CONTAINER_KEY)" \
  --backend-config="subscription_id=$(TF_BACKEND_SUBSCRIPTION_ID)" \
  --backend-config="resource_group_name=$(TF_BACKEND_RESOURCE_GROUP_NAME)"
```

- **One state key per workspace.** `TF_BACKEND_CONTAINER_KEY` is the isolation boundary between environments - two workspaces of the same domain differ by their key, nothing else.
- **`use_azuread_auth=true`** - state is accessed with the pipeline identity, never with a storage account key.
- The state storage accounts themselves are provisioned by the governance domain; see the [Azure landing zone](../azure/LANDING-ZONE.md).

## Variables and secrets

Pipeline settings come from Azure DevOps variable groups, layered:

| Group | Scope | Holds |
| :-- | :-- | :-- |
| `PROD-CORE-INFRASTRUCTURE` | shared by all workspaces | agent pool, tenant, common backend settings |
| `PROD-CORE-INFRASTRUCTURE-SECRETS` | shared by all workspaces | the ARM client secret reference |
| `PROD-<DOMAIN>` | one workspace | that workspace's backend key and domain settings |

Secrets are referenced, never written down: `TF_ARM_SECRET_REFERENCE` names a Key Vault backed variable. Nothing secret belongs in `terraform.tfvars` - a password lives in Key Vault and the tfvars carries the *name* of the secret, not its value.

## The tfvars file

`terraform.tfvars` is the only file in the estate that contains real identifiers, and it is meant to be read by humans:

```hcl
global_tags = {
  Lifecycle = "production"
  Owner     = "gl-sga-wanted-devops-infrastructure-owners"
  Project   = "Global infrastructure - corporate resources"
}

resource_groups = [{
  name     = "gl-rg-swc-contributors-persistence-prod-001"
  location = "swedencentral"
}]
```

- **Comment the decisions, not the syntax.** Why a server stays on a burstable SKU, why an option is left enabled for break glass access - that context has nowhere else to live.
- **Every workspace sets `global_tags`** with at least lifecycle, owner and project.
- Names follow the [Azure naming convention](../azure/NAMING-CONVENTION.md).

## Approvals

Production pipelines plan first, gate on a human, then apply. Two things worth knowing:

- `RUNTIME_ACTION: plan` on a pull request pipeline, `apply` only from the default branch.
- An in-pipeline `ManualValidation` gate **does not expand Entra groups** - only individual users resolve to an Approve button, so approvers are listed as UPNs and kept in sync with the group. An Environment approval does expand groups and is the better choice where it fits.

## The README is the runbook

A workspace README is not a project description - it documents how this environment is operated: how access is granted, which identifiers matter, what is deliberately inert and why, which work item introduced a construct. If an operator has to ask someone how a thing works, that answer belongs in the workspace README.

## Rules

1. **No `.tf` files in a workspace.** If you need code, the root module needs a change.
2. **One workspace, one state key, one environment.**
3. **No secret values in `terraform.tfvars`** - reference Key Vault by name.
4. **Never apply from a workstation.** The pipeline is the only path to production state.
5. **The module `ref` is a deliberate choice.** Bumping which revision of a domain an environment runs is a reviewable change.

> _The workspace is where infrastructure becomes a decision, not a script._
---
<sup><sub>_2024 &copy; All rights reserved - WANTED.solutions s.r.o. [<@wanted-solutions>](https://github.com/wanted-solutions)_</sub></sup>
