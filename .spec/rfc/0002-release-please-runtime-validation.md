# RFC 0002: Validate Release Please configuration and workflow outcomes

**Status:** Draft

## Summary

Strengthen Publishify's Release Please guidance so configuration is checked against the exact updater/runtime behavior and observed workflow output. TOML parsing and a green action status alone do not prove a version PR was created or that a historical tag was recognized.

## Problem

The Karen release workflow exposed gaps in the existing guidance. Its first selector correction still targeted TOML scalar `.value` nodes. Release Please v17.3.0 `GenericToml` uses a tagged TOML representation: Cargo.lock filtering must read the wrapped package name (`name.value`), while the update target is the version node itself (`version`). Ordinary TOML parsing cannot detect a selector that misses that node. The final hosted version PR diff confirmed the corrected selectors updated all seven local Cargo packages and the other configured version files.

The same run also showed why workflow conclusions need inspection: after version PR #12 merged into `develop`, Release Please logged `There are untagged, merged release PRs outstanding - aborting` and returned job success without opening another version PR. In this repository's policy, that is an expected lifecycle guard: GitHub Releases are skipped on `develop`, and the v0.12.0 production tag is created only after promotion to `main`. The promotion workflow opened PR #13, which is the next required stage. Treating this message as either a published release or a configuration failure would both be wrong. The current guidance does not require distinguishing this expected wait-for-production-tag state from a broken version calculation.

## Goals

- Require the release configuration to match the repository's existing release tag format and current version baseline.
- Require TOML extra-file selectors to be validated against the exact Release Please version used by the action, including its updater representation and array filtering behavior.
- Distinguish workflow job success from version PR creation, update, no-op, or abort; classify an untagged merged release PR as an expected wait state only when the production tag is intentionally deferred to main and a promotion is pending.
- Verify each branch transition independently: develop push to version PR, merged version PR to promotion PR, and main promotion to production release.
- Keep ordinary feature merges from allocating a promotion runner when no version PR has merged.

## Non-goals

- Do not prescribe Release Please as the required tool for every repository.
- Do not make Release Please create production tags or GitHub Releases on the development branch when repository policy reserves publishing for main.
- Do not add repository-specific versions, tokens, Apple credentials, or private configuration to Publishify.
- Do not replace repository CI with a local test framework solely to simulate GitHub Actions.

## Existing guidance to preserve

`SKILL.md` and `references/versioning-and-bumping.md` already cover the feature → develop → version PR → promotion PR → main release sequence, fine-grained PAT permissions and generated-PR CI behavior, idempotent promotion PRs, and stage-by-stage artifact checks. `references/desktop-cli-release.md` already contains the Tauri Apple credential mapping clarified by RFC 0001. This RFC adds Release Please updater/runtime and outcome diagnostics; it must not duplicate or weaken those existing rules.

## Design

Refine `references/versioning-and-bumping.md` and the Release Please portions of `SKILL.md`:

- Add a Release Please preflight that compares configured tag options with actual tags/releases. For an existing `vX.Y.Z` history, explicitly verify that manifest mode searches for that tag form; inspect the action log's searched tag before changing the manifest version or inventing a bootstrap value.
- Document the TOML updater as version-specific behavior, not ordinary TOML-to-JSON semantics. For Release Please v17.3.0, `GenericToml` uses a tagged parser and JSONPath pointers. The verified Cargo.lock selector shape is `$.package[?(@.name.value=='package-name')].version`; the filter reads the wrapped name, while the updater target stops at the wrapped version node. Plain manifest fields such as workspace package version use paths ending at the version node (for example `$.workspace.package.version`). Confirm these paths against the source or an executable updater for the exact action-bundled Release Please version before recommending them. Do not claim a plain TOML parser check proves updater compatibility.
- Require a focused updater check that applies configured extra-file selectors to representative files and confirms every expected version field changes exactly once while the TOML remains valid. If the exact updater cannot be exercised locally, mark that proof as pending and require the hosted Release Please run to show the generated PR diff updates all version files.
- Treat Release Please outcomes separately: action/job status, `prs_created`/PR outputs, log diagnostics, and actual PR state. A successful job with no PR is not a version bump. When logs report an untagged merged release PR, check whether the version PR was merged on the development branch, whether policy defers tags/releases to main, and whether a promotion PR is open. If all hold, report “waiting for production tag after promotion,” not a failed bump or a published release. Otherwise investigate the abort as a possible blocker. After main promotion, verify that the production release run creates the expected tag; only then should the outstanding-tag guard clear for the next version cycle.
- Make the workflow summary match that diagnosis. For the expected wait state, show no version PR was created, identify promotion as the pending stage, and state that no production tag/release exists yet. Do not emit the generic “completed without opening or updating” message as if it were a meaningful no-op. For an unexplained abort or a missing expected PR, fail or surface an actionable diagnostic rather than hiding it behind a green job.
- Trigger promotion work from a merged, labeled version PR into the development branch, then compare synchronized versions and idempotently create or reuse the develop-to-main PR. Do not start a promotion runner on every ordinary push to develop when there is no release PR merge.
- Keep release/tag creation on the production branch. After promotion, verify the exact main commit, tag, release, assets, signatures/notarization, and distribution PR as separate outputs. The distribution (Homebrew tap) PR is opened by an automated job in the release workflow after assets and checksums are verified, not by a manual step.

References: [Release Please v17.3.0 GenericToml updater](https://github.com/googleapis/release-please/blob/v17.3.0/src/updaters/generic-toml.ts), [Release Please tag and manifest behavior](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md#subsequent-versions), and [upstream GenericToml array-filter behavior](https://github.com/googleapis/release-please/issues/2455).

## Acceptance

- Guidance requires checking the action-bundled Release Please version, existing tag convention, and actual searched tag before editing manifest bootstrap values.
- Guidance records the v17.3.0 GenericToml Cargo.lock path shape and clearly marks it version-specific; it warns that ordinary TOML parsing does not validate Release Please selectors.
- Guidance requires an updater-level proof or an explicit hosted-run PR-diff proof for all configured version files.
- Guidance distinguishes successful workflow execution from version PR creation/update, no-op, and abort, and classifies the expected untagged-merged-PR wait state against promotion PR and production tag state.
- Guidance requires the workflow summary to report expected wait-for-promotion/tag state distinctly from a normal no-op or unexplained abort.
- Guidance describes the merged-version-PR promotion trigger and prevents promotion runner work on ordinary develop pushes.
- Guidance keeps version PR merge, promotion PR merge, tag/release creation, artifact verification, and package-manager PR as independently verified stages.
- RFC validation and `.spec` sync-check pass.
