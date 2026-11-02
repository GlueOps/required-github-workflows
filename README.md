# required-github-workflows

Centrally maintained GitHub Actions workflows that every GlueOps repository is
expected to run. The `glueops` org ruleset runs them from this repo's `main`, so
a fix here lands everywhere at once.

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
    uses: GlueOps/required-github-workflows/.github/workflows/glueops-basic-pr-checks.yml@<commit-sha>
```

The repo has no tags, so pin a commit SHA. The workflow needs no secrets.

Most repos don't call it at all: the `glueops` org ruleset runs it from this
repo's `main` on every PR and merge queue run.

> [!IMPORTANT]
> Fixing a PR title doesn't re-run the check on its own when it runs through the
> org ruleset: rulesets ignore `types` and only trigger on `opened`,
> `synchronize` and `reopened`. Re-run the failed job (it reads the title fresh)
> or push a commit. Repos calling it with `workflow_call` should include
> `edited` in their trigger, as above, so a retitle re-runs it.

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

This is release-please's `DEFAULT_HEADINGS` set. A commit with any other type is
**silently dropped**: it never appears in the changelog, and merged on its own it
cuts no release (with other commits it only adds a patch bump). That silence is
the reason this check exists.

Format: `<type>[optional scope][!]: <description>`

```
feat: add support for tenant DNS
fix(api): correct pagination cursor
deps: update argo-cd to 2.13.1
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
rejected: release-please drops such a commit (see above) instead of cutting the
major release it looks like it is asking for.

A `BREAKING CHANGE:` footer in the PR description (it becomes the commit body)
also works, but only as the description's very last paragraph, so prefer `!`.

## Version bumps

release-please derives the bump from the commits, not from any label:

| Commit | Bump |
| --- | --- |
| `!` or `BREAKING CHANGE:` footer | major (minor below 1.0 with `bump-minor-pre-major`) |
| `feat:` | minor |
| Any other recognized type | patch |
| Unrecognized type | none — the commit is dropped |

This workflow applies no labels; the bump comes only from the commit.
`.github/configs/labeler.yml` (synced into repos by `github-shared-files-sync`)
is no longer read.

## Conventions

Third-party actions are pinned to a full 40-character commit SHA with the
version in a trailing comment, so Renovate can offer upgrades while the
resolved code stays immutable:

```yaml
uses: amannn/action-semantic-pull-request@48f256284bd46cdaab1048c3721360e808335d50 # v6.1.1
```

