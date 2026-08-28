---
title: "Device Association for Windows Autopilot Device Preparation — What It Is and Why It Matters"
date: 2026-08-28
draft: false
categories: ["post"]
tags: ["Intune", "Autopilot", "Windows", "MDM", "Modern Workplace", "Endpoint Management"]
description: "Microsoft just announced Device Association for Windows Autopilot Device Preparation. Hardware-backed device trust before enrollment begins — here is what changed and why it matters for IT admins."
---

Microsoft announced Device Association for Windows Autopilot Device Preparation on August 27, 2026. It is one of those features that sounds incremental until you understand what problem it actually solves.

---

## The problem it solves

Traditional Autopilot ties the provisioning experience to the user signing in. That works well in most scenarios, but it breaks down when you need the *device* — not the user — to determine which policy and experience gets applied.

A concrete example: one employee enrolls multiple devices that serve different purposes. A shared kiosk, a field laptop, a standard office machine. With user-based assignment, whoever signs in gets their profile. With device association, each piece of hardware gets its own policy attached to it before enrollment even begins — regardless of who signs in.

That shift from user-centric to device-centric provisioning is the core of what this feature delivers.

---

## What device association actually does

Before this feature, Autopilot Device Preparation recognized devices based on the user signing in during OOBE. Device association adds a new step before that: binding the physical hardware to your tenant using hardware-backed attestation.

The association is stored in the device's UEFI firmware — not in the OS, not in a registry key. It persists across Windows reset, OS reinstallation, and removal of enrollment. That makes it genuinely durable in a way that software-based identifiers are not.

When an associated device connects to a network during OOBE, it finds its pre-association record automatically and completes the trust relationship. No manual intervention from the employee.

Associated devices are also automatically marked as corporate-owned, which means you no longer need to upload a separate corporate identifier for those devices. If your tenant blocks personally owned Windows devices via enrollment restrictions, device association handles that automatically.

---

## The workflow

The process is designed as a clear handoff between IT and the device:

1. **Create the device preparation policy** in Intune — apps, scripts, deployment settings, OOBE experience, optional device name template.
2. **Export device information** — during OOBE a technician opens the Autopilot menu and exports a DeviceLink CSV to a USB drive. For an existing device, the same data is available from Autopilot diagnostic logs.
3. **Pre-associate in Intune** — go to Devices > Enrollment > Device association > Devices, upload the CSV, and optionally assign a device preparation policy directly to the device.
4. **Complete association** — when the device connects to a network in OOBE, it finds the pre-association record and completes automatically. A technician can also trigger this manually from the Autopilot menu.
5. **Enroll and prepare** — the device receives the device-targeted policy, is marked corporate-owned, and presents the configured OOBE experience.

The association states to monitor in the Device association blade are Pre-associated (waiting to complete in OOBE), Associated (tenant affinity written to UEFI, ready for enrollment), and Pending removal.

---

## OOBE controls unlocked by device association

Beyond the policy targeting story, device association also unlocks a set of OOBE customization options that were not previously available in Device Preparation:

- Pre-configure language and region
- Automatically set keyboard layout and skip the keyboard selection screen (on ethernet; Wi-Fi connections still show it)
- Hide the Microsoft Software License Terms page
- Hide privacy settings during OOBE
- Apply a device name template using serial number or a randomized value
- Hide account-change options on company sign-in and domain error pages

For large deployments these controls meaningfully reduce the number of decisions an employee has to make during first boot and help create a consistent, repeatable experience across the fleet.

---

## Lifecycle and decommissioning

The durable nature of the UEFI-stored association is useful during normal device reuse inside the organization — reset and redeploy, the association follows the hardware.

When a device permanently leaves the organization (sold, recycled, transferred), the association must be removed. Because it is stored on the device rather than purely in the service, clearing a completed association requires running a local script on the physical device. An administrator or partner with physical access to the hardware performs this as part of the decommissioning workflow.

This is intentional design: the association is meant to be durable during the device's life inside the organization, not a soft record that can be cleared remotely without physical control of the hardware.

---

## Coexistence with traditional Autopilot

Device association is part of Autopilot Device Preparation and coexists with traditional Autopilot deployments in the same tenant. The precedence rule is straightforward: if a device is associated, the Device Preparation deployment takes precedence; if it is not associated, the traditional Autopilot registration takes precedence.

That gives organizations a clean migration path — introduce device association for new hardware while existing Autopilot registrations continue working without changes.

---

## Requirements

- Physical Windows 11 device with TPM 2.0 enabled and in a healthy state
- Virtual machines are not supported — device association relies on hardware-backed identity verification

---

## Why this matters for Modern Workplace admins

Device association is a meaningful step toward hardware-rooted device identity in Intune-managed environments. The UEFI persistence story is particularly significant: this is not a soft Intune object that disappears on a wipe. The trust follows the hardware, which is how enterprise device management should work.

For organizations running large Autopilot deployments, the ability to target policy at the device rather than the user, combined with the extended OOBE controls, makes provisioning more predictable and reduces the surface for configuration drift.

Worth testing in a pilot group if you are running Autopilot Device Preparation.

---

## References

- [Microsoft Tech Community — Introducing device association for Windows Autopilot device preparation](https://techcommunity.microsoft.com/blog/intunecustomersuccess/introducing-device-association-for-windows-autopilot-device-preparation/4550603)
- [Overview of Windows Autopilot device association](https://learn.microsoft.com/autopilot/device-preparation/device-association/overview)
- [Requirements for Windows Autopilot device association](https://learn.microsoft.com/autopilot/device-preparation/device-association/requirements)
- [Set up Windows Autopilot device preparation with device association](https://learn.microsoft.com/autopilot/device-preparation/tutorial/user-driven/entra-join-device-association)
