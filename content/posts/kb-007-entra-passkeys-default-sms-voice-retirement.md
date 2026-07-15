---
title: "Microsoft Entra: Passkeys by Default & SMS/Voice Retirement — Admin Advisory"
date: 2026-07-15
draft: false
tags: ["Microsoft Entra", "Passkeys", "MFA", "Identity", "IAM", "Modern Workplace"]
description: "Microsoft Entra is making passkeys the default authentication method and retiring Microsoft-provided SMS and voice MFA. Here's what it is, why it's happening, and what admins need to do before February 1, 2027."
---

## TL;DR

- **September 1, 2026** — Passkeys become default. Users enabled for SMS/Voice are auto-enabled for passkeys and prompted to register.
- **February 1, 2027** — Microsoft-provided SMS and Voice MFA fully retired. No opt-out. Users with no other MFA method will be blocked until they register a passkey.
- **Action required now** — Audit who in your tenant uses SMS/Voice and start migrating them to passkeys.

---

## 1. What Are Passkeys?

Passkeys are phishing-resistant, passwordless credentials that replace both passwords and weak MFA methods like SMS OTP. Instead of a shared secret, they use a cryptographic key pair:

- The **private key** lives on the user's device (never leaves it).
- The **public key** is registered with the service (Microsoft Entra).
- Authentication is proven via biometric or device PIN — no password, no SMS code.

They are resistant to phishing, SIM-swap attacks, and credential replay. Microsoft Entra ID supports two types:

| Type | How it works | Best for |
|---|---|---|
| **Synced passkey** | Stored in a platform credential manager (iCloud Keychain, Google Password Manager) and synced across devices | Users already using a platform credential manager |
| **Device-bound passkey** | Tied to a specific device — Microsoft Authenticator, FIDO2 hardware key, Windows Hello | High-security environments, shared device scenarios |

Passkeys are included in all Microsoft Entra plans at no additional cost.

---

## 2. Why Is Microsoft Doing This?

SMS and voice OTP are among the weakest MFA methods available. They are vulnerable to:

- **SIM-swap attacks** — attacker takes over your phone number via the carrier.
- **Real-time phishing** — attacker tricks the user into entering the OTP on a fake site.
- **SS7 protocol attacks** — interception at the telecom network level.

Microsoft's position is clear: as organizations move toward AI-driven workflows, phishing-resistant authentication is no longer optional. Passkeys are the industry direction — Apple, Google, and Microsoft have all committed to the FIDO2/WebAuthn standard.

---

## 3. Retirement Timeline

| Date | What happens | What you should do |
|---|---|---|
| **September 1, 2026** | Passkeys auto-enabled for SMS/Voice users. Registration Campaign set to Microsoft Managed. Users prompted to register a passkey at next MFA sign-in (skippable). | Notify end users. Start passkey deployment. |
| **September 18, 2026** | Telecom providers available in Microsoft Security Store for review. | Evaluate providers if SMS/Voice is required for compliance. |
| **October 30, 2026** | Customers can select and configure a telecom provider from the Security Store. | Configure provider if needed for regulated scenarios. |
| **February 1, 2027** | Microsoft-provided SMS/Voice fully retired. Users with no other MFA method hit a **blocking** passkey registration prompt. **No opt-out.** | All users must be on a phishing-resistant method by this date. |

---

## 4. Action Plan for Admins

### Step 1 — Audit who uses SMS or Voice

Run the Microsoft-provided PowerShell script to identify affected users:

```powershell
# Requires Global Reader, Authentication Policy Administrator, or Security Reader
# Script: https://github.com/microsoft/entra-sms-voice-usage-analyzer
```

Any non-zero result means your tenant is in scope and needs action.

### Step 2 — Enable Passkeys in your tenant

In the **Microsoft Entra admin center**:

1. Go to **Protection > Authentication methods > Policies**.
2. Enable **Passkey (FIDO2)**.
3. Target the security group of SMS/Voice users identified in Step 1.

### Step 3 — Configure a Registration Campaign

Drive adoption proactively before September 1:

1. Go to **Protection > Authentication methods > Registration campaign**.
2. Set **State** to **Microsoft Managed**.
3. Target your SMS/Voice user group.

Users will be prompted to register a passkey at their next MFA sign-in.

### Step 4 — Communicate to end users

A smooth rollout depends on communication, not just technical config. Microsoft recommends a phased approach:

- **Awareness** — SMS/Voice is retiring, here's what's changing and why.
- **Action** — how to register a passkey on their device (Windows Hello, iOS, Android).
- **Reminder** — follow-up for users who haven't registered yet.

Use [Microsoft's end-user communication templates](https://aka.ms/mfatemplates).

### Step 5 — Handle exceptions (regulated environments)

If your organization has a genuine regulatory or operational need to keep SMS/Voice:

- Starting **October 30, 2026**, configure a customer-managed telecom provider via the **Microsoft Security Store**.
- Document the business reason (which regulation, which scenario).
- Costs are per-message and vary by provider and region — evaluate before committing.
- Default all other users to passkeys.

### Step 6 — Verify before February 1, 2027

Re-run the audit from Step 1 to confirm no users remain on SMS/Voice only. If any do, they will hit a blocking registration prompt on February 1 — **there is no opt-out for this enforcement**.

---

## 5. Key Points to Remember

- Users already on passkeys, Windows Hello for Business, or FIDO2 keys are unaffected.
- SSPR (Self-Service Password Reset) is also affected — SMS/Voice retirement applies across all Entra authentication scenarios.
- A temporary opt-out for the September 1 changes will be available from August 1, 2026 — but the February 1, 2027 enforcement has no opt-out.
- This timeline applies to **public cloud only**. Other cloud environments (GCC, GCC-H, etc.) will follow on a later schedule.

---

## 6. References

- [Microsoft Docs — Passkeys by default and SMS/Voice retirement](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sms-voice-retirement)
- [Plan a passkey deployment in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/authentication/how-to-deploy-phishing-resistant-passwordless-authentication)
- [Enable passkeys (FIDO2) for your organization](https://learn.microsoft.com/en-us/entra/identity/authentication/how-to-authentication-passkeys-fido2)
- [SMS/Voice usage analyzer script](https://github.com/microsoft/entra-sms-voice-usage-analyzer)
- [End-user communication templates](https://aka.ms/mfatemplates)
