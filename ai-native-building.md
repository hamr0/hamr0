# How I build

I'm not a software engineer. I direct, design, architect and review, and AI agents write the code. This is a reflection on how that has gone, what I got wrong, and what I learned. I'm just a guy with an agent, working on my own problems most of the time.

## Why I build

My goal was simple. Learn AI building, teach it to myself, and then solve the things that had frustrated me for years: privacy, simple tools, bloated products, over-engineering.

All of it came down to simplifying and open-sourcing everything. I hated how the internet became a cesspool of harvesting data, "data mansions where we are the serfs". Most products I use revolve around two or three features.

So I build pockets of resistance, one small product at a time. I enjoy solving problems. That is what I thrive on, pushing the boundaries one small solution after another.

There is an ethical floor under all of it: "I want people to use the internet, not to be used by it." The idea I keep coming back to is capability without custody. Borrow the rails that already exist: email, maps, git, chat. Then there is no account to breach and no profile to sell. That is not a feature list. It is a posture, and it is why knowless can't email you, why plato won't host a photo of you, and why late.fyi forgets your email after the trip. If people go dormant, I have no business pinging them. If a project dies because nobody needs it, maybe it is time to move on.

## How I started

I started on Replit, and I got addicted. The hook was believing I was one prompt away from done. The dopamine was a new, beautiful UI appearing every 30 seconds. I bumped my spending limit $100 at a time, and within weeks I had burned $1,000. I only realized it was a gambling loop when I started putting limits on myself.

What I got were facades. Gorgeous interfaces with no connected backend, untested APIs, and nothing that worked in production. I built a unified API platform twice. The first attempt was all UI creep. The second was half as bad, and the UI still got me again.

Then came the manual project manager phase. I researched in DeepSeek, planned in OneNote, and spoon-fed the plan to Replit. It was better, but about half my time went to guardrailing: keeping two documents in sync, filling in missing details, and the AI never seeing the code as it evolved. I kept assuming it would get it.

Then I found the three-step method on YouTube: write a PRD, generate tasks, process them one at a time. I moved to the Claude CLI and to localhost. For the first time I could see the whole app. I could read the files, run it, and ask the agent to write down what it had done after every step. It was much easier, and the course correcting dropped to a small fraction of my time.

A few realities stuck, and they still hold:

- AI can't read your mind. It predicts words. If you want something, you have to say it.
- Planning is your job, not the AI's.
- Every model has amnesia. Docs have to remember for it, because chat history doesn't.
- A leading question gets a yes. Ask neutrally: "what could go wrong?" instead of "is this good?"
- It doesn't catch its own mistakes. Something outside it has to check.
- You need curiosity about how the pieces fit. Without it you stall at 80%.

My idea of an MVP changed too. It used to be a polished UI. Now it is one core feature that works, with nothing around it that doesn't have to be there.

## Privacy

**wearehere.** It started with ten simple browser extensions, later folded into one. I thought I knew how data-hungry browsers were from the surface. I didn't. Going down those ten extensions horrified me. It strengthened my resolve to at least inform people when I can't block.

**privpn and privcloud.** privpn is a personal VPN: WireGuard through your own VPS, one script, no third party. It is the first time I drew a hard line: my data, my VPS, my key. privcloud came from Google Photos and iCloud. It is a self-hosted server with Immich for photos, Navidrome for music, file backup and Tailscale for remote access, and it replaces the cloud bundle a phone normally pulls you into.

**8een and zkagent.** The push for age verification with full disclosure pushed me to 8een, a zero-knowledge proof over the EU mDoc, and then to zkagent, where a passport or ID chip answers one question and discloses nothing else.

**knowless.** I also hated, wholeheartedly, the password-and-email mess, and needing auth for plato pushed it further. That led to knowless. The mechanics are short:

- The email is salted-HMAC'd at the boundary.
- A magic link and a 30-day cookie are the whole session.
- Only the hashed ID is stored at rest.
- I cannot contact users. That is by design.

Knowless itself took several passes over the same question: what actually tells a human from a robot online, and which layers we kept for reasons that no longer hold? "Passwordless" mostly meant passwordful with extra steps. Once the answer got small enough, it felt earned instead of cute.

**justabit.** Privacy led me to contribute to CAMARA, and justabit came from my roaming and interconnect background. It brings the same "answer, don't disclose" idea to telecom network APIs. An issuer says yes or no, and never hands over the value behind the answer.

**The IETF draft.** Big tech's influence, even on standards, is what led me to the IETF.

## Products

**addypin.** It was an old product of mine that I loved, and I wanted to build it properly. I built it four times. Version three had accumulated everything: email collection, country pins, telemetry, device IDs, analytics, and PIN auth with bounceback hell. I wiped it. 1,900+ commits, gone. Version four took three days, and it is far simpler, built on knowless. What drove the wipe:

