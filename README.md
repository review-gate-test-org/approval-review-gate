# approval-review-gate

One of three repos that run the same three pull requests under a different approval path.
This repo shows the **custom required check** path: a GitHub App posts a `review-gate` commit
status, and the ruleset requires it in place of GitHub's built-in code-owner review.

Status key: **[verified]** = observed in this repo (evidence linked); **[unverified]** = expected
from the design or the lab repo, not observed here.

## What this path proves

A PR to `main` can merge when either:

- **code owners approved**: every changed file has an approval from one of its owners in
  [`CODEOWNERS`](.github/CODEOWNERS), or
- **Copilot approved** the head commit **and** every changed file is agent-approvable in
  [`AGENTOWNERS`](.github/AGENTOWNERS). Here, only `/backend/` is.

So a backend-only change can merge on Copilot's approval alone, while any change outside
`/backend/` (for example `README.md`) still needs a human.

## How it's configured

| Piece | Setting |
|-------|---------|
| [`CODEOWNERS`](.github/CODEOWNERS) | `*` → @venetrius @JacobiCamunda; `/backend/` → @review-gate-test-org/backend; `/client/` → @review-gate-test-org/frontend |
| [`AGENTOWNERS`](.github/AGENTOWNERS) | `/backend/ copilot` only |
| [`gate.js`](.github/review-gate/gate.js) | Decision logic. Reads `CODEOWNERS` and `AGENTOWNERS` from `main`, never from the PR. Posts `success` or `pending`, never `failure`. |
| [`review-gate.yml`](.github/workflows/review-gate.yml) | Runs `gate.js` from `main` on `pull_request_target`, `workflow_run`, a 15-min schedule and manual dispatch. |
| [`review-gate-trigger.yml`](.github/workflows/review-gate-trigger.yml) | Relays review events (no secrets, no permissions). |
| Environment `review-gate` | Limited to `main`. Holds the App key (`REVIEW_GATE_PRIVATE_KEY`) and client ID (`REVIEW_GATE_CLIENT_ID`). |
| GitHub App `review-gate-test-org-gate` (id 5441748) | Commit statuses: write; Contents, Pull requests, org Members: read. |
| Ruleset on `main` | Required checks `backend`, `client` (GitHub Actions) and `review-gate` (pinned to App 5441748); 1 approving review; **code-owner review off**; Copilot automatic review on. |

Some root paths are never agent-approvable, whatever `AGENTOWNERS` says, because Copilot code review
reads instructions and skills from the PR's head branch: `.github/`, `CODEOWNERS`, `docs/CODEOWNERS`,
root `AGENTS.md` / `CLAUDE.md` / `GEMINI.md` / `REVIEW.md`, and `.claude/skills/`, `.agents/skills/`.
The narrowed list comes from an injection experiment in the lab
([copilot-review-harness-playground#97](https://github.com/review-gate-test-org/copilot-review-harness-playground/issues/97)).
It is taken from lab PR
[copilot-review-harness-playground#17](https://github.com/review-gate-test-org/copilot-review-harness-playground/pull/17),
which is **still unmerged** in the lab (checked 2026-10-29).

```bash
node --test .github/review-gate/gate.test.js
```

## Observed outcomes

<!-- outcomes -->
_Pending: PRs not yet evaluated._
<!-- /outcomes -->

## Limits

- **Held runs on Copilot reviews.** GitHub holds workflow runs triggered by Copilot's review for
  manual approval ("Approve workflows to run"), so the gate can't react to the review event itself.
  See outcomes above for whether the prompt appeared here.
- **15-minute wait window.** The run started by the author's push polls for Copilot's review for up
  to 15 minutes. If Copilot is slower, the status stays `pending` until the next scheduled run
  (up to 15 more minutes) or a manual dispatch. [unverified here]
- **Stale status from schedule/dispatch runs.** These runs evaluate all open PRs without waiting. A
  run that starts before a push can post its result on the old head after a newer run, so the
  displayed status may lag until the next run. [unverified here]
- **App private key to protect.** Anyone with the key can post a green `review-gate` status on any
  repo the App is installed on (all repos in this org). The environment limits use to workflows on
  `main`, but key rotation and storage are on us.
- **Custom code to maintain.** `gate.js` re-implements CODEOWNERS matching and review semantics. Any
  drift from GitHub's behaviour is a bug we own.
- **The PR UI no longer says "code owner review required".** The built-in rule is off, so GitHub's
  merge box only shows a pending `review-gate` check. The reason is in the status description.
