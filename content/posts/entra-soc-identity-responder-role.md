---
title: "Entra SOC Identity Responder — The Right Role for Identity Incident Response"
date: 2026-09-04
draft: false
categories: ["post"]
tags: ["Microsoft Entra ID", "IAM", "Security", "Incident Response", "RBAC", "SOC", "Modern Workplace"]
description: "The Entra SOC Identity Responder role gives SOC analysts the minimum permissions needed to contain compromised accounts during active security incidents — without Global Admin. Here is what it does, how to assign it, and when to use it."
---

One of the most common anti-patterns in identity security is giving SOC analysts Global Administrator access — or User Administrator — because they need to disable accounts or revoke sessions during an incident. Those roles carry far more permission than incident response actually requires.

Microsoft added the **Entra SOC Identity Responder** role to address exactly this. It is a privileged built-in role designed for one specific job: identity containment during active security incidents from the Microsoft Defender portal.

---

## What it does

The role gives analysts four targeted actions — nothing more:

| Permission | What it allows |
|---|---|
| `microsoft.directory/users/disable` | Disable a compromised user account |
| `microsoft.directory/users/enable` | Re-enable the account once the incident is resolved |
| `microsoft.directory/users/invalidateAllRefreshTokens` | Force sign-out by invalidating all refresh tokens — kicks the attacker out of active sessions immediately |
| `microsoft.directory/users/password/update` | Reset the password for a compromised account |

That is the full scope. The role cannot read directory data, cannot modify group memberships, cannot touch Conditional Access, cannot create or delete users. It is purpose-built for containment, not administration.

The `invalidateAllRefreshTokens` and `password/update` actions are marked as privileged because they can directly affect authentication state. This means the role itself is a privileged role — it should be assigned via Privileged Identity Management (PIM) with just-in-time activation, not as a permanent standing assignment.

---

## Where it lives

This role is designed to be used from the **Microsoft Defender portal** (security.microsoft.com), not from the Entra admin center. During an active incident, analysts working an identity-related alert can take containment actions directly from the Defender incident timeline without switching portals.

That workflow matters: it keeps the containment action in context with the investigation, and the action is logged in both Defender and Entra audit logs for traceability.

---

## How to assign it

### Option A — Direct assignment (not recommended for production)

1. Go to **Microsoft Entra admin center** (entra.microsoft.com)
2. Navigate to **Identity > Roles & admins**
3. Search for **Entra SOC Identity Responder**
4. Click **Add assignments** and select the user or group

Direct assignment means the role is always active. Avoid this for permanent standing access.

### Option B — PIM with just-in-time activation (recommended)

1. Go to **Microsoft Entra admin center > Identity governance > Privileged Identity Management**
2. Select **Microsoft Entra roles**
3. Find **Entra SOC Identity Responder**
4. Configure the role settings — set maximum activation duration (e.g. 4 hours), require justification, optionally require approval
5. Add eligible assignments for your SOC analysts or the SOC team group
6. Analysts activate the role from PIM when an incident requires it, with a business justification recorded

PIM activation creates a full audit trail: who activated, when, for how long, and with what justification. That is the right posture for a privileged role touching authentication state.

### PowerShell assignment

For scripted or bulk assignment:

```powershell
Connect-MgGraph -Scopes "RoleManagement.ReadWrite.Directory"

$roleDefinition = Get-MgRoleManagementDirectoryRoleDefinition -Filter "displayName eq 'Entra SOC Identity Responder'"
$user = Get-MgUser -Filter "userPrincipalName eq 'analyst@yourdomain.com'"

New-MgRoleManagementDirectoryRoleAssignment -PrincipalId $user.Id -RoleDefinitionId $roleDefinition.Id -DirectoryScopeId "/"
```

---

## Containment actions in practice

When a compromised account is identified during an incident, the typical containment sequence is:

**1. Invalidate all refresh tokens immediately**
This forces the attacker out of any active sessions. If they have an active browser session or a token cached in a mail client, invalidating refresh tokens kills those sessions on next token refresh — typically within minutes.

**2. Reset the password**
Prevents the attacker from re-authenticating with the original credentials. Do this after invalidating tokens so both the session and the credential are revoked together.

**3. Disable the account**
For higher-severity incidents where you cannot immediately confirm the scope of compromise, disabling the account is the safest containment step. It blocks all authentication regardless of token state.

**4. Re-enable when resolved**
Once the account is confirmed clean — password reset, MFA re-registered, session invalidated, and any persistence mechanisms removed — the analyst re-enables the account.

All four actions leave audit log entries in Entra ID under the analyst's identity, providing a clear record of what was done and when.

---

## Why this matters for least privilege

The alternative is giving SOC analysts User Administrator or, worse, Global Administrator. Both roles carry significant permissions that have nothing to do with incident response:

- User Administrator can create and delete users, manage group memberships, reset passwords for all non-admin users, and manage licenses.
- Global Administrator can do everything.

Neither is appropriate for a role whose only job is to contain compromised accounts during active incidents.

The Entra SOC Identity Responder role gives analysts exactly what they need and nothing else. Combined with PIM for just-in-time activation, it is the correct way to handle identity containment without expanding the blast radius of the role itself.

---

## Considerations and limits

- The role can only reset passwords for **non-privileged users**. It cannot reset passwords for users who hold admin roles — those require Privileged Authentication Administrator or higher.
- The role is intended for use from the **Defender portal**. It works from the Entra admin center too, but the integrated Defender workflow is the design intent.
- Assign to a **group** rather than individual analysts where possible — this makes it easier to manage membership as the SOC team changes.
- If your environment uses PIM, configure **activation alerts** so the security team is notified whenever this role is activated.

---

## References

- [Microsoft Entra built-in roles — Entra SOC Identity Responder](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#entra-soc-identity-responder)
- [Privileged Identity Management overview](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
- [Microsoft Defender portal — Identity actions](https://learn.microsoft.com/en-us/defender-xdr/incident-response-overview)
