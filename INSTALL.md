# Focusdoro Widget — Install Guide

Download installers from the [GitHub Releases](https://github.com/code-with-ahsan/focusdoro-widget/releases) page. All builds are produced by GitHub Actions from tagged commits (`widget-v*`) in the Focusdoro source repository and published here.

The Focusdoro Widget is a desktop Pomodoro companion that mirrors timer state from the Focusdoro web app. The widget's bundle identifier is `com.focusdoro.widget` — this value is **load-bearing** for keyring entries, macOS Login Items, notification permissions, and Windows MSI upgrade behavior. Do not install another `.app` or installer with the same identifier from a different source.

---

## macOS (Apple Silicon)

**Apple Silicon (M1/M2/M3/M4) only.** Intel Mac users are out of scope for v3.0. (Intel + Rosetta is not supported because the widget's Hardened Runtime + JIT entitlement combination interacts poorly with Rosetta translation.)

Steps:

1. Download `Focusdoro_Widget_<version>_aarch64.dmg` from the release page.
2. Double-click the `.dmg` to mount it.
3. Drag the `Focusdoro Widget.app` to `/Applications`.
4. Eject the disk image and launch the app from Applications (Spotlight: type "Focusdoro Widget").

The `.app` is code-signed with a Developer ID Application certificate and notarized by Apple, so first-launch should succeed without any Gatekeeper warning. If Gatekeeper does hesitate (rare — typically only when the stapled notarization ticket fails to download), right-click the app → **Open** → confirm.

### Integrity verification (recommended)

Each release publishes a `SHA256SUMS.txt` file alongside the installers. To verify the downloaded `.dmg`:

```bash
shasum -a 256 Focusdoro_Widget_<version>_aarch64.dmg
```

Compare the output against the matching line in `SHA256SUMS.txt`. If they do not match, do not mount the `.dmg` — re-download or report the discrepancy.

The bundle identifier `com.focusdoro.widget` is what Keychain (Phase 36 OAuth/refresh token storage), `tauri-plugin-autostart` Login Items (Phase 39 launch-at-login), notification permissions (TCC), and the `focusdoro://` deep-link scheme all key off. Do not install another `.app` with the same identifier from a different source — doing so will overwrite the canonical bundle and may invalidate prior keyring/permission state.

---

## Windows

Steps:

1. Download `Focusdoro_Widget_<version>_x64_en-US.msi` from the release page.
2. Double-click the `.msi` to start the installer.
3. Click through the WiX installer prompts. Default install location is `C:\Program Files\Focusdoro Widget\`.

### SmartScreen bypass (v3.0 only)

When Windows shows **"Windows protected your PC"**, click **"More info"** → **"Run anyway"**.

This is expected: v3.0 ships **unsigned** (the `.msi` has no Authenticode code-signing certificate attached), so SmartScreen has no publisher reputation to evaluate. v3.1 will add code signing once the EV/OV certificate procurement is decided; SmartScreen will stop warning at that point.

**Known limitation:** v3.0 ships unsigned by design; SmartScreen reputation requires an EV certificate which is deferred to v3.1 (tracked as `WDIST-05` in [`.planning/REQUIREMENTS.md`](../../.planning/REQUIREMENTS.md)). Until then, users who prefer to verify before running can use the SHA256 procedure below.

### Integrity verification (recommended)

Each release publishes a `SHA256SUMS.txt` file alongside the installers. To verify the downloaded `.msi` matches the published hash, open PowerShell and run:

```powershell
Get-FileHash .\Focusdoro_Widget_<version>_x64_en-US.msi -Algorithm SHA256
```

Compare the output against the matching line in `SHA256SUMS.txt`. If they do not match, do not run the installer — re-download or report the discrepancy.

### Upgrade note (v1.0.0 → v1.1.0+)

If upgrading from `v1.0.0` to `v1.1.0+` leaves you with **two** entries in Add/Remove Programs ("Focusdoro Widget" listed twice with different versions), uninstall the older one manually. This is a one-time WiX UpgradeCode-stability issue resolved by `v1.1.0+`; subsequent upgrades will replace in place. The widget's WiX `UpgradeCode` UUID is pinned (see `apps/widget/src-tauri/tauri.conf.json` `bundle.windows.wix.upgradeCode`) and **MUST NOT change** in any future release — changing it breaks in-place upgrades for every existing Windows user.

---

## Linux

Two formats are published per release: a `.deb` package for Debian/Ubuntu-family distributions, and an `.AppImage` for everything else.

### `.deb` (Ubuntu/Debian and derivatives)

```bash
wget https://github.com/code-with-ahsan/focusdoro-widget/releases/download/widget-v<version>/Focusdoro_Widget_<version>_amd64.deb
sudo dpkg -i Focusdoro_Widget_<version>_amd64.deb
```

If `dpkg` reports missing dependencies (typically `libayatana-appindicator3-1` on a fresh system without the universe/contrib repos enabled), run:

```bash
sudo apt --fix-broken install
```

This will pull the missing runtime libraries from your distribution's package archive and complete the install.

**Supported distributions** (verified to ship `libayatana-appindicator3-1` in their default archives):

- Ubuntu 22.04 LTS (Jammy Jellyfish)
- Ubuntu 24.04 LTS (Noble Numbat)
- Debian 12 (Bookworm)
- Linux Mint 21+
- Pop!_OS 22.04+

**Older or unsupported distributions** — Ubuntu 20.04 (Focal Fossa) and Debian 11 (Bullseye) ship the older `libappindicator3-1` (note: no `-ayatana-` prefix) instead of `libayatana-appindicator3-1`. The `.deb`'s `Depends:` declaration will fail there. Use the `.AppImage` instead.

### `.AppImage` (universal Linux fallback)

```bash
wget https://github.com/code-with-ahsan/focusdoro-widget/releases/download/widget-v<version>/Focusdoro_Widget_<version>_amd64.AppImage
chmod +x Focusdoro_Widget_<version>_amd64.AppImage
./Focusdoro_Widget_<version>_amd64.AppImage
```

No install is required — the AppImage is a self-contained, double-clickable bundle (once the executable bit is set). It runs on any glibc-2.31+ distribution with FUSE available.

### Integrity verification (recommended)

Each release publishes a `SHA256SUMS.txt` file alongside the installers. To verify the downloaded `.deb` or `.AppImage`:

```bash
sha256sum Focusdoro_Widget_<version>_amd64.deb
sha256sum Focusdoro_Widget_<version>_amd64.AppImage
```

Compare each output against the matching line in `SHA256SUMS.txt`. If they do not match, do not install — re-download or report the discrepancy.

---

## Manual smoke procedure (maintainer)

After the first CI-signed `.dmg` is published, run these two smokes against the signed bundle. They cannot be exercised against dev binaries because dev binaries lack bundle identity — `tauri-plugin-autostart` Login Item registration and macOS TCC permission attribution both key off the signed bundle's notarized identity, not the unsigned dev binary's path.

These are the Phase 39 deferred smokes (D-15 reboot + D-09 tccutil) folded into Phase 40 per D-22.

### Smoke 1 — Autostart Login Item registration + reboot survival

1. Install the signed `.app` to `/Applications` (drag from the mounted `.dmg`).
2. Launch the app and sign in.
3. Open the settings panel and enable the **"Launch at login"** toggle.
4. Reboot the Mac.
5. After login completes, verify that:
   - The tray icon appears in the menu bar within ~5 seconds of login.
   - The login window does **NOT** appear (the autostart path must hide the login window per the Phase 39 D-15 contract).

If the login window appears on the autostart path, the bundled `tauri-plugin-autostart` is not invoking the silent-launch flag correctly against the signed bundle — capture as a Phase 40 blocker.

### Smoke 2 — TCC notification re-prompt

1. Open Terminal.app and run:

   ```bash
   tccutil reset Notifications com.focusdoro.widget
   ```

2. Relaunch the Focusdoro Widget app.
3. In the settings panel, toggle Notifications **OFF**, then back **ON**.
4. Verify that the macOS notification permission prompt fires.
5. Click **Allow**.
6. Verify that the cached permission state inside the app refreshes (no app restart required — the next test notification fires immediately).

If the prompt does not fire after the `tccutil reset`, TCC is failing to attribute the permission request to the signed bundle's `com.focusdoro.widget` identity — capture as a Phase 40 blocker.

Both smokes are captured pass/fail in the Plan 05 verification checkpoint task. Cross-reference: `.planning/phases/40-distribution-signing-ci/40-05-PLAN.md`.

---

## Maintainer notes

This section is operational documentation for the maintainer. End users do not need to read it.

### Bundle identifier (LOCKED per D-24)

`com.focusdoro.widget` MUST NOT change. Changing it invalidates:

- Keyring entries (Phase 36 WAUTH-03 — Google OAuth refresh tokens, magic-link state)
- macOS Login Items (Phase 39 D-12..D-15 — `tauri-plugin-autostart` registration)
- TCC permissions (Phase 39 D-09..D-11 — notification grants)
- `focusdoro://` deep-link scheme registration (Phase 36 D-08)
- Windows MSI UpgradeCode derivation (relative-to-identifier, used as a stable upgrade key)

This is the single most important invariant in the codebase. Phase-checker grep gates assert it on every `tauri.conf.json` diff.

### WiX UpgradeCode (LOCKED per RESEARCH §R-3)

The UUID in `apps/widget/src-tauri/tauri.conf.json` `bundle.windows.wix.upgradeCode` MUST NOT change after v1.0.0 ships. Current value (set in Phase 40 Plan 01): `429BA08E-7BB9-4280-821D-72125C60E290`.

Changing this UUID will:

1. Break in-place upgrades for every existing Windows user. The new `.msi` will install as a parallel product instead of upgrading; the user will see two Add/Remove Programs entries and have to uninstall the older one manually.
2. Reset all Windows user state that the WiX installer migrates (Start Menu shortcuts, file associations if any).

The UpgradeCode is load-bearing forever. Do not regenerate it for any reason.

### Apple API key rotation (RESEARCH §R-4)

To rotate the App Store Connect API key (`.p8`):

1. **Revoke the existing key.** In App Store Connect → **Users and Access** → **Integrations** → **Team Keys**, locate the existing key and click **Revoke**.
2. **Generate a new key.** Click **Generate API Key**, name it (e.g. `focusdoro-ci-2026Q2`), grant **Developer** role (sufficient for notarization), and click **Generate**. Download the `.p8` file. **You can only download the `.p8` ONCE — it cannot be re-downloaded.** Save it securely outside the repo.
3. **Base64-encode the .p8 locally.** On macOS:

   ```bash
   base64 -i AuthKey_<NEW_KEY_ID>.p8 | pbcopy
   ```

   On Linux:

   ```bash
   base64 -w 0 AuthKey_<NEW_KEY_ID>.p8
   ```

   (The `-w 0` flag prevents line wrapping; the resulting base64 is a single line ready to paste into a GitHub Secret.)
4. **Update GitHub Secrets.** In the repo's **Settings** → **Secrets and variables** → **Actions**:
   - Update `APPLE_API_KEY` to the new **Key ID** string (10-character alphanumeric shown next to the new key, e.g. `A1B2C3D4E5`).
   - Update `APPLE_API_KEY_BASE64` to the base64 contents you copied above.
   - Leave `APPLE_API_ISSUER` unchanged (it is per-team, not per-key).
5. **Push the next `widget-v*` tag.** CI will use the new key automatically — no code change required.

If the previous `.p8` file is still on disk, securely delete it (`rm -P AuthKey_<OLD>.p8` on macOS, `shred -u AuthKey_<OLD>.p8` on Linux).

### Required GitHub Secrets (exactly 10 mandatory)

Per CONTEXT.md D-18 ERRATA + ERRATA-2:

1. **`APPLE_CERTIFICATE`** — base64 of the Developer ID Application `.p12`
2. **`APPLE_CERTIFICATE_PASSWORD`** — password for the `.p12`
3. **`APPLE_SIGNING_IDENTITY`** — full identity string, e.g. `Developer ID Application: <Name> (TEAMID)`
4. **`APPLE_API_KEY`** — App Store Connect API Key ID string (10-character alphanumeric)
5. **`APPLE_API_ISSUER`** — App Store Connect Issuer UUID
6. **`APPLE_API_KEY_BASE64`** — base64 of the `AuthKey_<KEYID>.p8` file contents
7. **`GOOGLE_DESKTOP_OAUTH_CLIENT_ID`** — Phase 36 OAuth client ID (build-time inject)
8. **`GOOGLE_DESKTOP_OAUTH_CLIENT_SECRET`** — Phase 36 OAuth client secret (build-time inject)
9. **`FIREBASE_WEB_API_KEY`** — Phase 36 magic-link Web API key (build-time inject)
10. **`FIREBASE_DATABASE_URL`** — Phase 37 RTDB base URL (build-time inject via `option_env!` in `apps/widget/src-tauri/src/timer/rtdb.rs`). Per D-18 ERRATA-2.

Optional 10+1: **`APPLE_TEAM_ID`** (belt-and-suspenders). `tauri-action` infers it from the `APPLE_SIGNING_IDENTITY` suffix `(TEAMID)` when absent; the explicit secret avoids parsing assumptions but its absence is fine.

All 10 mandatory secrets are verified by the Plan 04 pre-flight checkpoint task.

---

*Last revised: 2026-06-04 (Phase 40 Plan 03).*