> Servers often have logs already. 90% of software is layers upon layers of monetization and fluff, like serving you a server log on a golden plate and having you pay for the plate to extract data you already have. AI makes it cheaper to expose there wasn't much art or craft behind those layered services. They were just part of my brainwashed thinking: collect email, track devices, see what you can harvest by default. Doing unto others what was being done unto me.

**plato.** It came from Reddit frustrations. I needed a lightweight phpBB with a Reddit-style feel. I love Discourse's design. I hate how bloated and heavily moderated it is. plato runs at ownsub.com, and needing auth for it is what pushed knowless.

**late.fyi.** It came from my own train experience. I was juggling three apps and a browser tab to track late trains. Now I email a train number and get told when it changes. No app, no account, no noise when the train is on time.

**sawt.** I love reading and I hate the lack of Arabic audio. This is where I taught myself some Python. I broke Arabic letters down, added diacritics, then went to IPA, before finding out it wouldn't work. The second version was simpler, using AWS Polly. Then I ran PRD, tasks, execute on it and it bloated, and I had to restart. The third version is simpler again and has a fully working UI. It is a personal project that made me learn programming for a while.

**ama.** It came from finding links with Google, from a vendor telling me to google their own pages. So it is the AI I already pay for, on the page I'm already reading.

**gitdone.** It was born out of wanting to do something with email, like verifying I got one. Mainly it was a question: can I prove my personhood with something I have used for years? I have had the same email address for more than twenty years. That led to DKIM, attestation and a digital notary: multi-party email workflows, every reply verified and stamped, no accounts. It now runs as signedreply.com.

## Libraries and agent rails

**mcp-gov.** It came from terribic, a personal API. Early on I knew AI agents and MCPs were going to be huge. I thought I could consolidate everything under one API and apply rwx rules to it, to guard agents early. That was the wrong angle. It broke into mcp-gov, a weak form of governance built on word guesses, and then the substrate moved under it: MCP made APIs plumbing. But the thinking carried on, into the whole authorization-for-agents line. Bad angle, great learnings.

**bareagent, bareguard and litectx.** They came from aurora, my first orchestration attempt. bareagent is the agent loop by itself, once the ceremony was stripped. litectx is its memory, pulled out. bareguard is one gate every agent action passes through: allow, ask a human, or deny, before it runs.

**barebrowse and baremobile.** They came from mcprune. Snapshot, click, type.

**multis.** It came from OpenClaw. I wanted an agent that lives in my chat apps, but not one that is risky and wreaks havoc. multis controls my machine, remembers conversations and searches my documents, built on the rails above.

**beeperbox.** It also came from OpenClaw, which reaches 50+ messengers through half-baked reverse-engineered bridges. I liked Beeper, but it had no MCP for agents to talk to. So I built one and put Beeper's Electron app in a Docker container: one container, every messenger, one shape.

**mailproof.** It came from gitdone: the DKIM checks, attestation and notary work got offloaded into it. I rewrote it twice. v1 was a quick, dirty carry-over of gitdone's work, and it was close to unusable. I cleaned it up by asking what I really needed, and ended with a lightweight, local, private tool: prove an email really came from its sender, and keep that proof checkable offline. It is now something agents can act on, with a simple CLI and MCP, and it works with any email client.

**pulselog and flightlog.** They came from my VPS going down without me knowing. I didn't want Sentry. I wanted something that reads the logs my server already writes, reminds me before the Let's Encrypt TLS certificate runs out, and sends me a weekly digest of how things are going. flightlog catches the errors in-process; pulselog watches from outside: health, TLS, disk and backups. Both are the fluff stripped down to the core: read JSONL, emit JSONL.

**bareloop and fwdloop.** bareloop came from relayfact and adaptlearn, and fwdloop from bareloop. This is the trust seam: once agents do real work, you need to know the work is real. I started from deterministic and asked how to make a fluid workflow that changes with the job. The agent builds its own workflow, checks it can't touch decide if it's green, and the budget is a cap it can't change. fwdloop keeps a human in the loop: FDE on rails. The verifier is code, not an LLM judge, because a judge is flaky and token hungry. Running them hardened the libraries underneath, because the loops found a lot of issues.

**barebox.** It came from the Meta hackathon. Reads pass, file writes can be undone, and anything irreversible waits for a yes. Many sandboxes, one proxy, spawning in milliseconds. I built it with Meta's Muse, which was weak, so I had to guide it tightly with Claude, and it made me appreciate what Claude already takes care of.

