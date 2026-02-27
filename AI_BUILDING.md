# I Spent a Year Building What AI Agents Are Missing

**Amr Hassan**
GitHub: [github.com/hamr0](https://github.com/hamr0) | LinkedIn: [linkedin.com/in/hamr](https://linkedin.com/in/hamr)

---

In 2025, I started using AI coding assistants and noticed something surprising: these tools were brilliant at writing code but terrible at doing anything real. They could not browse the web without hallucinating. They could not remember what happened five minutes ago. They could not use tools safely. And every framework that tried to fix these problems came wrapped in hundreds of megabytes of dependencies and abstractions.

So I started building. Not a single project -- a chain reaction. One problem led to the next, and each solution became a building block for whatever came after. Since then, that chain produced nine open-source frameworks. Every one of them exists because the previous one revealed a new gap.

Here is how the chain unfolded.

---

**It started with a failed idea.** Early on, I tried to build a unified API layer -- one interface to rule all the tool connections that AI agents need. The entire tech industry was moving toward API abstractions, and I thought I could simplify it. That project did not work. But it taught me the MCP protocol inside and out, and it planted the seed for **mcp-gov**: a governance layer that sits between an AI agent and its tools, preventing it from accidentally deleting files, dropping databases, or taking any destructive action. mcp-gov was born from failure, and it turned out to be one of the most important things I built.

**Then came the token problem.** I was watching AI agents browse the web and hemorrhage tokens. A single web page would cost 100,000 tokens when the agent only needed a fraction of that. I thought about how humans actually browse: you do not see the entire page, you see a focused window -- the button you want to click, the price you want to read, the form you want to fill. Everything else is noise. So I built **mcprune**, a pruning layer based on accessibility trees. It strips a page down to what matters -- the same focused view a human has -- cutting token costs by 75 to 95 percent. I also built two modes: one where the agent acts (clicks, types, navigates) and one where it just reads and extracts information, because sometimes you want an agent to browse and sometimes you want it to study.

**mcprune led straight to barebrowse.** Once I had proven that you could prune a page down to its essential information, the next question was obvious: why not build the entire browsing layer myself? The standard approach -- Playwright -- bundles a 200-megabyte browser, gets detected as a bot immediately, and locks you into its abstractions. I built **barebrowse** from scratch: it uses whatever browser is already installed on your machine, with your real cookies and login sessions. The agent browses as you, not as a robot. And the pruning I had proven in mcprune was baked in from day one. Three thousand lines of code, zero dependencies, passes every bot-detection obstacle course I have thrown at it.

**barebrowse led to baremobile.** If an agent can browse the web, why can it not use a phone? I built **baremobile** -- same philosophy, same patterns, same API shape, but for Android devices. An agent can tap, type, scroll, read screens, launch apps, even run directly on the phone itself through Termux. I reused barebrowse's pruning and snapshot patterns. Same building blocks, new surface.

**baremobile led back to bareagent.** Now I had browsing and mobile, but nothing to coordinate them. I built **bareagent** -- a lightweight orchestration engine that handles the think-act-observe loop, planning, retries, and failure recovery. It ties everything together. And because I had already built the pattern twice (browser, mobile), the orchestrator practically designed itself.

**bareagent led to multis.** Once I had an orchestration engine, I wanted to prove it could power a real product. Inspired by OpenClaw -- the idea of a personal assistant without enterprise bloat -- I built **multis**: a personal and small-business assistant that runs on a Raspberry Pi or an Android phone. It uses bareagent under the hood, which uses barebrowse and baremobile, which use the pruning from mcprune. Every layer I had built became a building block for the next.

---

**What kept the whole chain from falling apart.**

While I was building all of this, two projects ran quietly in the background as glue.

**Aurora** is a memory-aware framework based on ACT-R, a cognitive science model of how human memory works. It gives AI agents persistent memory that strengthens with use and fades with time -- the way ours does. But what matters for my own workflow is its code intelligence: Aurora hooks into the language server as a pre-edit check, so before my AI assistant modifies a file, it already knows what imports that file, what depends on it, what might break. It keeps me from shipping broken code into production. Aurora does not just give agents memory -- it gives them awareness of the codebase they are working in.

**Liteagent** is the other piece of glue. It runs friction analysis on my coding sessions -- scanning stashes, patterns, and mistakes -- and surfaces that context as hot memory. When the AI assistant's context window compresses (which happens constantly in long sessions), liteagent helps me recover. It tells the agent: here is what you were doing, here is what went wrong last time, here is the direction you should go. Without liteagent, I would lose hours to context resets. With it, I pick up where I left off.

Together, aurora and liteagent are why I can ship at the pace I do. They are not just projects -- they are the infrastructure that supports building everything else.

---

**How I build.**

The chain moved fast because the process is disciplined, not because I cut corners.

**POC first, always.** Every project starts with a 15-minute proof of concept -- the happy path plus a couple of edge cases. If the idea survives that, I design it properly and build it with tests. If it does not, I kill it and move on. The failed API project died this way. mcp-gov, mcprune, and barebrowse all survived. The POC never ships -- it just proves the idea is worth building for real.

**Vanilla first, dependencies last.** I follow a strict hierarchy: write it yourself in plain language first, then reach for the standard library, then -- only when the stdlib cannot do it in under 100 lines -- consider an external package. That external package must be maintained, lightweight, and widely adopted. The only exception is security-critical code (crypto, auth, sanitization), where vetted libraries are mandatory. This is why the entire bare suite runs on zero dependencies: it turns out you do not need them when you actually understand the protocol underneath.

**Test behavior, not implementation.** I follow the testing trophy, not the testing pyramid: few unit tests (only for pure logic and algorithms), many integration tests (real components working together), some end-to-end tests (critical user journeys). Tests should give you confidence to refactor freely -- if changing internal code without changing behavior breaks your tests, those tests are liabilities. The bare suite has 250+ passing tests across the three packages, and I can restructure internals without touching a single test file.

**Build incrementally.** After the POC graduates, I break the work into small independent modules. One piece at a time, each must work on its own before integrating with the next. This is why I could build barebrowse as 13 independent modules that each do one thing -- and why baremobile could borrow patterns from it without inheriting complexity.

---

**The pattern underneath all of it.**

None of these projects were planned as a portfolio. Each one was an itch I could not ignore -- a problem I hit while building the previous thing. And every time, I looked at what the rest of the industry was building, borrowed the ideas that worked, and obsessively simplified them into clean, open-source frameworks with no dependencies and no bloat.

The failed API project taught me MCP, which led to mcp-gov. mcprune taught me accessibility trees, which became barebrowse's core. barebrowse's patterns became baremobile's patterns. bareagent tied them together. multis proved the stack works end to end. Aurora and liteagent kept me productive while I built the rest.

This is the latest expression of a pattern that has run through my entire career. A decade at Vodafone building roaming products across six countries. A PM role at Meta. Consulting engagements and startups where I learned to build under extreme constraints. At Liberty Global, I took a black box and made it transparent in four months. Each phase taught me principles I applied to the next one. The AI frameworks are what happens when that compounding curiosity meets a domain full of unsolved problems.

All nine projects are open source, published on GitHub and npm, with full test suites. They represent over 1,000 contributions in the past year. I am not theorizing about what AI agents need. I built it, one problem at a time, each solution becoming the foundation for the next.

Whether you are looking for someone to build alongside you, consult on AI agent infrastructure, or join your team full time -- this is what I bring: hands-on depth in the problems that actually matter for AI agents, the curiosity to keep finding the next one, and 20 years of shipping under constraints to back it up.

I would love to talk about what you are building and where these building blocks might fit.

-- Amr

GitHub: [github.com/hamr0](https://github.com/hamr0) | LinkedIn: [linkedin.com/in/hamr](https://linkedin.com/in/hamr)
