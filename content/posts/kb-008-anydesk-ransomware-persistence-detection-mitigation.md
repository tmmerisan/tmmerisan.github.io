---
title: "AnyDesk Abused for Ransomware Persistence — Detection & Mitigation"
date: 2026-07-15
draft: false
tags: ["Ransomware", "AnyDesk", "RMM Abuse", "Detection", "Intune", "PowerShell", "Sigma"]
description: "32 ransomware groups use AnyDesk for persistence via a single CLI command. Here's how the attack works, how to detect it, and how to block it for free via Intune."
---

## 1. The Problem — RMM Tools as Ransomware Infrastructure

Remote Monitoring and Management (RMM) tools like AnyDesk are legitimate software used by IT teams worldwide. That legitimacy is exactly why ransomware groups abuse them — they blend into normal enterprise traffic and are often missed by security controls focused on known malware signatures.

According to the Ransomware Tool Matrix, **32 ransomware groups** use AnyDesk for persistence — making it the most abused tool in the RMM category. The technique is simple, reliable, and requires no custom malware.

---

## 2. How the Attack Works

The attacker establishes persistence with a single piped command:

```cmd
echo Admin#123 | AnyDesk.exe --set-password
```

This sets a known password on AnyDesk, giving the attacker permanent remote access to the compromised machine. Two details make this particularly interesting from a defensive perspective:

- **The password appears in plaintext** in the command line — visible in process creation logs if you're collecting them.
- **`--set-password` survives executable renaming** — attackers sometimes rename `AnyDesk.exe` to evade detection by filename, but the flag still works regardless of what the binary is called.

The attacker can then connect to the machine at any time using AnyDesk's infrastructure, even after the initial intrusion vector is closed.

---

## 3. Detection

### 3.1 Hunt for the command line pattern

The most reliable detection is process creation logging (Event ID 4688 with command line auditing enabled, or via EDR).

Hunt for:
- The `--set-password` flag in any process command line
- An `echo` pipe combined with `AnyDesk.exe` (or any renamed variant)

```powershell
# Search Windows Security event log for --set-password flag
Get-WinEvent -LogName Security | Where-Object {
    $_.Id -eq 4688 -and $_.Message -match "--set-password"
} | Select-Object TimeCreated, Message
```

> **Note:** Event ID 4688 command line logging must be enabled via Group Policy or Intune for this to work:
> `Computer Configuration > Administrative Templates > System > Audit Process Creation > Include command line in process creation events`

### 3.2 Sigma Rule

The community Sigma rule **"AnyDesk Piped Password Via CLI"** covers this technique. If you have a SIEM (Microsoft Sentinel, Splunk, Elastic), import and enable this rule. It detects:

- Process name matching `AnyDesk.exe` or renamed variants via `--set-password`
- The `echo` pipe pattern in the command line

Search for it in the [SigmaHQ repository](https://github.com/SigmaHQ/sigma) under `windows/process_creation`.

### 3.3 Key indicators to hunt

| Indicator | What it means |
|---|---|
| `--set-password` in any command line | AnyDesk password being set — investigate immediately |
| `echo \| AnyDesk.exe` pattern | Classic piped password setup |
| AnyDesk running from unusual paths (`%TEMP%`, `%APPDATA%`) | Likely attacker-dropped binary |
| AnyDesk running as SYSTEM or under unexpected parent process | Lateral movement or privilege escalation |

---

## 4. Mitigation

### 4.1 Free mitigation — Deny folder permissions via Intune

If AnyDesk is not an authorized tool in your environment, you can block its installation for free by pre-creating its expected folder path and denying write access to users. AnyDesk cannot create the folder if it already exists with deny permissions — so it fails to install.

This approach was shared by Nathan McNulty and works without any paid app control solution:

```powershell
# Deploy via Intune as a remediation script (run as SYSTEM)
# Source: https://github.com/nathanmcnulty/nathanmcnulty/blob/main/Intune/DenyFolderPermissions.ps1

(Get-ChildItem -Path C:\Users).FullName | Where-Object { $_ -ne "C:\Users\Public" } | ForEach-Object {

    $FolderPath = "$_\AppData\Local\AnyDesk"

    try {
        if (-not (Test-Path -Path $FolderPath)) {
            New-Item -Path $FolderPath -ItemType Directory -ErrorAction Stop | Out-Null
        }

        $acl = Get-Acl -Path $FolderPath
        $usersGroup = New-Object System.Security.Principal.NTAccount("Users")
        $accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule(
            $usersGroup,
            "FullControl",
            "ContainerInherit,ObjectInherit",
            "None",
            "Deny"
        )
        $acl.SetAccessRule($accessRule)
        Set-Acl -Path $FolderPath -AclObject $acl -ErrorAction Stop
    }
    catch {
        Write-Error "Failed to create folder or set permissions. Error: $_"
    }
}
```

**Deploy via Intune:**
1. Go to **Intune admin center > Devices > Scripts and remediations**
2. Create a new remediation script
3. Paste the script above as the **detection/remediation script**
4. Run as **SYSTEM**, 64-bit PowerShell
5. Target your device groups

### 4.2 Additional mitigations

- **Application control (WDAC / AppLocker)** — block unauthorized executables by publisher or hash. More robust than folder deny but requires more setup.
- **Enable process creation command line auditing** — essential for detecting this and many other techniques.
- **Block AnyDesk domains at firewall/proxy level** if not an authorized tool: `relay.anydesk.com`, `*.anydesk.com`
- **Allowlist authorized RMM tools** and alert on any other RMM binary executing in the environment.

---

## 5. Key Takeaways

- RMM abuse is one of the most common ransomware persistence techniques — not exotic malware, but legitimate tools turned against you.
- The AnyDesk `--set-password` technique is trivially simple but leaves a clear forensic trail in process creation logs **if you are collecting them**.
- The folder deny approach is free, low-risk, and deployable via Intune in minutes — a good quick win while a proper app control policy is being built.

---

## 6. References

- [Ransomware Tool Matrix](https://github.com/BushidoUK/Ransomware-Tool-Matrix)
- [SigmaHQ — AnyDesk Piped Password Via CLI](https://github.com/SigmaHQ/sigma)
- [Nathan McNulty — DenyFolderPermissions.ps1](https://github.com/nathanmcnulty/nathanmcnulty/blob/main/Intune/DenyFolderPermissions.ps1)
- [Microsoft Docs — Audit Process Creation](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/audit-process-creation)
