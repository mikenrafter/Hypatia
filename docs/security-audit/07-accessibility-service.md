# Accessibility Service Analysis

[← Back to Table of Contents](../../SECURITY_AUDIT.md) | [← Database Downloads](./06-database-downloads.md)

---

## Overview

Hypatia includes an accessibility service (`LinkScannerService`) that scans on-screen text for malicious domain names. Accessibility services are a well-known vector for malware on Android because they can read all screen content. This section analyzes Hypatia's use in detail.

## Why Accessibility Services Are Sensitive

Android accessibility services have the ability to:
- Read all text displayed on screen
- Monitor all app activity
- Perform actions on behalf of the user
- Overlay content on other apps

Malware frequently abuses these capabilities for credential theft, screen scraping, and click fraud. **Hypatia's use is limited to domain name scanning.**

## Hypatia's Implementation

### Service Declaration

From [`AndroidManifest.xml`](../../app/src/main/AndroidManifest.xml):

```xml
<service
    android:name=".LinkScannerService"
    android:exported="false"
    android:foregroundServiceType="specialUse"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
    android:label="@string/accessibility_service_label">
```

### Service Configuration

From [`accessibility_service_config.xml`](../../app/src/main/res/xml/accessibility_service_config.xml):

```xml
<accessibility-service
    android:accessibilityEventTypes="typeWindowContentChanged|typeViewTextChanged"
    android:accessibilityFlags="flagDefault"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:notificationTimeout="100"
    android:canRetrieveWindowContent="true" />
```

- **Event types:** Only `typeWindowContentChanged` and `typeViewTextChanged` — the minimum needed to detect when new text appears on screen
- **Can retrieve window content:** `true` — needed to read the actual text
- **Notification timeout:** 100ms — rate-limits how often events are processed

### What It Does

The service ([`LinkScannerService.kt`](../../app/src/main/java/us/spotco/malwarescanner/LinkScannerService.kt)) processes accessibility events as follows:

1. **Receives text change events** — `onAccessibilityEvent()` (line 69)
2. **Rate-limits processing** — Skips packages scanned within the last 1 second (lines 77–80)
3. **Prevents duplicate work** — Tracks active scanner threads per package (lines 83–93)
4. **Reads text from view nodes** — `scanViews()` recursively reads `AccessibilityNodeInfo.text` (lines 125–158)
5. **Extracts domain names** — Uses regex pattern matching (line 236)
6. **Checks against bloom filter** — `Database.domains.mightContain(domain)` (line 180)
7. **Notifies user if match found** — Shows a high-priority notification (lines 194–208)

### What It Does NOT Do

- ❌ Does NOT record or store screen content
- ❌ Does NOT transmit any data off-device
- ❌ Does NOT perform any actions on behalf of the user
- ❌ Does NOT interact with UI elements (no clicks, no input)
- ❌ Does NOT read passwords or form fields specifically
- ❌ Does NOT overlay content on other apps

### Domain Matching

The regex used for domain extraction ([`LinkScannerService.kt`](../../app/src/main/java/us/spotco/malwarescanner/LinkScannerService.kt), line 236):

```kotlin
private val hostnameRegex = Regex("^((?!-)[A-Za-z0-9-]{1,63}(?<!-)\\.)+[a-zA-Z0-9]{2,63}$")
```

This matches standard hostname patterns. Extracted domains are checked against the `hypatia-domains-bloom.bin` database.

### Rate-Limiting and Memory Bounds

The service includes multiple safeguards against resource abuse:

| Safeguard | Location | Detail |
|---|---|---|
| Package cooldown | Line 78 | 1-second cooldown per package |
| Thread deduplication | Lines 83–93 | Only one scanner thread per package at a time |
| Text cache limit | Line 140 | Clears text hash cache at 10,000 entries |
| Domain cache limit | Line 176 | Clears domain cache at 1,000 entries |
| Text length limit | Line 134 | Skips text longer than 10,000 characters |

### User Enablement

The accessibility service cannot be enabled programmatically. The user must:

1. Enable "Link Scanner" in Hypatia's menu, which opens Android's Accessibility Settings
2. Find Hypatia in the list of accessibility services
3. Manually toggle it on
4. Confirm the system warning dialog about accessibility permissions

This is a **four-step, fully user-initiated process**.

### Disabling

If the user disables the "Link Scanner" option in Hypatia's menu (setting `DOMAINS` preference to `false`), the service stops its foreground notification. The service itself is managed by the Android accessibility framework and can be disabled at any time from system settings.

## Summary

| Aspect | Assessment |
|---|---|
| Purpose | Scan on-screen text for malicious domains |
| Data collected | Domain names visible on screen (not stored or transmitted) |
| User action required | Manual 4-step enablement in system settings |
| Screen content storage | None — text is processed in memory and discarded |
| Network transmission | None — all checking is local against bloom filter |
| UI interaction | None — read-only observation |

---

[Next: Build Infrastructure & Scripts →](./08-build-scripts.md)
