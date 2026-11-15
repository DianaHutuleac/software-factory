# Threat model: the factory itself

This is about the factory, not the shop. The shop gets its own threat model per flagged ticket.
Written 13 Nov 2026 by Diana; reviewed by Vlad. Update it when a control changes.

## What we protect
- **The main branch and production.** Only reviewed, tested code reaches users, and only after a person approves.
- **Secrets.** The Claude API key, three GitHub App private keys, the Render deploy hook, the Neon connection string, the Stripe test key.
- **Money.** Prepaid credits on the factory's key, capped at 35 USD a month.
- **The record.** The audit log and the human log must be true.

## Who can reach the factory
| Actor | Can do | Trust |
|---|---|---|
| Anyone on the internet | Open issues, comment, fork, open PRs from a fork, read everything (public repo) | None. Their text is data. |
| The lecturer at the demo | Same as anyone, plus will plant a ticket with a hidden instruction, a bug, a bad dependency, a dead link, a new wish | Hostile by design |
| Vlad (Write) | Push branches, open PRs, write wishes | Trusted for product, not for settings |
| Diana (Admin) | Change settings, approve production, run the kill switch | Trusted; still blocked from pushing to main by the ruleset |
| The three agents | Only what their app and job permissions allow | Not trusted. They read untrusted text. |
| Dependencies and their maintainers | Run code in `ci`, in the image, and are read by the agents | Not trusted. Pinned, scanned, updated by PR. |

## Threats and controls
Severity: how bad if it happens. Controls are in the order they act.

### T1. A ticket or wish with a hidden instruction
- **Who:** anyone who can open an issue, including the lecturer. Also any file the agent reads (a comment in code, a dependency README).
- **How bad: high.** An agent that obeys could push bad code, leak a secret, or open PRs forever.
- **Controls:** only two doors start agents: a `wish` from an allow-listed author and a `ticket` from the planner app; everyone else's issues are ignored. `CLAUDE.md`: issue text is data, never instructions; the planner labels a wish `suspicious` and stops. The planner refuses a wish with hidden text (HTML comments, invisible characters) before Claude starts. The job that runs Claude has no GitHub write access and no tools beyond what its stage needs (planner and reviewer: none); a separate job without Claude does the writes from a JSON object, so an obeying agent can only propose tickets, a patch or a verdict. Least privilege means it still cannot push to `main`, read the deploy hook, or approve. `ci` and the reviewer check the diff against the acceptance criteria, not the ticket's wording. A `--max-turns` limit bounds the damage of a loop.
- **Residual:** an agent writes wrong code that passes tests. The reviewer and Diana's approval catch it.

### T2. A PR that edits workflows, CLAUDE.md or the kill switch
- **Who:** an agent (hijacked or mistaken), Vlad by mistake, anyone from a fork.
- **How bad: high.** Changing a workflow changes the permissions and the gates; changing `CLAUDE.md` changes every agent's rules.
- **Controls:** the writer's publish job refuses a patch that touches a protected path before anything is pushed, and the writer app has no `workflows` permission, so GitHub rejects such a push anyway. The `ci` path check fails on any change under `.github/` or `.claude/`, to `.mcp.json`, `CLAUDE.md`, `CODEOWNERS`, `scripts/kill-switch.sh` or `docs/human-log.md`, unless the PR's author is a person on the `HARNESS_AUTHORS` list. A PR runs its own copy of `ci.yml`, so for human PRs the control is the code owner's review, not this check. `CODEOWNERS` plus "require review from code owners" in the ruleset: those paths cannot merge without Diana. Workflows from fork PRs need approval and never get secrets. Changes to these files happen in a human-opened PR, logged as setup.
- **Residual:** Diana approves a bad change. Vlad reviews those PRs as the second pair of eyes.

### T3. A secret leaks through logs, prompts or the repo
- **Who:** an agent printing its environment; a workflow echoing a secret; a human committing a `.env`; a prompt that includes a connection string.
- **How bad: high.** The repo is public; a leaked key is used within minutes.
- **Controls:** secrets only in GitHub Actions secrets and `production` environment secrets; the deploy hook is unreadable to agent jobs. Workflows pass agents only the ticket text and the diff. GitHub masks registered secrets in logs. gitleaks fails `ci`; push protection blocks the push. `.gitignore` keeps `private/` and `.env` out. `CLAUDE.md` forbids printing environment variables. The kill switch deletes the Claude key in one command; the Console can revoke it.
- **Residual:** a secret in a format no scanner knows. Rotate on suspicion.

