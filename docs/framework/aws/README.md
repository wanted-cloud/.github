# AWS

AWS is a supported target of the [Cloud Framework](../terraform/README.md). The same three layers apply unchanged - [building blocks](../terraform/BUILDING-BLOCKS.md) wrap one resource each, a [root module](../terraform/ROOT-MODULES.md) composes a domain, a [workspace](../terraform/WORKSPACES.md) carries the values.

## Table of Contents

- [Status](#status)
- [Building blocks](#building-blocks)
- [Landing zone](#landing-zone)
- [What differs from Azure](#what-differs-from-azure)

## Status

The AWS building block set covers organisation, identity and the foundations of a Control Tower landing zone. Root modules and workspaces are defined per engagement rather than centrally, because AWS estates we run are customer owned.

## Building blocks

| Module | Wraps |
| :-- | :-- |
| `terraform-aws-organization` | the organisation and its roots |
| `terraform-aws-organization-unit` | an organisational unit |
| `terraform-aws-organization-account` | a member account |
| `terraform-aws-organization-policy` | service control and other organisation policies |
| `terraform-aws-control-tower-landing-zone` | the Control Tower landing zone |
| `terraform-aws-control-tower-control` | an individual Control Tower control |
| `terraform-aws-iam-identity-center` | IAM Identity Center |
| `terraform-aws-iam-identity-center-permission-set` | a permission set |
| `terraform-aws-iam-role`, `terraform-aws-iam-policy` | IAM roles and policies |
| `terraform-aws-iam-account-password-policy` | account password policy |
| `terraform-aws-kms-key` | a KMS key |
| `terraform-aws-s3-bucket` | an S3 bucket |
| `terraform-aws-vpc` | a VPC and its subnets |

## Landing zone

The AWS equivalent of our [Azure landing zone](../azure/LANDING-ZONE.md) is Organizations plus Control Tower: an organisation with organisational units in place of management groups, member accounts in place of subscriptions, service control policies in place of Azure policy, and IAM Identity Center permission sets in place of Entra group assignments. The principle carries over - accounts land in a default organisational unit, retirement is a move rather than a delete, and nothing workload related lives in the governance layer.

## What differs from Azure

- **The account is the isolation boundary.** Where Azure gives a subscription per domain and environment, AWS gives an account; the granularity is finer and the blast radius smaller.
- **There is no resource group.** Grouping is done with tags, so a consistent tag set is not optional.
- **Region is explicit everywhere.** Providers are aliased per region far more often than on Azure.
- **State backends are S3 plus DynamoDB** rather than a storage account container, but the rule is unchanged: one state key per workspace, no static credentials.

> _Same layering, different primitives._
---
<sup><sub>_2024 &copy; All rights reserved - WANTED.solutions s.r.o. [<@wanted-solutions>](https://github.com/wanted-solutions)_</sub></sup>
