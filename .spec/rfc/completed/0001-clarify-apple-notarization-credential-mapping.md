# RFC 0001: Clarify Apple notarization credential mapping

**Status:** Implemented

## Summary

Clarify that GitHub secret names and framework runtime environment variables are separate contracts. Correct the Apple API-key example for Tauri so `APPLE_API_KEY` contains the App Store Connect Key ID and `APPLE_API_KEY_PATH` points to the downloaded `.p8` file. Give a concrete secret-to-runtime mapping without presenting framework-specific names as universal Apple secret names.

## Problem

`references/desktop-cli-release.md` currently describes `APPLE_API_KEY` as the `.p8` file contents and `APPLE_API_KEY_ID` as the Key ID. Tauri v2 uses `APPLE_API_KEY` for the Key ID and `APPLE_API_KEY_PATH` for the private-key file path. Copying the current generic table into a Tauri workflow can put private-key contents where Tauri expects an ID, causing notarization to fail. The guide also needs to make clear that `gh secret list` confirms secret names only; it cannot validate secret values or prove the release job can authenticate to Apple.

## Goals

- Distinguish GitHub secret storage names from framework runtime variable names.
- Correct the Tauri v2 App Store Connect API-key mapping using current official Tauri documentation.
- Keep the example adaptable: other frameworks may require different runtime names or authentication inputs.
- Require release preflight to validate actual credential availability and authenticate before publishable assets are built or released.

## Non-goals

- Do not change Karen's workflow or release credentials in this RFC.
- Do not prescribe one universal GitHub secret naming scheme for every framework.
- Do not document Apple ID authentication beyond preserving the existing separate authentication option.
- Do not put credential values, private keys, or repository-specific secrets in the skill.

## Design

Update `references/desktop-cli-release.md`:

- Replace the single “typical values” table with separate GitHub secret storage and Tauri runtime mapping examples. Use distinct example GitHub names for the Key ID, `.p8` contents, and Issuer ID.
- In the Tauri v2 example, write the private key to a restricted temporary file in the trusted macOS release job, and map runtime `APPLE_API_KEY` to the Key ID and `APPLE_API_KEY_PATH` to that file path. Map the Issuer ID to `APPLE_API_ISSUER`.
- State that secret names are repository choices and must be mapped to the selected framework's documented variables. Do not imply that a GitHub secret named `APPLE_API_KEY` must contain private-key contents.
- Preserve the existing guidance that missing or invalid credentials block macOS release and that `gh secret list` reveals names only. Require an authentication preflight before producing release assets.
- Link the [Tauri v2 macOS signing and notarization documentation](https://v2.tauri.app/distribute/sign/macos/) as the source for this example.

## Acceptance

- The guide distinguishes GitHub secret names from runtime environment variables.
- The Tauri v2 example explicitly maps Key ID to `APPLE_API_KEY`, `.p8` file path to `APPLE_API_KEY_PATH`, and Issuer ID to `APPLE_API_ISSUER`.
- The GitHub secret examples use distinct storage names for Key ID, `.p8` contents, and Issuer ID, and write `.p8` contents to a restricted temporary file.
- The example never maps raw `.p8` contents directly to Tauri's `APPLE_API_KEY`.
- The guide says `gh secret list` confirms names only and requires release authentication preflight.
- The Tauri example links to the official Tauri v2 macOS signing and notarization documentation.
- `specify validate` and `specify sync-check` pass for this RFC and its index/task entries.
