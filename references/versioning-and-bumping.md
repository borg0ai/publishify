# Version Bump and Promotion Guide

Use this guide when designing or reviewing CI for a repository that publishes a CLI, desktop app, or both. A version bump is part of the release flow. A check that versions happen to match does not tell the repository how to select or create the next version.

## 1. Find the repository's version rules first

Inspect the full release path before proposing a bump:

1. Read applicable `AGENTS.md`, release docs, changelog policy, CI workflows, tags, and recent release PRs.
2. Find release automation and configuration, such as Release Please, Changesets, semantic-release, cargo-release, a project script, or a hand-reviewed release PR. Read the current official docs for the tool actually used.
3. Find the authoritative version source for each product. Search app, CLI, workspace, updater, and packaging manifests. Examples include `package.json`, workspace manifests, `Cargo.toml`, `pyproject.toml`, Tauri/Electron/Wails config, and release manifests. Do not assume all files share one version.
4. Check the latest published Git tag and release. Compare them with the checked-out branch and the proposed PR. Do not infer the next version from a stale manifest alone.
5. Record the result in plain terms: what file/tool owns the version, which products share it, what event performs the bump, and what tag format CI expects.

If existing policy is clear, preserve it. If it is missing or contradictory, show the repository owner a specific proposed rule and ask for a decision before introducing a new release policy. Do not silently choose a versioning tool or tag format.

## 2. Choose whether this change needs a bump

Use the repository's release labels, commit convention, or release notes as the authority when present. Otherwise, explain the usual SemVer choice and confirm unclear or breaking changes with the owner:

- **Patch**: backward-compatible bug fix or small maintenance change.
- **Minor**: backward-compatible user-visible feature.
- **Major**: change that breaks an existing user, CLI, config, data, or API contract.
- **No release**: docs-only or internal changes when repository policy says they do not ship.

Respect the repository's policy for `0.x` versions; do not assume pre-1.0 projects use the same rules as `1.x`. In a monorepo, bump only products affected by the change unless the repository deliberately releases them together. If impact is unclear, ask rather than guessing.

## 3. Put the bump in the correct branch and event

For the common `feature → develop → main → release` flow:

- Feature PRs into `develop` run full CI. They normally do not edit shipped version files or create release tags. Feed release automation through its required commit messages, labels, or changeset files instead.
- A push to `develop` after feature merges runs the version tool. It calculates the next version from repository policy and opens or updates a version PR whose base is `develop`. The PR contains tool-generated version/changelog changes and passes normal `develop` checks.
- Merge the version PR into `develop`. Configure this step to update version files only; it must not create a production tag, GitHub Release, sign artifacts, or publish packages.
- After that merge, a promotion workflow compares the release version on `develop` with the version on `main` (or the last published version, as repository policy defines). If a release is pending, it opens or updates a PR with `head=develop` and `base=main`. Search for an existing PR with that head/base pair first so repeated pushes update one PR rather than create duplicates. This preserves a strict rule that only `develop` may be the source of a `main` PR.
- The `develop → main` PR runs full CI, a source-branch/repository guard, and synchronized version/release-policy checks. Merge is the production release decision. Do not skip required review or branch protection to automate this flow.
- A push to `main` after promotion triggers release work against that exact commit and version. Create the tag/release, build CLI and GUI artifacts, sign/notarize, and publish only after all required checks pass. Never tag a moving branch head or publish before the version is fixed.

### Workflow trigger and token contract

For this flow, inspect the actual release tool and encode these trigger boundaries explicitly:

| Event | Required action | Must not happen |
| --- | --- | --- |
| `pull_request` targeting `develop` | Full PR CI; collect commit/changeset release input | Release tag or production publishing |
| Push to `develop` | Calculate/update version PR targeting `develop` | Sign, notarize, or publish production assets |
| Version PR merged into `develop` | Detect that `develop` version is ahead; create/update `develop → main` PR | Create production tag/release |
| `pull_request` targeting `main` | Verify source is same-repo `develop`; full CI and version checks | Expose release secrets or publish |
| Push to `main` or approved release tag | Build exact release commit, validate version, sign/notarize, publish assets | Build an ambiguous branch head or bypass failed gates |

Make promotion creation idempotent: use `gh pr list --head develop --base main` or the GitHub API to find and reuse the existing PR. If no release is pending, do not open a promotion PR. If version comparison cannot prove a bump, fail closed and report why.

The version and promotion jobs need narrowly scoped write permissions. A job that opens PRs needs `pull-requests: write`; a tool that commits version files may also need `contents: write`. Check whether repository Actions settings allow workflow-created PRs. Events created with the repository's `GITHUB_TOKEN` generally do not start another workflow run, so required CI may not run for a bot-created PR. Use an approved GitHub App installation token or narrowly scoped PAT when PR checks must trigger normally, or explicitly dispatch the required validation workflow and verify its result. Never put such credentials in fork PR jobs. See [GitHub's `GITHUB_TOKEN` event behavior](https://docs.github.com/en/actions/concepts/security/github_token).

Release tools differ. Configure version-PR mode on `develop` without production publication (for example, verify Release Please's [`target-branch` and `skip-github-release`](https://github.com/googleapis/release-please-action) behavior in its current docs). Keep actual tags and GitHub Releases in the trusted `main` pipeline. For Changesets, verify [`pr-base-branch` and version/publish modes](https://github.com/changesets/action). Do not copy these settings blindly into another tool.

These are default boundaries, not a mandate to replace a working release tool. If the current tool creates a release PR on another branch or targets `develop`, follow its verified configuration and document how that fits the branch policy.

## 4. Apply the bump through the owning tool

Use the repository's existing command or automation. Do not edit generated lockfiles, changelog output, updater metadata, or derived app versions by hand when a tool owns them.

Examples of tool behavior to discover and preserve:

- **Release Please**: inspect its config, manifest, commit parsing, release PR target, and manifest/monorepo settings. Let its release PR own generated version and changelog edits. Do not make a second manual bump for the same commits.
- **Changesets**: inspect package selection and release branch behavior. Add the required per-package changeset for each affected package, then let the configured version PR update package versions and changelogs.
- **semantic-release**: inspect plugins and commit rules. It may calculate the next version and tag during the trusted release job without committing a version bump to the promotion PR. Do not add competing version edits.
- **Project-specific command**: use its documented version/release command and verify exactly which files it changes before staging.

For every explicit bump, ensure the release PR explains the old version, new version, bump type, user-visible reason, and affected products. Avoid a second bump if an open release PR already includes the same unreleased changes.

## 5. Validate versions before publishing

Add or preserve a required check for the promotion/release path. It should verify the repository's actual contract, not demand that unrelated version strings are identical:

1. The selected version is valid for the repository's format and is greater than the last published version when a release is intended.
2. All files that must move together agree: source manifests, CLI/app metadata, updater config or manifest, package lock data generated by the project tool, release notes, and tag. Check only fields the project defines as synchronized.
3. The release tag uses the repository's exact convention (`v1.2.3`, `1.2.3`, or another configured format) and points at the promoted commit.
4. No same-version tag or release already exists. A retry must reconcile remote state, not overwrite a published tag.
5. Version validation and release creation run only on trusted PR/push/tag events. Pull requests from forks never receive release write permissions or signing secrets.
6. CI fails before publishing if a required bump is absent, malformed, inconsistent, or already published. If no release is due, the configured policy must explicitly allow a no-release promotion.

Report separately whether the bump was selected, generated, validated, merged, tagged, and published. A green version check does not prove that a release or its assets exist.
