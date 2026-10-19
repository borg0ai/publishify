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
| Push to `main` | One release workflow (section 7): verify, build each product, publish tag and release | Build an ambiguous branch head or bypass failed gates |

Make promotion creation idempotent: use `gh pr list --head develop --base main` or the GitHub API to find and reuse the existing PR. If no release is pending, do not open a promotion PR. If version comparison cannot prove a bump, fail closed and report why.

The version and promotion jobs need narrowly scoped write permissions. A job that opens PRs needs `pull-requests: write`; a tool that pushes a release branch or commits version files also needs `contents: write`. Check whether repository Actions settings allow workflow-created PRs. Events created with the repository's `GITHUB_TOKEN` do not normally start another workflow run; `pull_request` runs caused by that token can remain approval-required. Use a fine-grained PAT when generated PRs must trigger normal CI automatically. Never expose the PAT to fork PR jobs. See [GitHub's `GITHUB_TOKEN` event behavior](https://docs.github.com/en/actions/concepts/security/github_token).

### Set up a bot credential

#### Fine-grained PAT setup

Use a fine-grained PAT owned by a maintainer who can access the repository. This is the only bot-token setup Publishify recommends. The token expires and remains tied to its owner's account, so record its renewal date. If organization policy blocks PATs, stop and ask the owner to resolve policy; do not substitute another credential type.

1. Have the maintainer who will own the automation open **GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**. See [GitHub's fine-grained PAT guide](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens).
2. Set **Resource owner** to the repository owner, **Repository access** to **Only select repositories**, and select this repository. Choose an expiration date the maintainer can renew.
3. Grant **Pull requests: Read and write**. Grant **Contents: Read and write** only if the release tool pushes branches or commits version files. Do not grant Administration or unrelated repository access.
4. Save the token immediately as repository secret `RELEASE_BOT_TOKEN`. GitHub will not show it again. Pass `${{ secrets.RELEASE_BOT_TOKEN }}` through the tool's documented token input or `GH_TOKEN` environment variable. Never print it.

Store this as a repository secret under **Settings → Secrets and variables → Actions**, named `RELEASE_BOT_TOKEN`. Pass it to the selected tool using that tool's documented input:

```yaml
# Release Please step
with:
  token: ${{ secrets.RELEASE_BOT_TOKEN }}
```

```yaml
# Changesets action step
with:
  github-token: ${{ secrets.RELEASE_BOT_TOKEN }}
```

```yaml
# gh CLI step
env:
  GH_TOKEN: ${{ secrets.RELEASE_BOT_TOKEN }}
run: gh pr create --head develop --base main --title "Promote develop to main" --body "Automated promotion"
```

Do not put the PAT value in YAML, logs, artifacts, or fork-PR jobs. Keep workflow `permissions:` minimal; the PAT's repository permissions control API writes. If a tool uses `GITHUB_TOKEN` by default, explicitly pass `RELEASE_BOT_TOKEN` to its documented token input or it will keep using the default token.

PAT automation stops when token expires or its owner loses repository access. For organization repositories, organization policy may block PATs or require approval; ask an organization owner to approve the fine-grained token.

#### Verify bot PR checks

After wiring the token, make a safe test PR or wait for the next generated PR. Confirm the bot PR has the expected head/base branches, Actions starts the normal `pull_request` CI checks without an approval-required state, and `gh pr checks <number>` reports required checks. Also inspect workflow permissions and logs to ensure no credential value is printed. A token step succeeding is not proof that branch protection checks ran.

Release tools differ. Configure version-PR mode on `develop` without production publication (for example, verify Release Please's [`target-branch` and `skip-github-release`](https://github.com/googleapis/release-please-action) behavior in its current docs). Keep actual tags and GitHub Releases in the trusted `main` pipeline. For Changesets, verify [`pr-base-branch` and version/publish modes](https://github.com/changesets/action). Explicitly pass `RELEASE_BOT_TOKEN` using the tool's documented token input. Do not copy settings blindly into another tool.

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

## 6. Build the Release Please workflow

Apply when the repository uses Release Please. Build these pieces in order; a green job alone proves none of them.

### 1. Match config to existing tag history

- Pin the action to an exact Release Please version and note it; every later check targets that version.
- Set tag options (`include-v-in-tag`, `include-component-in-tag`, `tag-separator`) so manifest mode searches the repo's real tag form (for example `vX.Y.Z`). Keep `.release-please-manifest.json` equal to the latest published tag's version; never invent a bootstrap value.
- Use `"separate-pull-requests": false` for one version PR across packages, and `"include-component-in-tag": false` for plain `vX.Y.Z` tags. On the development branch run version-PR mode only: target `develop`, `skip-github-release: true`. Tags and releases come from the `main` pipeline.

### 2. Write TOML `extra-files` selectors for the pinned version

`GenericToml` is not plain TOML-to-JSON: v17.3.0 parses scalars as wrapped nodes. Filters read the wrapped value; the update target is the wrapped version node.

```json
{ "type": "toml", "path": "Cargo.lock", "jsonpath": "$.package[?(@.name.value=='my-crate')].version" }
{ "type": "toml", "path": "Cargo.toml", "jsonpath": "$.workspace.package.version" }
{ "type": "toml", "path": "apps/desktop/src-tauri/Cargo.toml", "jsonpath": "$.package.version" }
```

Confirm each path against the [v17.3.0 updater source](https://github.com/googleapis/release-please/blob/v17.3.0/src/updaters/generic-toml.ts) and [array-filter issue](https://github.com/googleapis/release-please/issues/2455). Prove it before merging: run the pinned updater (for example `release-please release-pr --dry-run`) or a script on representative files, and require every version field to change exactly once with valid TOML. If that cannot run locally, require the first hosted version PR diff to show every configured file updated.

### 3. Run version-PR mode on `develop` with a token guard

Fail fast when the bot token is missing; a `GITHUB_TOKEN`-created PR will not start CI.

```yaml
on:
  push: { branches: [develop] }
permissions: { contents: write, pull-requests: write }
concurrency: { group: "release-please-${{ github.ref }}", cancel-in-progress: false }
steps:
  - name: Require workflow-enabled GitHub token
    env: { RELEASE_PLEASE_TOKEN: "${{ secrets.RELEASE_PLEASE_TOKEN }}" }
    run: |
      [ -n "$RELEASE_PLEASE_TOKEN" ] || { echo "::error::Configure repository secret RELEASE_PLEASE_TOKEN"; exit 1; }
  - uses: googleapis/release-please-action@v4
    id: release
    with:
      token: ${{ secrets.RELEASE_PLEASE_TOKEN }}
      target-branch: develop
      skip-github-release: true
      config-file: release-please-config.json
      manifest-file: .release-please-manifest.json
```

Add no reporting step after the action: the version PR is the output, and a run that opens nothing needs no summary. Keep `skip-github-release` in mind when a run looks idle: Release Please aborts by design while a merged version PR is still `autorelease: pending`, until the `main` release retags it (stage 5).

### 4. Promote from the merged version PR

```yaml
on:
  pull_request: { types: [closed], branches: [develop] }
permissions: { contents: read, pull-requests: write }
concurrency: { group: promote-develop-to-main, cancel-in-progress: false }
jobs:
  promote:
    if: >-
      github.event.pull_request.merged == true &&
      contains(github.event.pull_request.labels.*.name, 'autorelease: pending')
```

Steps, each with `GH_TOKEN` set to the bot token:

1. Check out `develop` with `fetch-depth: 0`, run the repo's version-sync check, then compare the product version on `develop` with `origin/main`. Equal means no promotion.
2. Stop while another open version PR on `develop` still carries `autorelease: pending`.
3. Look for an open PR with `gh pr list --base main --head "$GITHUB_REPOSITORY_OWNER:develop" --state open`; reuse it, else `gh pr create --head develop --base main` with the version in the title.

Ordinary feature merges lack the label, so they never start this job.

### 5. Retag the merged version PR after publishing

With `skip-github-release: true` Release Please never creates the tag or release, so it never replaces `autorelease: pending` with `autorelease: tagged` on its merged version PR. Every later run then aborts with `untagged, merged release PRs outstanding` and opens no further version PR. The `main` pipeline must do it: after `publish`, add a job of its own that creates the `autorelease: tagged` label if missing and relabels merged PRs with `autorelease: pending` whose base is the development branch.

```yaml
tag-release-pr:
  needs: [prepare, publish]
  if: needs.prepare.outputs.proceed == 'true' && github.event_name == 'push'
  steps:
    - env: { GH_TOKEN: "${{ github.token }}" }
      run: |
        set -o pipefail
        gh label create "autorelease: tagged" --repo "$GITHUB_REPOSITORY" --color EDEDED 2>/dev/null || true
        gh pr list --repo "$GITHUB_REPOSITORY" --base develop --state merged \
          --label "autorelease: pending" --json number --jq '.[].number' |
        while read -r n; do
          gh pr edit "$n" --repo "$GITHUB_REPOSITORY" \
            --remove-label "autorelease: pending" --add-label "autorelease: tagged"
        done
```

Pass `--repo "$GITHUB_REPOSITORY"` to every `gh` call so the job needs no checkout, and grant the workflow `pull-requests: write`. Keep it a separate job so its failure cannot block package-manager PRs or undo a published release. Gate it on a push event: a `workflow_dispatch` rebuild of an old tag must not tag a newer version PR still waiting for promotion. The default `run:` shell is `bash -e` without `pipefail`, so set it for any pipeline whose first command can fail.

### 6. Recover a failed release

With the tag created in the publish job (section 7), a failed build leaves no tag and a re-run works. If an older workflow already pushed a tag before failing, recover through the normal path: fix on the development branch, promote to `main` by PR, then `workflow_dispatch` with the existing tag. Do not weaken rulesets or dispatch from another branch. The version PR stays `autorelease: pending` until the release really exists.

### 7. Prove each stage separately

1. `develop` push → version PR with every version file updated.
2. Merged version PR → promotion PR.
3. `main` promotion → tag, release, assets, signatures/notarization.
4. Published release → merged version PR relabelled `autorelease: tagged`; the next `develop` push opens or updates the next version PR.
5. Verified assets → automated tap PR (`desktop-cli-release.md`).

## 7. Build one release workflow

One workflow handles every product in the repository; each product's job comes from its profile in `references/profiles/`. Only workflows with different triggers stay separate: PR CI, version PR (push to `develop`), promotion (version PR merged).

Trigger: push to `main`, plus `workflow_dispatch` with an existing tag. Jobs, in order, each `needs:` the previous:

1. **verify**: rerun CI for the exact `main` commit, or confirm its required checks passed.
2. **prepare**: read the version from source, run the repo's version-sync check, compute `v<version>`. Stop with no error if that tag already exists.
3. **build-<product>**: one job or matrix per product (CLI archives, app installers), from its profile.
4. **verify-assets**: exact expected asset names, signatures, notarization, checksums.
5. **publish**: create the tag and GitHub Release at the verified commit, only now. A failed build leaves no tag.
6. **bookkeeping**: version-tool follow-up (Release Please: relabel merged version PRs `autorelease: pending` → `autorelease: tagged`, push events only, section 5).
7. **homebrew**: for any macOS asset, open the PR against `luban-homebrew` (base `develop`; formula for a CLI, cask for a macOS app) after the release is public and verified. Linux Homebrew is not covered yet.

Rules: jobs 6 and 7 are idempotent and re-runnable alone. A dispatch on an existing tag rebuilds that tag's source, never moves the tag, and never flips labels for a newer version PR. Pin actions to commit SHAs, use `RELEASE_BOT_TOKEN`, add `--repo "$GITHUB_REPOSITORY"` to every `gh` call, and give each job the least `permissions`. Before relying on a release-created tag, read `gh api repos/{owner}/{repo}/rulesets` for rules targeting tags; if `v*` is protected, report it and ask the owner to allow the release job.
