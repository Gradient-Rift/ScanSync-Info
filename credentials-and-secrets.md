# ScanSync Credentials and Secrets Management

This document describes how credentials and secrets are managed in the ScanSync project, where to find them, and how to configure them.

---

## Table of Contents

1. [Overview](#overview)
2. [Build-Time Secrets](#build-time-secrets)
   - [Release Keystore](#release-keystore)
   - [Google Services Configuration](#google-services-configuration)
3. [API Credentials (Committed)](#api-credentials-committed)
   - [Google OAuth Web Client ID](#google-oauth-web-client-id)
   - [Microsoft MSAL Configuration](#microsoft-msal-configuration)
   - [Debug Keystore](#debug-keystore)
4. [CI/CD Secrets (GitHub)](#cicd-secrets-github)
5. [Runtime Credentials](#runtime-credentials)
6. [External Services Configuration](#external-services-configuration)
7. [Quick Reference](#quick-reference)

---

## Overview

ScanSync uses a layered approach to credential management:

| Layer                         | Storage                                     | Committed to Git | Examples                            |
| ----------------------------- | ------------------------------------------- | ---------------- | ----------------------------------- |
| **Build-time secrets**  | `local.properties`, environment variables | No               | Release keystore passwords          |
| **Firebase config**     | `google-services.json`                    | No               | Firebase API keys, project IDs      |
| **API client IDs**      | `strings.xml`, `msal_config.json`       | Yes              | OAuth client IDs (not secrets)      |
| **Debug keystore**      | `keystores/debug.keystore`                | Yes              | Shared debug signing key            |
| **CI/CD secrets**       | GitHub Secrets                              | No               | Encoded keystores, service accounts |
| **Runtime credentials** | Android Keystore                            | Device-local     | User OAuth tokens                   |

---

## Build-Time Secrets

### Release Keystore

**Purpose:** Signs production APK/AAB for Google Play distribution.

**Location:**

- File: `../release.keystore` (relative to `app/` directory, i.e., project root)
- Configuration: `local.properties`

**Configuration in `local.properties`:**

```properties
RELEASE_STORE_FILE=../release.keystore
RELEASE_KEY_ALIAS=scansync
RELEASE_STORE_PASSWORD=<your-password>
RELEASE_KEY_PASSWORD=<your-password>
```

**Alternative (Environment Variables):**

```bash
export KEYSTORE_PATH=/path/to/release.keystore
export KEYSTORE_PASSWORD=<password>
export KEY_ALIAS=scansync
export KEY_PASSWORD=<password>
```

**How it's used:** `app/build.gradle.kts` reads from environment variables first, then falls back to `local.properties`:

```kotlin
storeFile = file(
    System.getenv("KEYSTORE_PATH")
        ?: localProperties.getProperty("RELEASE_STORE_FILE")
        ?: "../release.keystore"
)
```

**To regenerate/create:**

```bash
keytool -genkey -v -keystore release.keystore -alias scansync -keyalg RSA -keysize 2048 -validity 10000
```

---

### Google Services Configuration

**Purpose:** Firebase configuration (Authentication, Firestore, AI Logic/Gemini).

(not committed)

**Per-Build-Type Locations:**

- Dev/Debug: `app/src/dev/google-services.json` or `app/src/debug/google-services.json`
- Release: `app/src/release/google-services.json`

**How to obtain:**

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Select project: `scansync-2025`
3. Project Settings > General > Your apps > Android app
4. Download `google-services.json`

**Contents include (not secrets, but project-specific):**

- Firebase project ID
- API keys (restricted to your app's SHA-1)
- Client IDs
- Firestore database URLs

---

## API Credentials (Committed)

These are client IDs that are safe to commit because they require additional validation (SHA-1 fingerprint, redirect URIs).

### Google OAuth Web Client ID

**Purpose:** Google Sign-In, Drive API, Sheets API, Gmail API authorization.

**Location:** `app/src/main/res/values/strings.xml`

```xml
<string name="google_client_id">YOUR_WEB_CLIENT_ID.apps.googleusercontent.com</string>
```

**How to obtain/update:**

1. Go to [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
2. Select project: `ScanSync`
3. Find "Web client" under OAuth 2.0 Client IDs
4. Copy the Client ID

**Note:** Use the **Web Client ID**, not the Android Client ID. The Credential Manager API requires the web client ID.

---

### Microsoft MSAL Configuration

**Purpose:** Outlook email sending via Microsoft Graph API.

**Location:** `app/src/main/res/raw/msal_config.json`

```json
{
  "client_id": "YOUR_AZURE_APP_CLIENT_ID",
  "authorization_user_agent": "DEFAULT",
  "redirect_uri": "msauth://com.scansync.ocr/YOUR_SIGNATURE_HASH=",
  "broker_redirect_uri_registered": false,
  "account_mode": "SINGLE",
  "authorities": [
    {
      "type": "AAD",
      "authority_url": "https://login.microsoftonline.com/common"
    }
  ]
}
```

**How to obtain/update:**

1. Go to [Azure Portal](https://portal.azure.com/) > App registrations
2. Select your app or create new
3. Copy Application (client) ID
4. Configure redirect URI: `msauth://com.scansync.ocr/<SHA1_SIGNATURE_HASH>`

**To get signature hash:**

```bash
keytool -exportcert -alias androiddebugkey -keystore keystores/debug.keystore -storepass android | openssl sha1 -binary | openssl base64
```

---

### Debug Keystore

**Purpose:** Consistent SHA-1 fingerprint across all developers for Google API validation.

**Location:** `keystores/debug.keystore`

**Credentials (intentionally public):**

| Property       | Value               |
| -------------- | ------------------- |
| Store Password | `android`         |
| Key Alias      | `androiddebugkey` |
| Key Password   | `android`         |

**SHA-1 Fingerprint:** `4E:7A:55:4D:BE:6D:30:65:CC:98:EF:0D:27:E9:4F:22:9D:B3:08:02`

**Why committed:** Ensures all developers have the same SHA-1, so Google APIs work without registering each developer's fingerprint.

---

## CI/CD Secrets (GitHub)

GitHub Actions uses these secrets for automated builds.

**Location:** GitHub > Repository Settings > Secrets and variables > Actions

| Secret Name              | Description                                        | How to Create                       |
| ------------------------ | -------------------------------------------------- | ----------------------------------- |
| `GOOGLE_SERVICES_DEV`  | Base64-encoded dev `google-services.json`        | `base64 -w0 google-services.json` |
| `GOOGLE_SERVICES_PROD` | Base64-encoded production `google-services.json` | `base64 -w0 google-services.json` |
| `KEYSTORE_BASE64`      | Base64-encoded `release.keystore`                | `base64 -w0 release.keystore`     |
| `KEYSTORE_PASSWORD`    | Release keystore password                          | Plain text                          |
| `KEY_ALIAS`            | Key alias (e.g.,`scansync`)                      | Plain text                          |
| `KEY_PASSWORD`         | Key password                                       | Plain text                          |
| `PLAY_SERVICE_ACCOUNT` | Google Play API service account JSON               | From GCP Console                    |

**To encode files for GitHub secrets:**

```bash
# Encode google-services.json
cat app/google-services.json | base64 -w0

# Encode keystore
cat release.keystore | base64 -w0
```

---

## Runtime Credentials

User OAuth tokens are stored securely on-device using Android Keystore.

### CredentialStore

**Location:** `app/src/main/java/com/scansync/ocr/security/CredentialStore.kt`

**How it works:**

1. Uses AES-256-GCM encryption
2. Keys stored in Android Keystore (hardware-backed when available)
3. Keys cannot be extracted from device

**Usage:**

```kotlin
val credentialStore = CredentialStore(context)
val encrypted = credentialStore.encrypt("oauth_token_here")
val decrypted = credentialStore.decrypt(encrypted)
```

**Where tokens are stored:**

- Local: Room database (encrypted)
- Cloud: Firestore `users/{userId}` document (encrypted)

---

## External Services Configuration

### Google Play Console

**In-App Products:**

- Location: Play Console > Monetize > In-app products
- Product IDs must match `CreditPackage.kt`: `credits_100`, `credits_500`, `credits_1000`, `credits_2000`, `credits_5000`

**License Testing:**

- Location: Play Console > Setup > License testing
- Add test email addresses for free test purchases

### Firebase Console

**Project:** `scansync-2025`

**Services configured:**

- Authentication (Google Sign-In)
- Firestore Database
- AI Logic (Gemini)
- App Check (pending enforcement)

**SHA-1 Fingerprints registered:**

- Debug: `4E:7A:55:4D:BE:6D:30:65:CC:98:EF:0D:27:E9:4F:22:9D:B3:08:02`
- Release: (your release keystore SHA-1)

**To get release SHA-1:**

```bash
keytool -list -v -keystore release.keystore -alias scansync
```

### Google Cloud Console

**Project:** `ScanSync`

**APIs enabled:**

- Google Sign-In
- Google Drive API
- Google Sheets API
- Gmail API

**OAuth Consent Screen:** Configured for external users (pending verification for Drive scope)

---

## Quick Reference

### File Locations Summary

| File                     | Path                                       | Committed | Contains                          |
| ------------------------ | ------------------------------------------ | --------- | --------------------------------- |
| `local.properties`     | `/local.properties`                      | No        | SDK path, release keystore config |
| `google-services.json` | `/app/google-services.json`              | No        | Firebase configuration            |
| `strings.xml`          | `/app/src/main/res/values/strings.xml`   | Yes       | Google OAuth client ID            |
| `msal_config.json`     | `/app/src/main/res/raw/msal_config.json` | Yes       | Microsoft OAuth config            |
| `debug.keystore`       | `/keystores/debug.keystore`              | Yes       | Shared debug signing key          |
| `release.keystore`     | `/release.keystore`                      | No        | Production signing key            |

### Common Tasks

**Build debug APK:**

```bash
./gradlew assembleDebug
```

**Build release bundle:**

```bash
VERSION_CODE=10 VERSION_NAME="1.0.9" ./gradlew bundleRelease
```

**Verify keystore SHA-1:**

```bash
# Debug
keytool -list -v -keystore keystores/debug.keystore -alias androiddebugkey -storepass android

# Release
keytool -list -v -keystore release.keystore -alias scansync
```

**Register new SHA-1 in Firebase:**

1. Firebase Console > Project Settings > Your apps
2. Add fingerprint under the Android app

---

## Security Notes

1. **Never commit** `local.properties`, `google-services.json`, or `release.keystore`
2. **Backup** your release keystore securely - losing it means you cannot update your app
3. **Rotate** secrets if compromised
4. OAuth client IDs are safe to commit (validated by SHA-1 + package name)
5. Debug keystore is intentionally public (standard credentials)
