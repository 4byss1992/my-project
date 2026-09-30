# Full Technical Breakdown — NCPTC "SafePhone" & "Vantage MDM"

Deep static analysis of two Android device-management / monitoring apps
rebranded for "NCPTC." Decompiled with **jadx 1.5.0**; manifest, certificates,
components and policy XML via **androguard 4.1.4**. Findings below are drawn
from the decompiled Java (class/method level), not just manifest metadata.

> Scope note: this is a defensive/transparency analysis of what the apps can
> do and where they send data. Package/class names are as recovered from the
> shipped binaries (identifier names are obfuscated to `b.a.a.a.*`; logic and
> strings are intact).

---

## 0. TL;DR

| | App A — SafePhone | App B — Vantage MDM |
|---|---|---|
| Package | `com.ncptc.safephone.mdm` | `com.infoweise.vantage.vantagemdm.ncptc` |
| Label / version | `spmonitor` / DROID-8.8.500.23 (523) | `NCPTC` / DROID-V-1.1.109 (109) |
| Nature | Covert monitoring agent (SecureTeen-derived) | Full enterprise MDM (VantageMDM) |
| Headline capabilities | Stealth SMS capture, call recording, Gmail reading, browser-URL logging, location, screenshots, VPN web-filtering, remote SMS commands | Device-owner control, remote screen streaming, remote lock/wipe, silent install, kiosk, Knox, VPN web-filtering |
| Backend | `dashboard.vt-pathway.net` | `ippc-server.vantagemdm.com`, `ippc-mdm.vantagemdm.com:8443` |
| Shared traits | Komodia traffic redirection, weakened TLS, cleartext allowed, aggressive self-persistence, target SDK 28 | same |

Both apps are built on a **common SecureTeen/VantageMDM code lineage** and even
interoperate: SafePhone can read its server URL from Vantage's content provider
(`content://com.infoweise.vantage.vantagemdm.CastingContentProvider/serverUrl`).

---

## 1. Identity, signing & build posture

| Field | SafePhone | Vantage |
|---|---|---|
| Signer (self-signed) | CN=`ncptc`, O=`spark` | CN=`vantagemdm`, O=`sparkeye` |
| Cert serial | 551624075 | 1765014245 |
| Validity | 2022-12-07 → **2072** (50 yr) | 2022-12-07 → **2047** (25 yr) |
| SHA-1 | `4ED6B44DBE86500B9499E1440788E4936F8311AC` | `06390327A53D6C552219E0A51BF867739AE4BD02` |
| minSDK / targetSDK | 21 / **28** | 16 / **28** |

Both self-signed with decade-plus certificates and pinned to **API 28
(Android 9)** — deliberately below Android 10, where Google restricted
background access to SMS, call logs, `content://` providers, accessibility, and
overlay windows. Staying on target 28 preserves the broad background reach the
monitoring features depend on.

**Lineage:** hardcoded URLs across both apps reference `secureteen.com`,
`vt-pathway.net`, `paycomputermonitoring.com` and `vantagemdm.com` — i.e. the
commercial **SecureTeen** parental-monitoring platform and its **VantageMDM**
enterprise sibling, rebadged as "NCPTC."

---

## 2. App A — SafePhone (`com.ncptc.safephone.mdm`)

**Surface:** 41 activities, 31 services, 9 receivers, 1 provider, 34 permissions.

### 2.1 Data capture

**SMS interception — `receiver/StealthSMSReceiver`.** Registered on
`SMS_RECEIVED`. Reconstructs every incoming message from the PDUs, reads
`getDisplayOriginatingAddress()` + `getDisplayMessageBody()`, logs them, and
checks whether the message is a **remote command** (`b.a.a.a.g.b.a(...)`). If it
is, it calls `abortBroadcast()` so the control SMS never reaches the inbox
(covert command channel) and schedules `EmailSMSCmdProcessor`. Message bodies
are queued for upload (`---dellOldStoredSMS---` / `---dellOldStoredMMS---`
markers in the sync payload).

**Call logging + recording — `receiver/StealthOutgoingCallReceiver` +
`service/EmailCallRecordingService` + `SecureIncomingCallRegService`.** Outgoing
numbers are captured on `NEW_OUTGOING_CALL` and persisted; call recording is
driven by a strategy/format matrix pushed from the server
(`STRATEGY_A..D` × `MODE_1..3`) and started via `MediaRecorder`.

**Gmail reading — `gmail/GmailReaderService`.** Spawns a worker
(`b.a.a.a.k.b`) that reads Gmail content on the device.

