# Windows Client — `Internet_Monitoring_Install.exe` (Pay Computer Monitoring)

Static analysis of the Windows installer for the "Internet Monitoring" /
**Pay Computer Monitoring** desktop client — the Windows counterpart to the
Android SafePhone/Vantage monitoring apps documented elsewhere in this repo.
This artifact also provides the **organizational attribution** for the "NCPTC"
brand used across the whole family.

> Analysis is 100% static — the binary was not executed. PE structure,
> imports, version info and the Authenticode certificate chain were parsed with
> `pefile` / `asn1crypto`; strings/endpoints via `strings` + `grep`.

## File identity

| Field | Value |
|-------|-------|
| Original filename | `Internet_Monitoring_Install.EXE` |
| Product / description | "Internet Monitoring" — **Pay Computer Monitoring** |
| File / product version | 1.0.0.1 |
| Type | PE32 executable (x86 / i386 GUI), 8 sections, ~4.8 MB |
| Build timestamp | **2021-06-16** |
| SHA-256 | `2f0c5b52c5c105962604c35a60cc28bf837b4b3642148d38ded5d16ddf26a3d8` |
| MD5 | `df572a4f8ce3947dd803320e5c6e89e7` |
| Code signed | **Yes — EV code-signing (GlobalSign)** |

## Organizational attribution — who "NCPTC" is

The Authenticode **Extended-Validation code-signing certificate** (chained to
GlobalSign EV CodeSigning CA – SHA256 – G3) is held by:

> **NATIONAL CYBER PROTECTION & TRAINING CORPORATION**
> 1000 Valley Forge Rd #111, Southeastern, Pennsylvania, US
> Business Category: Private Organization · Incorporation: Pennsylvania, US ·
> Reg. serial 3061900
> Certificate contact e-mail: **jmetzner@ippctech.net**

So **NCPTC = National Cyber Protection & Training Corporation** (Pennsylvania).
The `ippctech.net` domain ties directly to the `ippc-` naming seen throughout
the Android backend infrastructure (`ippc-server.vantagemdm.com`,
`ippc-mdm.vantagemdm.com:8443`), the download bucket (`ippc-bucket-downloads`),
and the database schema (`ippc.dbo.*`) referenced by this installer — confirming
a single organization behind the Windows and Android products.

## Imports (capability surface)

`WS2_32.dll` (sockets), `urlmon.dll` (`URLDownloadToFileA`), `ADVAPI32.dll`
(`RegSetValueExA`, registry/service APIs), `SHELL32.dll` (`ShellExecuteA/Ex`),
`SHLWAPI`, `VERSION`, `gdiplus`, `ole32`/`OLEAUT32`/`oledlg`/`OLEACC`
(OLE/MFC UI), `USER32` (incl. `SetWindowsHookExA` / `CallNextHookEx` /
`UnhookWindowsHookEx`), `WINMM`, `WINSPOOL`. `GetAsyncKeyState` is also
referenced.

## Behavior (downloader / bootstrap)

This EXE is a **bootstrap stub**, not the full product — it fetches and launches
the real payload:

1. **Second-stage download** — retrieves
   `https://ippc-bucket-downloads.s3.amazonaws.com/setup-12422-15.exe` (AWS S3)
   via `URLDownloadToFileA`, writes it to `…\Temp\setup.exe`, and runs it with
   `ShellExecuteA`.
2. **Encrypted config** — pulls
   `http://www.paycomputermonitoring.com/install/stp.enc` over **cleartext HTTP**.
3. **Case-keyed provisioning** — calls
   `…/IC/GetSetting.aspx?type=generic&sproc=ippc.dbo.sp_getOrganizationNameByCase&string=`
   and `…sp_getOrganizationDescByCase…`, i.e. enrollment is tied to a **case**
   (a supervision/deployment case), resolved against the `ippc` database.
4. **Antivirus interference** — detects McAfee
   (`McAfee\Common Framework\mctray.exe`) and prompts the user to **uninstall
   their antivirus** before continuing.
5. **Components** — references component executables `WMPROC.exe` and
   `rnappp7.exe`; uses `RegSetValueExA` for registry persistence and
   `GetAsyncKeyState` (key-state polling).
6. **OS gate** — requires Windows 7 or greater.

## Deployment context — "contact your officer"

User-facing strings indicate a **supervised-monitoring** deployment rather than
a consumer parental-control product:

- *"This version of the monitoring software is already installed. If you need to
  uninstall, please contact your officer."*
- *"Please contact your officer, or consult a support technician at
  http://paycomputermonitoring.com/contactus.html or 855-855-6278 about how to
  proceed."*
- Installation-error strings direct users to `support@paycomputermonitoring.com`
  with codes `INS0111` / `INS0112` / `INS0117`.

The repeated reference to "your officer" plus case-based provisioning is
consistent with court-, agency-, or employer-supervised computer monitoring —
the desktop analogue to the Android SafePhone/SpMonitor line.

## Indicators

**Hosts / URLs**
- `https://ippc-bucket-downloads.s3.amazonaws.com/setup-12422-15.exe` (2nd stage)
- `http://www.paycomputermonitoring.com/install/stp.enc` (encrypted config, cleartext)
- `http://paycomputermonitoring.com/contactus.html`
- `…/IC/GetSetting.aspx?...sproc=ippc.dbo.sp_getOrganizationNameByCase&string=`

**Dropped / referenced files:** `…\Temp\setup.exe`, `WMPROC.exe`, `rnappp7.exe`

**Signer:** NATIONAL CYBER PROTECTION & TRAINING CORPORATION (GlobalSign EV
CodeSigning CA – SHA256 – G3)

**Support:** `support@paycomputermonitoring.com`, 855-855-6278

## Relationship to the rest of the family

| Layer | Artifact |
|-------|----------|
| Signing org | **National Cyber Protection & Training Corporation (NCPTC)** |
| Internal platform / infra | **IPPC** (`ippc-*` hosts, S3 bucket, `ippc.dbo` schema, `ippctech.net`) |
| Windows brand | Pay Computer Monitoring — "Internet Monitoring" |
| Android brands | SafePhone (`spmonitor`), Vantage MDM; shared SecureTeen lineage |
| Common pattern | Minimal installer/updater pulls the real payload + encrypted config from vendor infra |
