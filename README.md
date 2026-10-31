# required-github-workflows

Centrally maintained GitHub Actions workflows that every GlueOps repository is
expected to run. Consuming repos call them via `workflow_call` rather than
copying them, so a fix here lands everywhere at once.

Please point to a tagged release.

## Workflows

### `glueops-basic-pr-checks.yml` — GlueOps Standard Checks

Validates that a pull request will produce the release
[release-please](https://github.com/googleapis/release-please) is expected to
cut. It checks the PR title against the conventional commit types
release-please actually recognizes.

#### Usage

```yaml
name: "GlueOps Standard Checks"

on:
  pull_request:
    types: [opened, edited, synchronize, reopened]
  merge_group: # only needed if the repo uses a merge queue

jobs:
  PR_CHECKS_AND_LABELS:
    uses: GlueOps/required-github-workflows/.github/workflows/glueops-basic-pr-checks.yml@<tag>
    secrets: inherit
```

Most repos don't call it at all: the `glueops` org ruleset runs it from this
repo's `main` on every PR and merge queue run.

> [!IMPORTANT]
> Fixing a PR title doesn't re-run the check on its own when it runs through the
> org ruleset: rulesets ignore `types` and only trigger on `opened`,
> `synchronize` and `reopened`. Re-run the failed job (it reads the title fresh)
> or push a commit. Repos calling it with `workflow_call` should include
> `edited` in their trigger, as above, so a retitle re-runs it.

### Required in the calling repository

`.github/configs/labeler.yml` — distributed by
[`github-shared-files-sync`](https://github.com/internal-GlueOps/github-shared-files-sync).
`actions/labeler` resolves `configuration-path` against the **calling** repo, not
this one.

> [!NOTE]
> This file and the `Add default PR Labels` step are **deprecated**. The only
> label they apply, `include-in-release-notes`, exists for
> `.github/release.yml` — GitHub's auto-generated release notes config — which
> release-please ignores, since it writes its own release body from the
> changelog. They are retained only because existing repository rules still
> require them. See [Deprecations](#deprecations).

## What gets validated

Only the **PR title**. The org allows squash merges only, with
`squash_merge_commit_title: PR_TITLE` and `squash_merge_commit_message: PR_BODY`,
so the title is the commit release-please parses. Commit messages inside the PR
are deliberately not checked, since they never land.

In a merge queue (`merge_group`), the job runs and passes without re-checking:
there is no PR title there, and it was already validated on the PR.

### Accepted types

```
feat  fix  perf  deps  revert  docs  style  chore  refactor  test  build  ci
```

This is release-please's `DEFAULT_HEADINGS` set. Types outside it are **silently
dropped** — release-please logs the parse failure at `debug` level and the commit
simply never appears in the changelog. That silence is the reason this check
exists.

Format: `<type>[optional scope][!]: <description>`

```
feat: add support for tenant DNS
fix(api): correct pagination cursor
chore(deps): bump argo-cd to 2.13.1
feat!: drop v1 API
feat(api)!: drop v1 API
```

GitHub's "Revert" button titles the PR `Revert "..."`, which fails the check;
retitle it to `revert: ...`.

### Breaking changes

Put `!` after the type/scope in the PR title:

```
feat!: drop support for Kubernetes 1.28
```

`breaking:`, `major:` and `hotfix:` are **not** release-please types and are
rejected. release-please parses them as an ordinary type and falls through to a
patch bump, so a commit written that way produces a patch release instead of the
major one it looks like it is asking for.

A `BREAKING CHANGE:` footer in the PR description (it becomes the commit body)
also works, but only as the description's very last paragraph, so prefer `!`.

## Version bumps

release-please derives the bump from the commits, not from any label:

| Commit | Bump |
| --- | --- |
| `!` or `BREAKING CHANGE:` footer | major |
| `feat:` | minor |
| Any other recognized type | patch |
| Unrecognized type | none — the commit is dropped |

This workflow no longer applies `patch`/`minor`/`major`/`enhancement` labels.
They were consumed only by `.github/release.yml`, which release-please ignores —
it writes its own release body from the changelog. Renovate never read them
either; `auto-merge.json` gates on `matchUpdateTypes`, Renovate's internal
update classification, not GitHub labels.

## Deprecations

The `Add default PR Labels` step and `.github/configs/labeler.yml` are on their
way out. They apply a single label, `include-in-release-notes`, to every pull
request, and nothing consumes it: its only reader is `.github/release.yml`, and
this org releases through release-please, which never reads that file.

They remain only because existing repository rules still require them. To
retire: drop `.github/configs/labeler.yml` and `.github/release.yml` from
`github-shared-files-sync`, then delete the step from this workflow.

## Conventions

Third-party actions are pinned to a full 40-character commit SHA with the
version in a trailing comment, so Renovate can offer upgrades while the
resolved code stays immutable:

```yaml
uses: amannn/action-semantic-pull-request@48f256284bd46cdaab1048c3721360e808335d50 # v6.1.1
```

The job id `PR_CHECKS_AND_LABELS` is load-bearing: it forms the branch
protection check name `PR_CHECKS_AND_LABELS / PR_CHECKS_AND_LABELS` in every
calling repo. Renaming it silently breaks required status checks.
