---
title: "Microsoft Entra Tenant Security Baseline: What to Lock Down First"
date: 2026-11-05
draft: false
categories: ["post"]
tags: ["Microsoft Entra ID", "Tenant Security", "Conditional Access", "IAM", "Hardening", "Intune", "Identity"]
description: "A practical security baseline for a Microsoft Entra tenant, split into what to do this week and what to do within 30 days. Phishing-resistant MFA, privileged access, device trust, app consent and logging, with read-only checks to verify each one."
---

New vulnerabilities and incident headlines change how urgent security feels. They rarely change the fundamentals. Most tenants share the same handful of gaps: weak sign-in methods, too many admins, unmanaged devices, loose app consent, and logs that expire before anyone looks at them.

This is the baseline I would check first in a Microsoft Entra tenant. It is split into two horizons, **this week** and **within 30 days**, and everything uses features you already pay for. For each control you get what to do, why it matters and how to verify it.

> **Licensing note.** Conditional Access needs Entra ID P1 or P2. Privileged Identity Management and Identity Protection risk policies need P2. A tenant without P1 can still turn on Security Defaults as a minimum.

---

## 1. Identities and sign-in

### This week

**Require phishing-resistant MFA, starting with administrators.**
Use a Conditional Access policy with the **Require authentication strength** grant control and the built-in **Phishing-resistant MFA** strength. That strength accepts FIDO2 security keys and passkeys, Windows Hello for Business and certificate-based authentication. SMS and voice do not satisfy it. Microsoft recommends enforcing it at minimum on roles such as Global Administrator, Application Administrator, Authentication Administrator and Billing Administrator. Start there, then widen to all users.

Microsoft is also moving the whole platform in this direction: passkeys become the default method and Microsoft-provided SMS and voice MFA are being retired. The timeline and action plan are in [Entra Passkeys by Default and SMS/Voice Retirement](/posts/entra-passkeys-default-sms-voice-retirement/).

**Block legacy authentication.**
Legacy protocols cannot do modern MFA, so they bypass everything above. Create a Conditional Access policy targeting client apps of type "Other clients" and set the grant to Block. Run it in report-only mode first and check the sign-in logs for anything that still depends on legacy protocols.

**Build proper emergency access accounts.**
Microsoft recommends two cloud-only emergency access accounts with a permanent Global Administrator assignment. Give them a different credential from your regular admins (a FIDO2 key is the usual choice), keep them out of any policy that could lock them out, store the credentials offline, alert on every sign-in and test them on a schedule.

**Count your Global Administrators.**
Microsoft's guidance is fewer than five, and fewer than ten privileged role assignments overall. Anything beyond what is strictly needed is extra blast radius.

### Within 30 days

