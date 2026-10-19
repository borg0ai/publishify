# Profile: Wails app

Load when the repo has `wails.json` or a Wails v3 `Taskfile` build. Rules below are for macOS DMG releases; other platforms are unverified.

## Version files

- Wails config, frontend `package.json`, and any Info.plist or generated version file the repo defines: unverified list, read the repo first.
- Add a repo script with two commands: `read <sha>` returns the synchronized version, `compare <ref>` exits non-zero unless the version is newer than that ref. Promotion, tagging and release validation all call it.

## Build and package

- One build job per architecture (arm64 and amd64) on a matching runner; each produces its own DMG.
- Check each built binary's architecture with `lipo -archs` against the matrix value before signing.
- Name assets `<App>-v<version>-macos-<arch>.dmg`; the verify job compares the exact asset list.

## Signing and notarization

- Identity must be `Developer ID Application: ... (<TEAMID>)` for the configured Team ID.
- Store notary credentials with `xcrun notarytool store-credentials`, prove them with `notarytool history`, then sign and notarize the app (`wails3 tool sign ... --notarize`).
- After signing: `codesign --verify --deep --strict`, and confirm `TeamIdentifier` equals the Team ID. Fail the job on any mismatch.

## Updater and manifest

- None verified. Do not add an updater stage unless the app has one; confirm against current Wails docs first.

## Distribution

- macOS app: cask in `luban-homebrew`, one URL and checksum per architecture, opened after the release is public and the exact asset list is verified.

## Traps

- `wails3` tooling must be installed in the job before any task runs.
- A DMG-only release must verify that no other assets were attached.
