# RFC 0002: Guide agents to build Release Please workflows correctly

**Status:** Draft

## Summary

Teach Publishify to make an agent write Release Please configuration and workflows that work on the first hosted run: tag options matching existing history, TOML selectors proven against the pinned updater, a workflow step that reports real release state, and a promotion job gated on the merged version PR. The guidance lives in `references/versioning-and-bumping.md` section 6, with a pointer from `SKILL.md`.

## Motivation

The Karen release workflow shipped with TOML selectors that parsed fine but missed Release Please v17.3.0's wrapped `GenericToml` nodes, and its job went green without opening a version PR because a merged release PR was still untagged (an expected wait, since tags are deferred to `main`). Both were unspecified in the skill, so the agent built a workflow that could not tell the author which case applied. The skill should specify what to build so these cases cannot occur.

## Goals

An agent following the skill builds a workflow that:

- Uses tag options and a manifest version matching the repository's existing tags, with the action pinned to an exact Release Please version.
- Runs version-PR mode only on the development branch (`skip-github-release`), leaving tags and releases to `main`.
- Uses TOML `extra-files` selectors proven against the pinned updater, or marks the proof pending until the hosted version PR diff shows every file updated.
- Includes a step after the action that queries PR state and writes the job summary: version PR created, waiting for production tag after promotion, or no releasable commits. It fails on a merged untagged release PR with no open promotion PR.
- Starts promotion only from a merged version PR carrying the release label, and creates or reuses the `develop → main` PR idempotently.
- Opens the Homebrew tap PR from an automated release-workflow job after assets and checksums are verified.

## Non-goals

- Do not require Release Please in every repository.
- Do not let Release Please create tags or GitHub Releases on the development branch when policy reserves publishing for `main`.
- Do not add repository-specific versions, tokens, or credentials to Publishify.
- Do not replace repository CI with a local framework that simulates GitHub Actions.

## Existing guidance to preserve

`SKILL.md` and `references/versioning-and-bumping.md` already cover the feature → develop → version PR → promotion PR → main release sequence, fine-grained PAT setup, idempotent promotion PRs, and stage-by-stage artifact checks. `references/desktop-cli-release.md` holds the Tauri Apple credential mapping from RFC 0001 and the tap PR rules. This RFC adds Release Please build guidance without duplicating or weakening those rules.

## Design

Section 6 of `references/versioning-and-bumping.md`, "Build the Release Please workflow", gives the agent ordered build steps with copyable snippets: config matched to tag history, version-specific selectors, the state-reporting step, and the label-gated promotion job. `SKILL.md` step 4 points to it, and step 8 requires the tap PR to be a release-workflow job.

Version-specific facts (v17.3.0 tagged TOML parsing, the `autorelease: pending` label behavior) stay marked as version-specific and must be confirmed against the source or a hosted run before an agent relies on them.

References: [GenericToml updater v17.3.0](https://github.com/googleapis/release-please/blob/v17.3.0/src/updaters/generic-toml.ts), [manifest tag behavior](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md#subsequent-versions), [array-filter issue](https://github.com/googleapis/release-please/issues/2455).

## Acceptance

- Guidance tells the agent to pin the action, match tag options to existing tags, and run version-PR mode only on `develop`.
- Guidance gives the v17.3.0 Cargo.lock and workspace selector shapes, marked version-specific, and requires an updater-level proof or hosted PR-diff proof.
- Guidance includes the state-reporting step and requires an explicit failure for an unexplained abort.
- Guidance gates promotion on a merged, labeled version PR so ordinary `develop` pushes start no promotion runner.
- Guidance makes the tap PR an automated release-workflow job and keeps version PR, promotion PR, release, artifacts, and tap PR as separately proven stages.
- RFC validation and `.spec` sync-check pass.
