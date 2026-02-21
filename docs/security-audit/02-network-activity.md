# Network Activity Analysis

[← Back to Table of Contents](../../SECURITY_AUDIT.md) | [← Executive Summary](./01-executive-summary.md)

---

## Overview

Hypatia's network activity is limited to a single purpose: **downloading malware signature databases**. No user data, file contents, file names, scan results, or device information is ever transmitted from the device.

## Outbound Connections

### Database Download Endpoints

All network requests originate from `Database.Downloader` ([`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt), line 51) and connect to one of three configured database sources defined in [`DatabaseSource.kt`](../../app/src/main/java/us/spotco/malwarescanner/DatabaseSource.kt):

| Source | URL | Purpose |
|---|---|---|
| MaintainTeam (Codeberg) | `https://maintainteam.codeberg.page/HypatiaDatabases/` | Primary database mirror |
| MaintainTeam (GitHub) | `https://maintainteam.github.io/HypatiaDatabases/` | Mirror database |
| AXP OS | `https://lav.axpos.org/db/` | Alternative database source |
| Custom | User-defined | User can set any URL |

### Files Downloaded

For each configured source, the app downloads:

1. `gpg.key` — The GPG public key for signature verification
2. `hypatia-md5-bloom.bin` — MD5 hash bloom filter + `.sig` GPG signature
3. `hypatia-sha1-bloom.bin` — SHA-1 hash bloom filter + `.sig` GPG signature
4. `hypatia-sha256-bloom.bin` — SHA-256 hash bloom filter + `.sig` GPG signature
5. `hypatia-md5-extended-bloom.bin` (optional) — Extended MD5 database + `.sig`
6. `hypatia-domains-bloom.bin` (optional) — Domain blocklist bloom filter + `.sig`

### Connection Properties

From `Database.Downloader.doInBackground()` ([`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt), lines 65–215):

- **Protocol:** HTTPS (via `HttpURLConnection`)
- **Connect timeout:** 90 seconds
- **Read timeout:** 30 seconds
- **User-Agent:** `"Hypatia"`
- **Conditional downloads:** Uses `If-Modified-Since` header to avoid re-downloading unchanged databases (HTTP 304)
- **No cookies, no authentication tokens, no tracking headers**

### Tor/Onion Routing Support

When the user enables onion routing ([`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt), lines 76–79):

- Traffic is routed through a local SOCKS proxy at `127.0.0.1:9050` (Orbot)
- The app waits up to 60 seconds for Orbot to become available ([`Utils.kt`](../../app/src/main/java/us/spotco/malwarescanner/Utils.kt), lines 199–208)
- Orbot availability is checked by attempting a TCP connection to `127.0.0.1:9050`

### VirusTotal Lookup (User-Initiated Only)

When a malware detection occurs, the user can optionally look up the file hash on VirusTotal. This is done by opening a browser intent to `https://www.virustotal.com/gui/file/{sha256hash}` ([`NotificationPromptActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/NotificationPromptActivity.kt), lines 59–63). **This is user-initiated and requires explicit confirmation.**

## Data Exfiltration Analysis

### What Is NOT Sent

- ❌ File contents
- ❌ File names or paths
- ❌ File hashes
- ❌ Scan results
- ❌ Device information
- ❌ User credentials
- ❌ Location data
- ❌ Contact information
- ❌ Any personally identifiable information

### Verification

A complete search of the codebase for network-related classes confirms:

- `HttpURLConnection` is only used in `Database.Downloader` for downloading databases
- `URL` class is only used in `Database.Downloader`
- No `OkHttp`, `Retrofit`, `Volley`, or other HTTP client libraries are present
- No `WebSocket`, `Socket` (except the Orbot port check), or other network APIs are used
- No `Firebase`, `Analytics`, `Crashlytics`, or telemetry SDKs are included

## Network Permission Justification

| Permission | Usage |
|---|---|
| `INTERNET` | Downloading signature databases |
| `ACCESS_NETWORK_STATE` | Checking network availability before attempting downloads |

---

[Next: Disk & Storage Activity →](./03-disk-activity.md)
