---
title: "Stale Accounts and Old Passwords in Active Directory - Audit and Remediation"
date: 2026-07-15
draft: false
tags: ["Active Directory", "Identity", "Security", "PowerShell", "IAM"]
description: "Enabled accounts with passwords unchanged for years are one of the most common and exploited weaknesses in Active Directory. Here's how to find them and what to do about it."
---

## 1. The Reality of Active Directory

Every AD environment accumulates history. Migrations, acquisitions, staff turnover, legacy applications — they all leave traces. The result is almost always the same: enabled accounts that nobody remembers, service accounts with passwords set years ago, and exceptions that were "temporary" and never cleaned up.

This is exactly the low-hanging fruit attackers look for first. Why spend time trying to break into a well-protected account with MFA when there is a 20-year-old enabled account sitting next to it — or a service account with a 5-year-old password and access to something critical?

> **Attackers don't need to be sophisticated. They just need to find your exceptions.**

---

## 2. What to Look For

The most common findings in a stale account audit:

| Finding | Risk |
|---|---|
| Enabled user accounts with passwords older than 365 days | High - likely forgotten or unmanaged |
| Service accounts with passwords never changed | High - often over-privileged and never rotated |
| Accounts with no recent logon activity but still enabled | Medium - forgotten accounts that could be taken over |
| Accounts with "Password never expires" flag | High - exempt from policy, often forgotten |

---

## 3. Audit with PowerShell

### 3.1 Find enabled accounts with passwords older than 365 days

```powershell
Get-ADUser -Filter 'enabled -eq $true' -Properties Name, PwdLastSet, lastlogonTimestamp |
    Select-Object Name,
        @{N='PwdLastSet'; E={[DateTime]::FromFileTime($_.PwdLastSet)}},
        @{N='LastLogonTimestamp'; E={[DateTime]::FromFileTime($_.lastlogonTimestamp)}} |
    Where-Object { $_.PwdLastSet -le (Get-Date).AddDays(-365) } |
    Sort-Object -Property PwdLastSet
```

### 3.2 Find accounts with "Password never expires"

```powershell
Get-ADUser -Filter 'PasswordNeverExpires -eq $true -and Enabled -eq $true' `
    -Properties Name, PwdLastSet, PasswordNeverExpires |
    Select-Object Name, PasswordNeverExpires,
        @{N='PwdLastSet'; E={[DateTime]::FromFileTime($_.PwdLastSet)}} |
    Sort-Object PwdLastSet
```

### 3.3 Find enabled accounts with no recent logon (90+ days)

```powershell
$cutoff = (Get-Date).AddDays(-90)

Get-ADUser -Filter 'Enabled -eq $true' -Properties lastlogonTimestamp |
    Select-Object Name,
        @{N='LastLogon'; E={[DateTime]::FromFileTime($_.lastlogonTimestamp)}} |
    Where-Object { $_.LastLogon -lt $cutoff } |
    Sort-Object LastLogon
```

### 3.4 Find service accounts specifically

```powershell
Get-ADUser -Filter 'Enabled -eq $true' -Properties Name, PwdLastSet |
    Where-Object { $_.Name -match "^(svc|sa|srv|service)[-_]" } |
    Select-Object Name,
        @{N='PwdLastSet'; E={[DateTime]::FromFileTime($_.PwdLastSet)}} |
    Sort-Object PwdLastSet
```

---

## 4. The Real Work - What to Do With the Results

Finding the accounts is the easy part. The real work is the investigation:

**For each account found, answer:**

- Is the account still needed?
- Who owns it?
- Is it a user account or a service account?
- Where is it used?
- What can it access?

**Then act:**

- **No longer needed** - disable first, wait 30 days, then delete. Never delete immediately.
- **Service account, still in use** - rotate the password, document where it is used, set a recurring reminder to rotate again.
- **User account, still active** - force password reset, enroll in MFA if not already.
- **Unknown owner** - escalate to management, disable pending confirmation.

> **Never delete an account without first disabling it and waiting. Something will break if the account was still in use somewhere undocumented.**

---

## 5. Automate and Repeat

- **Scheduled task or runbook** - run the PowerShell queries monthly and export results for review.
- **Microsoft Entra ID Governance - Access Reviews** - automate periodic reviews of group memberships and account activity.
- **ADProbe** - a free, open-source PowerShell-based tool that scans Active Directory for vulnerabilities and persistence methods.

---

## 6. Key Takeaways

- Every AD has stale accounts. The question is whether you know about them before an attacker does.
- Old passwords on enabled accounts are one of the most exploited weaknesses in enterprise environments.
- PowerShell native queries are enough to get a full picture. No extra tooling required to start.
- The audit is quick. The remediation takes time, especially service accounts. Start now.

---

## 7. References

- [ADProbe - Free AD Security Scanner](https://github.com/MarkoH17/ADProbe)
- [Microsoft Docs - Get-ADUser](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-aduser)
- [Microsoft Entra ID Governance - Access Reviews](https://learn.microsoft.com/en-us/entra/id-governance/access-reviews-overview)
