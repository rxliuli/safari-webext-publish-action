# Safari WebExtension Publish Action

Sign, upload, and submit Safari web extensions (macOS + iOS) to App Store Connect via GitHub Actions.

Auto-detects platforms from the Xcode project schemes — if your project only has a macOS scheme, iOS steps are skipped automatically (and vice versa).

## Modes

| Mode | Runner | What it does |
|------|--------|-------------|
| `build` (default) | `macos-26` | Build, sign, and upload the binary to App Store Connect |
| `submit` | `ubuntu-latest` | Create/update the App Store version and submit for review via API |

Split into two jobs so the expensive macOS runner only handles the build, while the review submission runs on a cheap Linux runner.

## Prerequisites

- **Apple Developer account** with:
  - App IDs registered for your bundle identifier (app + extension)
  - Provisioning profiles (Mac App Store / App Store Connect)
  - Signing certificates (3rd Party Mac Developer Application + Installer, Apple Distribution for iOS)
  - App Store Connect API key

## Usage

### Build + Submit (recommended)

```yaml
safari:
  needs: version
  runs-on: macos-26
  steps:
    - uses: actions/checkout@v5
    - uses: pnpm/action-setup@v6
    - uses: actions/setup-node@v6
      with:
        node-version: 24
        cache: 'pnpm'
    - run: pnpm install
    - run: pnpm build:safari

    - uses: rxliuli/safari-webext-publish-action@v2
      with:
        mode: build
        project-path: '.output/My Extension'
        bundle-identifier: 'com.example.my-extension'
        team-id: 'ABCDE12345'
        app-signing-identity: '3rd Party Mac Developer Application: Your Name (ABCDE12345)'
        installer-signing-identity: '3rd Party Mac Developer Installer: Your Name (ABCDE12345)'
      env:
        APPLE_CERTIFICATE_BASE64: ${{ secrets.APPLE_CERTIFICATE_BASE64 }}
        APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
        APPLE_MACOS_PROVISIONING_PROFILE_BASE64: ${{ secrets.APPLE_MACOS_PROVISIONING_PROFILE_BASE64 }}
        APPLE_MACOS_EXTENSION_PROVISIONING_PROFILE_BASE64: ${{ secrets.APPLE_MACOS_EXTENSION_PROVISIONING_PROFILE_BASE64 }}
        APPLE_IOS_PROVISIONING_PROFILE_BASE64: ${{ secrets.APPLE_IOS_PROVISIONING_PROFILE_BASE64 }}
        APPLE_IOS_EXTENSION_PROVISIONING_PROFILE_BASE64: ${{ secrets.APPLE_IOS_EXTENSION_PROVISIONING_PROFILE_BASE64 }}
        APPLE_API_KEY: ${{ secrets.APPLE_API_KEY }}
        APPLE_API_KEY_ID: ${{ secrets.APPLE_API_KEY_ID }}
        APPLE_API_ISSUER: ${{ secrets.APPLE_API_ISSUER }}

safari-submit:
  needs: [version, safari]
  runs-on: ubuntu-latest
  steps:
    - uses: rxliuli/safari-webext-publish-action@v2
      with:
        mode: submit
        bundle-identifier: 'com.example.my-extension'
        version: ${{ needs.version.outputs.version }}
      env:
        APPLE_API_KEY: ${{ secrets.APPLE_API_KEY }}
        APPLE_API_KEY_ID: ${{ secrets.APPLE_API_KEY_ID }}
        APPLE_API_ISSUER: ${{ secrets.APPLE_API_ISSUER }}
```

### Build only (v1 compatible)

If you only need to upload the binary without auto-submitting for review, use just the `build` mode (or omit `mode` entirely — `build` is the default):

```yaml
safari:
  runs-on: macos-26
  steps:
    # ... checkout, setup, build steps ...
    - uses: rxliuli/safari-webext-publish-action@v2
      with:
        project-path: '.output/My Extension'
        bundle-identifier: 'com.example.my-extension'
        team-id: 'ABCDE12345'
        app-signing-identity: '3rd Party Mac Developer Application: Your Name (ABCDE12345)'
        installer-signing-identity: '3rd Party Mac Developer Installer: Your Name (ABCDE12345)'
      env:
        APPLE_CERTIFICATE_BASE64: ${{ secrets.APPLE_CERTIFICATE_BASE64 }}
        APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
        APPLE_MACOS_PROVISIONING_PROFILE_BASE64: ${{ secrets.APPLE_MACOS_PROVISIONING_PROFILE_BASE64 }}
        APPLE_MACOS_EXTENSION_PROVISIONING_PROFILE_BASE64: ${{ secrets.APPLE_MACOS_EXTENSION_PROVISIONING_PROFILE_BASE64 }}
        APPLE_IOS_PROVISIONING_PROFILE_BASE64: ${{ secrets.APPLE_IOS_PROVISIONING_PROFILE_BASE64 }}
        APPLE_IOS_EXTENSION_PROVISIONING_PROFILE_BASE64: ${{ secrets.APPLE_IOS_EXTENSION_PROVISIONING_PROFILE_BASE64 }}
        APPLE_API_KEY: ${{ secrets.APPLE_API_KEY }}
        APPLE_API_KEY_ID: ${{ secrets.APPLE_API_KEY_ID }}
        APPLE_API_ISSUER: ${{ secrets.APPLE_API_ISSUER }}
```

