# RFC 0003: Cover Release Please retagging, failed-release recovery and Tauri updater outputs

**Status:** Approved

## Summary

Fix and extend the Release Please guidance so a `feature → develop → main` flow keeps producing version PRs after the first release: relabel the merged version PR once the production release is published, recover a release that failed after its tag was pushed, and note two workflow and Tauri details that hid these failures.

## Problem

`references/versioning-and-bumping.md` section 6 says the `main` promotion creates the tag and "clears the pending label". With `skip-github-release: true` Release Please never creates that tag or release, so it never clears the label. Observed on a real repository: the merged version PR stayed `autorelease: pending` after v0.12.0, and every later run aborted with `untagged, merged release PRs outstanding`, so no further version PR was opened. A second failure, in the release workflow's first job, pushed the tag before failing and left no published release; a re-run skips because the tag exists. The Tauri updater needs a build flag and a manifest that the guidance does not mention.

## Goals

- Correct the stage-3 statement and add a stage that relabels the merged `autorelease: pending` version PR to `autorelease: tagged` after publish, gated on a push event and isolated in its own job.
- Add a recovery stage for "tag exists, release failed": fix through the normal PR path, then dispatch the release for the existing tag; never weaken rulesets or dispatch from another branch around environment limits.
- Note, inside the retag stage, that the default `run:` shell lacks `pipefail`.
- Add a Tauri updater row: build archives with `--config` (confirm the CLI honours it), and generate `latest.json` in the publish job because `tauri build` does not.
- Bump the skill version.

## Non-goals

- Repeating the existing `setup-node` cache note.
- Choosing a release tool or changing the existing flow model.
- Repository-specific history; the text stays general.

## Design

Edit `references/versioning-and-bumping.md` (stage 3 wording, two new stages before "Prove each stage separately", and that list's third item), `references/desktop-cli-release.md` (one Tauri table row) and `SKILL.md` and `skill-release.json` (version only; the `pipefail` note lives with the retag stage). Keep each addition short and general.

## Acceptance

- `node scripts/validate-skill.mjs <skill>` from the skillify skill passes.
- The corrected stage no longer claims the main promotion clears the label.
- `specify validate` and `sync-check` pass.
- The new text was derived from a run observed on a real repository; it has not been re-run against a fresh Release Please setup.
