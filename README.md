# prince-claude-pr-review-workflow

[![Lint workflows](https://github.com/princelundgren/prince-claude-pr-review-workflow/actions/workflows/lint.yml/badge.svg)](https://github.com/princelundgren/prince-claude-pr-review-workflow/actions/workflows/lint.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

A reusable GitHub Actions workflow in which Claude reviews each pull request a person opens. It
posts inline comments on the lines it flags and one summary comment whose first line is
`## Claude review of <head sha>`, so whoever merges can check that the review saw the latest
commit.

It runs [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action) with a
review prompt and a locked-down setup, so each repository that calls it needs only a short caller
file and one secret.

## What it does

- **Hunts for defects instead of ticking a checklist:** it reads each changed file whole, follows
  every other use of what the change touches, lists what the change should have touched and
  didn't, and checks the pull request's text against the code, in seven areas: correctness,
  security, performance, gaps, false text, tests, and code quality and design.
- **Reads the repository's `AGENTS.md` first**, so its own rules apply to the review.
- **Holds a workflow change to a hardening checklist:** a change under `.github/workflows/` or
  `.github/actions/` is checked against the
  [`github-actions-hardening`](.github/actions/review-skills/github-actions-hardening/SKILL.md)
  skill.
- **Sorts what it finds** into 🔴 Critical Issues, 🟡 Suggestions, 🔵 Nits and ✅ Checked, and
  doesn't repeat an issue it already posted. Every push starts another review, so fix a 🔵 only
  in a push that also fixes something else; otherwise reply why not and resolve the thread.
- **Skips what it can't safely review:** drafts, pull requests from forks, and anything a bot
  opened or sent.

## How to call it

Add `.github/workflows/claude-code-review.yml` to your repository:

```yaml
name: Claude review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

permissions: {}

# A person's new push cancels the review still running, so only the latest commit is reviewed. An
# event a bot sends gets a group of its own, so it can't cancel a review: the called job skips it.
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}-${{ github.event.sender.type == 'User' && 'review' || github.run_id }}
  cancel-in-progress: true

jobs:
  review:
    permissions:
      contents: read
      pull-requests: write
    # @v1 moves on purpose: only the maintainer of prince-claude-pr-review-workflow can move its v*
    # tags (a tag ruleset), and each release then reaches every caller without a pull request here.
    uses: princelundgren/prince-claude-pr-review-workflow/.github/workflows/claude-review.yml@v1
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

- **The secret:** set `CLAUDE_CODE_OAUTH_TOKEN` on the repository. Pro and Max users generate one
  with `claude setup-token`
  ([claude-code-action's setup guide](https://github.com/anthropics/claude-code-action/blob/main/docs/setup.md)).
  Pass it by name, not with `secrets: inherit`, so the review gets nothing else.
- **The concurrency group belongs in the caller only:** a called workflow sees its caller's
  `github.workflow`, so the same group in both would cancel the caller.
- **An Actions allow list:** if your repository allows only selected actions and reusable
  workflows, allow this repository, `anthropics/claude-code-action`, `oven-sh/setup-bun`, which
  that action uses, and `actions/checkout`, unless actions created by GitHub are allowed.
- **No GitHub App needed:** Claude posts with the job's own `GITHUB_TOKEN`.

## What it guards

Claude reads text the pull request's author controls, so the workflow keeps the token away from it:

- No default token permissions, and a job token that can read the code and write to pull requests,
  nothing more
- Claude Code's subprocess environment scrub, so the commands Claude runs don't get the Claude
  token
- Reads blocked outside the checkout, and a refusal of any pull request with a symlink
- `--setting-sources user`, so settings a pull request adds under `.claude/` don't apply
- An exact allow list of the three `gh` commands the review needs
- GitHub-hosted `ubuntu-24.04`, never a self-hosted runner

Each step's comment in [`claude-review.yml`](.github/workflows/claude-review.yml) says why.

## Versions

- **`@v1`** is a moving tag: the latest release with the same caller contract. A change that
  needs callers to change gets `v2`. It trusts whoever can move the tag: a tag ruleset lets only
  the maintainer create, move or delete `v*` tags.
- **`@<commit sha> # v1.0.0`** locks a caller to one exact release. Each release also gets an
  immutable `v1.x.y` tag, so Dependabot can propose the next one.

A change to the review is first reviewed by its own version: this repository's
[`claude-code-review.yml`](.github/workflows/claude-code-review.yml) calls the workflow beside it.
Once it has merged, the maintainer tags the merge commit and moves `v1` to it:

```sh
git fetch origin && git merge-base --is-ancestor <sha> origin/main \
  && git tag -s v1.x.y <sha> -m "v1.x.y: <summary>" \
  && git tag -f -s v1 <sha> -m "v1 -> v1.x.y" \
  && git push origin v1.x.y && git push -f origin v1
```

## Contributing

Issues and pull requests are welcome. [`AGENTS.md`](AGENTS.md) has the conventions; `Lint
workflows` runs [actionlint](https://github.com/rhysd/actionlint) on every pull request.

## License

[MIT](LICENSE). Two parts come from elsewhere:

- The review prompt is adapted from GitHub Docs'
  [Review code](https://docs.github.com/en/copilot/tutorials/customization-library/prompt-files/review-code)
  prompt file, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); the comment above
  `prompt:` in `claude-review.yml` describes the changes.
- The `github-actions-hardening` skill is from
  [github/awesome-copilot](https://github.com/github/awesome-copilot), MIT, with a shorter
  `description:`; [its LICENSE](.github/actions/review-skills/github-actions-hardening/LICENSE)
  names the source commit.