**rwxmap and rwxlite.** The IETF draft led to rwxmap, and rwxmap to rwxlite. rwxmap labels every API operation as read, write or can't-be-undone, so an agent gate knows what a call does before it runs. Mechanical rules got it to about 85%. For the doubtful rows I used a paid judge. Then I shaped a small model in the shape of that judge: the same three questions, JSON answers and a confidence. Then teacher and student distillation. Every round was preregistered, with hashes written before scoring. It failed seven exams before one passed. The result is rwxlite: 23 MB, milliseconds per call, free, offline and private, and it lifts the exact score from 85% to 91% on a blind exam. The discipline mattered more than the model.

## From vibe coding to agentic coding

Replit was vibe coding: prompt, generate, iterate, and rarely read the code. It is fast and fun, and fine for a demo. It is also how I ended up with gorgeous facades and a $1,000 bill. Agentic coding is the same agent with structure around it: understand what is already there, plan, execute, test, validate. The human stays at both ends, the understanding and the validation, and the agent does the middle.

| Workflow stage | Vibe coding | Agentic coding |
|---|---|---|
| Starting / framing | Prompt | Understand and plan |
| Building | Generate | Execute and test |
| Refining | Iterate | Validate and understand |

The goal is to cover what I don't know. I'm not an engineer, so I can't rely on my own eye to spot a bad pattern, a missed reuse or a security hole. Three habits do that work for me:

- **Understand first, then spec and POC.** Before any code I make sure I understand the problem, then discuss the spec and a small POC with the agent. That is where my principles get written in, so every module carries them from the start: simple over clever, no dependencies unless they earn their place, every line has a purpose.
- **Modular POCs.** Each piece has to work on its own before it graduates, so a wrong assumption dies small and early instead of surfacing in the middle of a big build.
- **Self-review after every branch.** An agent scrutinizes the code that was just written, against those same principles, and hands back nits, ideas and changes. I decide what goes in. The check sits outside the author, because the author doesn't catch its own mistakes.

How I got here, one phase at a time. Each one is what I believed, then what I learned:

- **Prompting, on Replit.**
  - *Believed:* I'm one prompt away.
  - *Learned:* gorgeous facades that didn't work, and $1,000 gone.
- **Three steps, on Claude.**
  - *Believed:* PRD, tasks, execute, test-heavy, one PRD per piece, and the pieces will connect.
  - *Learned:* they didn't. Disconnected, non-working products and lots of tests, because things didn't connect and the AI assumed a lot of fluff. On aurora I wrote about 20 PRDs that way and spent the rest of the time making them talk to each other. Adding memory (a friction log, local memory and stashes) is what started holding it together.
- **Modules and POCs.**
  - *Believed:* build in modules, all on main. If the POC works, build it out, move on, test later.
  - *Learned:* better, but still bugs and design problems.
- **Skills and agent rules.**
  - *Believed:* my flow is mine to repeat by hand, and if it passes, it's right.
  - *Learned:* it repeats, and models lie more convincingly as they get better. Light skills first (code-review, release), turning every repetition into a skill. Then heavy skills: my standards became default agent rules, and self-review, branch-review and release catch smelly code, missed reuse and what I didn't think of, into a fix ledger. Ask "what did I miss?" and "what did you fake?"

Through every phase, I start by poking: a problem I'm sitting with, and a fix I think might work. Then we go look at what has been tried, who built what, why it failed, and which assumptions were load-bearing. Before, I built on intuition and found the prior art halfway through. Now I walk into a POC already knowing the scene. Some sessions are dry, and some ideas come fast but none of them bait me to bite. That is a signal too, a quieter one. If Claude is repeating itself, I'm not asking the right question yet. Then comes the PRD and the POC. The PRD locks an initial shape. The POC is where most of it changes, and sometimes the product dies there. I ask Claude to argue against me, and roughly half the time the feature loses, because I can do it myself, or do without it, or someone has already done it well enough.

Skills and agent rules are where I work now. I don't write the code, so what I catch in review is the work. I use branches heavily. Review fills in what is under-specified, what I would normally miss, and what I'm not technical enough to see. About half my usage is review. Memory still holds it all together, and I'm still improving it.

I keep learning the same way. Mostly YouTube talks and other people's challenges. Reddit, for new repos. And my own loops, which keep showing me where to improve.

Most of these didn't start as ideas. They fell out of other projects or out of friction. The way I noticed was retrospective: I'd build, then look back at what drew me in, and the next product was usually sitting in that residue.

## Still going

It is not finished. The question keeps moving. It started as "what's bloat?" Then it became "how do I trust an agent's work?" Then whatever is next. Every repo carries the seed of the next one, and the next one is usually hiding in what I just built.

I'm not saying this is the right way to build. It is the way that works for me, with the limits I have.

I don't think the answer is a bigger product. It is pockets of resistance: small, open, private, one small solution at a time. The decisions stay mine.

Do more with less.

— Amr

GitHub: [github.com/hamr0](https://github.com/hamr0) | LinkedIn: [linkedin.com/in/hamr](https://linkedin.com/in/hamr)
