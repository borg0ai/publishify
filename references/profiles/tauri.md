# Profile: Tauri app (with Rust CLI crates)

Load when the repo has `tauri.conf.json` or `src-tauri/`. A repo that also ships a Rust CLI crate adds a CLI build job to the same release workflow.

## Version files

Keep these in sync through the version tool's `extra-files`, and add a repo script that checks they agree:

- Workspace `Cargo.toml` (`$.workspace.package.version`) and each shipped crate's `Cargo.toml` (`$.package.version`).
- App `package.json` and `tauri.conf.json` (`$.version`, type `json`).
- `Cargo.lock`: one selector per local package, `$.package[?(@.name.value=='<crate>')].version` (Release Please v17.3.0 tagged TOML; see `versioning-and-bumping.md` section 6).

## Build and package

- One build job per target. Run `tauri build` with `--config '{"bundle":{"createUpdaterArtifacts":true}}'` on release builds only; some CLI versions ignore a `TAURI_CONFIG` environment variable, so build once with a throwaway key to confirm.
- Linux runners need the WebKitGTK and tray build dependencies for the pinned Tauri version.
- A CLI crate is a separate job: `cargo build --release --target <triple>`, archive per target, checksums.
- Jobs that do not install dependencies set `package-manager-cache: false` on `setup-node`.

## Signing and notarization

- macOS needs a Developer ID Application certificate and App Store Connect API key; map secret names to the variables Tauri reads (see `desktop-cli-release.md`).
- Preflight before building: fail on missing or empty secrets, import the certificate into a temporary keychain, check the identity is present, and prove `xcrun notarytool history` authenticates.

## Updater and manifest

- Archive plus `.sig`: macOS `.app.tar.gz`, Linux AppImage. `tauri build` does not write `latest.json`; generate and validate it in the publish job from the verified assets.
- The CLI warns when the signing key does not match the configured public key; treat that as an error.
- Windows updater artifacts: unverified.

## Distribution

- macOS app: cask in `luban-homebrew` pointing at the verified DMG or archive. Rust CLI: formula.

## Traps

- `tauri build` success is not proof the updater signature exists; assert the `.sig` file.
- Do not publish `latest.json` before every platform archive is verified.
