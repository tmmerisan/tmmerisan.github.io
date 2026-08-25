---
title: "TN-002 - EntraFalcon: Lightweight Entra ID Posture Assessment Tool"
date: 2026-08-25
draft: false
categories: ["note"]
tags: ["Microsoft Entra ID", "Security", "Tools", "PowerShell", "Identity"]
description: "EntraFalcon by Compass Security is a lightweight PowerShell tool that assesses your Entra ID posture and outputs an interactive HTML report. No app registration required."
---

Found this while browsing: [EntraFalcon](https://github.com/CompassSecurity/EntraFalcon) by Compass Security.

A lightweight PowerShell tool that scans your Microsoft Entra ID tenant and generates an interactive HTML report covering:

- Privileged objects and role assignments
- Risky or misconfigured service principals
- Guest user exposure
- Conditional Access gaps
- General identity posture findings

No extra app registration needed — it uses your existing authenticated session via Microsoft Graph PowerShell. That makes it quick to run in environments where creating new app registrations requires approval.

Useful for a quick posture check before a security review, or as a periodic sanity check on your tenant configuration. Not a replacement for a full identity audit, but a solid starting point that gives you a visual, shareable report in minutes.

Worth keeping in the toolkit if you manage Entra ID.

Source: [github.com/CompassSecurity/EntraFalcon](https://github.com/CompassSecurity/EntraFalcon)
