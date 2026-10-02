# APK Breakdown — NCPTC "SafePhone" & "Vantage MDM"

Static analysis of two Android device-management / monitoring apps rebranded
for "NCPTC." Both are built on established commercial surveillance/MDM
codebases. Analysis performed with androguard (manifest, certs, components,
policy XML) plus string/endpoint extraction from the compiled DEX.

> **Versions analyzed:** SafePhone `DROID-8.8.500.23` (code 523) and Vantage MDM
> `DROID-V-1.1.109` (code 109). A newer Vantage build, `DROID-V-1.1.151`
> (code 151), was also examined — see the
> [version diff](vantage-version-diff.md). The security-relevant weaknesses
> below (cleartext + user-CA trust, Komodia redirection, hardcoded test
> artifacts) are unchanged in the newer build.

## 1. Identity & signing

| | App A — SafePhone | App B — Vantage MDM |
|---|---|---|
| Package | `com.ncptc.safephone.mdm` | `com.infoweise.vantage.vantagemdm.ncptc` |
| Display label | `spmonitor` | `NCPTC` |
| Version | DROID-8.8.500.23 (code 523) | DROID-V-1.1.109 (code 109) |
| Size | ~3.9 MB | ~22 MB |
| minSDK / targetSDK | 21 / 28 | 16 / 28 |
| Signer (self-signed) | CN=`ncptc`, O=`spark` | CN=`vantagemdm`, O=`sparkeye` |
| Cert validity | 2022 → 2072 (50 yr) | 2022 → 2047 (25 yr) |

Both are self-signed with unusually long-lived certificates and target
**API 28 (Android 9)** — the last version before Google restricted background
access to SMS, call logs, accessibility, and overlay windows. That target
choice is typical of monitoring software needing the pre-Android-10 model.

Real vendors, revealed by hardcoded URLs:
- **App A (SafePhone)** — built on the **SecureTeen** parental-monitoring
  stack (`secureteen.com`, `paycomputermonitoring.com`, `vt-pathway.net`).
- **App B (Vantage)** — **VantageMDM** by "infoweise/sparkeye"
  (`vantagemdm.com`), sharing SecureTeen lineage.
- **Both** embed **Komodia** redirector endpoints
  (`thor/rodimus/optimus.komodia.com/url.php`). Komodia is the
  SSL-interception / traffic-redirection SDK known from the Superfish/Lenovo
  incident, used here to route/filter device traffic through a local VPN.

## 2. What these apps are

Neither is a normal consumer app. Both are full remote-administration +
surveillance agents: an admin/parent console pushes policies and pulls data.
App A leans toward **covert child/employee monitoring** (call recording,
stealth SMS/call capture, Gmail reading). App B is a broader **enterprise
MDM** with Samsung Knox integration, kiosk mode, and remote screen sharing.

## 3. Permissions

**App A (SafePhone) — 34 permissions.** Surveillance-heavy:
- `READ_SMS`, `RECEIVE_SMS`, `READ_CALL_LOG`, `READ_CONTACTS`, `GET_ACCOUNTS`
- `ACCESS_FINE/COARSE/BACKGROUND_LOCATION`
- `READ_HISTORY_BOOKMARKS`, `PACKAGE_USAGE_STATS`, `GET_TASKS`
- `BIND_ACCESSIBILITY_SERVICE` + `SYSTEM_ALERT_WINDOW`
- `KILL_BACKGROUND_PROCESSES`, `REORDER_TASKS`, `REQUEST_DELETE_PACKAGES`
- `RECEIVE_BOOT_COMPLETED`, `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`, `WAKE_LOCK`
- C2DM/FCM push (remote command channel)

**App B (Vantage) — 52 permissions.** Enterprise/Knox-heavy:
- Full **Samsung Knox** suite: `KNOX_ENTERPRISE_DEVICE_ADMIN`,
  `KNOX_KIOSK_MODE`, `KNOX_FIREWALL`, `KNOX_VPN`, `KNOX_HW_CONTROL`,
  `KNOX_REMOTE_CONTROL`, `KNOX_APP_MGMT`, `KNOX_PHONE_RESTRICTION`, etc.
- `INSTALL_PACKAGES` + `REQUEST_INSTALL_PACKAGES` +
  `DOWNLOAD_WITHOUT_NOTIFICATION` — silent app install
- `WRITE_SECURE_SETTINGS`, `READ_PRIVILEGED_PHONE_STATE` — privileged
  (normally OEM/system-only) settings
- Bluetooth/WiFi/network control, `SET_WALLPAPER`, launcher shortcut install
- No SMS/call-log read (unlike App A) — more "manage device" than "record person"

## 4. Components — what they do

