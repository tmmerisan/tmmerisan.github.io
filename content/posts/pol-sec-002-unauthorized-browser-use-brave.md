---
title: "POL-SEC-002 - Unauthorized Browser Use Policy: Brave"
date: 2026-08-01
draft: false
categories: ["policy"]
tags: ["Policy", "Security", "Browser", "Endpoint", "Template"]
description: "Free policy template covering the use of Brave browser on corporate endpoints. Addresses Tor integration, VPN bypass, DLP evasion, and shadow IT risks."
---

> **Free template.** This policy is provided as a starting point for IT and security teams. Adapt it to your organisation's needs, jurisdiction, and tooling. Attribution appreciated but not required.

---

## Document Information

| | |
|---|---|
| **Reference** | POL-SEC-002 |
| **Version** | 1.0 |
| **Issue Date** | August 2026 |
| **Scope** | All employees and contractors |
| **Classification** | Internal Use |
| **Owner** | IT Department / Cybersecurity |
| **Next Review** | August 2027 |

---

## 1. Purpose

This policy defines the organisation's position on the use of Brave browser on corporate endpoints and establishes the rules for browser software management across the work environment.

Its purpose is to protect the organisation's network perimeter, data loss prevention controls, and security monitoring capabilities from being bypassed through the use of unauthorised browser software with built-in anonymisation and traffic evasion features.

---

## 2. Scope

This policy applies to:

- All employees in the organisation.
- Contractors, consultants, and external workers who access company systems or information.
- All corporate-managed devices, regardless of operating system.
- Personal devices used for work-related activities (BYOD) that access corporate resources.

---

## 3. Background: Why Brave Is a Specific Risk

Brave is a legitimate consumer browser with a privacy-first design. However, several of its built-in features create specific risks in a corporate environment that do not exist with standard enterprise browsers:

| Feature | Corporate Risk |
|---|---|
| **Tor integration** (Private Window with Tor) | Allows users to route traffic through the Tor network, bypassing corporate proxy, network filtering, and SOC visibility entirely. Traffic becomes uninspectable. |
| **Brave VPN** | Routes traffic through Brave's own VPN infrastructure, evading corporate DLP controls, Netskope/CASB inspection, and network logging. |
| **Aggressive ad and tracker blocking** | May interfere with corporate monitoring tools, telemetry, or web applications that depend on specific scripts loading correctly. |
| **Non-managed installation** | Users install Brave independently without IT involvement, creating an unmanaged browser outside MDM policy scope, ADMX controls, and enterprise configuration. |

The combination of Tor and VPN features means a user running Brave can effectively create an uninspectable tunnel out of the corporate network from a managed endpoint — bypassing controls that the organisation has invested in and is audited against.

---

## 4. Policy Position

### 4.1 Brave is not an approved corporate browser

Brave is not included in the organisation's approved software catalogue. Its installation on corporate devices is not permitted without explicit authorisation from the IT department and Cybersecurity team.

The absence of an explicit block does not imply approval. Software not listed in the approved catalogue is unauthorised by default.

### 4.2 Approved browsers

The following browsers are approved for corporate use:

- **Microsoft Edge** — primary recommended browser. Fully manageable via Intune ADMX policies, integrates with Entra ID and Microsoft 365, supports enterprise identity separation.
- **Google Chrome** — permitted where required for specific business applications, subject to the organisation's Chrome usage policy (see POL-SEC-003).
- Any other browser formally approved and communicated by the IT department.

### 4.3 Existing installations

Users who have installed Brave on a corporate device prior to this policy must:

1. Uninstall Brave from the device within the grace period communicated by IT.
2. Migrate any bookmarks or saved data to an approved browser.
3. Report any business use case that prompted the installation to the Service Desk so IT can evaluate alternatives.

Failure to comply within the grace period will result in forced removal via MDM and escalation per section 6.

---

## 5. IT Enforcement Actions

The IT department will enforce this policy through the following technical controls:

**Detection:**
- Inventory scan of installed applications via MDM to identify Brave installations across the estate.
- Alert on new Brave installations via endpoint management tooling.

**Remediation:**
- Deployment of an application uninstall script via MDM targeting devices with Brave detected.
- Addition of Brave to the organisation's software restriction policy / application control blocklist.

**Prevention:**
- Block Brave download domains at the web filtering / proxy layer where technically feasible.
- Enforce application allowlisting on managed endpoints to prevent unauthorised software installation.

---

## 6. Associated Risks

The use of Brave on corporate devices may result in:

- Complete bypass of network security controls (DLP, CASB, proxy filtering, SOC monitoring) via Tor or VPN features.
- Exfiltration of corporate data through uninspectable traffic channels.
- Regulatory non-compliance where network monitoring and data protection controls are required by audit or contractual obligation.
- Inability to investigate security incidents involving traffic routed through Tor or Brave VPN.
- Reputational and legal exposure if corporate data is involved in an incident traced to an unmanaged browser.

---

## 7. Consequences of Non-Compliance

Violations of this policy will be handled in accordance with the organisation's disciplinary framework and applicable labour legislation.

- **First instance / unintentional:** formal notice, mandatory removal, user education.
- **Repeated or intentional use of Tor/VPN features to bypass controls:** escalation to HR and management, potential disciplinary action.

Any security incident arising from browser-related policy violations must be reported to the IT department and Cybersecurity team immediately.

---

## 8. Exception Process

Users or teams with a legitimate business need for Brave (e.g. web development, browser compatibility testing) may request an exception by:

1. Opening a ticket with the Service Desk describing the business justification.
2. Obtaining approval from their area manager.
3. Receiving formal written authorisation from the IT department and Cybersecurity team.

Approved exceptions will be time-limited, scoped to specific devices, and reviewed periodically. Tor and VPN features must remain disabled on any exception-approved installation.

---

## 9. Responsibilities

| Role | Responsibility |
|---|---|
| **Employees / Users** | Uninstall Brave if present. Use only approved browsers. Report business needs via Service Desk. |
| **Area Managers** | Communicate policy to their teams. Escalate non-compliance. Approve exception requests where justified. |
| **IT Department / Service Desk** | Detect and remediate Brave installations. Manage exception requests. Maintain approved software catalogue. |
| **Cybersecurity / SOC** | Define blocking controls. Investigate incidents involving network bypass. Maintain application blocklist. |
| **Management** | Approve this policy and ensure enforcement resources are in place. |

---

## 10. Validity and Review

This policy comes into force on the date of its approval and remains valid for one year, at which point it will be reviewed and updated as necessary.

---

## Adaptation Guide

When adapting this template to your organisation:

- Review section 4.2 and replace with your actual approved browser list.
- Adjust section 5 enforcement actions to match your MDM platform (Intune, Jamf, etc.) and security tooling (Netskope, Zscaler, etc.).
- Reference POL-SEC-003 if you have a separate Chrome usage policy.
- Define the grace period in section 4.3 based on your remediation timeline.
- Align section 7 with your HR disciplinary framework and local labour law.

---

*Template authored by [Tony Merisan](https://tonymerisan.com) — Enterprise IT Engineer. Licensed for free use and adaptation. Attribution appreciated.*