**On-screen capture / browsing — `service/MyAccessibilityService` (1,208
lines).** Configured with `accessibilityEventTypes=0xFFFFFFFF` and
`canRetrieveWindowContent=true`. It:
- reads the **URL bar** of Chrome (`com.android.chrome:id/url_bar`), Samsung
  Internet, Opera Mini, UC Browser, CM Browser and Firefox → browsing history;
- **redirects/blocks URLs** in those browsers (`redirectURL()`), enforcing the
  server's category/URL filter lists;
- watches `com.android.settings` and `com.google.android.permissioncontroller`
  screens — used to detect and interfere with attempts to change permissions or
  remove the app (anti-tamper);
- targets Gmail, YouTube, Facebook Messenger (`com.facebook.orca`) and Google
  search for content control.

**Location — `location/LocationReportingService`, `PcmLocationService`,
`EmailGPSService`.** Continuous location (fine/coarse/**background**) reported
to the server and optionally emailed.

**Screenshots / screen — `capture/CaptureTransparentScreen`.** Screen capture
with a server-controlled `SCREENSHOT_IGNORE_LIST`; images uploaded as `.png`.

**App inventory / installs / USB — `app/detail/ApplicationsListProvider`,
`receiver/AppInstalledObserver`, `receiver/USBDetectReceiver`.**

### 2.2 Network traffic filtering (VPN + Komodia)

`dnsvpn/VpnImpl` + `netutil/ServiceSinkhole` implement a local `VpnService`
("sinkhole") used to filter/redirect device traffic. The filter integrates
**Komodia** redirectors — `b.a.a.a.x.f` hardcodes:
```
http://thor.komodia.com/url.php?version=w21&guid=
http://optimus.komodia.com/url.php?version=w21&guid=
http://rodimus.komodia.com/url.php?version=w21&guid=
```
Komodia is the traffic-redirection/SSL-interception SDK known from the
Superfish/Lenovo incident. Web filtering is applied via `SET_VPN` and the
category/URL lists.

### 2.3 Command & exfil channel

- **Server config — `b.a.a.a.x.f.i()`:** prod `https://dashboard.vt-pathway.net`
  (block page `https://mobile.vt-pathway.net/block/block-test.html`); a QA build
  flag swaps in `qa-dashboard.vt-pathway.net`. The base URL can also be pulled
  from the Vantage app's content provider or a device-stored override.
- **Command pull:** `/secure/cmd/cb?mappingId=<id>` returns `jobs` / `jobs_new`.
  Observed job types include `PARENTAL_CATEGORIES`, `PARENTAL_URLS`,
  `PARENTAL_SAMSUNG_BROWSER_URLS`, `SAFE_SEARCH`, `YOUTUBE_SAFE_SEARCH`,
  `SCREENSHOT_IGNORE_LIST`, `SET_VPN`, `TURN_OFF_DATA`.
- **Push:** `/secure/psh/notification.do` (FCM/C2DM wake).
- **Upload:** `service/DataChangeService` queues records
  (`ArrayBlockingQueue`, cap 50) uploaded as `<type>_ANDROID_1.0.json`;
  `LogFileUploadService` ships logs. HTTP via `b.a.a.a.l.b` (`HttpURLConnection`,
  POST).
- **TLS:** `b.a.a.a.l.b` installs a **custom `X509TrustManager`** whose
  `checkServerTrusted` catches `CertificateException` from the validity check
  and `return`s (swallowing expiry/validity failures) instead of throwing;
  paired with `cleartextTrafficPermitted="true"`. Transport security is
  deliberately weakened, consistent with the Komodia MITM design.

### 2.4 Persistence / anti-removal

- `WatchDogService`, `TaskProtector`, `SecureObserverService`,
  `SecureSecondService` — mutually restart each other and the accessibility
  service if killed.
- `PhoneBootCompletedReceiver` (`RECEIVE_BOOT_COMPLETED`) restarts everything on
  boot; `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` + `WAKE_LOCK` keep it alive.
- Device-admin (`admin/DeviceAdminRightshandler`, policies `force-lock`,
  `watch-login`, `reset-password`, `expire-password`) plus the accessibility
  watch on Settings/PermissionController resist uninstall.
- Kiosk mode (`kiosk/KioskLauncherScreen`, `FakeHome`) can lock the device to a
  controlled launcher.

---

## 3. App B — Vantage MDM (`com.infoweise.vantage.vantagemdm.ncptc`)

**Surface:** 77 activities, 34 services, 13 receivers, 4 providers, 52
permissions (incl. the full Samsung **Knox** suite).

### 3.1 Device control (Device Owner / Admin)

`enterprise/receiver/AdminDeviceOwner` is a `DeviceAdminReceiver` used for
**Device Owner** provisioning (`enterprise/provision/SetupManagementActivity`,
`EULAActivity`, `ProvisioningSuccessActivity`). Through `DevicePolicyManager` it
can, among other things:
- `setGlobalSetting("adb_enabled","1")` — toggle ADB;
- `retrieveSecurityLogs(...)` — pull enterprise security logs;
- enforce the full device-admin policy set declared in
  `enterprise_device_admin.xml`: **`wipe-data`** (remote wipe),
  **`disable-camera`**, `force-lock`, `reset-password`, plus dozens of `mdm-*`
  controls (firewall, VPN, phone-restriction, location, email, hardware-control,
  APN, browser-settings, tethering, date-time).

Privileged manifest permissions back this up: `WRITE_SECURE_SETTINGS`,
`READ_PRIVILEGED_PHONE_STATE`, `INSTALL_PACKAGES` +
`DOWNLOAD_WITHOUT_NOTIFICATION` (silent app install), and the Knox set
(`KNOX_ENTERPRISE_DEVICE_ADMIN`, `KNOX_KIOSK_MODE`, `KNOX_FIREWALL`,
`KNOX_VPN`, `KNOX_HW_CONTROL`, `KNOX_REMOTE_CONTROL`, `KNOX_APP_MGMT`, …).

### 3.2 Remote screen streaming

`screensharetobrowser/recorder/RecordService` uses **`MediaProjection` +
`VirtualDisplay` + `ImageReader`** (`createVirtualDisplay("MainScreen", …)`) to
capture the live screen. `screensharetobrowser/server/ServerService` runs a
socket-based HTTP server that serves those frames, and
`screensharetobrowser/socket/MainActivity` drives the session — i.e. a remote
operator can **watch the device screen live in a browser**. Additional
`ScreenMonitoringService` and `AppsUsageService` track screen/app usage.

### 3.3 Content filtering (VPN + Komodia)

`vpn/VpnConnectionTransparentScreen` + `dnsvpn/VpnImpl` provide the same local
VPN web-filter, again with the **Komodia** redirectors. Server-driven category
filtering is extensive — job/category constants include `ADULT_CONTENT`,
`PORNOGRAPHY`, `GAMBLING`, `DATING`, `VIOLENCE`, `SUICIDE`, `KEYWORD`,
`WHITELIST`/`BLACKLIST`/`BYPASS`/`CATEGORY_FILTERING`, `WEBCLIP_URL`.

### 3.4 Command & exfil channel

- **Servers — `util/MpcUtil`:** `https://ippc-server.vantagemdm.com`,
  `https://ippc-mdm.vantagemdm.com:8443/mdm`, `https://ns-cp.vantagemdm.com`,
  `cp.vantagemdm.com`; shares SecureTeen infra (`cp.secureteen.com`,
  `support.secureteen.com`, `secureteen.com/login.php`).
- **API endpoints:** `/secure/device/subscribe`, `/secure/map/device/new`,
  `/secure/mdm/validate/user`, `/secure/validate/usr/pwd`,
  `/secure/subuser/create`, `/secure/location/set`, `/secure/upload/file`,
  `upload_file.php`, plus the `cmd/cb?mappingId=` command pull.
- **Commands:** `LOCK_DEVICE`, `LOCK_WITH_EXISITING_PASSWORD` [sic],
  `REMOVE_ADMIN_RIGHTS`, `START_FUNCTIONING` / `STOP_FUNCTIONING`,
  `PCM_SELF_REMOVED`, `KOMODIA`, `MDMINTERFACE`, `WEBCLIP_URL`.
- **Identifiers collected:** `ICCID`, `SERIAL`, `SIMMCC`, `SIMMNC` (SIM/hardware
  identifiers).

### 3.5 Leftover / test infrastructure shipped in the build

Hardcoded in the shipped APK:
- `http://65.19.151.196:8080/secure/settings/get.do?subUserId=514713` — a raw
  IP over **cleartext** with a concrete `subUserId`;
- `http://10.200.200.221:82/web_clips.php?mappingId=` — an internal **LAN**
  address;
- `https://ns-cp.vantagemdm.com/web_clips.php?mappingId=74fb839a-da1b-4af8-9aca-d9bde0473c19`
  — a concrete mapping GUID.

`network_security_config.xml` additionally **trusts user-installed CAs**
(`<certificates src="user"/>`) on top of cleartext — the weakest of the two
transport postures, and exactly what a MITM/interception design needs.

### 3.6 Persistence

`AutoSyncWatchDogService`, `WatchDogService`, `TaskProtector`,
`SecureObserverService`, `SecureServiceLauncher`, `CacheMaintainerService` +
`PhoneBootCompletedReceiver` + battery-optimization exemption; kiosk stack
(`KioskModeActivity`, `NITBKioskLauncherScreen`, `KioskSecureSettingScreen`).

---

## 4. Side-by-side capability matrix

| Capability | SafePhone | Vantage |
|---|:--:|:--:|
| Read/intercept SMS (with covert command channel) | ✅ | — |
| Call logging | ✅ | — |
| Call **recording** | ✅ | — |
| Gmail content reading | ✅ | — |
| Browser URL logging (6 browsers) | ✅ | partial |
| Accessibility screen-content capture | ✅ | ✅ |
| Location tracking (background) | ✅ | ✅ |
| Screenshots | ✅ | — |
| **Live remote screen streaming** | — | ✅ |
| App-usage monitoring | ✅ | ✅ |
| Local VPN web filtering + Komodia | ✅ | ✅ |
| Remote **lock** | ✅ (device-admin) | ✅ |
| Remote **wipe** | — | ✅ (`wipe-data`) |
| **Disable camera** | — | ✅ |
| **Silent app install** | — | ✅ |
| Toggle ADB / pull security logs | — | ✅ (device owner) |
| Samsung Knox controls | — | ✅ |
| Kiosk lockdown | ✅ | ✅ |
| Self-persistence / anti-removal | ✅ | ✅ |
| Weakened TLS / cleartext allowed | ✅ | ✅ (also trusts user CAs) |

---

## 5. Security & privacy assessment

- **Surveillance breadth (SafePhone):** messages, calls (incl. recordings),
  email, location, browsing, screenshots and on-screen text — with an explicitly
  **covert** design ("Stealth" components, SMS-command hiding via
  `abortBroadcast`). On a device the user doesn't control, this is
  full-spectrum spyware behavior.
- **Control breadth (Vantage):** device-owner power to wipe, lock, disable
  camera, silently install apps, toggle ADB, and **watch the screen live** — a
  complete remote-administration channel.
- **Weakened transport security (both):** cleartext permitted; SafePhone's
  trust manager swallows certificate-validity errors; Vantage trusts
  user-installed CAs. Combined with Komodia, traffic is designed to be
  interceptable/redirectable.
- **Hygiene red flags (Vantage):** production build ships hardcoded raw-IP
  cleartext endpoints, an internal LAN address, a concrete `subUserId` and a
  mapping GUID — leftover test/staging wiring.
- **Persistence:** both are built to survive being stopped or uninstalled
  (watchdog webs, boot restart, battery exemption, accessibility watch on
  Settings/PermissionController, device-admin/kiosk).

Both belong to legitimate software *categories* (parental control / enterprise
MDM) and these capabilities are the point of such products **when deployed with
consent on a device the deployer owns or administers**. The concern is the
combination of covert operation, weakened transport security, shipped test
infrastructure, and anti-removal persistence — on any device where the person
using it did not knowingly consent, either app represents a serious privacy and
security exposure.

---

## 6. Indicators (endpoints & signing)

**SafePhone hosts:** `dashboard.vt-pathway.net`, `mobile.vt-pathway.net`,
`qa-dashboard.vt-pathway.net`, `qa-mobile.vt-pathway.net`,
`mobile.paycomputermonitoring.com`, `www.secureteen.com`,
`{thor,optimus,rodimus}.komodia.com`.

**Vantage hosts:** `ippc-server.vantagemdm.com`,
`ippc-mdm.vantagemdm.com:8443`, `ns-cp.vantagemdm.com`, `cp.vantagemdm.com`,
`server.vantagemdm.com`, `cp.secureteen.com`, `support.secureteen.com`,
`{thor,optimus,rodimus}.komodia.com`, `65.19.151.196:8080`, `10.200.200.221:82`.

**Signing SHA-1:** SafePhone `4ED6B44DBE86500B9499E1440788E4936F8311AC`;
Vantage `06390327A53D6C552219E0A51BF867739AE4BD02`.
