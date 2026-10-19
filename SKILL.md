---
name: publishify
description: Design, refine, and review GitHub Actions CI/CD for repositories shipping CLI tools and desktop apps, including PR gates, signed cross-platform GitHub Releases, framework auto-updaters, and Homebrew/Buzz distribution. Use when changing .github/workflows build, verification, release, or package-manager pipelines.
license: MIT
metadata:
  version: "0.1.9"
---

# Publishify

Shape repository CI/CD around its real branch, build, signing, and distribution contracts. For repositories shipping both CLI and desktop app, treat them as separate products with separate artifacts and install/update paths. Inspect before editing. Preserve working release behavior and fail closed when required release proof is missing.

## Use this skill for

- Pull request verification from feature branches into development branches.
- Promotion pull requests from development branches into production branches, including version checks.
- Release pipelines triggered by production branch updates or release tags.
- Cross-platform desktop builds, signing, notarization, update metadata, and package-manager pull requests.
- Reviewing existing GitHub Actions for gaps, unsafe permissions, broken dependencies, or release races.

Do not use this skill to perform a one-off application release, bump package versions without CI changes, or publish artifacts manually. Use the repository's release skill or documented release workflow for those tasks.

## Workflow

1. **Map repository contract.** Read applicable `AGENTS.md`, all `.github/workflows/**`, local composite actions, package/build manifests, release configuration, version sync tools, release docs/RFCs, and Git status. Identify protected branches, PR base branches, release trigger, version authority, supported OS/architectures, CLI and GUI entrypoints, artifact names, updater requirements, secrets, and package distribution target. Treat CLI and GUI as separate deliverables; do not assume same target support or package format. Treat dirty user changes as owned; do not overwrite them.
2. **Describe current flow.** Trace events → jobs → `needs` dependencies → artifacts → release/promotion side effects. Record which jobs are required checks and which jobs have write permissions. Never infer support from a matrix name alone; verify actual build targets and artifact output.
3. **Choose smallest safe change.** Keep existing workflows when they already satisfy the requested behavior. Separate PR validation, promotion/versioning, and production release when their triggers or permissions differ. Prefer reusable workflows only when duplication is real. State any platform or secret gap before making edits.
4. **Implement branch gates and automatic promotion flow.** Treat `pull_request.branches` as a base-branch filter; it does not restrict PR source branches. Run full CI for every PR targeting the development branch. For production-branch PRs, add a required source-branch guard that passes only when both `github.event.pull_request.head.ref` equals the configured development branch and `github.event.pull_request.head.repo.full_name` equals the base repository. Then run full CI plus the repository's version-bump/release-policy check. When policy is missing and the intended flow is `feature → develop → main`, implement this default: merged feature changes trigger the version tool; the tool opens/updates a version PR into `develop`; merging that PR makes the shipped version visible on `develop`; a promotion workflow then opens/updates a PR with head `develop` and base `main`; merging promotion triggers production release. Keep tag/release creation off `develop`. Use [references/versioning-and-bumping.md](references/versioning-and-bumping.md) for event, token, idempotency, and tool-specific requirements. For Release Please, also follow its section 6 (tag-matched config, selector proof, state-reporting step, label-gated promotion job). Checking the repo identity prevents a fork from spoofing the `develop` branch name. Keep required check names stable. Do not run publishing jobs for untrusted pull request code.
5. **Implement release trigger.** Trigger release work only from the configured production branch or approved release tag. Pin release commit SHA and tag explicitly; do not build an ambiguous moving branch head. Serialize releases where jobs mutate the same release. Grant write permissions only to jobs that create releases or PRs.
6. **Build supported targets.** Derive separate CLI and GUI matrices from verified product support and native dependency constraints. Build every supported OS/architecture and installable format. Upload immutable, clearly named GitHub Release assets and checksums. A successful compile is not proof of installability; inspect actual installers, archives, executable permissions, runtime dependencies, and updater metadata. Read [references/desktop-cli-release.md](references/desktop-cli-release.md) for framework-specific checks.
7. **Sign, notarize, and prepare updater.** Treat OS code signing/notarization and updater artifact/manifest signing as different checks. Inspect the project's framework version, current official framework docs, configured update provider, client endpoint, and release assets. Before enabling macOS release, verify the release GitHub Environment exists, the trusted release job references it, and every required Apple secret is configured at that environment or another intentionally scoped level. Derive secret names and required values from the actual workflow/framework auth method; never assume one universal naming scheme. Check names with GitHub settings or `gh secret list` (which shows names only), and add a workflow preflight that maps secrets to environment variables and fails on missing/empty values without printing them. Validate the Developer ID Application certificate and private key can be imported on the macOS runner, the identity is valid and matches the expected Team ID, and the selected `notarytool` authentication can authenticate before building/publishing release assets. Keep credentials in GitHub secrets or protected environments; expose them only to trusted release jobs. Do not expose them to pull-request workflows, forks, or untrusted code. For macOS direct distribution, verify Developer ID signing and notarization/stapling where required. Fail release when required signatures, notarization, update metadata, or platform artifacts are missing. Test an installed previous version updating from published release before claiming updater support. Read [references/desktop-cli-release.md](references/desktop-cli-release.md) for the Apple credential checklist.
8. **Publish package-manager update through PR.** Inspect the actual `luban-homebrew` tap and Buzz.app install contract. Classify CLI and GUI packages separately (formula, cask, or supported alternative), supported OS/architectures, asset URL, and checksum contract. For `luban-homebrew`, target `develop` unless the user specifies another target. Verify the target branch exists; if missing, stop and ask whether to create it from the verified tap default branch. Never silently fall back to `main`. Automate it as a release-workflow job that `needs:` the asset publish/verify job: download public assets, compute checksums, update the formula/cask, open a release-specific head branch PR with the bot token (`gh pr create --repo <owner/repo> --head <release-branch> --base <target> --title <title> --body-file <file>`), then check it with `gh pr checks`. Never push directly to the target; never leave the PR manual. Homebrew upgrades are separate from in-app auto-updates. See [references/desktop-cli-release.md](references/desktop-cli-release.md).
9. **Use `gh` for GitHub state.** Check `gh auth status` and `gh repo view --json nameWithOwner,defaultBranchRef` before GitHub operations. Read current rules with `gh api repos/{owner}/{repo}/rulesets --paginate` and effective branch rules with `gh api repos/{owner}/{repo}/rules/branches/{branch}`. Read workflow runs and PR checks with `gh run list`, `gh run view`, and `gh pr checks`. Never print tokens or secret values; `gh secret list` exposes names only.
10. **Configure branch rules when requested.** Use GitHub Rulesets through `gh api`, not guessed UI state. Preserve existing rules and bypass actors. Prepare and review a complete JSON payload before writing. Create with `gh api repos/{owner}/{repo}/rulesets -X POST --input <ruleset.json>`; update a discovered ruleset by ID with `gh api repos/{owner}/{repo}/rulesets/{ruleset_id} -X PUT --input <ruleset.json>`. Ruleset writes require repository Administration write permission. If API access is unavailable, report exact permission/API error and give user UI steps. Re-read rulesets and effective branch rules after mutation. See [GitHub rulesets REST API](https://docs.github.com/en/rest/repos/rules) and [`gh api` manual](https://cli.github.com/manual/gh_api).
11. **Validate workflow changes.** Run available YAML/action linters and repository validation commands. Check expression syntax, event filters, `permissions`, secret exposure, job dependencies, matrix coverage, artifact names, and failure conditions. Use local workflow simulation only if the repository already supports it. Do not claim hosted workflow success from static validation.
12. **Review final diff and remote state.** Confirm only requested workflow/config/docs paths changed, no secrets entered files or logs, release jobs cannot run from untrusted PRs, and failed build/sign/notary/artifact checks block publication. Report local validation and remote GitHub state separately.

## Diagnose post-job cache failures

When a GitHub Actions run fails during `Post job cleanup` with `Path Validation Error: Path(s) specified in the action for caching do(es) not exist`, inspect the exact failing job log with `gh run view <run-id> --job <job-id> --log-failed`. Identify the action named by the failed post step; do not infer the owner from nearby warnings. A preceding runtime deprecation warning such as Node's `DEP0040` is not the cause unless the log marks it as an error or the job exits there.

For `actions/setup-node@v5`, automatic package-manager caching is enabled when a valid `packageManager` field is detected in `package.json`, even when the workflow omits an explicit `cache` input. If the job only validates metadata and does not need dependency caching, set `package-manager-cache: false` on that setup-node step. If caching is needed, verify the selected manager, cache path, lockfile, and setup/install steps create the required path before post-job save. Avoid installing dependencies solely to satisfy an unused cache in a lightweight guard. Preserve the explicit cache behavior of build/test jobs that benefit from it. See the [setup-node v5 caching documentation](https://github.com/actions/setup-node/blob/v5.0.0/README.md#caching-global-packages-data).

Validate the fix with a new hosted run for the affected PR/job. Static inspection can confirm the workflow input, but cannot prove GitHub's post-job cache save succeeded.

## Release flow model

```text
feature branch --pull request--> develop
  required checks: lint + typecheck + tests + build smoke

push to develop after feature merges
  version tool calculates bump -> opens/updates version PR into develop
  version PR checks -> merge version PR (no tag/release here)
  merged version on develop -> bot opens/updates PR: head develop, base main

develop --promotion pull request--> main
  required checks: CI + version alignment + release policy

main update or approved release tag
  version/release creation -> OS build matrix -> signing/notarization
  -> artifact and updater verification -> GitHub Release
  -> automated tap PR job against the configured target branch (default develop)
```

When this branch model is requested, implement every arrow and verify each trigger. A version check alone does not create a bump PR or promotion PR. The version PR must target the configured development branch; the promotion PR must have the development branch as its head so a strict main source-branch guard passes. Keep release/tag creation on `main`. Follow repository branch names and release trigger when they differ.

## GitHub branch protection setup

Workflow YAML cannot enable repository branch protection. After adding workflows, configure **Settings → Rules → Rulesets** (or **Settings → Branches → Branch protection rules**):

- **Production branch (`main`)**: require pull requests; require the promotion-source guard and all CI/version checks; block force pushes and deletion; prevent direct updates by requiring PRs; avoid bypass actors unless repository policy explicitly names them.
- **Development branch (`develop`)**: require pull requests; require the complete PR CI check set; block force pushes and deletion; prevent direct updates. Require branches to be up to date only when the repository accepts the resulting CI reruns. If a merge queue is enabled, include the `merge_group` event in required workflows and verify its checks before making them mandatory.
- Add the exact stable check names reported by GitHub after the workflow has run. A renamed job changes the required check and can leave PRs permanently blocked.
- Keep release write permissions on trusted `push`/tag jobs only. PR workflows get read-only permissions and no signing secrets.

GitHub's base-branch filter does not enforce "main accepts PRs only from develop." The required source-branch guard provides that check in Actions; branch protection makes the guard mandatory. Check GitHub settings/API state separately and report it as unconfigured until verified. Do not claim a workflow file alone enforces branch protection.

When user asks to configure branch rules, use this order:

1. Inspect authentication, repository identity, existing repository rulesets, and effective rules for each target branch with read-only `gh` commands.
2. Build a proposed ruleset payload from actual workflow check names and current rules. Do not invent status-check contexts or discard existing bypass restrictions.
3. Before writing rules that can block merges or direct pushes, show the exact branch patterns, required checks, and bypass behavior. Apply only when user authorizes that configuration.
4. Create or update via `gh api` using the [repository rulesets API](https://docs.github.com/en/rest/repos/rules). Re-read configured and effective rules. Confirm `main` requires PR and promotion guard, `develop` requires complete PR CI, and both reject force push/deletion.
5. Verify source-branch enforcement with a PR check: a PR into `main` passes only for the same-repository `develop` branch; other branches and fork branches named `develop` fail. Rulesets match target refs; workflow guard enforces source branch and repository identity.

For pull request creation, inspect destination repository, head branch, base branch, asset URLs, and checksums first. Workflow jobs create promotion and tap PRs; outside a workflow, use `gh pr create` only when the user requested it. Do not merge unless explicitly asked.

## Safety rules

- Inspect current worktree and preserve unrelated edits. Never reset, clean, or stage broad paths.
- Never weaken required checks to make a workflow green. Report platform jobs deliberately disabled and why.
- Never invent a release version or bump every feature PR by default. Follow the repository's version authority and bump timing; if none exists, propose a concrete policy before wiring automation.
- Never expose signing credentials to fork PRs or untrusted workflow contexts.
- Never publish unsigned artifacts when signing/notarization is part of the product contract.
- Never update a Homebrew formula/cask before release asset URL and checksum are verifiable.
- Never push directly to a package tap's protected/default/target branch; the release job submits a PR to the configured target (default `develop`).
- Treat release creation, asset upload, and tap PR as separate stages. Verify each stage independently and report partial completion.
- Do not invent action inputs, branch names, secrets, artifact paths, supported platforms, or successful checks. Verify against repository config and official action documentation when an action's behavior is uncertain.
