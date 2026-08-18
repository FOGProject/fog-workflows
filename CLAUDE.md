# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Further documentation

The official FOG Project docs describe these workflows in more depth than this file does:

- [Stable Release Workflow](https://docs.fogproject.org/en/latest/development/stable-release-workflow/)
  — the `stable-releases.yml` pipeline end to end.
- [Version Sync Automation](https://docs.fogproject.org/en/latest/development/version-sync-automation/)
  — the `fog-version.sh` / `apply-fog-version.sh` / `update-language.sh` design and how
  `update-lang-fix-psr-and-sync-version.yml` and the local `pre-commit` hook both drive off it.

Consult these before making non-trivial changes to either workflow — this file summarizes intent,
those pages are the maintained source of truth.

## What this repository is

`fog-workflows` contains **no application code**. It is a standalone GitHub Actions
repository that hosts the release-automation, cross-repo maintenance, and pre-release
distro install-validation workflows for the
[FOGProject/fogproject](https://github.com/FOGProject/fogproject) repo. It exists separately
from `fogproject` so that these workflows can run on their own schedule, independent of
activity/permissions in the main repo.

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

`update-lang-fix-psr-and-sync-version.yml`, `stable-releases.yml`, and
`fog-release-immortality.yml` authenticate
to the `FOGProject` org as a GitHub App (`create-github-app-token` using
`vars.FOG_WORKFLOWS_APPID` / `secrets.FOG_WORKFLOWS_PRIVATE_KEY`), not the default
`GITHUB_TOKEN`. This is the standard for every job in this repo's centralized workflows, not
just cross-repo ones: it means every action taken by these unified workflows — dispatching
`run_all_distros.yml`, commits, PRs, releases — is consistently attributed to the same GitHub
App bot identity, rather than a mix of the bot and the generic `github-actions[bot]` depending
on which repo happens to be involved. Even same-repo-only jobs (e.g. the `run-install-tests` /
`check-all-tests-completed-successfully` jobs that just dispatch/poll `run_all_distros.yml`
here) get their own `create-github-app-token` step scoped to `fog-workflows` for this reason —
don't "simplify" those back to the default `GITHUB_TOKEN` just because the call is same-repo.

- **`update-lang-fix-psr-and-sync-version.yml`** — the core drift-correction workflow. Runs
  daily (`10 10 * * *`) and sweeps every "watched" `fogproject` branch (`dev-branch`,
  `working-1.6`, and any `rc-*`/`feature-*` branch), or a single branch when given `branch` as a
  `workflow_call`/`workflow_dispatch` input. For each branch, in one job (one commit, one push):
  1. Checks out that branch of `fogproject` (full history/tags — needed for step 3).
  2. Runs `fogproject`'s own `.githooks/lib/update-language.sh` (gettext regen) and
     `php-cs-fixer fix --rules=@PSR2` over the whole tree, staging whatever changed — the same
     scripts/tools its local `pre-commit` hook uses, so this repo doesn't duplicate that logic.
     `update-language.sh` (and its `require-tools.sh` companion) already exist on `dev-branch`
     and `working-1.6`; `rc-*`/`feature-*` branches cut before either landed there skip this step
     with a visible `::warning::` rather than failing the job.
  3. Runs `fogproject`'s own `.githooks/lib/fog-version.sh` to compute the correct
     `FOG_VERSION`/`FOG_CHANNEL` — this repo deliberately does not duplicate that math. Passes
     its `local` flag as `1` when step 2 staged changes (a commit is about to happen, same as
     when `pre-commit` calls it mid-commit) so the version already accounts for that commit
     instead of drifting again the next day, or `0` when nothing needs syncing.
  4. If drifted, runs `fogproject`'s `.githooks/lib/apply-fog-version.sh` to patch
     `packages/web/lib/fog/system.class.php`.
  5. If anything from steps 2/4 is staged, commits as the GitHub App bot and pushes directly to
     that branch (no PR — mirrors what the local hook does), with a message describing whichever
     combination of translations/PSR2/version actually changed.
  6. For `dev-branch` and `working-1.6`, also commits an updated `badges/<branch>.json` in
     *this* repo via the Contents API, using its own App token scoped to `fog-workflows`. Not
     `github.token`: under `workflow_call` that is the *caller's* token, which cannot write to
     another repository.
  - It runs on a **schedule** and on **merge**, and the difference between those two entry
    points matters. The schedule is the only cover for direct pushes, for merged fork PRs, and
    for `rc-*`/`feature-*`. The merge path is `fogproject`'s `sync-generated-files.yml`, which
    calls this over `workflow_call` on `pull_request: types: [closed]` for merges into
    `working-1.6`/`dev-branch`, and exists because the pre-commit hook is client-side and never
    runs for a PR merged in the web UI.
  - **`pull_request`, not `pull_request_target`** — worth knowing, because the latter looks like
    the better choice (it is the variant that gets secrets on a fork PR) and was tried first. It
    never fired once: GitHub reads a `pull_request_target` workflow from the repository's
    **default** branch (`stable`), not from the PR's base branch, so a stub on
    `working-1.6`/`dev-branch` is never consulted — it did not even register as a workflow, and
    four PRs merged without it running. `pull_request` is read from the base branch. The cost is
    that a merged fork PR has no secrets, so the stub's same-repo guard skips it and the
    schedule picks it up.
  - It is **not push-triggered**, on purpose: an earlier push-triggered design (a stub in
    `fogproject` calling this as a reusable workflow) caused a runaway loop — the bot's own
    fixup push re-triggered the stub, which re-ran this workflow, which pushed another fixup,
    producing ~30 unwanted commits in ~20 minutes. A merge event is safe where a push event is
    not, because the bot pushes directly to the branch and a direct push is not a PR merge — it
    cannot raise the event that calls this again. See the comment block at the top of the file
    before changing the trigger model.
  - Never runs against `stable` — that branch's version is owned exclusively by
    `stable-releases.yml`.
  - This used to be two separate workflows (a version check and a generated-files sync) on a
    30-minute schedule offset, trusting the gap to keep the version check from racing a regen
    commit. They're merged into one job now — see the comment block at the top of the file.

- **`stable-releases.yml`** — the monthly release pipeline (`11 11 11 * *`, i.e. 11:11 UTC on
  the 11th — intentionally ~1 hour after the daily sweep so it never reads a stale version on
  release day). Chained jobs:
  1. Calls `update-lang-fix-psr-and-sync-version.yml` for `dev-branch` to make sure translations,
     PSR2 formatting, and its version are all current first.
  2. Opens a `dev-branch` → `stable` PR in `fogproject` titled with the release tag (skips
     everything below if the computed tag matches the previous release tag).
  3. Triggers `run_all_distros.yml` (same repo — see below) and polls for its result.
  4. Merges the PR only if validation passed; closes it (and posts to Discord) if not.
  5. Tags `stable`, builds release notes (commit log minus routine/merge commits, closed
     issues since the previous tag, new security advisories — and patches those advisories'
     `patched_versions`), and publishes a GitHub Release.
  6. Updates `badges/stable.json` here, then opens/merges a `stable` → `dev-branch` PR to sync
     the release commit back.
  7. Announces success/failure to Discord (`secrets.DISCORD_WEBHOOK`) at the relevant steps.
  - Every job gets its own App token scoped to whichever repo it actually touches — tokens
    aren't shared across jobs. `run-install-tests` / `check-all-tests-completed-successfully`
    only dispatch/poll `run_all_distros.yml` in this same repo, so their token is scoped to
    `fog-workflows` itself rather than `fogproject`.

- **`fog-release-immortality.yml`** — monthly no-op (`workflow_dispatch` + `20 0 1 * *`) that
  exists solely to keep GitHub from disabling the *other* scheduled workflows' `schedule`
  triggers for inactivity (GitHub auto-disables scheduled workflows after 60 days with no repo
  activity). Uses `PhrozenByte/gh-workflow-immortality` against this repo.

- **`reusable_distro_workflow.yml`** — the actual FOG install-test logic for one distro,
  invoked via `workflow_call` by every `distro_*.yml` wrapper below. It:
  1. Ensures `crun` is >= 1.14.3 (apt on the `ubuntu-24.04` runner is often too old), upgrading
     via apt or, failing that, downloading a release from the `containers/crun` GitHub API
     (with a pinned fallback URL if the API call is rate-limited/fails).
  2. Derives a distrobox container name from the `distro` input (last path segment, `:`
     replaced with `_`).
  3. Installs Distrobox and creates a rootless-but-`--root` container from the given image,
     adding `git systemd <package_man> sudo` plus `libpam-systemd` when `package_man` is `apt`,
     plus any `additional_packages` input.
  4. Clones `FOGProject/fogproject` (`dev-branch`) and runs `bin/installfog.sh -y` inside the
     container as root.

- **`distro_*.yml`** (e.g. `distro_alma_10.yml`, `distro_debian_12.yml`) — one thin wrapper per
  distro/version. Each just sets `name`, triggers (`workflow_call` + `workflow_dispatch`), and
  calls `reusable_distro_workflow.yml` with `distro` (an OCI image ref), `package_man` (`apt`
  or `dnf`), and optionally `additional_packages` (e.g. Fedora needs `gawk`).
  - To add a new distro: copy an existing `distro_*.yml` that uses the same package manager,
    update `name`, the job id, and the `with:` block. No other file needs to change —
    `run_all_distros.yml` discovers new `distro_*.yml` files automatically.

- **`run_all_distros.yml`** — orchestrator, `workflow_dispatch`-only. Builds a matrix by
  listing every `distro_*.yml` file directly in `.github/workflows/` (a non-recursive glob,
  so it deliberately does **not** pick up `not-release-required-distros/`), dispatches each
  one via `gh workflow run`, then polls `gh run list` / `gh run view` for each to reach a
  terminal `conclusion`. At least one distro must pass for the overall run to succeed (see
  `aggregate-results` job) — this is deliberately lenient because flaky/rate-limited base
  images shouldn't block a release on their own.

- **`not-release-required-distros/distro_*.yml`** — additional per-distro wrappers (Fedora,
  Rocky, extra Ubuntu versions) that are *not* part of the release-gating matrix — they exist
  for standalone `workflow_dispatch` runs from the Actions tab only, since `run_all_distros.yml`
  doesn't glob into subdirectories. Otherwise identical in shape to the top-level `distro_*.yml`
  wrappers.

## Conventions to preserve when editing workflows

- Always use a GitHub App token (`create-github-app-token`), scoped to only the repositories a
  job actually touches, instead of the default `GITHUB_TOKEN` or a broadly-scoped PAT — even
  for jobs that only act within this repo. This is a deliberate standard, not an oversight: it
  keeps every action across all of this repo's unified/centralized workflows attributed to the
  same GitHub App bot identity, so run history and audit trails stay consistent regardless of
  which repo a given job happens to touch.
- Don't trigger version syncing from any event the sync bot's own push can raise. In practice
  that means **never `push`**, per the incident documented in
  `update-lang-fix-psr-and-sync-version.yml`: the bot pushes its fixup directly to the branch,
  a `push` trigger sees it, and the workflow feeds itself. Scheduled,
  `workflow_call`/`workflow_dispatch`, and `pull_request: types: [closed]` (a merge — which the
  bot's direct push is not) are all safe for the same reason, and all three are in use. The
  test to apply to a new trigger is that one question, not whether it happens to be a cron.
- A reusable workflow's `permissions:` block is a **request**, and a caller granting less is a
  hard startup error — `The workflow is requesting 'contents: write', but is only allowed
  'contents: read'` — not a silent capping. No jobs run and no logs are written, so it surfaces
  only as "a workflow file issue". Keep a reusable workflow's request as low as it genuinely
  needs (these workflows do their writing with App tokens, so `contents: read` is usually
  right); raising it forces every caller to grant the same, and breaks the ones that don't.
- Separately from that safety question, check **which ref a trigger is read from** before
  relying on it. Most events — including `pull_request_target`, `schedule` and
  `workflow_dispatch` — are read only from the repository's default branch, so a workflow file
  on a non-default branch is silently never consulted; `push`, `create` and `pull_request`
  resolve per-ref. A workflow that is correct but never runs looks exactly like one that ran and
  found nothing to do, so confirm a new trigger actually produced a run rather than assuming it.
- Keep version/translation-generation logic (`fog-version.sh` / `apply-fog-version.sh` /
  `update-language.sh`) living in `fogproject`, not duplicated here — this repo should only
  *call* those scripts. PSR2 formatting is the one exception, since fogproject only ever invokes
  `php-cs-fixer` directly rather than wrapping it in a script of its own.
- Badge JSON files under `badges/` are shields.io endpoint format and are written by workflow
  steps via the GitHub Contents API (get current `sha`, then PUT); don't hand-edit them except
  for one-off corrections.
- Each matrix/job step that reports status should append to `$GITHUB_STEP_SUMMARY` rather than
  relying on job logs, so a run's overall state is visible from the Summary page alone.
- Every `distro_*.yml` wrapper supports both `workflow_call` and `workflow_dispatch`, so it can
  run standalone from the Actions tab as well as via `run_all_distros.yml`.
- `package_man` is only ever `"apt"` or `"dnf"`; `reusable_distro_workflow.yml` branches on this
  string directly rather than using a lookup table.
- Distro-validation runners are pinned to `ubuntu-24.04`.