- **Move privileged roles to Privileged Identity Management.** Eligible instead of permanent, activation with justification, time-limited, with an alert on every activation. The only permanent assignments should be the emergency accounts.
- **Use narrow roles for narrow jobs.** An incident responder who only needs to disable accounts and revoke sessions does not need User Administrator. See [Entra SOC Identity Responder](/posts/entra-soc-identity-responder-role/) for a least-privilege example.
- **Close the MFA registration gap.** Find every enabled user who has no method beyond a password with [SC-002: Get Users Without MFA](/posts/sc-002-get-users-without-mfa/).
- **Turn on risk-based policies** in Identity Protection (P2) for sign-in risk and user risk.
- **Clean up stale identities.** Old enabled accounts and service accounts are the cheapest way in. The audit is in [Stale Accounts and Old Passwords in Active Directory](/posts/ad-stale-accounts-old-passwords-audit/).
- **Review dynamic groups that drive access.** Frozen or drifting membership silently breaks Conditional Access and licensing. See [Entra ID's memberOf Retirement](/posts/entra-memberof-retirement-audit/).

---

## 2. Devices

### This week

**Require a compliant device for corporate data.**
Add a Conditional Access policy that requires the device to be marked compliant by Intune (or Microsoft Entra hybrid joined) for Microsoft 365 and for administrator sign-ins. Combined with phishing-resistant MFA, this makes a stolen password or a replayed session much less useful to an attacker.

**Stop personal accounts from syncing in the browser.**
A managed laptop with a personal browser profile can quietly copy corporate credentials into an account you do not control. The policy and the enforcement settings are in [POL-SEC-003](/posts/pol-sec-003-chrome-personal-accounts-corporate-devices/).

### Within 30 days

- **Put every corporate device under MDM and EDR.** Intune for management and compliance, Defender for visibility when something on a laptop starts behaving badly.
- **Control what can be installed.** Remote-access tools are a common persistence path, as shown in [AnyDesk Abused for Ransomware Persistence](/posts/anydesk-ransomware-persistence-detection-mitigation/). Browsers with built-in anonymisation are covered by [POL-SEC-002](/posts/pol-sec-002-unauthorized-browser-use-brave/).
- **Allowlist browser extensions** for Edge and Chrome through Intune policy instead of letting users install anything.
- **Provision with hardware-backed trust.** [Device Association for Autopilot Device Preparation](/posts/autopilot-device-association-windows-device-preparation/) ties a device to your tenant in UEFI, so the trust survives a reset.
- **Roll out a password manager** so credentials stop living in browsers and spreadsheets, and so credentials are only filled on the correct domain.

---

## 3. Applications and service principals

### This week

**Restrict user consent.**
By default, users can consent to apps for permissions that do not require admin consent. Microsoft recommends allowing user consent only for apps from verified publishers and only for low-risk permissions. Pair that with the admin consent workflow so users can request approval instead of being blocked with no path forward.

**Restrict who can register applications.**
Most users do not need to create app registrations. Turn that off for the default user role and give the capability to the people who actually build apps.

### Within 30 days

- **Audit service principals and app registrations.** Look for credentials that never expire, owners who have left, and permissions nobody remembers granting. Prefer certificates, managed identities and workload identity federation over client secrets.
- **Remove what is unused.** Every enterprise app and credential you delete is one less thing to defend.
- **Run a posture scan.** [EntraFalcon](/posts/tn-002-entrafalcon-entra-id-posture-assessment/) gives you a quick, shareable report of privileged objects and risky assignments.

---

## 4. Logging and detection

Controls prevent. Logs are what you have when prevention fails.

### This week

**Export the logs and keep them somewhere an attacker cannot reach.**
Entra keeps audit and sign-in logs for 7 days on the Free tier and 30 days with P1 or P2. That is short for an investigation. Use diagnostic settings to route AuditLogs and SignInLogs to a Log Analytics workspace, a storage account, an event hub or a partner solution, and set the retention you actually need. Restrict who can modify or delete that workspace.

**Alert on the high-signal events:**

- A new privileged role assignment
- Any sign-in by an emergency access account
- Authentication method changes and new MFA registrations
- Conditional Access policy changes
- New federated domains and new credentials on app registrations

Two starting queries, assuming the logs are in Log Analytics:

```kusto
AuditLogs
| where TimeGenerated > ago(24h)
| where Category == "RoleManagement"
| where OperationName has "Add" and OperationName has "role"
| project TimeGenerated, OperationName, InitiatedBy, TargetResources, Result
```

```kusto
SigninLogs
| where TimeGenerated > ago(24h)
| where UserPrincipalName in~ ("<emergency-account-1>@<tenant>.onmicrosoft.com", "<emergency-account-2>@<tenant>.onmicrosoft.com")
| project TimeGenerated, UserPrincipalName, AppDisplayName, IPAddress, ResultType
```

### Within 30 days

- Turn each query above into a scheduled alert that reaches a human.
- Add the Microsoft Graph activity logs to the same workspace if you want to see which applications call which APIs.
- Rehearse containment so the first time you disable an account and revoke sessions is not during a real incident.

---

## Quick read-only checks

Three checks you can run today. All of them only read configuration.

**Who holds Global Administrator (active assignments):**

```powershell
Connect-MgGraph -Scopes "RoleManagement.Read.Directory","Directory.Read.All"

$role = Get-MgDirectoryRole -Filter "displayName eq 'Global Administrator'"
Get-MgDirectoryRoleMember -DirectoryRoleId $role.Id | ForEach-Object {
    [PSCustomObject]@{
        Id   = $_.Id
        Type = $_.AdditionalProperties['@odata.type']
        UPN  = $_.AdditionalProperties['userPrincipalName']
    }
}
```

PIM-eligible assignments do not appear here. Review those in the PIM blade.

**Which Conditional Access policies exist and what state they are in:**

```powershell
Connect-MgGraph -Scopes "Policy.Read.All"

Get-MgIdentityConditionalAccessPolicy -All |
    Select-Object DisplayName, State |
    Sort-Object State, DisplayName
```

Look for policies stuck in report-only mode that were validated months ago and never enforced.

**App registration and user consent settings:**

```powershell
Connect-MgGraph -Scopes "Policy.Read.All"

$p = Get-MgPolicyAuthorizationPolicy
"Users can register apps : $($p.DefaultUserRolePermissions.AllowedToCreateApps)"
"User consent policies   : $($p.DefaultUserRolePermissions.PermissionGrantPoliciesAssigned -join ', ')"
```

If the consent policies include the legacy default, users can consent to any permission that does not require admin consent. Moving to a policy limited to low-risk permissions from verified publishers is the safer setting.

---

## The baseline at a glance

| Control | Horizon | Where to verify |
|---|---|---|
| Phishing-resistant MFA for admins, then everyone | This week | Conditional Access, authentication strengths |
| Block legacy authentication | This week | Conditional Access, sign-in logs |
| Two emergency access accounts, monitored | This week | Roles and administrators, sign-in alerts |
| Fewer than five Global Administrators | This week | Roles and administrators |
| Compliant device required for corporate access | This week | Conditional Access, Intune compliance |
| Restrict user consent and app registration | This week | Enterprise applications, user settings |
| Logs exported and alerts on privileged changes | This week | Diagnostic settings, Log Analytics |
| PIM for all privileged roles | 30 days | Privileged Identity Management |
| MFA registration gap closed | 30 days | SC-002 report |
| MDM and EDR on every corporate device | 30 days | Intune, Defender |
| Extension and software allowlists | 30 days | Intune policies |
| Service principal and credential audit | 30 days | App registrations, EntraFalcon |
| Risk-based policies | 30 days | Identity Protection |

---

## What this baseline does not cover

This is the identity and device layer of a Microsoft tenant. It does not cover developer workstations, source code, supply chain or internet-facing appliances. Those matter, and they need their own baselines. But a tenant where an attacker can sign in as an administrator with a phished password has already lost, whatever else is in place.

---

## References

- [Best practices for Microsoft Entra roles](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/best-practices)
- [Manage emergency access accounts in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)
- [How Conditional Access authentication strength works](https://learn.microsoft.com/entra/identity/authentication/concept-authentication-strength-how-it-works)
- [Configure how users consent to applications](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent)
- [Microsoft Entra diagnostic settings: log options](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-diagnostic-settings-logs-options)
- [Privileged Identity Management overview](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