**App A (SafePhone): 41 activities, 31 services, 9 receivers.**
- `EmailCallRecordingService`, `SecureIncomingCallRegService` — records calls
  and emails them out (matches `MediaRecorder`/`CallRecord` code refs)
- `StealthSMSReceiver`, `StealthOutgoingCallReceiver` — covert SMS/outbound-call capture
- `GmailReaderService` — reads Gmail content
- `MyAccessibilityService` (`accessibilityEventTypes=0xFFFFFFFF`,
  `canRetrieveWindowContent=true`) — captures all on-screen text across apps
- `ServiceSinkhole` + `VpnImpl` + `dnsvpn` — local VPN/DNS filter (Komodia-tied)
- `LocationReportingService`, `EmailGPSService`, `PcmLocationService` — location exfil
- `CaptureTransparentScreen` — screen capture
- `WatchDogService`, `TaskProtector`, `SecureObserverService` — anti-kill persistence
- `USBDetectReceiver`, `AppInstalledObserver` — monitor USB and new installs
- `KioskLauncherScreen`/`FakeHome` — lock device to a controlled launcher
- `MDMBridgeActivity`, `DeviceAdminRightshandler` — admin control surface

**App B (Vantage): 77 activities, 34 services, 13 receivers.**
- `enterprise.provision.*` + `AdminDeviceOwner` — Device Owner provisioning
- `screensharetobrowser.*` — `RecordService` + `ServerService` + socket
  `MainActivity`: live screen streaming to a browser
- `ScreenMonitoringService`, `AppsUsageService` — screen/app-usage monitoring
- `MyAccessibilityService`, `SecureObserverService`, `WatchDogService`,
  `TaskProtector`, `SecureServiceLauncher` — content capture + persistence
- Kiosk stack, WebClip/policy management, WiFi/Bluetooth/hotspot controllers
- `CastingContentProvider`, Picasso, Google Analytics

## 5. Control surface (device-admin + accessibility)

**App A device-admin policies:** `force-lock`, `watch-login`,
`reset-password`, `expire-password` — modest.

**App B device-admin policies:** the full Exchange/MDM policy set —
`wipe-data`, `disable-camera`, `force-lock`, `reset-password`, plus dozens of
`mdm-*` controls (firewall, VPN, phone-restriction, location, email,
hardware-control, APN, browser-settings). Can remotely wipe the device,
disable the camera, and lock nearly everything.

**Accessibility (both):** subscribe to every event type with full
window-content retrieval — the standard technique for reading anything on
screen. App A also requests `canRequestFilterKeyEvents` and enhanced web
accessibility.

## 6. Backend endpoints

**App A →** `dashboard.vt-pathway.net`, `mobile.vt-pathway.net`,
`mobile.paycomputermonitoring.com`, `secureteen.com/login`, Komodia redirectors.

**App B →** `cp.vantagemdm.com`, `ns-cp.vantagemdm.com`,
`server.vantagemdm.com`, `ippc-mdm.vantagemdm.com:8443/mdm`,
`ippc-server.vantagemdm.com`, plus hardcoded test/leftover endpoints:
`http://65.19.151.196:8080/secure/settings/get.do?subUserId=514713` and
`http://10.200.200.221:82/web_clips.php` (a raw IP over cleartext and an
internal-LAN address baked into the shipped build).

## 7. Security-relevant traits

- **Cleartext traffic allowed** in both (`cleartextTrafficPermitted="true"`).
  App B additionally **trusts user-installed CAs** — weakening TLS and
  consistent with the Komodia MITM design.
- **Komodia traffic interception** in both.
- **Aggressive persistence** (watchdog + task-protector + boot receivers +
  battery-optimization exemption) to survive attempts to stop them.
- **Covert operation** in App A (explicit "Stealth" components; call recording).
- **Hardcoded internal/test infrastructure** shipped in App B (raw IP, LAN
  address, a fixed `subUserId`).
- **Old target SDK (28)** to retain broad background access.

## 8. Bottom line

- **App A (SafePhone/spmonitor)** — a covert monitoring agent
  (SecureTeen-derived): records calls; captures SMS/calls/Gmail/location/
  on-screen text; filters web via a Komodia VPN; hides and persists itself.
- **App B (Vantage MDM)** — a full enterprise MDM (VantageMDM): device-owner
  provisioning, remote wipe/lock/camera-disable, silent app install, kiosk
  lockdown, live screen streaming, Knox integration.

Both are legitimate *categories* of software (parental control / enterprise
MDM), but both carry extensive surveillance and remote-control capability,
weak transport security, and persistence designed to resist removal. On a
device you do not own or administer, either would be a serious privacy and
security concern.
