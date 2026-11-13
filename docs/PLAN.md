# Plan for M1: one ticket to production

Due Tue 1 Dec 2026, 23:59. One PDF on CV with a video link (10 minutes max) and one A4 page.
This file is the source for that page. Everything here is something either of us can explain at the demo.

## Our choices (decided 13 Nov 2026)

| Choice | What we chose | Why (the demo answer) |
|---|---|---|
| Team | Two people. **Diana Hutuleac** builds the harness: what agents may touch, the gates, the audit log, the cost cap, the kill switch. **Vlad-Andrei Mocan** builds how the agents work: planner, writer, reviewer, the product and its tests. He is the product owner. | Nothing is checked by the person who built it. The person who asks for a feature (Vlad) is not the one who lets it into production (Diana). |
| Track | Complex: a new product, a small online shop. | Accounts, two roles, personal data and Stripe in test mode exercise every harness item. Every feature is a ticket, so a lot goes through the factory, which is graded. |
| Stack | Vlad's choice, the language he knows best, as one server-rendered app with one Dockerfile and a test suite. Recorded here before session 4. | One deployable unit means one pipeline and one deploy. Small PRs that an agent can finish in one run. |
| Platform | GitHub, public repo `DianaHutuleac/software-factory`. | Protected branches and required reviewers for environments only work in public repos on free plans. Nothing secret lives in the repo anyway. |
| Hosting | Render, free web service, deployed from a Docker image. Database: Neon free Postgres. | Deployable from GitHub Actions with one deploy-hook URL. Render's free Postgres expires after 30 days; Neon's does not. |
| Agents and models | Planner: Sonnet. Writer: Claude Code GitHub Action, Sonnet. Reviewer: a separate job, fresh context, its own prompt and GitHub identity, Sonnet. Watcher (M2): Haiku. | Sonnet keeps the cost under the cap. The reviewer is a second agent because it has a fresh context, a different prompt and a different identity, not because it is a different model. Deterministic checks (tests, Semgrep, gitleaks) catch what a same-model reviewer might share as a blind spot. |
| Spend | A Claude Console API key with prepaid credits, used only by the factory. Cap: **20 USD per month**, email alert at 10 USD. | A number we can say out loud. Topping up after the cap is the human decision "spend above the cap". |
| Human gate | Diana is the required reviewer of the `production` environment. | A named person clicks approve. Logged every time. |

## The 8 harness items

For each: the mechanism (setting, file or command), what the M1 video shows, and the risk it stops.

### 1. Least privilege (must have)
- **Mechanism:** every workflow starts with `permissions: {}` and each job adds only what it needs (for example `contents: read`, `issues: write`). One GitHub App per agent (`factory-planner`, `factory-writer`, `factory-reviewer`), installed on this repo only, each with the smallest permission set for its stage. The Claude API key is passed only to the steps that call Claude. The Render deploy hook is an **environment secret** of `production`, so no agent job can even read it. The ruleset on `main` (item 2) applies to apps and admins alike.
- **Video:** the `permissions:` block of the writer job; the three apps' permission pages; a push to `main` with the writer's token being refused.
- **Risk stopped:** a hijacked agent (for example through a ticket with a hidden instruction) pushing to `main`, reading the deploy hook, or acting outside this repo.

### 2. Protected branch (must have)
- **Mechanism:** a repository ruleset on `main`: pull request required, required status check `ci`, one approving review required, stale approvals dismissed on new pushes, review from code owners required, no force push, no deletion, **no bypass list** (admins included). `CODEOWNERS` gives Diana `.github/`, `CLAUDE.md` and `scripts/`.
- **Video:** the ruleset page; `git push origin HEAD:main` refused for Diana too.
- **Risk stopped:** anyone, human or agent, changing production code without the pipeline and a review.

### 3. Human gate (must have)
- **Mechanism:** GitHub Environment `production` with Diana as required reviewer. The deploy job targets it and waits. Only after approval does the job get the deploy-hook secret and call Render.
- **Video:** the waiting deploy, Diana clicking approve, the live site changing.
- **Risk stopped:** an agent, or a merged bug, reaching users without a person deciding so. This is human decision 1, "deploy to production".

