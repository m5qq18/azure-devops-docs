---
title: Change application connection and security policies for organizations
titleSuffix: Azure DevOps Services
description: Manage security policies for accessing organization through conditional access, OAuth, SSH, and personal access tokens (PATs).
ms.subservice: azure-devops-organizations
ms.assetid: 2fdfbfe2-b9b2-4d61-ad3e-45f11953ef3e
ms.topic: how-to
ms.author: chcomley
author: chcomley
ms.date: 05/14/2025
monikerRange: 'azure-devops'
---

# Change application connection & security policies for your organization

[!INCLUDE [alt-creds-deprecation-notice](../../includes/alt-creds-deprecation-notice.md)]

This article shows how to manage your organization's security policies that determine how users and applications can access services and resources in your organization. You can access most of these policies in **Organization settings**.

## Prerequisites

| Category | Requirements |
|--------------|-------------|
|**Permissions**| <ul><li>Org-level policies: [Project Collection Administrator](../security/look-up-project-collection-administrators.md)</li><li>Tenant-level policies: [Azure DevOps Administrator](../security/look-up-azure-devops-administrator.md)</li></ul>|

[!INCLUDE [manage-policies](../../includes/manage-policies.md)]


## Restrict authentication methods

To allow seamless access to your organization without repeatedly prompting for user credentials, applications can use authentication methods, like [OAuth](../../integrate/get-started/authentication/oauth.md), [SSH](../../repos/git/use-ssh-keys-to-authenticate.md), and [personal access token (PATs)](use-personal-access-tokens-to-authenticate.md). By default, all existing organizations allow access for all authentication methods.

You can limit access to these authentication methods by disabling these application connection policies:
- **Third-party application access via OAuth**: Enable Azure DevOps OAuth apps to access resources in your organization through OAuth. This policy is defaulted to *off* for all new organizations. If you want access to [Azure DevOps OAuth apps](../../integrate/get-started/authentication/azure-devops-oauth.md), enable this policy to ensure these apps can access resources in your organization. This policy doesn't impact [Microsoft Entra ID OAuth app access](../../integrate/get-started/authentication/entra-oauth.md).
- **SSH authentication**: Enable applications to connect to your organization's Git repos through SSH.
- Tenant admins can [**restrict global personal access token creation**](manage-pats-with-policies-for-administrators.md#restrict-creation-of-global-pats-tenant-policy), [**restrict full-scoped personal access token creation**](manage-pats-with-policies-for-administrators.md#restrict-creation-of-full-scoped-pats-tenant-policy), and [**enforce maximum personal access token lifespan**](manage-pats-with-policies-for-administrators.md#set-maximum-lifespan-for-new-pats-tenant-policy) through tenant-level policies on the _Microsoft Entra_ settings page. Add Microsoft Entra users or groups to exempt them from these policies.
- Organization admins can [**restrict personal access token creation**](manage-pats-with-policies-for-administrators.md#restrict-personal-access-token-creation-organization-policy) in their respective organizations. Subpolicies allow admins to permit the creation of packaging-only PATs or the creation of any-scope PATs to allowlisted Microsoft Entra users or groups.

When you deny access to an authentication method, no application can access your organization through that method. Any application that previously had access encounter authentication errors and lose access.

## Enforce conditional access policies 

Microsoft Entra ID allows tenants to define which users can access Microsoft resources through their [Conditional Access policy (CAP) feature](/azure/active-directory/conditional-access/overview). Tenant admins can set conditions that users must meet to gain access, such as requiring that the user:

- Be a member of a specific Entra security group
- Belong to a certain location and/or network
- Use a specific operating system
- Use an enabled device in a management system

Depending on which conditions the user satisfies, you can then permit them access, require additional checks like multifactor authentication, or block access altogether. Learn more about [conditional access policies](/azure/active-directory/active-directory-conditional-access) and [how to set one up for Azure DevOps](/azure/active-directory/conditional-access/concept-conditional-access-cloud-apps) in the Entra docs.

### CAP support on Azure DevOps

When you sign in to the web portal of a Microsoft Entra ID-backed organization, Microsoft Entra ID always performs validation for any Conditional Access policies set by tenant administrators. Since [modernizing our web authentication stack to use Microsoft Entra tokens](https://devblogs.microsoft.com/devops/full-web-support-for-conditional-access-policies-across-azure-devops-and-partner-web-properties/), we now also enforce validation for Conditional Access policies on all interactive (web) flows. 

* Using PATs on REST API calls that rely on Microsoft Entra requests requires that any sign-in policies set by tenant admins are also met. For example, if a sign-in policy requires a user sign in every seven days, you must also sign in every seven days to continue using PATs.
* If you don't want any CAPs to be applied to Azure DevOps, remove Azure DevOps as a resource for the CAP.
* We support MFA policies on web flows only. For non-interactive flows, if the user doesn't satisfy a CAP, they aren't prompted for MFA and are blocked instead.

#### IP-based conditions

If the **Enable IP Conditional Access policy validation on non-interactive flows** policy is enabled, we check IP fencing policies on non-interactive flows, such as when you use a PAT to make a REST API call.

We support IP-fencing conditional access policies (CAPs) for both IPv4 and IPv6 addresses. If your IPv6 address is being blocked, ensure that the tenant administrator configured CAPs to allow your IPv6 address. Additionally, consider including the IPv4-mapped address for any default IPv6 address in all CAP conditions.

If users access the Microsoft Entra sign-in page via a different IP address than the one used to access Azure DevOps resources (common with VPN tunneling), check your VPN configuration or networking infrastructure. Ensure to include all used IP addresses within your tenant administrator's CAPs.

## Policies by Level

| Policy | Org-level | Tenant-level |
|--------------|-------------|
| [**Third-party application access via OAuth**](#restrict-authentication-methods) | ✅ | |
| [**SSH authentication**](#restrict-authentication-methods) | ✅ |  |
| [**Log audit events**](../audit/azure-devops-auditing.md) | ✅ |  |
| [**Restrict personal access token creation**](manage-pats-with-policies-for-administrators.md#restrict-personal-access-token-creation-organization-policy) | ✅ |  |
| [**Allow public projects**](../projects/make-project-public.md) | ✅ |  |
| [**Additional protections when using public package registries**](https://devblogs.microsoft.com/devops/changes-to-azure-artifact-upstream-behavior/) | ✅ |  |
| [**Enable IP Conditional Access policy validation on non-interactive flows**](#enforce-conditional-access-policies) | ✅ |  | 
| [**External guest access**](add-external-user.md) | ✅ |  |
| [**Allow team and project administrators to invite new users**](../security/restrict-invitations.md) | ✅ |  |
| [**Request access** allows users to request access to the organization with a provided internal URL](disable-request-access-policy.md) | ✅ |  |
| [**Allow Microsoft to collect feedback from users**](../security/data-protection.md#data-privacy) | ✅ |  |
| [**Restrict organization creation**](azure-ad-tenant-policy-restrict-org-creation.md) |  | ✅ | 
| [**Restrict global personal access token creation**](manage-pats-with-policies-for-administrators.md#restrict-creation-of-global-pats-tenant-policy) | | ✅ |
| [**Restrict full-scoped personal access token creation**](manage-pats-with-policies-for-administrators.md#restrict-creation-of-full-scoped-pats-tenant-policy)| | ✅ |
| [**Enforce maximum personal access token lifespan**](manage-pats-with-policies-for-administrators.md#set-maximum-lifespan-for-new-pats-tenant-policy) | | ✅ |

