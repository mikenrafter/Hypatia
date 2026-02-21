# Database / Payload Download Analysis

[← Back to Table of Contents](../../SECURITY_AUDIT.md) | [← Permissions Analysis](./05-permissions-analysis.md)

---

## Overview

The "payloads" downloaded by Hypatia are **malware signature databases** — serialized Guava BloomFilter objects containing hashes of known malware. These are data files, not executable code.

## Database Format

### Bloom Filters

Signature databases are serialized [Guava `BloomFilter`](https://guava.dev/releases/snapshot/api/docs/com/google/common/hash/BloomFilter.html) objects. A bloom filter is a probabilistic data structure that can tell you:
- **Definitely not in the set** — a hash is not known malware
- **Probably in the set** — a hash might be known malware (with a configurable false positive rate)

The databases contain only hash strings (MD5, SHA-1, SHA-256) and cannot contain executable code.

### Database Files

| File | Contents | False Positive Rate |
|---|---|---|
| `hypatia-md5-bloom.bin` | ~7.6M MD5 hashes of known malware | 0.00001 |
| `hypatia-sha1-bloom.bin` | ~100K SHA-1 hashes of known malware | 0.00001 |
| `hypatia-sha256-bloom.bin` | ~2.2M SHA-256 hashes of known malware | 0.00001 |
| `hypatia-md5-extended-bloom.bin` | ~52M additional MD5 hashes (optional) | 0.00001 |
| `hypatia-domains-bloom.bin` | ~4.5M malicious domain names (optional) | 0.00001 |

## Download Process

### Step 1: Initiate Download

The download is triggered by the user tapping "Update Database" in the menu ([`MainActivity.kt`](../../app/src/main/java/us/spotco/malwarescanner/MainActivity.kt), line 498):

```kotlin
Database.updateDatabase(this,
    Database.signatureDatabases as ConcurrentLinkedQueue<SignatureDatabase>
)
```

### Step 2: Download Files

`Database.updateDatabase()` ([`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt), lines 263–307) downloads:

1. The GPG public key (`gpg.key`)
2. Each signature database file (`.bin`)
3. Each signature's GPG detached signature (`.bin.sig`)

Downloads use `HttpURLConnection` with:
- `If-Modified-Since` conditional requests (HTTP 304 support)
- 90-second connect timeout, 30-second read timeout
- Files written with `.new` suffix, then renamed atomically

### Step 3: Verify Signatures

Before loading any database, GPG signature verification is performed ([`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt), lines 403–425):

```kotlin
val verifier = GPGDetachedSignatureVerifier(getSigningKey(context))
// ...
val validated = verifier.verify(databaseLocation, databaseSigLocation, publicKey)
if (validated) {
    // Load the database
} else {
    log("Database validation error")
}
```

### Step 4: Load into Memory

Verified databases are deserialized from files into `BloomFilter` objects ([`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt), lines 349–384):

```kotlin
val bf: BloomFilter<String?> = BloomFilter.readFrom(databaseLoading, Funnels.stringFunnel(Charsets.US_ASCII))
```

## GPG Signature Verification

The [`GPGDetachedSignatureVerifier.java`](../../app/src/main/java/us/spotco/malwarescanner/GPGDetachedSignatureVerifier.java) implements PGP signature verification using BouncyCastle:

1. Reads the detached signature file
2. Extracts the `PGPSignature` object
3. Looks up the signing key by Key ID in the downloaded public keyring
4. Verifies the signature against the database file contents

### Signing Key IDs

| Source | Key ID | Defined In |
|---|---|---|
| MaintainTeam | `5298C0C0C3E73288` | [`DatabaseSource.kt`](../../app/src/main/java/us/spotco/malwarescanner/DatabaseSource.kt) |
| AXP OS | `14C17E7F99EABF3F` | [`DatabaseSource.kt`](../../app/src/main/java/us/spotco/malwarescanner/DatabaseSource.kt) |

The signing key can be overridden by the user via the "Signing Key" menu option.

## Database Sources (Server-Side Generation)

The `scripts/` directory contains shell scripts and a Java tool that are used **server-side** to generate the bloom filter databases. These do NOT run on the Android device:

| Script | Purpose | Source Data |
|---|---|---|
| [`0clamav.sh`](../../scripts/0clamav.sh) | Extracts hashes from ClamAV databases | ClamAV (GPL-2.0) |
| [`0eset.sh`](../../scripts/0eset.sh) | Extracts hashes from ESET malware IOCs | ESET (BSD-2-Clause) |
| [`0targetedthreats.sh`](../../scripts/0targetedthreats.sh) | Extracts hashes from targeted threats dataset | Nex/botherder (CC BY-SA 4.0) |
| [`0stalkerware.sh`](../../scripts/0stalkerware.sh) | Extracts hashes from stalkerware indicators | Echap (CC BY 4.0) |
| [`0threatfox.sh`](../../scripts/0threatfox.sh) | Extracts hashes from ThreatFox | abuse.ch (CC0) |
| [`0threatview.sh`](../../scripts/0threatview.sh) | Downloads hashes from ThreatView | ThreatView |
| [`0cybercure.sh`](../../scripts/0cybercure.sh) | Downloads hashes from CyberCure API | CyberCure |
| [`0sanesecurity.sh`](../../scripts/0sanesecurity.sh) | Extracts hashes from SaneSecurity | SaneSecurity |
| [`0malshare-bulk.sh`](../../scripts/0malshare-bulk.sh) | Generates URLs for MalShare daily lists | MalShare |
| [`0malshare-combine.sh`](../../scripts/0malshare-combine.sh) | Combines MalShare hash files | MalShare |
| [`0avast-covid19.sh`](../../scripts/0avast-covid19.sh) | Extracts hashes from Avast COVID-19 IOCs | Avast |
| [`0genbloom.sh`](../../scripts/0genbloom.sh) | Generates bloom filter files and HTML index | — |
| [`Main.java`](../../scripts/Main.java) | Java tool that reads raw hash files and creates bloom filter `.bin` files | — |

## Self-Test Mechanism

The app includes a self-test to verify databases loaded correctly ([`Database.kt`](../../app/src/main/java/us/spotco/malwarescanner/Database.kt), lines 460–471):

```kotlin
fun selfTest(): Boolean {
    return signaturesMD5!!.mightContain("903616d0dbe074aa363d2d49c03f7362")
            && signaturesMD5!!.mightContain("faf325d9d4b2a6c9457405eb31870b22")
            && signaturesSHA1!!.mightContain("fc4a3e802894cc2229be77ec6f082d1aab744e54")
            // ...
}
```

These are hashes of known test strings (e.g., `"HypatiaHypatiaHypatia"`) embedded in the databases during generation.

## Old Database Detection

The app warns users who are using deprecated database URLs or signing keys ([`Utils.kt`](../../app/src/main/java/us/spotco/malwarescanner/Utils.kt), lines 259–271):

```kotlin
val OLD_DATABASES: Array<String?> = arrayOf(
    "https://raw.githubusercontent.com/MaintainTeam/HypatiaDatabases/refs/heads/main/",
    "https://maintainteam.github.io/HypatiaDatabasesTests/"
)
```

## Summary

| Aspect | Assessment |
|---|---|
| Download content | Bloom filter data files (not executable) |
| Signature verification | GPG detached signatures via BouncyCastle |
| Source trust | Known open-source malware hash databases |
| Integrity checking | Databases rejected if GPG verification fails |
| User control | User can change database source and signing key |

---

[Next: Accessibility Service →](./07-accessibility-service.md)
