# Disk & Storage Activity Analysis

[← Back to Table of Contents](../../SECURITY_AUDIT.md) | [← Network Activity](./02-network-activity.md)

---

## Overview

Hypatia interacts with the file system for three purposes: **scanning files** (read-only), **managing signature databases** (read/write in app-private storage), and **responding to malware detections** (delete with user confirmation).

## Read Operations

### Manual Scan — `MalwareScanner.kt`

The manual scan ([`MalwareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScanner.kt)) reads files to compute their cryptographic hashes. The scan scope is user-configured in [`MainActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/MainActivity.kt) lines 412–462:

| Scan Option | Paths Scanned |
|---|---|
| System | `/`, `/system`, `/data`, `/vendor`, `/product`, `/apex`, `/cache`, `/firmware`, `/oem`, `/odm`, and related |
| Apps | All installed app directories (`sourceDir`, `dataDir`, `nativeLibraryDir`, `publicSourceDir`) |
| External Storage | `Environment.getExternalStorageDirectory()` (typically `/storage/emulated/0`) |
| Removable Storage | `/storage` |

**File size limits** ([`Utils.kt`](../../app/src/main/java/us/spotco/malwarescanner/Utils.kt)):
- Manual scan: 500 MB max per file (`MAX_SCAN_SIZE`)
- Realtime scan: 250 MB max per file (`MAX_SCAN_SIZE_REALTIME`)

### Realtime Monitoring — `MalwareScannerService.kt`

The realtime scanner ([`MalwareScannerService.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScannerService.kt)) uses [`RecursiveFileObserver.kt`](../../app/src/main/java/us/spotco/malwarescanner/RecursiveFileObserver.kt) to monitor external storage for file changes:

- Watches for `MOVED_TO` and `CLOSE_WRITE` events
- Monitors directories up to 8 levels deep
- Only scans files ≤ 250 MB
- Runs as a foreground service with a persistent notification

### Share Scanner — `ShareScanner.kt`

When files are shared to Hypatia via Android's share intent ([`ShareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/ShareScanner.kt)):

- Resolves the file URI to a real path when possible
- For unsupported URIs, copies the file to the app's cache directory for scanning
- Cache copies are deleted after scanning

### How Files Are Read

In `MalwareScanner.getFileHashes()` ([`MalwareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScanner.kt), lines 432–472):

```kotlin
val fis: InputStream = FileInputStream(file)
val buffer = ByteArray(4096)
// Computes MD5, SHA-1, SHA-256 in a single pass
```

Files are read in 4 KB chunks. Only cryptographic hash digests are computed — **file contents are never stored, copied, or transmitted**.

## Write Operations

### Database Storage

Signature databases are stored in app-private storage ([`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt), line 326):

```kotlin
databasePath = File(context.filesDir.toString() + "/signatures/")
```

This is the app's internal storage directory, inaccessible to other apps. Downloaded database files are written here with a `.new` suffix during download, then renamed to the final name atomically.

### Self-Test Files

`Utils.writeSelfTestFiles()` ([`Utils.kt`](../../app/src/main/java/us/spotco/malwarescanner/Utils.kt), lines 225–246) writes test files to external storage when the user explicitly triggers a self-test. These files contain known test strings (e.g., `"HypatiaHypatiaHypatia-MD5"`) that match the self-test entries in the signature databases.

### Share Scanner Cache

`ShareScanner.handleUnsupportedUri()` ([`ShareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/ShareScanner.kt), lines 177–193) may temporarily copy a shared file to the app's cache directory. The copy is deleted after scanning.

## Delete Operations

### File Deletion (User-Initiated)

When malware is detected, the user can choose to delete the file via a notification action. The delete flow requires **explicit user confirmation** through an `AlertDialog`:

1. Detection notification appears with a "Delete" action button
2. User taps "Delete"
3. `NotificationPromptActivity` shows a confirmation dialog ([`NotificationPromptActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/NotificationPromptActivity.kt), lines 78–120)
4. User confirms by tapping "Yes"
5. `File.delete()` is called

Only files on external storage (paths starting with `~/`) can be deleted. System files cannot be deleted through this mechanism.

### App Uninstallation (User-Initiated)

For detected malware in installed apps (paths starting with `/data/app`), the user can trigger an uninstall via Android's standard `ACTION_UNINSTALL_PACKAGE` intent ([`NotificationPromptActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/NotificationPromptActivity.kt), lines 122–148). This delegates to the system package manager, which shows its own confirmation dialog.

### Cache Cleanup

After scanning shared files, cached copies are deleted ([`MalwareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScanner.kt), lines 325–333).

## Summary

| Operation | Scope | User Consent Required |
|---|---|---|
| Read files for hashing | User-selected scan targets | Implicit (user initiates scan) |
| Monitor file changes | External storage | User enables realtime scanner |
| Write signature databases | App-private storage only | Automatic during update |
| Write self-test files | External storage | User explicitly triggers |
| Delete detected files | External storage only | Explicit dialog confirmation |
| Uninstall detected apps | Via system package manager | Explicit dialog + system confirmation |

---

[Next: Memory & Process Activity →](./04-memory-process-activity.md)
