# Human log

Every action a person took in this factory. The last column names one of the three human decisions
(**deploy to production**, **spend above the cap**, **destructive data change**), or **setup** for
building the harness, or **product owner wish** for a wish. Agents never write here. Times are Bucharest time.

| When | Who | Action | Decision |
|---|---|---|---|
| 2026-11-13 21:21 | Diana | Created the public repo `DianaHutuleac/software-factory` and pushed the first commit (README, `.gitignore` keeping `private/` out). | setup |
| 2026-11-13 | Diana | Turned on secret scanning with push protection and Dependabot alerts. | setup |
| 2026-11-13 | Diana | Set Actions workflow permissions to read-only; left "allow Actions to create and approve pull requests" off. | setup |
| 2026-11-13 | Diana, Vlad | Session 1: confirmed team split and the shop track; chose Render plus Neon, Sonnet for planner and writer and Opus for the reviewer, a 35 USD monthly cap with an alert at 20 USD, and audit records as ticket comments plus run artifacts. | setup |
| 2026-11-13 | Diana | Opened the session 1 PR (plan, factory threat model, agent rules, issue forms, this log). | setup |
| 2026-11-13 21:22 | Diana | Invited `pseudocod` (Vlad) as collaborator with Write access, not Admin. | setup |
| 2026-11-13 22:36 | Diana | Reviewed and merged PR #1 (session 1 documents) into `main`. | setup |
| 2026-11-13 23:10 | Diana | Session 2: opened the PR with the agent workflows (planner, writer, ci, reviewer, deploy, audit) and first drafts of the agent prompts, written with Claude Code (Fable 5.2). | setup |
