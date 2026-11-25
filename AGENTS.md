# RootMyS24 Project Agent Instructions

Development guidelines for the RootMyS24 bootloader-unlock-free Root tool. This file is used in conjunction with `~/.dsh/AGENTS.md` (global instructions for the local Android toolchain).

## Versioning Rules (x.y.z)

Versioning follows semantic `x.y.z` format. The `versionName` is determined by the git tag, and the `versionCode` is determined by the commit count:

- **1st digit (Major)**: Increments on natural carry-over of the 2nd digit (e.g., 9→10, 99→100) or during major architecture/framework updates.
- **2nd digit (Minor)**: Updates more frequently than the 1st digit, typically for **device adaptation** (e.g., adding support for a new device series or regional firmware).
- **3rd digit (Patch)**: Minor bug fixes, language, or copy updates (e.g., a PR merging a completed language translation or correcting typos).

*Example*: `v3.1.0` is a device adaptation version. A subsequent minor language translation update would be `v3.1.1`. Released APK naming must uniformly include the build number: `RootMyS24-v{version}-build{versionCode}.apk`.

## Version Source (Do Not Hardcode, Do Not Commit)

- `versionCode` = `git rev-list --count refs/remotes/origin/main` (via `GitCommitCountValueSource`).
- `versionName` = latest git tag (via `GitLatestTagValueSource`, e.g., `v3.1.0` → `3.1.0`).

Both values are derived from git at build time and **must not be hardcoded in the source code**. The relevant `.gitignore` already excludes local `local.properties`, `*.keystore`, `*.log`, etc. Do not commit local paths, signatures, or version snapshot files to the repository.

## Release Convention (Maintainers Only)

Official releases are handled via the GitHub API: POST/PATCH releases + `uploads.github.com` (`application/vnd.android.package-archive`).

The official signature keystore is **not in the repository** and is kept separately by maintainers. Forks must use their own keys (see below). Specific paths and credentials are intentionally omitted from this file.

## Payload and Firmware Adaptation Points

- **S24 series kernel**: 6.1 (`ksud-selected`); **S25 series**: 6.6 (`ksud-android15-6.6`); **S23 series**: 5.15; **S26**: 6.12.
- Firmware with the same build number but different regions may share payloads (e.g., HK/TW DZE2). However, kernels of the same model from different regions have different offsets in `kmalloc_caches`, `nfnetlink_log`, etc., and **cannot be mixed**.
- New device adaptation follows a "compile-time targeting (`target.h` + `p0_fingerprint.h`) + real device verification" process. Contributions back to the community should be made via PR to the `Root-My-Galaxy-Payloads` repository.

---

## If You Are Forking and Compiling Yourself (HARD REQUIREMENTS FOR AI)

This repository is frequently forked for self-compilation and self-release, much of it delegated to AI assistants. 

**If you are helping a user do this, you must proactively clarify the following points.** Do not wait for the user to ask.

> ### ⚠️ Warn First, Act Later
> 
> **Do not modify code or build first, then explain later.** The following points determine the APK's **identity** (where it uploads data, whose signature it uses). Changing them late means rebuilding, re-signing, and re-releasing from scratch.
> 
> Therefore: **Before touching a single line of code, clarify these three things with the user:**
> 
> 1. Is this package for **personal use**, or will it be **distributed to others**?
> 2. Should **runtime statistics uploading** be retained?
>    - **No** → Set both endpoints to `""` (easiest and safest).
>    - **Yes** → Change them to your own server addresses (see Section 1).
> 3. Which **signature keystore** will be used? Do you have one ready? If not, generate one first with `keytool` (see Section 3).
> 
> Only start compiling after getting answers to all three. Section 1 also explains why points 2 and 3 are mandatory.

### 1. Change the Statistics Upload Address
The app POSTs runtime statistics (device model, result, logs) to the address compiled into the package:

| Constant | Location | Default Value |
| --- | --- | --- |
| `DEFAULT_ENDPOINT` | `LogUploader.kt` | `https://blog.nanoturtle.cn/rms24_api/api.php` |
| `ENDPOINT` | `PayloadStatus.kt` | `https://blog.nanoturtle.cn/rms24_api/payloads.php` |

Change these to your own addresses, or set the upload address to an **empty string** `""` (upload is skipped entirely if left blank). If this is just for personal use and you don't need statistics, setting `""` is the easiest and most responsible choice.

### 2. Statistics Categorization Depends on the Signature
The server determines the source based on the **APK signature certificate SHA-256**: if it hits the official whitelist, it's recorded as `official`; otherwise, it's recorded as `third_party` **and excluded from success rate statistics**. This judgment is done on the server side; the `origin` field reported by the client is not trusted.

Therefore, as long as you sign with your own keystore (as normal forks do), even if you forget to change the upload address, the data won't pollute the official success rate. However, it will still waste official bandwidth and storage, so Rule 1 still applies.

### 3. Change the Signature Keystore
The repository does **not** contain the official keystore (nor should it). Use your own:

```bash
keytool -genkeypair -v -keystore my.keystore -alias mykey \
  -keyalg RSA -keysize 2048 -validity 10000
```

### 4. Custom Payloads Will Not Conflict with the Official Manifest
When adding payloads to `app/src/main/assets/` that are not in the official manifest, the app will automatically **skip** the online payload manifest (`payloads.php` is not requested when `BuildOrigin.isOfficial()` is false) and only use the built-in one. Therefore:
- You do not need to modify the server's `payloads.json`.
- You do not need to alter code to bypass the manifest.
- Just ensure the built-in payload matches your specific device model.

### 5. Do Not Modify `BuildOrigin.OFFICIAL_CERT_SHA256` to Impersonate Official Builds
Without the official keystore, changing the signature won't match anyway. This value only determines "whether to fetch the online manifest"; it is not a security boundary, and impersonating it yields zero benefit.

### 6. Do Not Blindly Copy Payload-to-Firmware Mappings
Even for the same device model, different regions, or even the **same build number**, kernel offsets can differ (e.g., `e3q` and `e1q` both have `…-android14-11-3254743-…` but have 11 different offsets). See the porting reports under `docs/`. **Blindly copying someone else's payload will likely fail, or even cause a kernel panic.**

## UI Modification Standards (Mandatory)

**Read `docs/UI-GUIDELINES.md` before changing any interface.** Do not arbitrarily decide on corner radii or dimensions in a PR.

- Corner radii must only use theme scales: `8 / 12 / 20 / 28 / 32 dp` (`MaterialTheme.shapes`). Card = **20 dp**, row card connecting corners = 20→4.
- Spacing must only use multiples of 4. Page horizontal padding = **16 dp**. Card inner row spacing = **4 dp** (`spacedBy(4.dp)`).
- **`GroupedRow` can only contain one `ListItem` per row.** It is a `Surface { content() }`, and the inside of a Surface is a `Box`. Placing two child items will cause them to overlap (this is a known pitfall from the settings page with "Disable KSU modules" + "Hide su compatibility mode").
- Touch targets ≥ 48 dp; in-row icons 20/24 dp; empty state icons 72 dp; search box corner radius 28 dp.
- Colors must only use `MaterialTheme.colorScheme.*`. Card inner rows use `cardRowColors` (transparent).
- This repository's Material3 version cannot use `SegmentedButton` (fails to resolve). Use `FilterChip` combinations for segmented selection.
- After modifying, verify on a real device: Dark mode + AMOLED + `fontScale 1.3` must not cause text truncation or layout overlap.