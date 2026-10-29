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
| Copilot code review settings | *Allow Copilot to approve pull requests* on (UI only, no API) |
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
Observed 2026-10-29. Each PR got two Copilot reviews: the first ran with Copilot approvals **off**
(the default), the second after turning on *Allow Copilot to approve pull requests* (see Limits).

| PR | Change | Copilot verdict (1st → 2nd) | `review-gate` | Mergeable | Held "Approve workflows to run" |
|----|--------|-----------------------------|---------------|-----------|---------------------------------|
| A [#1](https://github.com/review-gate-test-org/approval-review-gate/pull/1) | backend only, benign | 🟢 Approval recommended (COMMENTED) → 🟢 **Approved** (APPROVED) | ✅ success: *Approved by copilot (all files agent-approvable)* | **Yes, no human** (`mergeable=true`) | Yes: 2 held runs |
| B [#2](https://github.com/review-gate-test-org/approval-review-gate/pull/2) | backend only, returns mutable backing list | 🟡 Changes recommended → 🔵 Needs a closer look (both COMMENTED) | ⏳ pending: *Needs code owner approval (@review-gate-test-org/backend); copilot has not approved* | **No**, blocked | Yes: 3 held runs |
| C [#3](https://github.com/review-gate-test-org/approval-review-gate/pull/3) | backend + README | 🟢 Approval recommended (COMMENTED) → 🟢 **Approved** (APPROVED) | ⏳ pending: *Needs code owner approval (...); copilot may not approve 1 file(s)* | **No**, blocked (review decision is `APPROVED`, gate still pending) | Yes: 2 held runs |

All rows **[verified]** via the REST API: reviews, commit statuses, `mergeable` / `mergeStateStatus`, and
workflow runs with conclusion `action_required` (for example [run](https://github.com/review-gate-test-org/approval-review-gate/actions/runs/40860928213) on A). The held runs were checked
through the API, not the PR UI. A's merge state is `unstable`, not `clean`, only because its first gate run
crashed (see Limits). That check is not required.

The green status on A was posted by a **manual dispatch** ([run](https://github.com/review-gate-test-org/approval-review-gate/actions/runs/40863754608)), not by an automatic trigger.
See Limits for why.

The three PRs differ in content from those in `approval-builtin-copilot`. They were opened before the PR
content was aligned across repos.
<!-- /outcomes -->

## Limits

- **Copilot approvals must be turned on.** By default Copilot only *recommends* approval in a
  COMMENTED review, which the gate ignores. Repo Settings → Copilot → Code review → *Allow Copilot
  to approve pull requests* has no REST API; it was set in the UI. **[verified]** The second reviews were
  `APPROVED` and counted toward the ruleset's required approval (`reviewDecision=APPROVED`).
- **Held runs on Copilot reviews.** Every `Review gate trigger` run started by Copilot's review ended
  `action_required` (7 of 7). So the gate can't react to the review event without a human clicking
  "Approve workflows to run". **[verified]**
- **15-minute wait window.** Only the run started by the author's push waits for Copilot's review.
  Here Copilot's first reviews arrived about 2 min after the push. The second, re-requested reviews came
  after any wait had ended, so nothing picked them up automatically. **[verified]**
- **The scheduled fallback did not fire.** The 15-min `schedule` trigger produced no run in about 42 min
  after the workflow landed on `main` (GitHub doesn't guarantee schedule timing, especially on new repos).
  A's green status needed a manual dispatch. **[verified]**
- **Transient API errors leave a stale status.** On A and B the push-triggered run crashed with
  `HTTP 503 Service Unavailable` while polling ([run](https://github.com/review-gate-test-org/approval-review-gate/actions/runs/40858341883), [run](https://github.com/review-gate-test-org/approval-review-gate/actions/runs/40858347023)). The poll's
  `pulls.get` is not inside `gate.js`'s error handling. The status stayed at *Waiting for copilot…* until
  the manual run, and the crashed run still shows as a failed (non-required) check. **[verified]**
- **Stale status from schedule/dispatch runs.** These runs evaluate all open PRs without waiting. One
  that reads a PR just before a push can post its result on the old head after a newer run. **[unverified here]**
- **Copilot's verdict is not deterministic.** The same commit of B got 🟡 and then 🔵. Both block. **[verified]**
- **App private key to protect.** Anyone with the key can post a green `review-gate` status on every
  repo the App is installed on (all repos in this org). The environment limits use to workflows on
  `main`, but storing and rotating the key is on us. **[by design]**
- **Custom code to maintain.** `gate.js` re-implements CODEOWNERS matching and review semantics. Any
  drift from GitHub's behaviour is a bug we own. **[by design]**
- **The PR UI no longer says "code owner review required".** The built-in rule is off, so the merge box
  shows a pending `review-gate` check. The reason is only in the status description. **[unverified: PR UI not inspected]**
