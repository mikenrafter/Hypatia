# Executive Summary

> **Audit Date:** 2026-02-21
> **Repository:** `mikenrafter/Hypatia` (fork of `Divested-Mobile/Hypatia` via `MaintainTeam/Hypatia`)
> **Version Audited:** 3.18
> **License:** AGPL-3.0-or-later

[← Back to Table of Contents](../../SECURITY_AUDIT.md)

---

## Overall Assessment

**Hypatia is a legitimate, open-source malware scanner for Android.** After thorough analysis of every source file, build script, manifest, and CI configuration in this repository, **no malicious behavior was identified**. The application does exactly what it claims: it downloads malware signature databases and scans files on the device locally.

### Key Findings

| Category | Risk Level | Summary |
|---|---|---|
| [Network Activity](./02-network-activity.md) | 🟢 Low | Downloads only signature databases from known, documented sources. No data exfiltration. |
| [Disk & Storage Activity](./03-disk-activity.md) | 🟡 Medium | Broad file-system read access for scanning. Can delete files with user confirmation. |
| [Memory & Process Activity](./04-memory-process-activity.md) | 🟢 Low | Standard Android threading. Wake locks for scan/update operations. |
| [Permissions](./05-permissions-analysis.md) | 🟡 Medium | Requests broad permissions (all files, all packages, accessibility), all justified for stated purpose. |
| [Database Downloads](./06-database-downloads.md) | 🟢 Low | GPG-verified signature databases from Codeberg/GitHub Pages. |
| [Accessibility Service](./07-accessibility-service.md) | 🟡 Medium | Used for link scanning. Can read all screen content. User must explicitly enable. |
| [Build Infrastructure](./08-build-scripts.md) | 🟢 Low | Standard Gradle Android build. Server-side scripts for database generation only. |
| [Potential Concerns](./09-potential-concerns.md) | 🟡 Medium | Some areas could benefit from hardening. See detailed recommendations. |

### What This App Is NOT

- ❌ **Not spyware** — No data leaves the device. All scanning is local.
- ❌ **Not a downloader/dropper** — Downloaded databases are GPG-verified bloom filters, not executable code.
- ❌ **Not ransomware** — File deletion requires explicit user confirmation via dialog.
- ❌ **Not a keylogger** — The accessibility service only reads text to check for malicious domains.
- ❌ **Not a cryptominer** — CPU usage is bounded by a capped thread pool.

### What This App IS

- ✅ An open-source Android malware scanner powered by ClamAV-style signature databases
- ✅ Uses bloom filters for O(k) hash lookups against known malware signatures
- ✅ Computes MD5, SHA-1, and SHA-256 hashes of files in a single pass
- ✅ Provides real-time file monitoring via Android's `FileObserver`
- ✅ Optionally routes database downloads through Tor/Orbot for privacy
- ✅ GPG-verifies all downloaded databases before loading them

---

[Next: Network Activity →](./02-network-activity.md)
