# Permissions Analysis

[← Back to Table of Contents](../../SECURITY_AUDIT.md) | [← Memory & Process Activity](./04-memory-process-activity.md)

---

## Overview

Hypatia requests a broad set of Android permissions, which is expected for a malware scanner that needs to read all files on the device. Every permission has a clear, documented justification.

## Declared Permissions

From [`AndroidManifest.xml`](../../app/src/main/AndroidManifest.xml):

### Network Permissions

| Permission | Risk | Justification | Code Reference |
|---|---|---|---|
| `INTERNET` | Normal | Download signature databases | [`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt) `Downloader` class |
| `ACCESS_NETWORK_STATE` | Normal | Check network availability before downloads | [`Utils.kt`](../../app/src/main/java/us/spotco/malwarescanner/Utils.kt) `isNetworkAvailable()` |

### Storage Permissions

| Permission | Risk | Justification | Code Reference |
|---|---|---|---|
| `MANAGE_EXTERNAL_STORAGE` | **Dangerous** | Read all files for scanning; delete detected malware | [`MainActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/MainActivity.kt) `requestAllPermissions()` |
| `READ_EXTERNAL_STORAGE` | Dangerous | Read files for scanning (Android ≤ 10) | Legacy fallback |
| `WRITE_EXTERNAL_STORAGE` | Dangerous | Delete detected files, write self-test files (Android ≤ 10) | Legacy fallback |

**Note:** `MANAGE_EXTERNAL_STORAGE` is the most privileged storage permission available. The app requests this on Android 11+ via `Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION`, which opens the system settings for the user to explicitly grant access.

### Package Permissions

| Permission | Risk | Justification | Code Reference |
|---|---|---|---|
| `QUERY_ALL_PACKAGES` | Normal | Enumerate installed apps to scan their APKs | [`MainActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/MainActivity.kt) `startScanner()` |
| `REQUEST_DELETE_PACKAGES` | Normal | Uninstall detected malicious apps via system dialog | [`NotificationPromptActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/NotificationPromptActivity.kt) `UNINSTALL_APP` |

### Service Permissions

| Permission | Risk | Justification | Code Reference |
|---|---|---|---|
| `FOREGROUND_SERVICE` | Normal | Run realtime scanner as foreground service | [`MalwareScannerService.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScannerService.kt) |
| `FOREGROUND_SERVICE_SPECIAL_USE` | Normal | Android 14+ foreground service type declaration | [`MalwareScannerService.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScannerService.kt) |
| `WAKE_LOCK` | Normal | Prevent device sleep during scans | [`MainActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/MainActivity.kt) wake lock acquisition |
| `RECEIVE_BOOT_COMPLETED` | Normal | Restart realtime scanner after device reboot | [`EventReceiver.kt`](../../app/src/main/java/us/spotco/malwarescanner/EventReceiver.kt) |
| `POST_NOTIFICATIONS` | Dangerous (Android 13+) | Show malware detection notifications | [`MalwareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScanner.kt) `logResult()` |

### Accessibility Service

The accessibility service is declared in the manifest but is **not a runtime permission** — it requires the user to manually navigate to Android's Accessibility Settings and enable it:

```xml
<service
    android:name=".LinkScannerService"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
    ...>
```

Configuration ([`accessibility_service_config.xml`](../../app/src/main/res/xml/accessibility_service_config.xml)):
- Event types: `typeWindowContentChanged`, `typeViewTextChanged`
- Can retrieve window content: `true`
- Feedback type: `feedbackGeneric`

## Package Queries

```xml
<queries>
    <package android:name="org.torproject.android" />
</queries>
```

The app only queries for Orbot (Tor proxy) to check if onion routing is available.

## Permission Request Flow

In [`MainActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/MainActivity.kt) `requestAllPermissions()` (lines 145–165):

1. **Android 13+:** Requests `POST_NOTIFICATIONS`
2. **Android 6–10:** Requests `READ_EXTERNAL_STORAGE` and `WRITE_EXTERNAL_STORAGE`
3. **Android 11+:** Opens system settings for `MANAGE_EXTERNAL_STORAGE` (requires user to manually toggle)

All permission requests use the standard Android `ActivityResultContracts.RequestPermission()` API. No permissions are granted silently or through exploits.

## Exported Components

| Component | Exported | Protection |
|---|---|---|
| `MainActivity` | Yes | Standard launcher activity |
| `ShareScanner` | Yes | Receives `ACTION_SEND` intents for file scanning |
| `NotificationPromptActivity` | No | Internal notification actions only |
| `MalwareScannerService` | No | Internal service |
| `LinkScannerService` | No | Protected by `BIND_ACCESSIBILITY_SERVICE` permission |
| `EventReceiver` | Yes | Responds to system broadcasts only (`BOOT_COMPLETED`, `PACKAGE_REPLACED`) |

## Summary

All permissions are consistent with the app's stated purpose as a malware scanner. The most sensitive permissions (`MANAGE_EXTERNAL_STORAGE`, accessibility service) require explicit user action in system settings and cannot be granted programmatically.

---

[Next: Database Downloads →](./06-database-downloads.md)
