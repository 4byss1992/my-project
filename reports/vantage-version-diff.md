# Version Diff — Vantage MDM `DROID-V-1.1.109` → `DROID-V-1.1.151`

Comparison of two builds of the Vantage MDM app
(`com.infoweise.vantage.vantagemdm.ncptc`). Both decompiled with jadx 1.5.0 and
diffed at the manifest, component, and endpoint level. v1.1.109 is the build
covered in the [full breakdown](full-breakdown.md); v1.1.151 is a newer release.

## Build metadata

| Field | v1.1.109 | v1.1.151 | Δ |
|-------|----------|----------|---|
| versionCode | 109 | **151** | ↑ |
| versionName | DROID-V-1.1.109 | DROID-V-1.1.151 | — |
| minSdk | 16 | **21** | ↑ (drops Android <5 support) |
| targetSdk | 28 | 28 | unchanged |
| Signing cert | CN=vantagemdm, serial 1765014245 | **same** | identical signer (same author key) |
| SHA-1 | `06390327…E4BD02` | `06390327…E4BD02` | identical |

The signing certificate is unchanged, confirming both builds come from the same
developer. `targetSdk` stays at **28** — the app still avoids the stricter
Android 10+ background/permission model.

## Permissions (52 → 54)

**Added in v1.1.151:**
- `android.permission.POST_NOTIFICATIONS` — Android 13 runtime notification
  permission (keeps the foreground/persistence notifications working on newer OS).
- `android.permission.QUERY_ADVANCED_PROTECTION_MODE` — queries the device's
  Advanced Protection state (Android 14+).

**Removed:** none. The full Knox suite, `WRITE_SECURE_SETTINGS`,
`READ_PRIVILEGED_PHONE_STATE`, `INSTALL_PACKAGES`,
`DOWNLOAD_WITHOUT_NOTIFICATION`, etc. all carry over unchanged.

## Components

| | v1.1.109 | v1.1.151 |
|---|---|---|
| Activities | 77 | **79** |
| Services | 34 | 34 |
| Receivers | 13 | **14** |
| Providers | 4 | 4 |

**New classes in v1.1.151:**

- `ui/ProvisioningModeActivity` — handles
  `android.app.action.GET_PROVISIONING_MODE` and chooses between
  `PROVISIONING_MODE_FULLY_MANAGED_DEVICE` and `PROVISIONING_MODE_MANAGED_PROFILE`.
- `ui/AdminPolicyComplianceActivity` — handles
  `android.app.action.ADMIN_POLICY_COMPLIANCE` (the post-enrollment compliance
  callback).
- `receiver/PackageReceiver` — a `BroadcastReceiver` for
  `MY_PACKAGE_REPLACED`, `PACKAGE_ADDED`, `PACKAGE_REMOVED`, `PACKAGE_REPLACED`.
- `model/w0` — supporting model class.

**New manifest intent actions:** `GET_PROVISIONING_MODE`,
`ADMIN_POLICY_COMPLIANCE`, `PROFILE_PROVISIONING_COMPLETE` (alongside the
existing `DEVICE_ADMIN_*` set).

### What changed, functionally

1. **Modern enterprise enrollment.** The new provisioning activities implement
   the Android 10+ **managed-provisioning** callbacks used by QR-code and
   zero-touch enrollment, and let the enroller pick **fully-managed device** vs
   **managed work profile**. v1.1.109 lacked these; v1.1.151 is built to be
   deployed through standard Android Enterprise enrollment flows.
2. **Self-update + app-inventory persistence.** `PackageReceiver` responds to
   `MY_PACKAGE_REPLACED` (re-establish itself after its own update) and to
   install/remove/replace of *other* packages (app-inventory monitoring).
3. **Newer-OS compatibility.** `POST_NOTIFICATIONS` and
   `QUERY_ADVANCED_PROTECTION_MODE` keep the agent functional on Android 13/14.

## Backend endpoints

| | v1.1.109 | v1.1.151 |
|---|---|---|
| Primary control panel | `https://ns-cp.vantagemdm.com` | **`https://ncptc-cp.vantagemdm.com`** |
| MDM server | `ippc-mdm.vantagemdm.com:8443/mdm` | unchanged |
| App server | `ippc-server.vantagemdm.com` | unchanged |
| SecureTeen infra | `cp.secureteen.com`, `support.secureteen.com` | unchanged |
| Komodia redirectors | present | **still present** |

The control-panel host moved to a **NCPTC-dedicated** subdomain
(`ncptc-cp.vantagemdm.com`), suggesting a tenant/branding split for the NCPTC
deployment.

## What did **not** change

- **Weakened TLS.** `network_security_config.xml` still sets
  `cleartextTrafficPermitted="true"` **and** trusts user-installed CAs
  (`<certificates src="user"/>`) — unchanged from v1.1.109.
- **Komodia** traffic redirectors — still embedded.
- **Leftover test artifacts — all still shipped** in v1.1.151:
  `65.19.151.196:8080`, `subUserId=514713`, `10.200.200.221:82`, and mapping GUID
  `74fb839a-da1b-4af8-9aca-d9bde0473c19`. The newer build did **not** clean these
  up.
- Core capability set (Device-Owner control, remote screen streaming, kiosk,
  Knox, VPN filtering, persistence) is carried over intact.

## Bottom line

v1.1.151 is an **incremental modernization**, not a redesign: it adds
Android-Enterprise managed-provisioning support (fully-managed vs work-profile),
a package-event receiver for self-update and app-inventory, and Android 13/14
permission compatibility, and it points the control panel at an NCPTC-dedicated
host. The security-relevant weaknesses from v1.1.109 — cleartext + user-CA trust,
Komodia redirection, and the hardcoded test artifacts — are **unchanged and
still present**.
