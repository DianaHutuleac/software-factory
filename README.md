# Software factory (SSDLC, UPT)

## How the factory works

1. **Backlog**: a wish becomes a ticket with acceptance criteria and waits in the backlog.
2. **Write**: the writer agent implements the ticket on its own branch, tests first.
3. **Test**: CI runs the tests and checks on the pull request.
4. **Review**: the reviewer agent checks the pull request against the ticket.
5. **Deploy**: a human decides when the merged change goes to production.
6. **Watch**: the watcher agent monitors the running shop and reports problems.
