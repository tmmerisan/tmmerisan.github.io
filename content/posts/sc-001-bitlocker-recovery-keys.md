---
title: "SC-001 - BitLocker Recovery Keys Extractor"
date: 2026-08-03
draft: false
categories: ["script"]
tags: ["BitLocker", "PowerShell", "Windows", "Security", "Endpoint"]
description: "PowerShell script to extract BitLocker recovery keys from local volumes and export them to a text file. Useful for auditing and manual key escrow verification."
---

## What it does

Extracts BitLocker recovery keys from all locally protected volumes and saves them to a timestamped file at `C:\Temp\BitLockerKeys_YYYYMMDD_HHMMSS.txt`.

Useful for:

- Auditing BitLocker key availability before a device is reprovisioned
- Manual verification when keys have not escrowed correctly to Entra ID / Intune
- Local key recovery in offline or unmanaged environments

## Requirements

- Windows 10/11 or Windows Server with BitLocker module available
- PowerShell run as Administrator
- BitLocker active on at least one volume

## Source

Script available on GitHub:

**[SC-001-Get-BitLockerRecoveryKeys.ps1](https://github.com/tmmerisan/it-ops-scripts/tree/main/scripts/security)**

```powershell
# Run as Administrator
# Output saved to C:\Temp\BitLockerKeys_<timestamp>.txt

Import-Module BitLocker -ErrorAction SilentlyContinue

$volumes = Get-BitLockerVolume | Where-Object { $_.ProtectionStatus -eq "On" }

foreach ($vol in $volumes) {
    Get-BitLockerVolume -MountPoint $vol.MountPoint |
        Select-Object -ExpandProperty KeyProtector |
        Where-Object { $_.KeyProtectorType -eq "RecoveryPassword" }
}
```

## Notes

- If the BitLocker module is not available, the script will warn and exit cleanly.
- If BitLocker is active but no recovery password protector is found, the script warns — this usually means the volume uses a TPM-only protector with no recovery key set, which is worth flagging.
- In Intune-managed environments, recovery keys should be escrowed automatically to Entra ID. Use this script to verify locally when escrow is suspected to have failed.

## Related

- [KB-010 - FortiClient Force Uninstall](/posts/kb-010-forticlient-force-uninstall-corrupted-msi/) *(example of endpoint remediation workflow)*
- Microsoft Docs — [Get-BitLockerVolume](https://learn.microsoft.com/en-us/powershell/module/bitlocker/get-bitlockervolume)
