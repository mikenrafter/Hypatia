# Potential Security Concerns & Recommendations

[← Back to Table of Contents](../../SECURITY_AUDIT.md) | [← Build Infrastructure & Scripts](./08-build-scripts.md)

---

## Overview

While Hypatia is not malware, this section documents areas where the codebase could be hardened or improved from a security perspective.

## Concern 1: Custom Database Server Allows Arbitrary URLs

**Severity:** 🟡 Medium
**Location:** [`MainActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/MainActivity.kt) `showCustomServerDialog()` (lines 557–583)

**Description:** Users can set any URL as the database server. While this flexibility is useful, it means a user could be tricked into pointing to a malicious server.

**Mitigating factors:**
- All databases must pass GPG signature verification before loading
- The signing key is separately configurable, providing defense in depth
- The user must actively change the setting

**Recommendation:** Consider implementing a warning dialog when setting a custom URL that differs from known sources, and consider certificate pinning for the default sources.

## Concern 2: Signing Key Override

**Severity:** 🟡 Medium
**Location:** [`MainActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/MainActivity.kt) `mnuSigningKey` handler (lines 266–291)

**Description:** Users can override the GPG signing key used to verify databases. If both the server URL and signing key are changed, an attacker could serve arbitrary bloom filter databases.

**Mitigating factors:**
- An attacker controlling both would only be able to influence scan results (false positives/negatives), not execute code
- Bloom filter databases cannot contain executable code
- Both settings must be changed by the user

**Recommendation:** Consider requiring a confirmation dialog that explicitly warns about the security implications of changing the signing key.

## Concern 3: Deprecated AsyncTask Usage

**Severity:** 🟢 Low
**Location:** [`MalwareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScanner.kt), [`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt)

**Description:** The app uses `AsyncTask`, which is deprecated since Android 11 (API 30). While functional, it has known issues with lifecycle management and memory leaks.

**Recommendation:** Migrate to Kotlin coroutines or `java.util.concurrent` for background tasks.

## Concern 4: Bloom Filter False Positives

**Severity:** 🟢 Low (by design)
**Location:** [`MalwareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScanner.kt) `checkSignature()` (lines 382–430)

**Description:** Bloom filters are probabilistic data structures that can produce false positives (reporting a clean file as malware). The configured false positive rate is 0.00001 (0.001%).

**Mitigating factors:**
- The false positive rate is very low
- Detected files require user action (delete/ignore) — no automatic deletion
- Users can verify detections on VirusTotal

**Recommendation:** This is inherent to the bloom filter approach and is well-documented. No change needed.

## Concern 5: HTTP Download Without Certificate Pinning

**Severity:** 🟢 Low
**Location:** [`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt) `Downloader` class

**Description:** Database downloads use `HttpURLConnection` over HTTPS without certificate pinning. While TLS provides transport security, certificate pinning would prevent MITM attacks using rogue certificates.

**Mitigating factors:**
- HTTPS provides baseline transport security
- GPG signature verification catches any tampering regardless of transport security
- Tor routing option provides additional anonymity

**Recommendation:** Consider adding certificate pinning for the default database sources. However, the GPG verification layer makes this a low-priority concern.

## Concern 6: EventReceiver Exported with Broad Intent Filters

**Severity:** 🟢 Low
**Location:** [`AndroidManifest.xml`](../../app/src/main/AndroidManifest.xml) lines 88–103

**Description:** The `EventReceiver` is exported and listens for `BOOT_COMPLETED`, `PACKAGE_REPLACED`, and `PACKAGE_ADDED` system broadcasts.

**Mitigating factors:**
- These are standard system broadcasts that any app can register for
- The receiver only calls `considerStartService()`, which checks user preferences before acting
- No sensitive operations are performed

**Recommendation:** No change needed. This is standard Android practice for boot-persistent services.

## Concern 7: Static Context Reference

**Severity:** 🟢 Low
**Location:** [`Utils.kt`](../../app/src/main/java/us/spotco/malwarescanner/Utils.kt) line 43

**Description:** `Utils.context` holds a static reference to the application context. While using the application context (not an activity context) mitigates memory leak concerns, static mutable state can lead to subtle bugs.

**Recommendation:** Consider using dependency injection or the `Hypatia.appContext` companion property consistently instead of a mutable static field.

## Concern 8: Debug Signing Keys in Repository

**Severity:** 🟢 Low
**Location:** `debugkey.pk8`, `debugkey.x509.pem`

**Description:** Debug signing keys are committed to the repository. These are only used for debug builds and do not affect release builds.

**Mitigating factors:**
- Debug keys are separate from release signing keys
- Debug builds have distinct application IDs (`.debug` suffix)
- This is common practice for Android open-source projects

**Recommendation:** Consider adding a note in the README clarifying that release builds use separate signing keys.

## Summary of Recommendations

| Priority | Recommendation |
|---|---|
| Medium | Warn users when setting custom database URLs |
| Medium | Add explicit warning when overriding the signing key |
| Low | Migrate from AsyncTask to coroutines |
| Low | Consider certificate pinning for default sources |
| Low | Remove or document debug signing keys |
| Low | Clean up static context usage |

---

[← Back to Table of Contents](../../SECURITY_AUDIT.md)
