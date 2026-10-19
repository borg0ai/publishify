# RFC 0004: Route release guidance by product type and language

**Status:** Approved

## Summary

Publishify's goal is the `feature → develop → main` CI/CD: feature PRs are verified into `develop`, a version PR and a promotion PR carry the release to `main`, and the push to `main` ends in a GitHub Release of the CLI or app. Make that outcome fit the product by keeping one shared spine and adding per-stack profiles the agent loads after detecting the product type and language, instead of leaving stack-specific choices (version files, build, signing, updater) to the agent on each run.

## Problem

The first two repositories built with this skill, one shipping a Rust CLI plus a Tauri app and one a Wails app, came out very different: tag created before vs after CI, token names, version comparison inline vs a repo script, promote skipping vs failing on a pending version PR, formula vs cask, one release workflow vs several. Some differences are real product differences, some are skill-version drift, and some are choices the skill left open. A shared spine fixes the last two; per-stack profiles cover the first. A CLI and a desktop app also need different build matrices, signing, update paths and package types, but the skill carries one mixed guide plus a framework table in `desktop-cli-release.md`.

## Goals

- Detect in `SKILL.md` step 1 each shipped product (CLI or app) and its stack from concrete signals (for example `tauri.conf.json`, `wails.json`, `Cargo.toml` binaries, `package.json` `bin`), then load the matching profile. A repository with several products loads several profiles.
- Build one release workflow for all products, triggered by the push to `main`, with these jobs in order:
  1. Verify the exact `main` commit: rerun CI or confirm the required checks passed for that SHA.
  2. Validate the version against the source and compute the tag.
  3. One build job per product, from its profile.
  4. Verify the built assets.
  5. Publish: create the tag and GitHub Release at that commit, only after every build and verification succeeded. A failed build leaves no tag, so a re-run works.
  6. Version-tool bookkeeping after publish (for Release Please, flip the merged version PR from `autorelease: pending` to `autorelease: tagged`).
  7. For any macOS asset, open the `luban-homebrew` PR (mandatory tap, base `develop`; formula for a CLI, cask for a macOS app) only after the release is public and verified. Linux Homebrew is planned and not covered yet.
- Jobs 6 and 7 are idempotent and re-runnable on their own, so a failed tap PR never forces a second publish. `workflow_dispatch` on an existing tag rebuilds that tag's source and re-runs only jobs that are missing; it never moves the tag or flips labels for a newer version PR.
- Before relying on a release-created tag, read the repository's tag rules (`gh api repos/{owner}/{repo}/rulesets`, checking rulesets whose target is `tag`). If `v*` tags are protected, report it and ask the owner to allow the release job; never weaken the ruleset or push the tag from elsewhere.
- Keep separate only the workflows whose triggers differ: PR CI, version PR (push to the development branch), and promotion (version PR merged).
- Shared spine for every profile: `develop → main` automation, tag validated against the source version, public-asset verification before bookkeeping, actions pinned to SHAs, the bot token named `RELEASE_BOT_TOKEN`, promote fails on a pending version PR, `--repo` on every `gh` call.
- Give each profile a short fixed contract of six fields: version files and `extra-files` selectors, build and package targets, signing and notarization, updater and manifest, `luban-homebrew` distribution for macOS assets (formula or cask; Linux later), and stack-specific CI traps.
- First profiles are limited to stacks with a verified repository: Tauri (with its Rust CLI crates) and Wails. Add Go CLI, Node CLI, Rust CLI-only and Electron only when a real repository needs them.
- A profile may override a spine default only where the stack requires it, and says why.

## Non-goals

- Do not invent profiles for stacks no repository ships yet; add one when a real repository needs it.
- Do not rewrite working repositories to match; existing release behaviour is preserved and differences are reported.
- Do not put repository-specific names, tokens or credentials into a profile.

## Design

- `SKILL.md`: fold detection into step 1 in a few lines (no new step) and link `references/profiles/<stack>.md`. Frontmatter `description` names the stacks.
- `references/versioning-and-bumping.md` and `desktop-cli-release.md` stay as the spine. First pass only adds profiles and the spine rules; duplicated rows move out later, once profiles are stable.
- `references/profiles/`: one short file per stack, each with the six contract fields above. Profile text states rules only, with no repository names or history. A claim not verified against a real repository is marked unverified (at least: the `autorelease: tagged` label name and the per-SHA required-check query); the agent then checks it against official docs before writing the workflow.

## Acceptance

- `SKILL.md` detects product type and stack in step 1 and names the profile it loaded.
- Behaviour check: an agent using the skill plans releases for a Tauri repository and a Wails repository. Both plans have one release workflow, create the tag only in the publish job after builds succeed, validate the tag against the source version, verify public assets before relabel, pin actions to SHAs, use `RELEASE_BOT_TOKEN`, and make promote fail on an open pending version PR.
- A repository with a CLI and an app gets both build jobs in the same release workflow; a CLI-only plan has no app signing or updater job, and an app-only plan has no CLI archive matrix.
- Each profile lists the six contract fields; unverified claims are marked.
- Tag-rule check: the plan reads the tag rules before choosing how the publish job creates the tag, and reports a blocking rule instead of working around it.
- Homebrew check: the plan opens the PR against `luban-homebrew` with base `develop`, only after the release is public and verified, and only for macOS assets.
- `node scripts/validate-skill.mjs`, `specify validate` and `sync-check` pass.
