# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Further documentation

The official FOG Project docs describe these workflows in more depth than this file does:

- [Stable Release Workflow](https://docs.fogproject.org/en/latest/development/stable-release-workflow/)
  — the `stable-releases.yml` pipeline end to end.
- [Version Sync Automation](https://docs.fogproject.org/en/latest/development/version-sync-automation/)
  — the `fog-version.sh` / `apply-fog-version.sh` design and how `check-fog-version.yml` and the
  local `pre-commit` hook both drive off it.

Consult these before making non-trivial changes to either workflow — this file summarizes intent,
those pages are the maintained source of truth.

## What this repository is

`fog-workflows` contains **no application code**. It is a standalone GitHub Actions
repository that hosts the release-automation and cross-repo maintenance workflows for the
[FOGProject/fogproject](https://github.com/FOGProject/fogproject) repo (and related repos
like `fogproject-install-validation`). It exists separately from `fogproject` so that these
workflows can run on their own schedule, independent of activity/permissions in the main repo.

There is no build, lint, or test tooling here — all "development" is editing YAML workflow
files under `.github/workflows/` and validating behavior via `workflow_dispatch` runs or by
reading Actions run logs/summaries in GitHub.

## Repository layout

- `.github/workflows/` — the actual product of this repo (see below).
- `badges/*.json` — [shields.io endpoint badge](https://shields.io/badges/endpoint-badge) JSON
  files (`dev-branch.json`, `working-1.6.json`, `stable.json`). These are read by badges in
  `fogproject`'s README and are committed to by the workflows themselves via the GitHub API
  (not by hand) whenever a version changes.
- `README.md` — one-line repo description.

## Workflows and how they relate

All three workflows authenticate to the `FOGProject` org as a GitHub App (`create-github-app-token`
using `vars.FOG_WORKFLOWS_APPID` / `secrets.FOG_WORKFLOWS_PRIVATE_KEY`), not the default
`GITHUB_TOKEN`, because they need to act across repos (`fogproject`,
`fogproject-install-validation`, and this repo itself).

- **`check-fog-version.yml`** — the core drift-correction workflow. Runs daily
  (`10 10 * * *`) and sweeps every "watched" `fogproject` branch (`dev-branch`, `working-1.6`,
  and any `rc-*`/`feature-*` branch), or a single branch when given `branch` as a
  `workflow_call`/`workflow_dispatch` input. For each branch it:
  1. Checks out that branch of `fogproject`.
  2. Runs `fogproject`'s own `.githooks/lib/fog-version.sh` (the same script its local
     `pre-commit` hook uses) to compute the correct `FOG_VERSION`/`FOG_CHANNEL` — this repo
     deliberately does not duplicate that math.
  3. If drifted, runs `fogproject`'s `.githooks/lib/apply-fog-version.sh` to patch
     `packages/web/lib/fog/system.class.php`, commits as the GitHub App bot, and pushes
     directly to that branch (no PR — mirrors what the local hook does).
  4. For `dev-branch` and `working-1.6`, also commits an updated `badges/<branch>.json` in
     *this* repo via the Contents API.
  - It is a **scheduled** trigger, not push-triggered, on purpose: an earlier push-triggered
    design (a stub in `fogproject` calling this as a reusable workflow) caused a runaway loop —
    the bot's own fixup push re-triggered the stub, which re-ran this workflow, which pushed
    another fixup, producing ~30 unwanted commits in ~20 minutes. See the comment block at the
    top of the file before changing the trigger model.
  - Never runs against `stable` — that branch's version is owned exclusively by
    `stable-releases.yml`.

- **`stable-releases.yml`** — the monthly release pipeline (`11 11 11 * *`, i.e. 11:11 UTC on
  the 11th — intentionally ~1 hour after `check-fog-version.yml`'s daily run so it never reads
  a stale version on release day). Chained jobs:
  1. Calls `check-fog-version.yml` for `dev-branch` to make sure its version is current first.
  2. Opens a `dev-branch` → `stable` PR in `fogproject` titled with the release tag (skips
     everything below if the computed tag matches the previous release tag).
  3. Triggers `fogproject-install-validation`'s `run_all_distros.yml` and polls for its result.
  4. Merges the PR only if validation passed; closes it (and posts to Discord) if not.
  5. Tags `stable`, builds release notes (commit log minus routine/merge commits, closed
     issues since the previous tag, new security advisories — and patches those advisories'
     `patched_versions`), and publishes a GitHub Release.
  6. Updates `badges/stable.json` here, then opens/merges a `stable` → `dev-branch` PR to sync
     the release commit back.
  7. Announces success/failure to Discord (`secrets.DISCORD_WEBHOOK`) at the relevant steps.
  - Every job needs its own App token scoped to whichever repo it touches
    (`fogproject` vs `fogproject-install-validation`) — tokens aren't shared across jobs.

- **`fog-release-immortality.yml`** — monthly no-op (`workflow_dispatch` + `20 0 1 * *`) that
  exists solely to keep GitHub from disabling the *other* two workflows' `schedule` triggers
  for inactivity (GitHub auto-disables scheduled workflows after 60 days with no repo activity).
  Uses `PhrozenByte/gh-workflow-immortality` against this repo.

## Conventions to preserve when editing workflows

- Prefer a GitHub App token (`create-github-app-token`) scoped to only the repositories a job
  actually touches, over the default `GITHUB_TOKEN` or a broadly-scoped PAT.
- Don't reintroduce push-triggered cross-repo workflows for version syncing — use scheduled
  and `workflow_call`/`workflow_dispatch` triggers instead, per the incident documented in
  `check-fog-version.yml`.
- Keep version-computation logic (`fog-version.sh` / `apply-fog-version.sh`) living in
  `fogproject`, not duplicated here — this repo should only *call* those scripts.
- Badge JSON files under `badges/` are shields.io endpoint format and are written by workflow
  steps via the GitHub Contents API (get current `sha`, then PUT); don't hand-edit them except
  for one-off corrections.
- Each matrix/job step that reports status should append to `$GITHUB_STEP_SUMMARY` rather than
  relying on job logs, so a run's overall state is visible from the Summary page alone.
