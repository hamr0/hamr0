# Doing more with less

**Amr Hassan**
GitHub: [github.com/hamr0](https://github.com/hamr0) | LinkedIn: [linkedin.com/in/hamr](https://linkedin.com/in/hamr)

---

I build small, local-first tools and privacy primitives — starting with **knowless**, an auth library that can't contact its users by design.

Most web products and software are full of fluff. The line I've been taking, product to product, is how to do more with less. The apps I actually use revolve around 2–3 features. The rest is bloat. For the last few months I've been trying to build only those 2–3 features in everything.

AI is what made that possible to say out loud. It exposed how thin the "art and craft" of layered SaaS actually was. Most of those layers were monetization and fluff — serving you a server log on a golden plate, and having you pay for the plate to extract data you already had. Once that was visible, the post-AI version of building got obvious: ship the 2–3 features, skip the rest.

Here's how the chain unfolded.

---

## The trajectory

**privpn — privacy starts here.** WireGuard through your own VPS. One script: VPS access, WireGuard install, peer management, connect/disconnect. No third party. The first time I drew a hard line: my data, my VPS, my key.

**weare____ suite — see what's being done to you.** Eight browser extensions, each exposing a different surveillance layer: cookies, hidden pixels, network traffic and data brokers, localStorage tracking, link redirect chains, canvas/WebGL/audio fingerprinting, dark patterns, toxic ToS clauses, form-data exfiltration before submit. All folded into **wearehere** — one popup, one risk score, all of the above at once, Chrome and Firefox. Network Map added in v3.2.

**privcloud — bring it home.** Self-hosted Fedora server: Immich for photos, Navidrome for music, file backup, Tailscale for remote access. Two modes — Immich-only on a laptop, or always-on home server set up from one script. Replaces the cloud bundle a phone normally pulls you into.

**The digital-identity reckoning — gitdone and addypin.** Both had been sitting around through a long back-and-forth: how do you do "digital identity" without the bloat? Once it clicked, both shipped in days. gitdone runs multi-party email workflows verified with DKIM, stamped on Bitcoin. addypin — a 14-year-old project — hit v4 the same week. No accounts, no profiles, no tracking. Email is the surface, the cryptography is the spine.

**knowless — the primitive that fell out.** While rebuilding addypin I realized I needed almost none of the auth I had assumed. What I actually needed was: a way to verify a user is the same person across sessions, and a way to email them exactly once — the sign-in link. That's it. So I pulled it out as a library and adopted it across addypin, gitdone, and plato. (See its own section below.)

**plato — the first product built natively on the new primitive.** A forum. Reddit-shaped, phpBB-spirited, Discourse-grade text-first. No accounts beyond knowless. No image, video, or file hosting — even funny pics, kept small on purpose. Pseudonymous because people deviate based on names and pics when they converse. Public modlog — three views (open, inbox, audit) consolidated into one set of pages, reusing the same code paths. Markdown files on disk, SQLite as an index, one HTTP port. Runs on a $5 VPS.

**bareguard — agent-side gate.** A runtime policy library every agent action passes through. One Gate, three call sites (`redact`, `check`, `record`), twelve primitives. Allow / ask-human / deny, before the action runs. Same minimalist promise as the rest of bare-suite — embed it, no daemon, no SaaS, no telemetry.

> A note on bareguard and knowless. They look similar — small gates that let humans, then agents, do exactly one thing — but they are not the same primitive. knowless gates humans into a session. bareguard gates agents into an action. Don't conflate them. The shape is shared because the discipline is shared.

**ama — ask me anything, on the page you're already on.** A browser extension that uses your existing Gemini, ChatGPT, and Claude session cookies — no sign-in, no API keys, no separate account. Find a page on a site whose owner tells you to "just google our own pages." Translate an old foreign-language site to English in place. Compare two pages side by side. The AI you already pay for, applied to the web you're already reading, without a fourth login.

**late.fyi — the "tired of three apps" instinct, productized.** I was juggling three apps and a browser tab to track late EU trains. Email a train number to `ICE145@late.fyi`, get told when something changes. No app, no account, no noise on time. Built after the juggling got annoying enough. Confirmed product, not scratch-pad.

---

## knowless — the actual mechanics

Most auth libraries default to maximum identity collection: email in plaintext, profile fields, recovery email, federation. Even nominally privacy-focused options store enough that a breach is materially harmful. knowless inverts the default.

- At login, email is **salted-HMAC'd** to verify the user exists.
- The library issues a magic link and a **30-day cookie**.
- The **hashed ID is what's stored at rest**. The plaintext email is discarded at the boundary.
- **I cannot contact users.** That is by design.

Two activation modes:

1. **Sign-in, then act.** Classic gate.
2. **Act, then sign-in or activate** if you want to keep what you did.

The philosophy: *if I want to hear from you, notify me when you're in the app, logged in. Otherwise — silent.* No newsletters. No re-engagement campaigns. No "we miss you" emails. If you're dormant, I have no business pinging you, and I have no way to.

In production at addypin, gitdone, and plato. One production dependency (nodemailer). Apache 2.0.

---

## addypin v3 → v4: the deletion

addypin v3 had everything you'd expect a 14-year-old product to accumulate. Email collection. Country pins. Telemetry. Device IDs. Umami analytics. Email-plus-PIN auth via Resend, with full bounceback hell. Layers and layers.

I deleted the entire project. **1,900+ commits, wiped.**

v4 is built on knowless. Magic links. No PIN. No analytics. The realization that drove the wipe:

> Servers often have logs already. 90% of software is layers upon layers of monetization and fluff — like serving you a server log on a golden plate and having you pay for the plate to extract data you already have. AI makes it cheaper to expose there wasn't much art or craft behind those layered services. They were just part of my brainwashed thinking — collect email, track devices, see what you can harvest by default. Doing unto others what was being done unto me.

That's not a refactor. That's the post-AI builder being honest about what was actually load-bearing.

---

## The kill story: mcp-gov

I had built mcp-gov twice over Replit — sophisticated, beautiful UI. It was what I knew most about before Claude. It ended up archived. The product positioning was wrong from the start.

> I thought API was the substrate to celebrate, and I built a city around it. Then MCP made API the plumbing. That's how mcp-gov came to be. I also didn't understand much compared to now.

The lesson: **positioning beats craft, and the substrate moves under you.** A beautiful UI on the wrong layer is just a tombstone with good typography. Kill it, and build the next thing on the new substrate. mcp-gov is in the archive for a reason.

---

## The bloat test

I don't have telemetry. I don't have user analytics. So how do I know if a feature is worth keeping?

> Would I use it? I ask Claude, and 50% of the time it's bloat — because I can do it myself, or without the feature.

That's the rule. If I wouldn't use it, and Claude can talk me out of it, it doesn't ship. Half the time, that's the correct answer.

---

## Who I serve, who I don't

I don't think people need help. If they're dormant, I've lost my value. Better to serve who needs me and wants me — and if the project dies, then maybe it's time to move on.

> I want people to use the internet, not to be used by it.

That's the ethical floor under "silent by default." It's why knowless can't email you. It's why plato won't host a photo of you. It's why addypin doesn't know your country. It's why late.fyi forgets your email after the trip. It's not a feature list — it's a posture.

---

## What got me here

The current lineage isn't a replacement — it's a new branch. The earlier work is still alive: bare-suite (bareagent, barebrowse, baremobile), liteagent, multis, aurora — stable, in use, picking up a fix or enhancement here and there. They're where I learned through building. bareagent and liteagent taught me what an agent loop actually is once you strip the framework. multis and aurora taught me orchestration the hard way — what's worth coordinating and what's just ceremony. barebrowse and baremobile taught me that "automation" is mostly a thin wrapper over a snapshot and a click. None of that is retired. The privacy / web-revival lineage above is what came out of that discipline once the lens shifted toward "do more with less" — same hands, same posture, different target.

---

## What changed with AI

A 14-year-old product (addypin) hit four versions in a year. gitdone and addypin both shipped in days once the idea clicked. plato came out fully formed on top of knowless. late.fyi went from annoyance to product in a weekend.

That velocity is not the AI writing code faster. It's the AI making the bloat visible. Once you can see that ten layers were doing the work of two, you stop building the other eight.

Do more with less. Build the 2–3 features. Skip the rest.

— Amr

GitHub: [github.com/hamr0](https://github.com/hamr0) | LinkedIn: [linkedin.com/in/hamr](https://linkedin.com/in/hamr)