## Inputs

### Common

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `mode` | No | `build` | `build` or `submit` |
| `bundle-identifier` | Yes | — | Bundle identifier (e.g. `com.example.my-extension`) |

### Build mode

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `project-path` | Yes | — | Path to the Xcode project directory (e.g. `.output/My Extension`) |
| `team-id` | Yes | — | Apple Developer Team ID |
| `app-signing-identity` | Yes | — | Code signing identity (e.g. `3rd Party Mac Developer Application: Name (TeamID)`) |
| `installer-signing-identity` | Yes | — | Installer signing identity (e.g. `3rd Party Mac Developer Installer: Name (TeamID)`) |
| `macos-deployment-target` | No | `12.0` | Minimum macOS deployment target |

### Submit mode

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `version` | Yes | — | App version string (e.g. `1.2.3`) |
| `release-notes` | No | `Bug fixes and improvements.` | What's New text for the App Store listing |

## Environment Variables

All secrets are passed as environment variables to keep parity with tools like `wxt submit`.

### Build mode

| Variable | Description |
|----------|-------------|
| `APPLE_CERTIFICATE_BASE64` | Base64-encoded `.p12` certificate containing signing identities |
| `APPLE_CERTIFICATE_PASSWORD` | Password for the `.p12` certificate |
| `APPLE_MACOS_PROVISIONING_PROFILE_BASE64` | Base64-encoded macOS app provisioning profile |
| `APPLE_MACOS_EXTENSION_PROVISIONING_PROFILE_BASE64` | Base64-encoded macOS extension provisioning profile |
| `APPLE_IOS_PROVISIONING_PROFILE_BASE64` | Base64-encoded iOS app provisioning profile (`.mobileprovision`) |
| `APPLE_IOS_EXTENSION_PROVISIONING_PROFILE_BASE64` | Base64-encoded iOS extension provisioning profile (`.mobileprovision`) |
| `APPLE_API_KEY` | Base64-encoded App Store Connect API key (`.p8` file) |
| `APPLE_API_KEY_ID` | App Store Connect API key ID |
| `APPLE_API_ISSUER` | App Store Connect API issuer ID |

### Submit mode

| Variable | Description |
|----------|-------------|
| `APPLE_API_KEY` | Base64-encoded App Store Connect API key (`.p8` file) |
| `APPLE_API_KEY_ID` | App Store Connect API key ID |
| `APPLE_API_ISSUER` | App Store Connect API issuer ID |

## How It Works

### Build mode

1. **Check macOS version** — Ensures the runner has macOS 26+ for Xcode 26 SDK
2. **Detect platforms** — Reads Xcode project schemes to determine macOS/iOS/both
3. **Setup certificates** — Imports `.p12` into a temporary keychain
4. **Install provisioning profiles** — Decodes and installs profiles for detected platforms
5. **Create entitlements** — Generates sandbox entitlements with app-identifier and team-identifier
6. **Build archives** — Runs `xcodebuild archive` without signing (macOS: universal binary x86_64+arm64)
7. **Sign & package** — macOS: manual `codesign` + `productbuild`; iOS: `xcodebuild -exportArchive`
8. **Upload** — Uses `xcrun altool` to upload to App Store Connect
9. **Cleanup** — Deletes temporary keychain and profiles

### Submit mode

1. **Authenticate** — Generates a JWT using the App Store Connect API key (ES256)
2. **Find app** — Looks up the app by bundle identifier
3. **Wait for builds** — Polls until builds for the specified version are processed
4. **Manage version** — Creates a new App Store version or updates an existing draft
5. **Attach build** — Links the processed build to the version
6. **Set release notes** — Updates the What's New text via `appStoreVersionLocalizations`
7. **Submit for review** — Creates (or reuses) a review submission and confirms it

## Preparing Secrets

### Certificate (.p12)

Export your signing identities from Keychain Access:
- "3rd Party Mac Developer Application" certificate
- "3rd Party Mac Developer Installer" certificate
- "Apple Distribution" certificate (for iOS)

Export all as a single `.p12` file, then:

```bash
base64 -i Certificates.p12 | pbcopy
# Paste as APPLE_CERTIFICATE_BASE64
```

### Provisioning Profiles

Create profiles at [Apple Developer Portal](https://developer.apple.com/account/resources/profiles/list):
- **Mac App Store Connect** profile for your app bundle ID
- **Mac App Store Connect** profile for your extension bundle ID (`.Extension` suffix)
- **App Store Connect** (iOS) profile for your app bundle ID
- **App Store Connect** (iOS) profile for your extension bundle ID

Then:

```bash
base64 -i MyApp_macOS.provisionprofile | pbcopy
# Paste as APPLE_MACOS_PROVISIONING_PROFILE_BASE64
```

### App Store Connect API Key

Create at [App Store Connect > Users and Access > Integrations > Keys](https://appstoreconnect.apple.com/access/integrations/api).

```bash
base64 -i AuthKey_XXXXXXXXXX.p8 | pbcopy
# Paste as APPLE_API_KEY
```

## License

MIT
