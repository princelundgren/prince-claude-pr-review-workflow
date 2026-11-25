# prince-claude-pr-review-workflow — instructions for agents

This repository is public: everything in it, its commits and its pull requests are public. Write
nothing about the maintainer's own machines, accounts, private repositories or employer.

- **Every caller on `@v2` runs what `v2` points at**, and a `v1` caller what `v1` does. Tagging a
  release and moving its major tag is the maintainer's, by hand (README.md, "Versions"): prepare
  the commands, never push a tag.
- **This repository's own review reads the Claude token from AWS,** as `v2` does, so it fails
  without the `CI_SECRETS_AWS_ACCOUNT` secret.
- **The review lives in the `review` action**, and `claude-review.yml` runs it on a hosted
  runner. The workflow pins the action by commit SHA, since a called workflow's checkout is the
  caller's repository, so a `./` path would point into it. A change to `.github/actions/review/`
  lands in one commit, and a second commit moves the pin to it. Pull requests merge with a merge
  commit (squash and rebase are off), so the pinned commit lands on `main`.
- **The prompt's sources:** keep the attribution comment at the top of `review-prompt.md` true when
  you change the prompt (CC BY 4.0). `github-actions-hardening/` is github/awesome-copilot's skill
  with a shorter `description:`; keep its LICENSE.
- **CI** runs on GitHub-hosted runners only, never with `pull_request_target` or `workflow_run`.
  Lint locally with `actionlint`, and shellcheck the review action's `run:` blocks as `lint.yml`
  does, since actionlint skips a composite action's steps.
- **Claude reviews each pull request a person opens from this repository**, once it is out of
  draft, with the pull request's own version of the workflow: read its
  `## Claude review of <sha>` summary, and fix or answer each 🔴 and 🟡 before the maintainer
  merges. Every push starts another review, so fix a 🔵 only in a push that also fixes something
  else; otherwise reply why not and resolve its thread. Forks, drafts and bots' pull requests get
  no review.
