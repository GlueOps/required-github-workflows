# required-github-workflows

Centrally maintained GitHub Actions workflows that every GlueOps repository is
expected to run. Consuming repos call them via `workflow_call` rather than
copying them, so a fix here lands everywhere at once.

Please point to a tagged release.

## Workflows

### `glueops-basic-pr-checks.yml` — GlueOps Standard Checks

Validates that a pull request will produce the release
[release-please](https://github.com/googleapis/release-please) is expected to
cut. It checks commit messages and the PR title against the conventional commit
types release-please actually recognizes.

#### Usage

```yaml
name: "GlueOps Standard Checks"

on:
  pull_request:
    types: [opened, edited, synchronize, reopened]

jobs:
  PR_CHECKS_AND_LABELS:
    uses: GlueOps/required-github-workflows/.github/workflows/glueops-basic-pr-checks.yml@<tag>
    secrets: inherit
```

> [!IMPORTANT]
> The caller's trigger **must** include `edited`. GitHub's default
> `pull_request` types are `opened`, `synchronize` and `reopened` — without
> `edited`, retitling a PR to fix a failed title check never re-runs the check
> and the PR stays red.

No configuration file is required in the calling repository.

## What gets validated

Two independent checks, because which text reaches the default branch depends on
the merge strategy:

| Merge strategy | What lands on the default branch | Covered by |
| --- | --- | --- |
| Squash, multiple commits | The PR title | `Validate PR Title` |
| Squash, single commit | That commit's message | `Validate PR Title` (`validateSingleCommit`) |
| Merge commit | Every commit, verbatim | `Validate Conventional Commit Messages` |
| Rebase | Every commit, verbatim | `Validate Conventional Commit Messages` |

The squash cases matter because `squash_merge_commit_title` is
`COMMIT_OR_PR_TITLE`, which means the PR title becomes the commit message
release-please parses.

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

Messages beginning with `Merge `, `Revert `, `Reapply ` or `Initial plan` are
exempt — git and GitHub generate those.

### Breaking changes

Use `!` after the type/scope, or a `BREAKING CHANGE:` footer in the body:

```
feat!: drop support for Kubernetes 1.28
```

`breaking:`, `major:` and `hotfix:` are **not** release-please types and are
rejected. release-please parses them as an ordinary type and falls through to a
patch bump, so a commit written that way produces a patch release instead of the
major one it looks like it is asking for.

Because a PR title is a single line, `!` is the only way to mark a breaking
change there — a `BREAKING CHANGE:` footer has nowhere to live.

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

## Known limitations

`webiny/action-conventional-commits` fetches PR commits with a single
unpaginated request, so it validates only the **first 30 commits** of a pull
request. Its error handling also returns an empty commit list on any API
failure, which the action reports as `No commits to check, skipping...` and
exits successfully — the check fails open. Both behaviors are upstream.

Neither affects squash merges, where only the PR title reaches the default
branch and `Validate PR Title` covers it.

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
