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

## Guided Apple signing and notarization setup

Use this procedure when a repository distributes a macOS app or CLI outside the Mac App Store and needs Developer ID signing and notarization. Guide the repository owner from discovery through a verified release run. Do not stop at listing secret names or hand them a generic secret checklist. Adapt the names, file conversion, runner, signing commands, and updater signing requirements to the repository's actual framework and workflow.

**Read the official Apple pages before writing setup instructions.** Open the pages linked below during the current task; do not rely on their titles, old notes, or memory. Read only the pages relevant to this repository's chosen certificate and notarization method, then check the framework's current signing docs too. Explain the steps in plain language the repository owner can follow: what they need before starting, where to click, what Apple will give them, which exact GitHub field gets each item, and how they can tell setup worked. Define terms such as CSR, private key, Team ID, and notarization the first time they appear. Keep Apple's UI labels exact, but explain what each one means in everyday words. Do not merely paste links or repeat Apple's technical language. If a page is unavailable or does not answer a detail, say so and verify another official source before giving that step.

Write the final repo-specific guide as a sequence the owner can follow without already knowing Apple signing: **Before you start → Get signing certificate → Get notarization login → Add GitHub secrets → Run and check release**. At each step use short, direct sentences: what to do, what result to look for, and what to do with that result. Use the actual repository's secret names and workflow jobs, not the examples below. If a choice depends on whether this repo makes a DMG, PKG, or updater archive, inspect its packaging config and tell the owner which choice applies here.

### 1. Discover this repository's exact contract

Before asking the owner to create credentials, inspect the app identifier, package format (DMG, PKG, ZIP, or app bundle), CI framework/version, macOS runner, release workflow, and its signing/notarization inputs. Read current official framework docs. Decide whether the output needs a **Developer ID Application** certificate (apps, binaries, DMGs) and whether a **Developer ID Installer** certificate is also needed (PKG installer signing). Identify whether notarization authenticates through an App Store Connect API key or Apple ID plus app-specific password. Prefer API-key authentication when the framework supports it, and use the framework's documented names and file format. Keep updater private keys separate from Apple credentials.

Produce a repo-specific setup checklist that says, for every value: exact GitHub secret/variable name, what it contains, which Apple page supplies it, which job consumes it, and how the workflow validates it without printing it. Do not ask the owner to guess names. If an agent has permission to inspect GitHub, check environment and secret **names** with `gh`; GitHub never reveals saved secret values. If configuration is missing or agent lacks write permission, give exact UI steps and wait for the owner to add the values before claiming readiness.

### 2. Create Apple Developer credentials

Before starting, explain that the owner needs an active Apple Developer Program membership, access to the correct team, and a Mac to create and export the signing certificate. A **Team ID** is Apple's short identifier for their developer team; it is not a password.

