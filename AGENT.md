# Obtainium Tarball APK Support — Agent Notes

## Context

User (Paul Swenson) requested support for Android APK releases distributed inside
tarball archives (`.tar.gz`, `.tgz`, `.tar.bz2`, `.tar.xz`).

Example app: **Plezy** (`https://github.com/edde746/plezy`) — releases Android
APKs as architecture-specific `.tar.gz` files that Obtainium previously could
not detect or install.

## Problem Analysis

Obtainium's GitHub source only recognized `.apk`, `.xapk`, and optionally `.zip`
files as valid release assets. Plezy releases look like this:

- `plezy-android-arm64-v8a.tar.gz`
- `plezy-android-armeabi-v7a.tar.gz`
- `plezy-android-x86_64.tar.gz`

The `.split('.').last` extension check returned `gz` for `.tar.gz`, so these
files were completely ignored. Even if recognized, Obtainium had no extraction
logic for tarballs — only ZIP/XAPK extraction via `flutter_archive`.

## Changes Made

### 1. `pubspec.yaml`

Added `archive: ^3.6.0` — a pure-Dart package for extracting `.tar.gz`,
`.tar.bz2`, and `.tar.xz` archives without shelling out to system binaries.

### 2. `lib/providers/source_provider.dart`

- Added `bool allowIncludeTarballs = false` to the `AppSource` base class
  (mirrors existing `allowIncludeZips` pattern).
- Extended `combinedAppSpecificSettingFormItems` to inject two new UI controls
  when `allowIncludeTarballs == true`:
  - `includeTarballs` toggle (default: false)
  - `tarballedApkFilterRegEx` text field for filtering APKs inside tarballs

### 3. `lib/app_sources/github.dart`

- Set `allowIncludeTarballs = true` in the GitHub constructor.
- Added `isApkContainer()` helper that properly checks filename suffixes:
  - `.apk`, `.xapk`
  - `.zip` (when `includeZips` enabled)
  - `.tar.gz`, `.tgz`, `.tar.bz2`, `.tar.xz` (when `includeTarballs` enabled)
- Replaced two inline `.split('.').last` checks with calls to `isApkContainer()`.
- Added `bool includeTarballs = additionalSettings['includeTarballs'] == true;`
  alongside the existing `includeZips` variable.

### 4. `lib/providers/apps_provider.dart`

- Added `archive` import with `as archive` prefix to avoid `ZipFile` naming
  collision with `flutter_archive`.
- Added `TARBALL` to `DownloadedDirType` enum.
- Added `extractTarballFile()` method using `archive` package:
  - Detects compression type from file extension
  - Decompresses with `GZipDecoder`, `BZip2Decoder`, or `XZDecoder`
  - Decodes TAR structure with `TarDecoder`
  - Writes each file to the destination directory
- Updated `downloadApp()`:
  - Detects tarballs by the **original asset name** (not the downloaded file
    path, since `downloadFile()` infers extension from `Content-Disposition`
    and gets `gz` instead of `tar.gz`).
  - Routes tarball downloads to `extractTarballFile()` instead of `unzipFile()`.
  - Uses `tarballedApkFilterRegEx` for tarball downloads and
    `zippedApkFilterRegEx` for ZIP/XAPK downloads.
  - Scans extracted directory **recursively** for APKs (tarballs may nest
    files inside subdirectories).
  - Returns `DownloadedDirType.TARBALL` for tarball containers.
- `installApkDir()` already handles any `DownloadedDirType` generically by
  scanning recursively for `.apk` and `.obb` files, so no changes were needed
  there.

### 5. `assets/translations/en.json`

Added English translation strings:
- `"includeTarballs"`: `"Include tarball files (tar.gz, etc.)"`
- `"tarballedApkFilterRegEx"`: `"Filter APKs inside tarball"`

Other locales will fall back to English until community translations are added.

## How to Use (for Plezy)

1. Open Obtainium → Add App
2. Enter `https://github.com/edde746/plezy`
3. In the app's settings, enable **"Include tarball files (tar.gz, etc.)"**
4. Obtainium will now detect the `plezy-android-*.tar.gz` assets
5. Optionally set **"Filter APKs inside tarball"** regex to match your
   architecture (e.g. `arm64-v8a`)
6. Save and update/install as normal

## Build Notes

The project requires Flutter >=3.38.0 (per `pubspec.yaml`). The repo's own
`docker/Dockerfile` specifies Flutter 3.29.3, which is too old for the current
pubspec constraints.

APK was built successfully using:
```bash
docker run --rm -v "${PWD}:/project" -w /project \
  ghcr.io/cirruslabs/flutter:3.41.0 \
  bash -c "flutter pub get && flutter build apk --release"
```

Note: Flutter's build tool reports "Gradle build failed to produce an .apk file"
because this project uses build flavors (`normal` and `fdroid`). The APKs are
actually generated at:
- `build/app/outputs/flutter-apk/app-normal-release.apk` ← standard release
- `build/app/outputs/flutter-apk/app-fdroid-release.apk` ← F-Droid flavor

The signed 64MB normal-release APK has been copied to the working directory.

## Files Modified

- `pubspec.yaml`
- `assets/translations/en.json`
- `lib/providers/source_provider.dart`
- `lib/app_sources/github.dart`
- `lib/providers/apps_provider.dart`

Branch: `feature/tarball-apk-support`
Pushed to: `https://github.com/pdscomp/Obtainium/tree/feature/tarball-apk-support`
