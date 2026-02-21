# Build Infrastructure & Scripts Analysis

[← Back to Table of Contents](../../SECURITY_AUDIT.md) | [← Accessibility Service](./07-accessibility-service.md)

---

## Overview

The repository contains a standard Android Gradle build system, CI/CD workflows, and server-side scripts for generating signature databases. None of the scripts run on the Android device.

## Android Build System

### Build Configuration

The app uses a standard Gradle build ([`build.gradle`](../../build.gradle), [`app/build.gradle`](../../app/build.gradle)):

- **compileSdkVersion:** 36
- **targetSdkVersion:** 34
- **minSdkVersion:** 21
- **Java/Kotlin version:** 17
- **ProGuard:** Enabled for release builds

### Dependencies

From `app/build.gradle` (referencing a version catalog):

| Dependency | Purpose | Risk Assessment |
|---|---|---|
| `commons-io` (Apache Commons IO) | File I/O utilities | 🟢 Well-known, trusted library |
| `bcpg` (BouncyCastle) | GPG signature verification | 🟢 Industry-standard crypto library |
| `guava` (Google Guava) | BloomFilter implementation | 🟢 Google's core Java library |
| `androidx.appcompat` | Android compatibility | 🟢 Official Android library |
| `androidx.documentfile` | Document file handling | 🟢 Official Android library |
| `androidx.core-ktx` | Kotlin Android extensions | 🟢 Official Android library |

No suspicious, unknown, or backdoored dependencies. All dependencies are mainstream, well-audited libraries.

### Build Variants

| Variant | Application ID | Notes |
|---|---|---|
| Debug | `org.maintainteam.hypatia.debug[.branch]` | Branch name appended for non-standard branches |
| Release | `org.maintainteam.hypatia` | Minified with ProGuard |

### Signing

The repository includes debug signing keys:
- `debugkey.pk8` — Debug signing private key
- `debugkey.x509.pem` — Debug signing certificate

These are standard Android debug keys used for development builds. Release builds use separate signing keys not included in the repository.

## CI/CD Workflows

The `.github/workflows/` directory contains:

| Workflow | Purpose |
|---|---|
| `ci.yml` | Continuous integration (build/test) |
| `release.yml` | Release builds |
| `validate-gradle-wrapper.yml` | Validates Gradle wrapper integrity |

The `validate-gradle-wrapper.yml` workflow is a security best practice that ensures the Gradle wrapper hasn't been tampered with.

## Server-Side Scripts (`scripts/`)

These scripts run on the database build server, NOT on Android devices. They convert raw malware hash data from various sources into the bloom filter format used by Hypatia.

### Database Generation Pipeline

```
Raw malware hashes → Shell scripts (extract/format) → Main.java (create bloom filters) → .bin files → GPG signed → Hosted on web server
```

### Script Analysis

#### `Main.java` — Bloom Filter Generator

The core database generator ([`scripts/Main.java`](../../scripts/Main.java)):

1. Reads hash files in various formats (`.hdb`, `.hsb`, `.md5`, `.sha1`, `.sha256`, `.loki`, `.txt`)
2. Validates each line as a hexadecimal hash of the correct length
3. Adds valid hashes to the appropriate bloom filter
4. Applies exclusions (known false positives)
5. Writes bloom filter binary files

This is a straightforward data processing tool with no network access or dangerous operations.

#### Shell Scripts — Data Extraction

Each shell script extracts hashes from a specific malware intelligence source:

| Script | Data Source | Operation |
|---|---|---|
| `0clamav.sh` | ClamAV databases | Extracts `.hdb`/`.hsb` files using `sigtool` |
| `0eset.sh` | ESET malware-ioc repo | Reads `samples.md5`, `samples.sha1`, `samples.sha256` |
| `0targetedthreats.sh` | targetedthreats repo | Parses CSV of targeted threat IOCs |
| `0stalkerware.sh` | stalkerware-indicators repo | Parses CSV of stalkerware hashes |
| `0threatfox.sh` | ThreatFox CSV export | Extracts SHA-256 hashes |
| `0threatview.sh` | ThreatView feeds | Downloads MD5/SHA-1 hash lists |
| `0cybercure.sh` | CyberCure API | Downloads MD5 hash feed |
| `0sanesecurity.sh` | SaneSecurity databases | Copies `.hdb`/`.hsb` files |
| `0malshare-bulk.sh` | MalShare daily lists | Generates download URLs |
| `0malshare-combine.sh` | MalShare hash files | Combines and deduplicates |
| `0avast-covid19.sh` | Avast COVID-19 IOCs | Extracts SHA-256 from CSV |
| `0genbloom.sh` | All raw hashes | Runs `Main.java` to create bloom filters |

All scripts perform straightforward text processing (sorting, deduplication, format conversion). None contain hidden functionality.

## Gradle Wrapper

The repository includes a Gradle wrapper (`gradlew`, `gradlew.bat`, `gradle/`) for reproducible builds. The CI includes `validate-gradle-wrapper.yml` to verify its integrity.

## Summary

| Component | Risk | Assessment |
|---|---|---|
| Android build | 🟢 Low | Standard Gradle with official dependencies |
| Dependencies | 🟢 Low | All well-known, mainstream libraries |
| CI/CD | 🟢 Low | Standard GitHub Actions with wrapper validation |
| Server scripts | 🟢 Low | Data processing only, no hidden behavior |
| Signing keys | 🟢 Low | Only debug keys in repo, release keys separate |

---

[Next: Potential Security Concerns & Recommendations →](./09-potential-concerns.md)