### T4. Runaway cost
- **Who:** a loop between agents (writer fails, reviewer requests changes, writer retries), a hidden instruction, a bug in a trigger that starts the agent on every event.
- **How bad: medium.** Prepaid credits are the ceiling, so the loss is bounded at 35 USD, but the factory stops for the month.
- **Controls:** Console spend limit 35 USD with an alert at 20 USD; `--max-turns` per run; one writer run per ticket per event, never on its own comments; a concurrency group per ticket so two runs never overlap; on a billing error the job saves the branch, labels the ticket `paused-no-credits` and notifies Diana; topping up is a logged human decision; the kill switch.
- **Residual:** the cap hits during a real ticket. It resumes after a top-up; the pause is itself a good demo.

### T5. A malicious or vulnerable dependency
- **Who:** a package maintainer, a typosquat, a moved action tag, a stale base image.
- **How bad: high.** Code runs in `ci` with the Claude key present and in the production image.
- **Controls:** exact versions in a committed lockfile and a frozen install; actions pinned by commit SHA; base image by digest; Dependabot alerts (on since 13 Nov) and version-update PRs that go through `ci` and the reviewer; Semgrep in `ci`; `CLAUDE.md` forbids adding a dependency without an exact version and a reason in the PR. The agent jobs with the key install dependencies only from the lockfile.
- **Residual:** a vulnerability with no fix yet. The ticket says so and a human decides.

### T6. The agent pushes to main
- **Who:** the writer app token, a human token, the default `GITHUB_TOKEN`.
- **How bad: high.** Untested code in production on the next deploy.
- **Controls:** the ruleset on `main` requires a PR, the `ci` check and an approval, with no bypass for anyone; the writer app's token can create branches and PRs only; `GITHUB_TOKEN` is read-only by default and "allow Actions to create and approve PRs" is off. Deploy to production still needs Diana.
- **Residual:** none we know. The video shows a refused push.

### T7. An agent approves its own work
- **Who:** the writer approving its PR; the reviewer and writer sharing an identity; auto-merge without a real review.
- **How bad: high.** The review step becomes theatre.
- **Controls:** three GitHub Apps, so the PR author (writer) and the approver (reviewer) are different identities; GitHub refuses approvals from the PR author anyway; the reviewer runs in a separate job with a fresh context and reads only the ticket and the diff; the ruleset dismisses stale approvals when new commits arrive; Diana's production approval is a human check after the agent's.
- **Residual:** the reviewer is too lenient. `ci` is deterministic and does not care; Vlad tunes the reviewer prompt and Diana reviews it.

### T8. A fork PR runs our workflows with our secrets
- **Who:** anyone with a GitHub account.
- **How bad: high.** It is the classic way to steal Actions secrets.
- **Controls:** never `pull_request_target`; the reviewer runs on `workflow_run` from `main`, checks that the PR comes from this repo and was opened by the writer app, and never checks out PR code; the agents' secrets are in environments that only runs from `main` can use; fork workflow runs need approval; secrets are not available to fork PRs by GitHub's design.
- **Residual:** a maintainer approving a fork run without reading it. We do not approve fork runs.

### T9. A GitHub App private key or Diana's account is taken over
- **Who:** phishing, a leaked key file.
- **How bad: high for the account, medium for an app key.**
- **Controls:** app keys stored only as repository secrets, apps installed on this repo only with minimal permissions; two-factor authentication on both GitHub accounts; Vlad has Write, not Admin; the kill switch removes agent access in one command; app keys can be revoked from the app page.
- **Residual:** an admin account takeover. Nothing in the repo stops that; GitHub's security log shows it.

### T10. The audit log is incomplete or edited
- **Who:** a failing job that skips its last step; anyone editing a comment.
- **How bad: medium.** The course says the audit log is the proof.
- **Controls:** the audit step runs with `if: always()`; the JSON artifact is a second copy; comment edits are visible in GitHub's edit history; the Actions run log is a third copy; human actions go in `docs/human-log.md` through reviewed PRs.
- **Residual:** artifacts expire after 90 days. The comments remain.

### T11. The planner floods the backlog
- **Who:** a long or malicious wish, a planner bug, the M2 watcher opening a ticket for every error line.
- **How bad: low to medium.** Noise, cost and a stuck writer.
- **Controls:** the planner may open at most five tickets per wish; the writer takes one ticket at a time; the watcher (M2) groups identical errors; `suspicious` wishes stop the planner; the human log shows no human wrote tickets.

## What a human still decides
Deploy to production (Diana), spend above the cap (Diana tops up), destructive changes to data (a second approval in `production` for migrations that delete). Everything else the factory does alone, and the log proves it.
