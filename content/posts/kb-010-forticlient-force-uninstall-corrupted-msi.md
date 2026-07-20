---
title: "FortiClient Force Uninstall - Corrupted MSI Recovery"
date: 2026-07-16
draft: false
tags: ["FortiClient", "Troubleshooting", "Windows", "PowerShell", "Modern Workplace"]
description: "How to force uninstall FortiClient when the MSI installer is corrupted or the original package is missing. Step by step from GUID extraction to manual registry cleanup."
---

## Symptom

When attempting to uninstall FortiClient via Add/Remove Programs or an MDM platform, the uninstaller throws an error similar to:

> *"The network resource is unavailable"* or *"Windows Installer cannot find the original package"*

This happens because the Windows Installer is looking for the original FortiClient.msi in a path that no longer exists.

---

## Option 1 — Force Uninstall via GUID

### Step 1 — Get the product GUID

Open PowerShell as Administrator:

```powershell
wmic product where "name like 'FortiClient%'" get IdentifyingNumber
```

Output example: {01CDBF14-709C-4840-B813-DC49A18A943C}

### Step 2 — Force uninstall

```powershell
msiexec /x {YOUR-GUID-HERE} /qn /norestart
```

Notes:
- /qn runs silently. Check Task Manager for msiexec.exe to confirm it is running.
- Remove /qn if you want to see the progress bar.
- The process can take 1-2 minutes.
- Restart after uninstall — FortiClient installs WFP network drivers that may not release cleanly without a reboot.

### Step 3 — Optional: capture uninstall log

```powershell
msiexec /x {YOUR-GUID-HERE} /qn /norestart /l*v C:\Temp\forticlient_uninstall.log
```

---

## Option 2 — Microsoft Fix It Tool

If the GUID method fails, use the Microsoft Program Install and Uninstall Troubleshooter:

1. Search for "Fix problems that block programs from being installed or removed" on support.microsoft.com.
2. Download and run it.
3. Select Uninstall mode and choose FortiClient from the list.

---

## Option 3 — FortiClientCleanupTool

Fortinet's dedicated cleanup tool removes all traces without relying on the original MSI:

1. Download FortiClientCleanupTool from support.fortinet.com.
2. Run as Administrator.
3. Restart after completion.

---

## Manual Post-Uninstall Cleanup

Run this after confirming FortiClient no longer appears in Programs and Features.

### Remove residual folders

```powershell
Remove-Item "C:\Program Files\Fortinet" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "C:\ProgramData\Fortinet" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:LOCALAPPDATA\Fortinet" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:APPDATA\Fortinet" -Recurse -Force -ErrorAction SilentlyContinue
```

### Remove registry keys

```powershell
Remove-Item "HKLM:\SOFTWARE\Fortinet" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "HKCU:\Software\Fortinet" -Recurse -Force -ErrorAction SilentlyContinue
```

> Warning: HKLM:\SOFTWARE\Fortinet may contain keys for other Fortinet products. Review with regedit before deleting if unsure.

### Restart

Restart the machine. After restarting verify FortiClient no longer appears in Programs and Features and that network connectivity is working normally.

---

## Managed Environments (Intune / SCCM / GPO)

If FortiClient is managed via Intune or SCCM, force uninstalling locally may cause the platform to reinstall it automatically on the next sync.

- Check for active deployments targeting the device before proceeding.
- For Intune: Apps > FortiClient > Device install status.
- Remove the assignment first, then uninstall.

---

## References

- Fortinet Support Portal: https://support.fortinet.com
- Microsoft Fix It Tool: https://support.microsoft.com/en-us/topic/fix-problems-that-block-programs-from-being-installed-or-removed-cca7d1b6-65a9-3d98-426b-e9f927e1eb4d
- Microsoft Docs msiexec: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/msiexec
