---
title: "SC-002 - Get Users Without MFA Registered (Entra ID / Microsoft Graph)"
date: 2026-09-04
draft: false
categories: ["script"]
tags: ["PowerShell", "Microsoft Graph", "Entra ID", "MFA", "Security", "Identity"]
description: "PowerShell script using Microsoft.Graph to identify all enabled users with no MFA authentication methods registered. Exports results to CSV. Designed for Entra ID P1/P2 environments."
---

## What it does

Connects to Microsoft Graph and audits all enabled users in the tenant. For each user it checks registered authentication methods and identifies anyone with no MFA method registered — only the default password method.

Results are exported to a timestamped CSV at `C:\Temp\UsersWithoutMFA_YYYYMMDD_HHMMSS.csv`.

Useful for:

- Identifying MFA gaps before enforcing Conditional Access policies
- Periodic compliance audits on authentication posture
- Reporting to management on MFA adoption across the tenant

---

## Requirements

**Module:** Microsoft.Graph

```powershell
Install-Module Microsoft.Graph -Scope CurrentUser
```

**Permissions required (delegated):**
- `UserAuthenticationMethod.Read.All`
- `User.Read.All`

The script will prompt for consent on first run if the permissions have not been granted previously.

---

## Notes

- The script filters out the `#microsoft.graph.passwordAuthenticationMethod` type — every user has a password method by default. Only additional methods (Authenticator app, FIDO2 key, phone, etc.) count as MFA.
- Large tenants with thousands of users will take time — the Graph API checks methods per user. Progress is shown via `Write-Progress`.
- Guest users are included if their accounts are enabled. Filter them out by adding `-Filter "accountEnabled eq true and userType eq 'Member'"` to the `Get-MgUser` call if needed.
- This approach uses the **Authentication Methods** endpoint, which reflects the modern Entra ID authentication method policy — not the legacy per-user MFA state from the old portal.

---

## Source

Script available on GitHub:

[SC-002-Get-UsersWithoutMFA.ps1](https://github.com/tmmerisan/it-ops-scripts/tree/main/scripts/identity)

---

## Related

- [Microsoft Graph — List authentication methods](https://learn.microsoft.com/en-us/graph/api/authentication-list-methods)
- [Microsoft Entra — Authentication methods policy](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods-manage)
