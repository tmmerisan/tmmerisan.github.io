---
title: "NTLM Coercion in Active Directory — Detection & Mitigation"
date: 2026-07-06
draft: false
tags: ["Active Directory", "NTLM", "Security", "Windows Server 2025", "PowerShell"]
description: "What NTLM coercion is, what changed in Windows Server 2025, and how to detect and mitigate the risk in Active Directory environments."
---

## 1. What is NTLM Coercion?

NTLM coercion is an attack technique where an adversary with network access forces a Windows machine — typically a Domain Controller — to authenticate against an attacker-controlled host. That authentication travels in NTLM format and can be captured and relayed (NTLM relay) to access other domain resources, or cracked offline.

The classic scenario:

1. The attacker runs a coercion tool (Coercer, PetitPotam, PrinterBug) against a DC.
2. The DC automatically attempts to authenticate against the attacker's IP.
3. The attacker captures the NTLMv2 hash and relays or cracks it.
4. Depending on privileges, this can lead to full domain compromise.

## 2. Common Coercion Vectors

| Vector | Protocol | Notes |
|---|---|---|
| Print Spooler (MS-RPRN) | RPC / SMB | Most historically exploited |
| EFS (MS-EFSRPC) | RPC | Requires EFS to be active |
| DFS (MS-DFSNM) | RPC / SMB | Common on DCs with DFS enabled |
| WebClient (WebDAV) | HTTP | Requires the service to be running |

## 3. What Changed in Windows Server 2025

Windows Server 2025 (and Server 2022 23H2 onwards) introduced significant changes that reduce the default attack surface:

- **Print Spooler over named pipes (SMB) is disabled by default** — the most historically exploited path no longer works in out-of-the-box configurations.
- **EFS behaves like WebClient** — the service exists but must be active to be exploitable. Not enabled by default.
- **DFS is common on DCs but not enabled by default** — if DFS is not in use, this vector does not apply.

This means tools like Coercer have limited success against Server 2025 in standard configurations. However, environments with legacy services enabled or non-hardened configurations remain vulnerable.

## 4. Detection

### 4.1 Check Print Spooler status on all DCs

```powershell
Get-Service -Name Spooler -ComputerName (Get-ADDomainController -Filter *).HostName |
    Select-Object MachineName, Status, StartType
```

The Print Spooler **should not be running on any Domain Controller**. If `Running` appears, it is an immediate risk.

### 4.2 Check EFS status

```powershell
Get-Service -Name EFS -ComputerName (Get-ADDomainController -Filter *).HostName |
    Select-Object MachineName, Status, StartType
```

### 4.3 Event Log correlation

Look for unusual outbound NTLM authentications from DCs:

```powershell
Get-WinEvent -LogName Security -ComputerName <DCName> |
    Where-Object { $_.Id -eq 4648 } |
    Select-Object TimeCreated, Message -First 50
```

**Event ID 4648** — Logon attempt using explicit credentials. An unexpected outbound authentication from a DC to an unknown or external IP is a sign of active coercion.

## 5. Mitigation

**Disable Print Spooler on all DCs** — if direct printing from the DC is not required, disabling it eliminates the most exploited vector.

```powershell
Get-ADDomainController -Filter * | ForEach-Object {
    Invoke-Command -ComputerName $_.HostName -ScriptBlock {
        Stop-Service -Name Spooler -Force
        Set-Service -Name Spooler -StartupType Disabled
    }
}
```

- **Enable EPA (Extended Protection for Authentication)** — makes it significantly harder to relay captured NTLM credentials.
- **Enforce SMB Signing** — prevents a captured hash from being relayed against SMB services.
- **Disable NTLM where possible** — favor Kerberos across the environment.
- **Monitor changes to high-risk services** — alert if Print Spooler or EFS are activated on a DC unexpectedly.

## 6. References

- Microsoft Security Advisory — Print Spooler hardening
- Microsoft Docs — Extended Protection for Authentication (EPA)
- [Coercer](https://github.com/p0dalirius/Coercer) — audit tool (authorized lab use only)
- [Rubeus](https://github.com/GhostPack/Rubeus) — Kerberos/NTLM audit tool
