# Contributing

Contributions are welcome. These projects are small and opinionated on purpose, and these rules keep them that way. I maintain them alone, so replies can take a few days.

## Start with an issue

1. **Open an issue first**, before writing code.
   - **Bug:** what you ran, what you expected, what happened, the version, and any logs.
   - **Feature:** the problem you're trying to solve, not only the solution you have in mind.
2. **We agree on the design in the issue.** I don't add every feature. If something doesn't fit how I see the project, I'll say so and explain why. It's about the project's direction, not about you or your idea.
3. **Then open a PR** linked to that issue. A PR without an agreed issue will usually be closed with a pointer back here.

Typos and small doc fixes can skip the issue and go straight to a PR.

**Security problems:** don't post the details publicly. Use "Report a vulnerability" on the repo's Security tab. If that button isn't there, open an issue that only says you found one, with no details, and I'll set up a private channel.

## What a PR needs

- **One concern per PR.** Small and focused beats big and sweeping.
- **A test that proves it** (for code changes). It should fail without your change and pass with it.
- **The tests pass.** The repo's README says how to run them. Say what you ran and what it printed.
- **No new dependencies and no changes to what users call or configure** (functions, CLI flags, settings, tools) unless we agreed on it in the issue.
- **Docs updated** if behaviour changes.

## Using AI

AI assistance is welcome. What I ask for is agentic coding, not vibe coding: you understand the problem, the design and the solution, and you can explain any line of your PR. Generated code you haven't read or run isn't ready for review.

## How I review

I check out every PR locally and run it through my own review and release checks. I may ask for changes, sometimes more than once, until it all checks out. Sometimes I'll land a reworked version of your PR myself, with you credited as co-author.

## Other ways to help

PRs aren't the only contribution, and often not the most useful one:

- Bug reports with logs and steps to reproduce.
- Testing on setups I don't have (other hardware, OSes, receivers, clients) and reporting what happened.
- Fixes to docs.

---

Contributions are accepted under the repo's license. If a repo has its own CONTRIBUTING.md, that one wins.
