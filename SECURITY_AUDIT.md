# Hypatia Security Audit — Table of Contents

> **Audit Date:** 2026-02-21
> **Repository:** `mikenrafter/Hypatia` (fork of `Divested-Mobile/Hypatia`)
> **Version:** 3.18
> **Auditor:** Automated code review

---

## ⚠️ Important Note

**After thorough analysis of the entire codebase, Hypatia is NOT malware.** It is a legitimate, open-source (AGPL-3.0) malware scanner for Android. This audit documents all network, disk, memory, process, and permission activities in the codebase for transparency and review.

---

## Audit Sections

### 1. [Executive Summary](./docs/security-audit/01-executive-summary.md)
Overall assessment, risk matrix, and key findings.

### 2. [Network Activity](./docs/security-audit/02-network-activity.md)
All outbound network connections, download endpoints, data transmission analysis, and verification that no user data is exfiltrated.

### 3. [Disk & Storage Activity](./docs/security-audit/03-disk-activity.md)
File system read operations (scanning), write operations (database storage, self-test files), and delete operations (malware removal with user confirmation).

### 4. [Memory & Process Activity](./docs/security-audit/04-memory-process-activity.md)
Threading model, thread pool bounds, wake locks, memory usage (bloom filters), garbage collection, foreground services, and boot receiver behavior.

### 5. [Permissions Analysis](./docs/security-audit/05-permissions-analysis.md)
Complete analysis of all declared Android permissions, their risk levels, justifications, and code references. Includes exported component analysis.

### 6. [Database / Payload Downloads](./docs/security-audit/06-database-downloads.md)
Detailed breakdown of what the app downloads, the bloom filter database format, the GPG signature verification process, and server-side database generation scripts.

### 7. [Accessibility Service](./docs/security-audit/07-accessibility-service.md)
Analysis of the `LinkScannerService` — what screen content it accesses, how it processes text, rate-limiting safeguards, and the multi-step user enablement process.

### 8. [Build Infrastructure & Scripts](./docs/security-audit/08-build-scripts.md)
Android build system, dependency analysis, CI/CD workflows, server-side database generation scripts, and signing key management.

### 9. [Potential Security Concerns & Recommendations](./docs/security-audit/09-potential-concerns.md)
Areas where the codebase could be hardened, including custom server URL handling, signing key overrides, deprecated API usage, and certificate pinning.

---

## Quick Reference

| Category | Risk | Key Finding |
|---|---|---|
| Network | 🟢 Low | Only downloads signature databases; no data exfiltration |
| Disk | 🟡 Medium | Broad read access for scanning; deletes only with user confirmation |
| Memory | 🟢 Low | Bounded thread pools; time-limited wake locks |
| Permissions | 🟡 Medium | Broad but justified for malware scanner functionality |
| Payloads | 🟢 Low | GPG-verified bloom filters; not executable code |
| Accessibility | 🟡 Medium | Reads screen text for domain scanning; user must explicitly enable |
| Build | 🟢 Low | Standard Gradle with mainstream, trusted dependencies |
| Overall | 🟢 **Not Malware** | Legitimate FOSS malware scanner |

---

## Files Analyzed

### Application Source (`app/src/main/java/us/spotco/malwarescanner/`)
- `Database.kt` — Database download, verification, and loading
- `DatabaseSource.kt` — Database source URL definitions
- `EventReceiver.kt` — Boot/package broadcast receiver
- `GPGDetachedSignatureVerifier.java` — PGP signature verification
- `Hypatia.kt` — Application class
- `HypatiaLogger.kt` — Logging infrastructure
- `LinkScannerService.kt` — Accessibility-based domain scanner
- `MainActivity.kt` — Main UI and user interactions
- `MalwareScanner.kt` — Core file scanning engine
- `MalwareScannerService.kt` — Realtime file monitoring service
- `NotificationPromptActivity.kt` — Detection notification actions
- `RecursiveFileObserver.kt` — Recursive directory monitoring
- `ShareScanner.kt` — Scan files shared to the app
- `SignatureDatabase.kt` — Database URL model
- `Utils.kt` — Utility functions

### Configuration
- `AndroidManifest.xml` — Permissions and component declarations
- `app/build.gradle` — Build configuration and dependencies
- `accessibility_service_config.xml` — Accessibility service scope

### Server-Side Scripts (`scripts/`)
- `Main.java` — Bloom filter database generator
- `0clamav.sh`, `0eset.sh`, `0targetedthreats.sh`, `0stalkerware.sh`, `0threatfox.sh`, `0threatview.sh`, `0cybercure.sh`, `0sanesecurity.sh`, `0malshare-bulk.sh`, `0malshare-combine.sh`, `0avast-covid19.sh`, `0genbloom.sh`
