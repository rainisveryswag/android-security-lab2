# 🔓 Android Security Lab — Rooting, Verified Boot & AVB

> **Academic lab** — GCDSTE · ENSA Marrakech · Mobile Application Security Module  
> Environment: Android Virtual Device (AVD) · Fictitious data only · Isolated network

---

## 📋 Table of Contents

- [Overview](#overview)
- [Scope](#scope)
- [Prerequisites](#prerequisites)
- [Lab Steps](#lab-steps)
- [Key Concepts](#key-concepts)
- [Risk Matrix](#risk-matrix)
- [Defensive Measures](#defensive-measures)
- [Security Standards](#security-standards)
- [Command Reference](#command-reference)
- [Deliverables](#deliverables)
- [Disclaimer](#disclaimer)

---

## Overview

This lab explores the concepts of **Android rooting**, **Verified Boot**, and **Android Verified Boot 2.0 (AVB)** in a controlled, isolated environment. The goal is to understand how privilege escalation impacts system integrity guarantees, and to observe the artifacts that a privileged attacker could access on a mobile device.

The lab is purely academic and conducted on a dedicated AVD. No personal data, personal accounts, or production devices are used at any point.

---

## Scope

| Field | Value |
|---|---|
| **Application** | `app-debug.apk` (test build) |
| **Environment** | Android Virtual Device (AVD) — Google APIs image |
| **Objective** | Understand rooting mechanics and their security impact |
| **Data** | Fictitious only — no real user data |
| **Network** | Isolated test network — no internet access |

---

## Prerequisites

- Android Studio with a configured AVD (API 29+ recommended)
- ADB installed and in `PATH`
- AVD running a **Google APIs** image (required for `adb root`)
- Clean AVD — no residual applications or personal accounts

```bash
# Verify ADB detects the device
adb devices
# Expected: emulator-5554   device
```

---

## Lab Steps

### Step 1 — Root the AVD

Start the emulator with a writable system partition and activate root privileges:

```bash
emulator -avd YOUR_AVD_NAME -writable-system
adb root       # Restart ADB server with root privileges
adb remount    # Remount /system as read/write
```

**Verify root access:**

```bash
adb shell id
# Expected: uid=0(root) gid=0(root)

adb shell getprop ro.boot.verifiedbootstate
# Expected: orange or yellow (verity disabled)

adb shell getprop ro.boot.veritymode
adb shell getprop ro.boot.vbmeta.device_state
adb shell "su -c id"
```

**Permissive option (disable dm-verity):**

```bash
adb disable-verity
adb reboot
adb remount
```

**Capture logs as evidence:**

```bash
adb logcat -d | tail -n 200 > logcat_root_check.txt
```

---

### Step 2 — Install and Launch the Test App

```bash
adb install app-debug.apk
```

Document the app version — security behaviors can differ between releases.

---

### Step 3 — Define 3 Repeatable Test Scenarios

| # | Scenario | Expected Result |
|---|---|---|
| 1 | Open the home screen | Normal load, no errors |
| 2 | Search for an item | Results displayed correctly |
| 3 | Open a detail view | Navigation without crash |

Each scenario must be fully reproducible — note exact inputs, buttons clicked, and capture screenshots at each step.

---

### Step 4 — Read Android Security Fundamentals

Reference: [source.android.com/docs/security](https://source.android.com/docs/security)

Key mechanisms to summarize in your report:

- **App sandboxing** — each app is isolated with a unique UID
- **Permission model** — granular access control to sensitive resources
- **System integrity** — protection against unauthorized modifications at runtime and boot

---

### Step 5 — Verified Boot & AVB Analysis

References:
- [Verified Boot](https://source.android.com/docs/security/features/verifiedboot)
- [AVB 2.0](https://source.android.com/docs/security/features/verifiedboot/avb)

**Chain of Trust:**

```
ROM Boot → Bootloader → Kernel → System → Application
   └── each component verifies the next before trusting it
```

**Boot state interpretation:**

| State | Meaning |
|---|---|
| 🟢 `GREEN` | System verified and intact — no modifications detected |
| 🟡 `YELLOW / ORANGE` | Modified system — custom image or rooting present |
| 🔴 `RED` | Integrity compromised — potentially dangerous |

**AVB 2.0** adds rollback protection (blocks downgrade to vulnerable versions) and cryptographic metadata per partition via the `vbmeta` structure.

---

### Step 6 — OWASP MASVS & MASTG

References:
- [MASVS](https://mas.owasp.org/MASVS/) — defines *what* to verify
- [MASTG](https://mas.owasp.org/MASTG/) — defines *how* to test it

**Key requirements:**

| ID | Requirement |
|---|---|
| **STORAGE-1** | Sensitive data (tokens, passwords, API keys) must be stored using proper encryption — never in plaintext |
| **NETWORK-1** | Network communications must use TLS with valid certificate verification |

**Test ideas (with root privileges):**

1. **Insecure storage check** — inspect `/data/data/<package>/shared_prefs/` for plaintext sensitive values
2. **Log leakage check** — run `adb logcat` during app execution and search for tokens, passwords, or PII

---

### Step 7 — Environment Reset (mandatory)

```bash
# Via Android Studio
# Device Manager → Wipe Data

# Via command line
adb emu avd wipe-data

# Via fastboot (lab device only)
fastboot erase userdata
```

**Evidence required:** screenshot of the Android setup wizard or fresh home screen post-wipe.

---

## Key Concepts

<details>
<summary><strong>What is rooting?</strong></summary>

Root = UID 0 (superuser) privileges on the device — the equivalent of a master key for every door in the system. It bypasses sandboxing, dm-verity, SELinux enforcement, and other OS-level protections. Useful in a controlled lab to observe low-level artifacts, but requires strict isolation, full traceability, and mandatory environment reset.

</details>

<details>
<summary><strong>What is dm-verity?</strong></summary>

dm-verity is a Linux kernel feature that verifies filesystem integrity on every read. Disabling it (`adb disable-verity`) allows modification of normally read-only system partitions — equivalent to removing a tamper-evident seal.

</details>

<details>
<summary><strong>What is Magisk?</strong></summary>

Magisk performs *systemless* rooting by modifying only the boot image without touching the system partition. This approach can bypass some integrity detection mechanisms.

</details>

---

## Risk Matrix

| # | Risk | Impact |
|---|---|---|
| 1 | Integrity not guaranteed | Biased conclusions about actual app security |
| 2 | Expanded attack surface | Exposure if device leaves the lab |
| 3 | Sensitive data exposed | Privacy violation if real data is present |
| 4 | System instability | Non-reproducible tests and inconsistent results |
| 5 | Mixed personal/test accounts | Potential personal data leak |
| 6 | Poor end-of-session cleanup | Sensitive data persists on the environment |
| 7 | Non-isolated network | Unintended effects on external systems |
| 8 | Insufficient traceability | Tests cannot be reproduced or audited |

---

## Defensive Measures

| # | Measure | Purpose |
|---|---|---|
| 1 | Isolated network | Block all uncontrolled external communication |
| 2 | Fictitious data only | Eliminate real data leak risk entirely |
| 3 | Dedicated AVD / device | Prevent cross-contamination with personal use |
| 4 | Snapshot or wipe after session | Leave no persistent trace |
| 5 | Detailed configuration log | Ensure test reproducibility |
| 6 | No personal accounts | Prevent data mixing |
| 7 | Strict APK whitelist | Minimize attack surface |
| 8 | Timestamped screenshots | Full traceability |

---

## Security Standards

| Standard | Description | Link |
|---|---|---|
| **OWASP MASVS** | Mobile Application Security Verification Standard | [mas.owasp.org/MASVS](https://mas.owasp.org/MASVS/) |
| **OWASP MASTG** | Mobile Application Security Testing Guide | [mas.owasp.org/MASTG](https://mas.owasp.org/MASTG/) |
| **Android Security** | Official Android security documentation | [source.android.com/docs/security](https://source.android.com/docs/security) |
| **Verified Boot** | Boot integrity documentation | [source.android.com/docs/security/features/verifiedboot](https://source.android.com/docs/security/features/verifiedboot) |

---

## Command Reference

```bash
# ── Connection ─────────────────────────────────────────────────────
adb devices

# ── Root activation ────────────────────────────────────────────────
emulator -avd YOUR_AVD -writable-system
adb root
adb remount

# ── Verification ───────────────────────────────────────────────────
adb shell id
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop ro.boot.veritymode
adb shell getprop ro.boot.vbmeta.device_state
adb shell "su -c id"

# ── Permissive option ──────────────────────────────────────────────
adb disable-verity
adb reboot
adb remount

# ── Logging ────────────────────────────────────────────────────────
adb logcat -d | tail -n 200 > logcat_root_check.txt

# ── Fastboot (lab device only) ─────────────────────────────────────
fastboot oem device-info
fastboot getvar avb_boot_state
fastboot boot magisk_patched.img   # temporary boot only — no flash

# ── Reset ──────────────────────────────────────────────────────────
adb emu avd wipe-data
# or: fastboot erase userdata
```

> **Troubleshooting:** If `adb root` returns `adbd cannot run as root in production builds`, this is expected on production images. Use a Google APIs emulator image or a dedicated lab device.

---

## Deliverables

- [ ] Rooting definition (4 sentences)
- [ ] Verified Boot / AVB diagram (chain of trust)
- [ ] 8 risks + 8 defensive measures
- [ ] MASVS: 2 summarized requirements
- [ ] MASTG: 2 test ideas
- [ ] Completed environment sheet
- [ ] Signed reset checklist + evidence (screenshots)

---

## Disclaimer

> ⚠️ **This lab is strictly academic.**  
> All manipulations are performed on a dedicated AVD or authorized lab device only.  
> **Never root, flash, or manipulate a personal device.**  
> Unlocking a bootloader or flashing system images can permanently damage a device ("brick") and voids manufacturer warranties.  
> In some jurisdictions, rooting may violate device terms of service or applicable law.  
> Always obtain explicit written authorization before conducting security tests.

---

<div align="center">
  <sub>ENSA Marrakech · GCDSTE · Mobile Application Security</sub>
</div>
