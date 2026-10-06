<div align="center">

# 📱 Mobile MDM/Monitoring APK Analysis

**Static reverse-engineering analysis of two Android device-management / monitoring applications rebranded for "NCPTC."**

![Type](https://img.shields.io/badge/type-security_research-blue)
![Platform](https://img.shields.io/badge/platform-Android-3DDC84?logo=android&logoColor=white)
![Analysis](https://img.shields.io/badge/analysis-static-orange)
![Tooling](https://img.shields.io/badge/tooling-jadx%20%7C%20androguard-purple)
![License](https://img.shields.io/badge/license-MIT-green)

</div>

---

## Overview

This repository documents a defensive, transparency-focused static analysis of
two Android applications. Both are commercial **device-management / monitoring**
apps (built on the SecureTeen / VantageMDM code lineage) redistributed under the
"NCPTC" brand. The goal is to document, at the class and method level, **what
each app is capable of** and **where it sends data** — the kind of write-up a
security or privacy reviewer would produce before allowing such software onto a
managed fleet.

| App | Package | Version | Nature |
|-----|---------|---------|--------|
| **SafePhone** (`spmonitor`) | `com.ncptc.safephone.mdm` | DROID-8.8.500.23 | Covert monitoring agent |
| **Vantage MDM** (`NCPTC`) | `com.infoweise.vantage.vantagemdm.ncptc` | DROID-V-1.1.109 | Full enterprise MDM |

---

## Reports

| Document | Description |
|----------|-------------|
| 📄 [**Executive Summary**](reports/executive-summary.md) | High-level breakdown — identity, permissions, components, endpoints, and bottom-line assessment. Start here. |
| 📚 [**Full Technical Breakdown**](reports/full-breakdown.md) | Deep, class-level analysis from decompiled source: data-capture services, VPN/Komodia filtering, command & exfil channels, TLS handling, persistence, and a full capability matrix. |
| 🔀 [**Vantage Version Diff**](reports/vantage-version-diff.md) | What changed between Vantage MDM `v1.1.109` and the newer `v1.1.151` — new provisioning/compliance components, permission and endpoint changes, and what stayed the same. |
| 🪟 [**Windows Installer**](reports/windows-installer.md) | Analysis of the `Internet_Monitoring_Install.exe` Windows client — a signed downloader stub whose EV certificate attributes the whole family to **National Cyber Protection & Training Corporation (NCPTC)**. |
| 🔬 [**Methodology**](reports/methodology.md) | Tools, versions, and the step-by-step static-analysis process used to produce the reports. |

---

## Key findings at a glance

- **SafePhone** — stealth SMS interception (with a hidden SMS command channel),
  call logging and recording, Gmail reading, browser-URL logging across six
  browsers, background location, screenshots, and local-VPN web filtering.
- **Vantage MDM** — Device-Owner control (remote **wipe**, **lock**,
  **disable camera**, silent app install, ADB toggle, security-log retrieval),
  **live screen streaming to a browser**, kiosk lockdown, and Samsung Knox
  integration.
- **Shared traits** — [Komodia](https://en.wikipedia.org/wiki/Komodia) traffic
  redirection, weakened TLS with cleartext permitted, aggressive
  self-persistence / anti-removal, and a deliberately old `targetSdk 28`.
- The two apps share a codebase and **interoperate** (SafePhone can read its
  server URL from Vantage's content provider).

Full details, endpoints, and signing indicators are in the
[reports](reports/).

---

## Methodology

| Stage | Tooling |
|-------|---------|
| Manifest, permissions, certificates, components, policy XML | [androguard](https://github.com/androguard/androguard) 4.1.4 |
| Decompilation to Java | [jadx](https://github.com/skylot/jadx) 1.5.0 |
| String / endpoint extraction | `strings`, `grep` over `classes.dex` and decompiled sources |

Analysis is **100% static** — no APK was installed or executed. Findings are
drawn from the shipped binaries; identifier names are obfuscated (`b.a.a.a.*`)
but program logic and string constants are intact.

---

## Repository structure

```
.
├── README.md                      # this file
├── LICENSE
└── reports/
    ├── executive-summary.md       # high-level breakdown
    ├── full-breakdown.md          # deep class-level analysis
    ├── vantage-version-diff.md    # v1.1.109 → v1.1.151 changes
    ├── windows-installer.md       # Windows "Internet Monitoring" client
    └── methodology.md             # tooling and process
```

---

## ⚠️ Disclaimer

This material is provided for **defensive security research, privacy review,
and educational purposes only**. It describes the capabilities and network
behavior of software already distributed to end users; it contains no exploit
code and no instructions for building or deploying surveillance tooling.

App and vendor names appear only to identify the artifacts analyzed. No
affiliation with, or endorsement by, any named vendor is implied. Analyze only
software you are authorized to review.

---

## License

Released under the [MIT License](LICENSE).