1. Explain where to find the Team ID in Apple Developer account membership details. Have the owner record it; workflow usually needs it to check the certificate belongs to the right team.
2. Walk them through [Certificates, Identifiers & Profiles](https://developer.apple.com/account/resources/certificates/list): **Certificates** → **+** → **Developer ID** → **Developer ID Application**. Explain that a **CSR** is a request created on their Mac so Apple can issue a certificate; its private key stays on that Mac. In Keychain Access, create the CSR, upload it, download Apple's `.cer` certificate, and open it to install it beside the matching private key. For a PKG installer, create **Developer ID Installer** too; do not ask for it when repo only signs an app/DMG.
3. In Keychain Access → **My Certificates**, tell them to expand the new certificate and confirm a private key appears under it. Then export that identity as a password-protected `.p12`. Explain that `.p12` is the file CI needs because it carries both certificate and private key; Apple's downloaded `.cer` by itself cannot sign. Have them record the exact identity shown by `security find-identity -v -p codesigning`, for example `Developer ID Application: Company Name (TEAMID)`.
4. For API-key notarization, walk them through [App Store Connect](https://appstoreconnect.apple.com/) → **Users and Access** → **Integrations** → **App Store Connect API** → **Team Keys**. Tell them to create a key with the minimum role that Apple's current notarization guide and this framework require. They download a `.p8` key and record its Key ID and Issuer ID. Explain that Apple only offers the private `.p8` for download once, so they must keep it safe before leaving the page. If this repo cannot use that method, guide them through Apple's linked app-specific-password steps instead; explicitly say this is a special password for the app, not their normal Apple account password.

Official Apple guides: [Developer ID certificates](https://developer.apple.com/help/account/certificates/create-developer-id-certificates/), [App Store Connect API keys](https://developer.apple.com/documentation/appstoreconnectapi/creating-api-keys-for-app-store-connect-api), [notarization workflow](https://developer.apple.com/documentation/security/customizing-the-notarization-workflow), and [app-specific passwords](https://support.apple.com/en-us/102654).

### 3. Add credentials to a protected GitHub Environment

1. In the target repository open **Settings** → **Environments** → **New environment**. Create the environment name used by the workflow (commonly `release` or `production`).
2. Restrict deployment branches/tags to the actual trusted production branch or release tag pattern. Add required reviewers when the repository needs human approval before signing or publishing.
3. Open that environment's **Environment secrets** → **Add secret**. Add each secret using the exact names discovered in step 1. Put signing credentials and notarization credentials at environment scope where possible. Put non-sensitive identifiers in environment variables if the framework supports that; never commit `.p12`, `.p8`, app-specific passwords, or encoded forms to the repository.
4. Ensure the macOS release job has `environment: <configured-name>` and only trusted release events can reach it. Pull-request jobs, including same-repository feature branches and forks, must not reference the environment or receive Apple secrets. Keep secret values out of command arguments where the framework offers safer environment/keychain input.

GitHub secret names belong to the repository. They do not define framework environment-variable names. Choose clear storage names, then map them to the selected framework's documented runtime contract.

Example GitHub secret storage for Tauri v2:

| GitHub secret name | Stored value |
| --- | --- |
| `APPLE_CERTIFICATE_BASE64` | Base64 encoding of password-protected `.p12` containing Developer ID identity and private key |
| `APPLE_CERTIFICATE_PASSWORD` | Password used to export that `.p12` |
| `APPLE_SIGNING_IDENTITY` | Exact Developer ID Application identity string from `security find-identity` |
| `APPLE_TEAM_ID` | Apple Developer Team ID |
| `APPLE_NOTARY_KEY_ID` | Key ID from App Store Connect; identifier, not private-key contents |
| `APPLE_NOTARY_PRIVATE_KEY` | Contents of downloaded `.p8` private-key file |
| `APPLE_NOTARY_ISSUER_ID` | Issuer ID from App Store Connect |

Map those GitHub secrets to Tauri v2 runtime inputs:

| Tauri runtime input | Source |
| --- | --- |
| `APPLE_CERTIFICATE` | `APPLE_CERTIFICATE_BASE64` secret value |
| `APPLE_CERTIFICATE_PASSWORD` | `APPLE_CERTIFICATE_PASSWORD` secret value |
| `APPLE_SIGNING_IDENTITY` | `APPLE_SIGNING_IDENTITY` secret value |
| `APPLE_API_KEY` | `APPLE_NOTARY_KEY_ID` secret value (Key ID) |
| `APPLE_API_KEY_PATH` | Path to a temporary file containing `APPLE_NOTARY_PRIVATE_KEY` secret value |
| `APPLE_API_ISSUER` | `APPLE_NOTARY_ISSUER_ID` secret value |

In the trusted macOS release job, write the private key to a restricted temporary file, then run authentication preflight before building release assets. Example step:

```yaml
- name: Authenticate and build macOS release
  env:
    APPLE_CERTIFICATE: ${{ secrets.APPLE_CERTIFICATE_BASE64 }}
    APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
    APPLE_SIGNING_IDENTITY: ${{ secrets.APPLE_SIGNING_IDENTITY }}
    APPLE_API_KEY: ${{ secrets.APPLE_NOTARY_KEY_ID }}
    APPLE_API_ISSUER: ${{ secrets.APPLE_NOTARY_ISSUER_ID }}
    APPLE_NOTARY_PRIVATE_KEY: ${{ secrets.APPLE_NOTARY_PRIVATE_KEY }}
  run: |
    set -eu
    key_path="$RUNNER_TEMP/AuthKey_${APPLE_API_KEY}.p8"
    umask 077
    printf '%s' "$APPLE_NOTARY_PRIVATE_KEY" > "$key_path"
    trap 'rm -f "$key_path"' EXIT
    xcrun notarytool history --key "$key_path" --key-id "$APPLE_API_KEY" --issuer "$APPLE_API_ISSUER" >/dev/null
    APPLE_API_KEY_PATH="$key_path" pnpm tauri build --bundles dmg
```

This mapping follows [Tauri v2 macOS signing and notarization documentation](https://v2.tauri.app/distribute/sign/macos/). Other frameworks may require different runtime names or file formats; follow their current official documentation. Never assume a GitHub secret named `APPLE_API_KEY` must contain private-key contents.

Only add the API-key trio **or** the documented Apple ID/app-specific-password inputs needed by the chosen auth method. Avoid duplicate credentials. If a workflow needs a base64 value, generate it locally and paste directly into GitHub's secret form; never print it into CI logs or commit it. For example, on macOS: `base64 -i DeveloperID.p12 | tr -d '\n'`.

### 4. Add fail-closed preflight and prove readiness

In the trusted macOS release job, before building publishable assets:

1. Map GitHub secrets to the framework's required environment variables. Check every required variable is non-empty with a shell preflight that prints only missing **names**, never values. A successful `gh secret list` proves only that a name exists, not that its value is correct or accessible to this job.
2. Import the `.p12` into a temporary keychain using the framework's supported action/commands; unlock only for the signing job and remove it afterward. Confirm the expected valid Developer ID identity and Team ID are present. Fail on missing private key, expired/revoked identity, wrong certificate type, or Team ID mismatch.
3. Authenticate to Apple's notary service using the configured method (for API key, verify key file, key ID, issuer ID; for Apple ID, verify Apple ID, app-specific password, Team ID). Use `xcrun notarytool` or the framework's supported equivalent. Do not put credentials into logged command text. Confirm authentication before release assets can be published.
4. Build and sign the actual release bundle. Verify signature with `codesign --verify --deep --strict --verbose=2` and inspect its designated requirement/Team ID. Submit the exact distributable package for notarization, wait for **Accepted**, staple where supported, then run `xcrun stapler validate` and `spctl -a -vv` on the distributed artifact. Fail the macOS release if any required stage fails; do not publish an unsigned/unnotarized macOS asset as though it were ready.
5. Run a trusted release workflow and inspect its run/checks. A configured secret name or green PR validation does not prove release credentials work. Report separately: GitHub environment configured, credential preflight passed, signing passed, notarization accepted, staple/Gatekeeper checks passed, asset published.

If Apple credentials are unavailable, continue only independent platform builds if repository policy allows. Mark macOS release as blocked and do not claim the release is complete or publish an unsigned macOS artifact.

## Homebrew and Buzz.app

Homebrew package update and app self-update are separate mechanisms. Inspect the current tap repository and Buzz.app implementation to learn which package types, architectures, and OS targets Buzz can install. Never assume a formula or cask automatically means Buzz supports that artifact.

- **CLI**: normally represent as a formula with platform-specific URLs/bottles only where supported. Validate formula against the tap's conventions and artifact matrix.
- **GUI on macOS**: a cask may install an app bundle from a supported archive/DMG/PKG. Match Buzz's cask support and expected app path.
- **Linux**: Homebrew runs on Linux; formulas commonly serve CLI tools. Use casks for GUI only when both the tap and Buzz support that Linux cask artifact/install behavior. Otherwise expose the GitHub Release package through a supported Linux installation path.
- **Windows**: Homebrew runs under WSL 2, not as native Windows package management. Add Windows to Buzz's Homebrew flow only if Buzz explicitly supports WSL/Homebrew integration and the package has been installed and verified there. Otherwise publish the Windows artifact on GitHub Releases and use Buzz's supported Windows installer source, if one exists.
- For `luban-homebrew`, target `develop` unless the user specifies another base branch. If the target branch is missing, stop and ask whether to create it from the verified tap default branch; never silently fall back to `main`. A GitHub Actions job in the release workflow opens each tap PR automatically; do not leave it to a human. In that job, verify release URL is public and immutable, checksum matches downloaded asset, formula/cask syntax passes tap checks, and PR base matches the requested target. Never put credentials or mutable local paths in a formula/cask.

Homebrew platform guidance: [Homebrew on Linux](https://docs.brew.sh/Homebrew-on-Linux), [Adding Software to Homebrew](https://docs.brew.sh/Adding-Software-to-Homebrew).
