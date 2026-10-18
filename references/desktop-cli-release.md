# Desktop and CLI Release Guide

Use this reference when repository ships a command-line tool and a desktop app, or when release behavior depends on framework.

## Separate deliverables

Treat CLI and GUI as separate release products even when they share one version and GitHub Release:

- **CLI**: publish executable/archive per supported OS and architecture. Verify executable permissions, runtime dependencies, stable asset names, checksums, and install command.
- **Desktop app**: publish the platform installer or bundle users can install. Verify app bundle contents, native dependencies, signing, notarization, and platform-specific updater assets.
- **Shared release**: use one immutable tag/commit only when repository version policy says both products release together. Otherwise, preserve separate version/tag conventions. Upload all expected public artifacts to GitHub Release so users can download without authentication.
- Do not treat a GitHub Release asset as an auto-update feed by itself. Confirm the app has a configured provider/endpoint, correctly versioned metadata, signature verification, and matching client behavior.

## Framework adapter checks

Detect framework and version from project manifests/configuration. Then read its current official release, packaging, signing, and updater documentation before changing workflows. Framework feature support changes; this list is a map for investigation, not a substitute for version-specific verification.

| Framework | Verify before changing CI/CD |
| --- | --- |
| Tauri | Bundle targets and platform runners; updater plugin/configuration; required updater signing key and emitted signature/manifest files; GitHub Release endpoint; OS code signing and notarization as separate requirements. Start with [Tauri distribution](https://v2.tauri.app/distribute/) and [Tauri updater](https://v2.tauri.app/plugin/updater/). |
| Electron | Actual packager/updater pair, commonly electron-builder plus electron-updater; OS-specific target formats; generated metadata alongside assets; GitHub provider/release visibility; macOS signing requirement for auto-update; Windows signing and update verification. Start with [electron-builder auto update](https://www.electron.build/features/auto-update/) and [publish configuration](https://www.electron.build/docs/publish/). |
| Velox | Verify the exact Velox project/version, host OS and target matrix, bundle outputs, signing/notarization steps, and whether an updater exists for that release. Do not assume Tauri's plugin or updater contract carries over. Start with the [Velox project documentation](https://github.com/velox-apps/velox). |
| Wails 3 | Verify exact Wails version, platform build tasks, bundle/archive format, updater provider, GitHub asset naming/matching rules, manifest/signature verification, and code-signing requirements. Wails v3 documentation evolves; use its current [updater docs](https://v3.wails.io/tutorials/04-self-update-a-wails-app/) and [FAQ](https://v3.wails.io/faq/). |
| Other framework | Find its official packaging and updater docs. If no supported updater contract exists, ship downloadable installers and state that auto-update is not implemented. |

Never infer that a framework supports macOS, Linux, Windows, or a specific architecture from its name. Confirm repository code, framework version, official docs, and real artifact output.

## Platform artifact and trust checks

- **macOS**: publish installable app format such as DMG/PKG/ZIP when project supports it. For direct distribution outside App Store, verify Developer ID code signature and notarization/stapling according to Apple's and framework's current requirements. If updater uses a separate archive or signature, verify it too. A signed updater archive does not prove the DMG is notarized.
- **Linux**: publish only formats project supports (for example AppImage, DEB, RPM, or tar archive). Record required system libraries and architecture. Verify updater behavior per format; package-manager formats may require elevated install and may not support silent self-update.
- **Windows**: publish project-supported installer/archive (for example MSI, NSIS/EXE, or ZIP). Verify Windows signing if product promises trusted publisher identity. Verify update metadata and permissions on a clean Windows runner/device.
- Test fresh installation and upgrade from previous release separately for each supported OS/architecture. Verify checksums and signatures against downloaded release assets, not only CI outputs.

## Homebrew and Buzz.app

Homebrew package update and app self-update are separate mechanisms. Inspect the current tap repository and Buzz.app implementation to learn which package types, architectures, and OS targets Buzz can install. Never assume a formula or cask automatically means Buzz supports that artifact.

- **CLI**: normally represent as a formula with platform-specific URLs/bottles only where supported. Validate formula against the tap's conventions and artifact matrix.
- **GUI on macOS**: a cask may install an app bundle from a supported archive/DMG/PKG. Match Buzz's cask support and expected app path.
- **Linux**: Homebrew runs on Linux; formulas commonly serve CLI tools. Use casks for GUI only when both the tap and Buzz support that Linux cask artifact/install behavior. Otherwise expose the GitHub Release package through a supported Linux installation path.
- **Windows**: Homebrew runs under WSL 2, not as native Windows package management. Add Windows to Buzz's Homebrew flow only if Buzz explicitly supports WSL/Homebrew integration and the package has been installed and verified there. Otherwise publish the Windows artifact on GitHub Releases and use Buzz's supported Windows installer source, if one exists.
- For `luban-homebrew`, target `develop` unless the user specifies another base branch. If the target branch is missing, stop and ask whether to create it from the verified tap default branch; never silently fall back to `main`. For each tap PR, verify release URL is public and immutable, checksum matches downloaded asset, formula/cask syntax passes tap checks, and PR base matches the requested target. Never put credentials or mutable local paths in a formula/cask.

Homebrew platform guidance: [Homebrew on Linux](https://docs.brew.sh/Homebrew-on-Linux), [Adding Software to Homebrew](https://docs.brew.sh/Adding-Software-to-Homebrew).
