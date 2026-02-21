# Memory & Process Activity Analysis

[← Back to Table of Contents](../../SECURITY_AUDIT.md) | [← Disk & Storage Activity](./03-disk-activity.md)

---

## Overview

Hypatia uses standard Android concurrency primitives for scanning operations. There is no evidence of excessive resource consumption, hidden background processes, or memory-based exploits.

## Threading Model

### Thread Pool Executor

The app uses a bounded `ThreadPoolExecutor` ([`Utils.kt`](../../app/src/main/java/us/spotco/malwarescanner/Utils.kt), lines 67–76):

```kotlin
fun getNewThreadPoolExecutor(threads: Int): ThreadPoolExecutor {
    return ThreadPoolExecutor(
        threads, threads,
        0L, TimeUnit.MILLISECONDS,
        LinkedBlockingQueue(32),
        ThreadPoolExecutor.CallerRunsPolicy()
    )
}
```

- **Max threads:** Capped at `min(availableProcessors, 4)`, minimum 2 ([`Utils.kt`](../../app/src/main/java/us/spotco/malwarescanner/Utils.kt), lines 78–88)
- **Queue size:** 32 pending tasks maximum
- **Overflow policy:** `CallerRunsPolicy` — excess tasks run on the calling thread (no unbounded growth)

### AsyncTask Usage

Both `MalwareScanner` and `Database.Downloader` extend `AsyncTask` (deprecated but functional). These run on the shared thread pool executor, not the default `AsyncTask` serial executor.

### Coroutine Usage

Database loading uses Kotlin coroutines with `Dispatchers.IO` ([`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt), lines 335–385):

- `loadDatabase()` is a `suspend` function with a 90-second timeout ([`MalwareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScanner.kt), lines 250–258)
- Domain database loading runs on `Dispatchers.IO`
- The `MalwareScannerService` uses its own `CoroutineScope(Dispatchers.IO + Job())` which is cancelled on service destruction

## Wake Locks

Wake locks prevent the device from sleeping during critical operations:

| Wake Lock | Location | Duration | Purpose |
|---|---|---|---|
| `Hypatia::ManualScanLock` | [`MainActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/MainActivity.kt), line 469 | 10 minutes max | Manual file scan |
| `Hypatia::UpdateLock` | [`MainActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/MainActivity.kt), line 497 | 3 minutes max | Database update download |

Both wake locks:
- Use `PARTIAL_WAKE_LOCK` (CPU only, screen can turn off)
- Have explicit timeout limits
- Are released when the operation completes (or at timeout)

## Memory Usage

### Bloom Filters

The primary memory consumers are Guava `BloomFilter` objects holding malware signatures:

- MD5 bloom filter: ~7.6 million entries (standard) or ~52 million (extended)
- SHA-1 bloom filter: ~100,000 entries
- SHA-256 bloom filter: ~2.2 million entries
- Domain bloom filter: ~4.5 million entries (optional)

The README states the app uses under 120 MB with default databases enabled.

### Large Heap

The AndroidManifest declares `android:largeHeap="true"` ([`AndroidManifest.xml`](../../app/src/main/AndroidManifest.xml), line 32), which requests additional heap space from the system. This is appropriate for an app that loads large bloom filters into memory.

### Garbage Collection

The app explicitly calls `System.gc()` after scans ([`MalwareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScanner.kt), lines 340–342), throttled to every 40 files scanned during realtime scanning to avoid performance impact:

```kotlin
if (userFacing || Utils.FILES_SCANNED.get() % 40 == 0) {
    System.gc()
}
```

### Hash Maps

Per-scan hash maps (`fileHashesMD5`, `fileHashesSHA1`, `fileHashesSHA256`) are cleared after each scan completes ([`MalwareScanner.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScanner.kt), lines 336–338).

## Foreground Services

### MalwareScannerService

Runs as an Android foreground service with `START_STICKY` ([`MalwareScannerService.kt`](../../app/src/main/java/us/spotco/malwarescanner/MalwareScannerService.kt)):

- Shows a persistent low-priority notification
- Restarts automatically if killed by the system
- Can be stopped by the user via the menu toggle
- Cleans up all `FileObserver` instances on destruction

### LinkScannerService

Runs as an accessibility service ([`LinkScannerService.kt`](../../app/src/main/java/us/spotco/malwarescanner/LinkScannerService.kt)):

- Managed by the Android accessibility framework
- Shows a foreground notification when active
- Rate-limits scanning per package (1-second cooldown)
- Clears cached scan results at 10,000 entries to prevent unbounded growth

## Boot Receiver

`EventReceiver` ([`EventReceiver.kt`](../../app/src/main/java/us/spotco/malwarescanner/EventReceiver.kt)) listens for:

- `BOOT_COMPLETED` — Restarts the realtime scanner service if previously enabled
- `PACKAGE_REPLACED` — Restarts the service if Hypatia itself was updated

The boot receiver only calls `considerStartService()`, which checks the user's `autostart` preference before starting anything.

## Summary

| Aspect | Assessment |
|---|---|
| Thread count | Bounded at 2–4 threads |
| Task queue | Bounded at 32 tasks |
| Wake locks | Time-limited, properly released |
| Memory growth | Bounded by bloom filter sizes; hash maps cleared per scan |
| Background services | Standard Android foreground services with notifications |
| Boot persistence | Only if user explicitly enabled realtime scanning |

---

[Next: Permissions Analysis →](./05-permissions-analysis.md)
