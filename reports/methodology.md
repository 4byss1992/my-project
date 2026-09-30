# Methodology

How the two APKs in this analysis were examined. All work is **static** — the
applications were never installed, launched, or granted any permission on a real
or emulated device.

## Environment

- Linux analysis host (headless)
- OpenJDK (for jadx)
- Python 3 virtualenv for androguard

## Tools

| Tool | Version | Role |
|------|---------|------|
| [androguard](https://github.com/androguard/androguard) | 4.1.4 | Parse `AndroidManifest.xml`, signing certificates, permissions, components, and `res/xml/*` policy files |
| [jadx](https://github.com/skylot/jadx) | 1.5.0 | Decompile `classes.dex` → readable Java |
| `unzip` | — | Extract `classes*.dex` and resources from the APK containers |
| `strings`, `grep` | — | Recover URLs, endpoints, command constants, and API paths from DEX and decompiled sources |

## Steps

1. **Triage** — confirm each file is a valid APK; record size and signing block.
2. **Manifest & metadata** — extract package, version, `min`/`targetSdk`,
   signing certificate (subject/issuer/validity/fingerprint), the full
   permission set, declared/custom permissions, and hardware features.
3. **Component inventory** — enumerate activities, services, receivers, and
   providers; identify the launcher/main activity and notable `<meta-data>`
   (device-admin and accessibility-service resource references).
4. **Policy XML** — decode the binary `res/xml/` files: device-admin policy set,
   accessibility-service configuration, and network-security configuration.
5. **Decompilation** — convert both DEX files to Java with jadx.
6. **Behavioral review** — read the key components at method level (data-capture
   services, VPN/filtering, command dispatch, upload/exfil, TLS handling,
   persistence) and corroborate with string/endpoint extraction.
7. **Reporting** — cross-check every claim against the recovered code before
   writing it up.

## Scope & limitations

- **Static only.** Dynamic behavior (runtime traffic, actual server responses,
  timing) was not observed. Capabilities described are those *present in the
  code*, which the server can enable or disable at runtime.
- **Obfuscation.** Identifier names are minified (`b.a.a.a.*`); class/method
  structure, control flow, and string constants remain intact, which is
  sufficient for capability and endpoint mapping.
- **No secrets included.** Where the binaries embedded concrete identifiers
  (e.g. a test `subUserId` or mapping GUID), they are cited as
  hygiene/indicator findings, not as credentials to be used.
