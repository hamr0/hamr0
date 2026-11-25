# Contributing

Contributions are welcome. These projects are small and opinionated on purpose, and these rules keep them that way. If a repo has its own `CONTRIBUTING.md`, that one wins.

## Start with an issue

1. **Open an issue first** — before writing code.
   - **Bug:** what you ran, what you expected, what happened, the version, and any logs.
   - **Feature:** the problem you're trying to solve, not only the solution you have in mind.
2. **We agree on the design in the issue.** I don't add every feature. If something doesn't fit how I see the project, I'll say so and explain why — it's about the project's direction, not about you or your idea.
3. **Then open a PR**, linked to that issue. A PR without an agreed issue will usually be closed with a pointer back here.

## What a PR needs

- **One concern per PR.** Small and focused beats big and sweeping.
- **A test that proves it.** It should fail without your change and pass with it.
- **The project's tests pass.** Say which command you ran and what it printed.
- **No new runtime dependencies** unless we agreed on them in the issue. Most of these packages have none, and I'd like to keep it that way.
- **No changes to the public surface** (exported functions, CLI flags, config, MCP tools) unless we agreed on them in the issue.
- **Docs updated** if behaviour changes.

## How I review

I check out every PR locally and run it through my own review and release checks. I may ask for changes, sometimes more than once, until it all checks out. Sometimes I'll land a reworked version of your PR myself, with you credited as co-author.

## Other ways to help

PRs aren't the only contribution, and often not the most useful one:

- Bug reports with logs and steps to reproduce.
- Testing on setups I don't have (other hardware, OSes, receivers, clients) and reporting what happened.
- Fixes to docs.
