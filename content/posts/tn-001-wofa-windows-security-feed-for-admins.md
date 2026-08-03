---
title: "TN-001 - WOFA: Windows Organized Feed for Admins"
date: 2026-08-03
draft: false
categories: ["note"]
tags: ["Windows", "Patch Tuesday", "CVE", "Tools", "Security"]
description: "WOFA is a free, machine-readable feed of Windows security update data — CVEs, exploitation status, and build versions per OS release. Sourced from MSRC and CISA KEV."
---

Stumbled across this while browsing: [WOFA](https://wofa.dev) — Windows Organized Feed for Admins.

It is essentially a clean, machine-readable aggregation of Windows security update data sourced from MSRC and CISA KEV. What it gives you at a glance:

- CVE count per OS version (Windows 10, 11, Server 2016 through 2025)
- Actively exploited CVEs flagged separately
- Latest Patch Tuesday build versions per release
- CISA KEV (Known Exploited Vulnerabilities) flags per version
- JSON feed and RSS available for automation

As of today (August 2026), it is tracking 11,711 CVEs across 13 OS versions, with 153 actively exploited. Windows Server 2025 has CVE-2026-56155 flagged as a KEV — worth checking if your Server 2025 fleet has the July 2026 update applied.

The JSON feed endpoint is public and free:
GET https://wofa.dev/v1/windows_data_feed.json

Useful if you want to pull patch status into a dashboard, a compliance check script, or a runbook. Inspired by [SOFA](https://sofa.macadmins.io) (the macOS equivalent from the Mac Admins community).

Worth bookmarking if you manage Windows endpoints and want a faster way to check exploitation status without digging through MSRC manually.

Source: [wofa.dev](https://wofa.dev) — built by Josh Tucker. Open source on [GitHub](https://github.com/Josh-Tucker/WOFA).