### 4. Pinned dependencies (must have)
- **Mechanism:** a committed lockfile with exact versions, installed in `ci` with the frozen-lockfile command for the stack (fails if the lockfile and manifest disagree); every GitHub Action referenced by commit SHA with the version as a comment; the Docker base image by digest; Dependabot version updates for the package ecosystem, Docker and GitHub Actions (Dependabot keeps the SHA pins current). Dependabot PRs go through `ci` and the reviewer like any other PR.
- **Video:** the lockfile, one SHA-pinned action, one digest-pinned base image, the Dependabot config.
- **Risk stopped:** a dependency silently changing between the test run and the deploy; a tag being moved to malicious code. Dependabot also gives the answer to "a vulnerable dependency" at the demo.

### 5. Secrets handling (must have)
- **Mechanism:** gitleaks runs in `ci` on every PR and fails the build on a match. GitHub secret scanning with push protection is on (since 13 Nov). Secrets live only in GitHub Actions secrets and environment secrets. Workflows pass the agents the ticket text and the diff, never a secret; the agent's environment holds only the Claude API key, and `CLAUDE.md` forbids printing environment variables or secret-shaped strings. Nothing from `private/` is committed (`.gitignore`).
- **Video:** a PR adding a made-up key turning `ci` red. If push protection blocks the push first, show that: it is the same control one layer earlier.
- **Risk stopped:** a key in a commit, a log line or a prompt that anyone can read in a public repo.

