---
title: "Onelogon — Netlogon Vulnerable Channel Bypass"
date: 2026-07-04
draft: false
tags: ["Active Directory", "Zerologon", "CVE-2020-1472", "Security", "PowerShell"]
description: "Vulnerability Overview, Exploitation Path & Remediation Guidance — Bypass del parche Zerologon via excepciones de compatibilidad legacy."
---

## 1. Summary

A publicly disclosed technique, referred to as "Onelogon", demonstrates a bypass of the Zerologon (CVE-2020-1472) remediation on Active Directory Domain Controllers. The bypass does not target a new flaw in the Netlogon Remote Protocol (MS-NRPC) itself — it targets legacy compatibility exceptions that administrators may have left enabled after the original Zerologon patch cycle (August 2020 – February 2021).

Where these exceptions are present and misconfigured, an attacker can re-establish an unauthenticated or weakly authenticated Netlogon secure channel, impersonate a computer account, and in some documented cases escalate to full domain compromise (credential dumping via DCSync-style replication, extraction of NTDS.dit).

This does not affect standard, fully patched Domain Controllers with no legacy exceptions configured. Exposure requires a non-default configuration.

## 2. Background: Zerologon (CVE-2020-1472)

Zerologon exploited a cryptographic flaw in the Netlogon session-key negotiation (AES-CFB8 with a static IV), allowing an attacker to authenticate to a Domain Controller as any computer account without credentials, and subsequently reset the machine account password, take over the DC, or extract domain hashes.

Microsoft's fix (rolled out in two phases, Aug 2020 and Feb 2021) enforced secure RPC for all Netlogon connections by default. To avoid breaking legacy/non-Windows devices that could not support secure RPC, Microsoft provided two temporary compatibility mechanisms:

- **Group Policy:** "Domain Controller: Allow vulnerable Netlogon secure channel connections"
- **Registry value** `VulnerableChannelAllowList` under `HKLM\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters`

Both were intended as a short-term bridge. In practice, many environments configured these during the 2020–2021 rollout under time pressure and never reverted them.

## 3. Exploitation Path (Conceptual)

At a high level, the Onelogon technique works as follows when a vulnerable configuration is present:

- The allow-list exempts one or more accounts from the Zerologon secure-RPC enforcement.
- An attacker with network access identifies an exempted account.
- The attacker establishes a Netlogon secure channel without modern authentication protections.
- Depending on privileges, this can lead to full Active Directory compromise.

No exploit code or step-by-step attack procedure is reproduced here. This document is scoped to detection and remediation.

## 4. Detection & Audit Procedure

### 4.1 Registry check — single DC

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" `
  -Name "VulnerableChannelAllowList" -ErrorAction SilentlyContinue
```

- Nothing returned → healthy state.
- `VulnerableChannelAllowList : *` → **CRITICAL**. Every account is exempt.
- `VulnerableChannelAllowList : DESKTOP-OLD01$,PRINTSRV$` → specific accounts exempt. Must be justified.

### 4.2 Registry check — all Domain Controllers

```powershell
$DCs = (Get-ADDomainController -Filter *).HostName

foreach ($dc in $DCs) {
    $val = Invoke-Command -ComputerName $dc -ScriptBlock {
        Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" `
            -Name "VulnerableChannelAllowList" -ErrorAction SilentlyContinue
    }
    [PSCustomObject]@{ DC = $dc; AllowList = $val.VulnerableChannelAllowList }
}
```

### 4.3 Group Policy check

```powershell
Get-GPO -All | ForEach-Object {
    $report = Get-GPOReport -Guid $_.Id -ReportType Xml
    if ($report -match "VulnerableChannelAllowList") {
        Write-Output "Match found in GPO: $($_.DisplayName)"
    }
}
```

Check every GPO linked to the Domain Controllers OU — not just Default Domain Controllers Policy.

### 4.4 Event log correlation

```powershell
Get-WinEvent -LogName System -ComputerName <DCName> |
    Where-Object { $_.ProviderName -eq "Netlogon" } |
    Select-Object TimeCreated, Id, Message -First 50
```

- **Event ID 5829/5830** → Netlogon allowed a vulnerable connection. Account is actively using the exception.
- **Event ID 5827/5828** → Netlogon denied a vulnerable connection.

## 5. Remediation

Do not delete the registry value or disable the GPO immediately — confirm first, then remediate, then re-verify.

- **Step 1 — Inventory:** record every account found, which DCs, and recent activity (Section 4.4).
- **Step 2 — Owner confirmation:** identify who owns the device and whether it still needs the exception.
- **Step 3a — If no longer needed:** remove only that account from the list.

```powershell
$current = (Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters").VulnerableChannelAllowList
$updated = ($current -split "," | Where-Object { $_ -ne "DESKTOP-OLD01$" }) -join ","

Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" `
    -Name "VulnerableChannelAllowList" -Value $updated
```

- **Step 3b — If the value is `*` or list is empty after cleanup:** remove the property entirely and disable the GPO.

```powershell
Remove-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" `
    -Name "VulnerableChannelAllowList" -ErrorAction SilentlyContinue
```

- **Step 3c — If a legacy device genuinely still needs the exception:** keep only that specific account name (never a wildcard), document the business reason, get security sign-off, and set a 90-day review reminder.
- **Step 4 — Re-verify:** re-run Section 4 checks on every DC and confirm via `gpupdate`.
- **Step 5 — Going forward:** alert on future changes to this registry value via a scheduled task or SIEM rule.

## 6. References

- CVE-2020-1472 — Netlogon Elevation of Privilege Vulnerability (Zerologon)
- Microsoft: `VulnerableChannelAllowList` / Netlogon secure channel enforcement guidance
- Neff, Holl, Borgolte — [Onelogon: Taking over Active Directory Accounts via Netlogon](https://github.com/rub-softsec/onelogon), USENIX WOOT 2026