### 6. Audit log (must have)
- **Mechanism:** every agent job ends with a step that writes one record in two places: a comment on the ticket, and a JSON line uploaded as a run artifact. Fields: agent, model, action, PR or commit, time, ticket number, run URL, tokens and cost (from the action's execution output). The run itself stays in the Actions log. Human actions go in `docs/human-log.md`.
- **Video:** the M1 ticket with its comments from planner, writer and reviewer, and one JSON artifact opened.
- **Risk stopped:** not being able to answer "who did what, when, for which ticket". This is the proof the course asks for.

### 7. Cost cap (nice to have at M1, must have at M2)
- **Mechanism:** spend limit of 20 USD per month on the factory's key in the Claude Console, email alert at 10 USD. Every agent run has a `--max-turns` limit passed to the action. **Out of credits:** when the Claude step fails with a billing or spend-limit error, the job pushes the work-in-progress branch (progress saved), labels the ticket `paused-no-credits`, comments with the run URL, and assigns Diana so GitHub emails her. A `resume` workflow (manual, or every 6 hours) retries tickets with that label once a cheap probe call succeeds; the writer continues on the existing branch. Topping up is logged as human decision 2, "spend above the cap".
- **Video:** the spend limit page and the alert setting; if time allows, a paused ticket.
- **Risk stopped:** a loop or a prompt attack burning money; the factory stops, keeps its work, and tells a human.

### 8. Kill switch (nice to have at M1, must have at M2)
- **Mechanism:** `scripts/kill-switch.sh`: disables every agent workflow (`gh workflow disable`), cancels running agent runs (`gh run cancel`), deletes the Claude API key secret (`gh secret delete`), prints how long it took. Undo: `gh workflow enable` and re-add the key. Only Diana (code owner) may change the script.
- **Video:** running it, with the stopwatch time said aloud; the disabled workflows page.
- **Risk stopped:** an agent misbehaving while nobody can reach the settings fast enough.

## The first ticket, end to end

Suggested first ticket (Vlad decides as product owner): "a visitor sees the catalogue page: product name and price, and a message when there are no products". Small, no personal data, so no threat model, and it can finish in one agent run. The account ticket comes second and shows the threat-model flag.

Only two doors start agents: a `wish` issue from the product owner (or the watcher bot in M2), and a `ticket` issue from the planner app. Issues typed by anyone else do nothing.

| Step | Who | What happens | Human? |
|---|---|---|---|
| 1 | Vlad | Opens a Wish issue with the form. Label `wish` is added by the form. | **Human: product owner wish** (not one of the three decisions). Logged. |
| 2 | Planner (Sonnet) | Starts on the `wish` label if the author is on the allow-list. Reads the wish as data. Writes one or more `ticket` issues from the template, links them to the wish, adds `needs-threat-model` where accounts, roles, personal data or payments are touched, adds `suspicious` and stops if the wish tries to instruct the agents. Writes its audit record. | No |
| 3 | Writer (Sonnet) | Starts only if the ticket's author is the planner app. Creates branch `ticket-<n>`, writes a failing test, then the code, runs the tests, opens a PR with the `factory-writer` app token so other workflows run on it. Turns on auto-merge. Writes its audit record. | No |
| 4 | `ci` | Tests, Semgrep, gitleaks, frozen-lockfile check, protected-path check (fails if the PR touches `.github/workflows/`, `CLAUDE.md`, `scripts/kill-switch.sh`). | No |
| 5 | Reviewer (Sonnet) | A separate job, fresh context, `factory-reviewer` app. Reads only the ticket and the diff. Checks the acceptance criteria and `CLAUDE.md`. Approves or requests changes with a comment. Writes its audit record. | No |
| 6 | GitHub | Auto-merge fires when `ci` is green and the approval is in. | No |
| 7 | Deploy | Builds the image, waits in the `production` environment. | No, it waits |
| 8 | Diana | Reads the ticket, the audit comments and the diff. Clicks approve. | **Human decision 1: deploy to production.** Logged. |
| 9 | Deploy | Calls the Render deploy hook; the site updates. | No |
| 10 | Both | Add the row to `docs/human-log.md`. | Logging only |

If anything needs a human to touch code, nobody edits the PR. We fix the harness in a separate PR and the ticket runs again, and we write where and why in the human log.

Other human decisions, for completeness: "delete my account" will be a ticket whose deploy needs the usual approval, and whose destructive migration (if any) runs only after a second approval in `production`. That is human decision 3, "destructive changes to data", shown at M2 or the demo.

## Schedule to 1 Dec

| Dates | Work | Owner | Model |
|---|---|---|---|
| Fri 13 Nov | Session 1 (this PR). Merge it. Add Vlad as collaborator (Write). | Diana | Fable |
| Sat 14 to Sun 15 Nov | Vlad: stack choice written into this file; Render and Neon accounts checked. Diana: session 2, the agent workflows; create the three GitHub Apps. Each reviews the other's PR. The included Fable allowance ends 17 Nov 09:59 (check: Mon 16 or Tue 17). | Both | Fable |
| Mon 16 to Fri 20 Nov | Session 3, repo settings (ruleset, environment, CODEOWNERS, secrets). Session 4, app skeleton with health page, one test, Dockerfile; `ci` and deploy green. Vlad rewrites the three agent prompts; Diana reviews them against hidden instructions. | Diana / Vlad | Sonnet; Opus if stuck |
| Sat 21 to Wed 25 Nov | Session 5: the first wish, the first ticket, the first approved deploy. Fix the harness, never the PR. Check the audit records. | Both | Factory agents; Opus for debugging |
| Thu 26 to Fri 27 Nov | Session 6: kill switch, cost cap, out-of-credits handling. Harness review by Vlad against the 8 requirements. | Diana / Vlad | Opus |
| Sat 28 to Sun 29 Nov | Rehearse and record the video; each presents their own part. Run the kill switch on camera. | Both | none |
| Mon 30 Nov | The A4 page from this file and the human log. Upload (a day early). | Both | Claude chat |
| Tue 1 Dec 23:59 | Deadline. Every member uploads the same PDF. | Both | none |

## Gotchas we already know
- A PR opened with the default `GITHUB_TOKEN` does not trigger other workflows. The writer opens PRs with its app token.
- Never use `pull_request_target` with code from the PR. Agent jobs never run on PRs from forks.
- Dependabot PRs run workflows without repository secrets. The reviewer needs the Claude key as a Dependabot secret too, or dependency PRs wait for Diana's manual approval.
- Run artifacts expire after 90 days; the ticket comments are the copy that lasts to the January demo.
- Check the current docs of `anthropics/claude-code-action` before session 2 and pin it by SHA.
