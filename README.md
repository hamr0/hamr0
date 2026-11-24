# Hey, I'm Amr 👋

📍 **NL** · <!-- STATS -->⭐ 209 stars | 🔱 20 forks | 📦 54 repos<!-- /STATS -->

> AI-Native Builder. **Verified agent workflows**, agent rails and identity/auth, and **privacy-first products** — no accounts, no tracking, open rails over rented infrastructure.

![Python](https://img.shields.io/badge/-Python-3776AB?style=flat-square&logo=python&logoColor=white)
![JavaScript](https://img.shields.io/badge/-JavaScript-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![Node.js](https://img.shields.io/badge/-Node.js-339933?style=flat-square&logo=node.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/-TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Claude](https://img.shields.io/badge/-Claude-000000?style=flat-square&logo=anthropic&logoColor=white)
![MCP](https://img.shields.io/badge/-MCP-121212?style=flat-square&logo=openai&logoColor=white)

AI-Native Builder and technical leader — shipped at Meta, Vodafone, Liberty Global,
and Cognizant. I take problems to working products: architecture, code, and the
decisions in between.

**Capability without custody.** I don't build new infrastructure — I reshuffle
the rails you already use (email, maps, git, chat apps, the logs your apps
already write) so the leverage lands on you, not a platform. Borrowed rails never need custody of your
data — privacy here isn't a promise, it's structure. No account to breach, no
profile to sell, no lock-in: you keep the function and the keys, never become
the product.

**Always up for hard problems and good teams — let's talk.**

*Simple over clever. Every line must have a purpose.*

---

*Three throughlines: **verified agent workflows** that run, **rails and identity** agents can trust, and **privacy-first products** that prove it — no accounts, no custody, no rented infrastructure.*

---

### ✅ Verified agent workflows

Agent builds it, you define done, a gate it can't touch checks it — humans keep the last word. My most active work. The arc: **run it once, verified → run it every day, with people.** What the agent may touch: see [Privacy, agent authN & authZ](#-privacy-agent-authn--authz).

- **[bareloop](https://github.com/hamr0/bareloop)** — *Closed loop.* "Automate this job — I don't know the best workflow." You say what done means and what the agent must never do; it builds its own workflow. The run is green only when checks the agent can't touch pass, inside a hard budget, and every run replays at $0. [WIP]
- **[fwdloop](https://github.com/hamr0/fwdloop)** — *Open loop.* "Automate this job — it has humans in it." You describe the steps and guardrails in plain language, and an agent builds the workflow. It pauses for a person where you said and runs on its trigger under budgets it can't change. [WIP]

---

### ⚡ baresuite — agent rails

Boilerplate pieces for agents and LLM tool calling · light enough to read in an afternoon, zero/low deps

*Hands — let the agent reach the world*

- **[barebrowse](https://github.com/hamr0/barebrowse)** — Gives an agent a real browser with your own logins, on Chromium or Firefox. It returns a short, pruned page the agent can click and type on, so it uses far fewer tokens. Replaces Playwright, Selenium, Puppeteer.
- **[baremobile](https://github.com/hamr0/baremobile)** — Gives an agent control of real Android and iOS devices: read the screen, tap, type, swipe, launch apps. The screen comes back as a pruned snapshot, with zero runtime dependencies. Replaces Appium, Espresso, XCUITest.
- **[beeperbox](https://github.com/hamr0/beeperbox)** — Plugs an agent into 50+ messengers (WhatsApp, iMessage, Signal, Telegram, Slack, …) through one MCP endpoint instead of one SDK each. Run it as a Docker appliance with headless Beeper bundled, or as a lite `npx` layer over your own Beeper.

*Rails — build and guard your own agent*

- **[bareagent](https://github.com/hamr0/bareagent)** — Small agent building blocks with zero required deps: loops, retries, budgets, and one call that splits a hard task, fans it out, verifies, and synthesizes (RLM). Provider-agnostic. Replaces LangChain, CrewAI, AutoGen.
- **[bareguard](https://github.com/hamr0/bareguard)** — One gate on every action an agent takes: allow, deny, or ask a human. It uses deterministic checks only, never an LLM, keeps one audit log, and has hard caps that halt the run. One production dependency.
- **[barebox](https://github.com/hamr0/barebox)** — A light sandbox you spawn per agent. Reads pass, file writes can be undone, and anything irreversible (pay, send, publish, push) waits for your yes. About 8 MB per sandbox, one shared proxy for many agents. Replaces Docker/VM isolation and approve-everything prompts.
- **[litectx](https://github.com/hamr0/litectx)** — Local search and memory for agents and humans. It indexes your code and docs and returns ranked `file:lines` pieces, plus what breaks if you change something. One SQLite file, no LLM inside.

*Apps — built with the suite:* **[bareloop](https://github.com/hamr0/bareloop)** · **[fwdloop](https://github.com/hamr0/fwdloop)** (see [Verified agent workflows](#-verified-agent-workflows) above)

<sub>Lineage: **[aurora](https://github.com/hamr0/aurora)** (archived) pioneered this — its memory became litectx, its orchestration became bareagent.</sub>

---

### 🪪 Privacy, agent authN & authZ

Prove who (or what) is acting without handing over who they are, and scope what it may do. Standards work: **IETF · CAMARA · AAIF**. Local-first, no telemetry.

- **[knowless](https://github.com/hamr0/knowless)** — Passwordless auth for Node.js: a magic link in, a session cookie out, nothing else stored. The email is hashed and discarded, so there is no profile to breach. The auth layer under everything I ship.
- **[8een](https://github.com/hamr0/8een)** — Age verification that answers one bit: proof in, true or false out. No name, no birthdate, no document, nothing stored, and proofs can't be linked across sites. The ZK verifier the EU didn't ship; replaces ID uploads and face scans. Built on `google/longfellow-zk`.
- **[zkagent](https://github.com/hamr0/zkagent)** — Your passport or ID-card chip answers one question, like "over 18?", and discloses nothing else. No issuer and nothing stored; it is the human root for agent auth, cited by an IETF individual draft. Open-source lab: `chiproof` on npm plus an Android scanner.
- **[justabit](https://github.com/hamr0/justabit)** — A standards effort to make telecom network APIs answer with one signed, expiring bit instead of disclosing data. Predicate, floor and nonce go in; never the timestamp, country or number behind it. Staged for CAMARA + AAIF, with rwxmap's r/w/x as its actionClass axis.
- **[rwxmap](https://github.com/hamr0/rwxmap)** — *Scope.* Labels every API operation r / w / x, like chmod, so an agent knows what a call does before it's made. It reads OpenAPI and picks the tighter letter when unsure. Emits bareguard grants and MCP hints.
- **[rwxlite](https://github.com/hamr0/rwxlite)** — *Scope, on the fly.* A 23 MB local classifier that gives r / w / x odds for any API operation in about 20 ms. Free, offline and private; it is rwxmap's default second pass.

---

### 🚀 Products — use them today

Websites and front ends built on borrowed rails. No accounts, no tracking, no central trust.

- **[addypin](https://addypin.com)** — Turn a map location into a six-character handle you can say out loud, shared as a link (addypin.com/HOUSE1) or an email address (`HOUSE1@addypin.com`). It opens in the other person's own map app. No accounts, no tracking, and links never expire. · [repo](https://github.com/hamr0/addypin)
- **[signedreply](https://signedreply.com)** — Run multi-party approvals over email: each person replies from a normal inbox, no accounts. Every reply is DKIM-verified, timestamped with OpenTimestamps and committed to a per-event git repo, so proofs verify offline even if the service dies. · [repo](https://github.com/hamr0/gitdone)
- **[mailproof](https://github.com/hamr0/mailproof)** — Proves an email really came from the address it says, and keeps that proof checkable offline years later. The engine extracted from signedreply: DKIM/DMARC verdicts, a tamper-evident git ledger you own, and the reply text the sender newly wrote.
- **[plato](https://ownsub.com)** — A self-hosted forum shaped like Reddit: subs, threads, votes, mods, with a public modlog. One program and one SQLite file, plain-text posts, magic-link sign-in with no profile. Live at ownsub.com. · [repo](https://github.com/hamr0/plato)
- **[late.fyi](https://late.fyi)** — Email a train number, get platform, delay and cancellation alerts back. European trains only. Your inbox is the protocol: no app, no account, no noise when the train is on time. · [repo](https://github.com/hamr0/latefyi)
- **[sawt](https://github.com/hamr0/sawt)** — Turns Arabic books (EPUB, DOCX, TXT) into audiobooks, locally, with a browser UI. Fiction gets a narrator voice and a dialogue voice; voices come from Gemini or ElevenLabs. [WIP]

**Self-hosted & privacy tools**

- **[ama](https://github.com/hamr0/ama)** — A browser sidebar that answers questions about a whole website, not just the page you're on. It crawls and reads relevant pages, translates foreign-language content, and replies in English using your existing ChatGPT, Claude or Gemini subscription. No API keys, no accounts.
- **[privpn](https://github.com/hamr0/privpn)** — Turns a cheap Linux VPS into your own WireGuard VPN with one script that handles setup, peers and connect/disconnect. Your server, your keys. Nothing rests on a provider's "no-logs" promise.
- **[privcloud](https://github.com/hamr0/privcloud)** — A self-hosted home server for photo backup, music streaming, files and remote access. One script takes a fresh Fedora install to fully running. Replaces iCloud, Google Photos and Google Drive.
- **[wearehere](https://github.com/hamr0/wearehere)** — A browser extension that shows who is tracking you on the page and acts on it. It scores each site, cleans up tracker cookies, blurs your device fingerprint, and flags hostile terms in the fine print. Local only, no account.

<details><summary>↳ ten earlier scanners, now folded into wearehere</summary>

- **[wearecooked](https://github.com/hamr0/wearecooked)** — cookie + tracking-pixel scanner; local only. [ARCHIVED]
- **[wearebaked](https://github.com/hamr0/wearebaked)** — network traffic dashboard + data-broker detector. [ARCHIVED]
- **[weareleaking](https://github.com/hamr0/weareleaking)** — flags tracking IDs and personal data in local storage. [ARCHIVED]
- **[wearelinked](https://github.com/hamr0/wearelinked)** — shows redirect chains and strips tracking parameters from links. [ARCHIVED]
- **[wearewatched](https://github.com/hamr0/wearewatched)** — detects fingerprinting calls and silent permission use. [ARCHIVED]
- **[weareplayed](https://github.com/hamr0/weareplayed)** — dark-pattern scorecard for sites. [ARCHIVED]
- **[wearetosed](https://github.com/hamr0/wearetosed)** — scores terms of service so you know what you agree to. [ARCHIVED]
- **[wearesilent](https://github.com/hamr0/wearesilent)** — detects sites capturing form input and keystrokes before you submit. [ARCHIVED]
- **[wearesold](https://github.com/hamr0/wearesold)** — detects connections to known data-broker companies. [ARCHIVED]
- **[wearecounted](https://github.com/hamr0/wearecounted)** — detects hidden tracking infrastructure on a page. [ARCHIVED]

</details>

---

### 🧪 Labs — side experiments

Graduate or get archived. Assembled from the bare suite; they build no primitives.

- **[edgelms](https://github.com/hamr0/edgelms)** — Zero-training experiments with one frozen local embedder on endpoint data: anomaly detection, routing and entity resolution. Measured on real public data with pre-registered verdicts: it wins at similarity and recall, and loses at judgment.
- **[relayfact](https://github.com/hamr0/relayfact)** — An autonomous senior-dev runner assembled from bareagent, litectx and bareguard, with no primitives of its own. A plain-English request goes in; a verified result or a decision-ready escalation comes out. Its line led to bareloop.
- **[adaptlearn](https://github.com/hamr0/adaptlearn)** — Tests whether an agent's harness, not its plan or code, can improve across runs. Only proven successes (real test passes) are passed on to the next run. Sibling of relayfact; its graduate is **bareloop**.
- **[coding-assistant](https://github.com/hamr0/coding-assistant)** — Six bash scripts that wrap `llama.cpp` to run quantized coding models on your laptop. Terminal chat, or an OpenAI-compatible local server. No GPU, Docker or Python needed.

---

### 🛠️ Building with AI & ops

- **[liteagents](https://github.com/hamr0/liteagents)** — A skills toolkit for AI coding assistants: ten specialist agents plus commands like `/stash` and `/remember` that carry project context across sessions. Works with Claude, Opencode, Ampcode and Droid.
- **[multis](https://github.com/hamr0/multis)** — A local-first chatbot and assistant that lives in your chat apps (Telegram, plus WhatsApp, Signal and more through Beeper). It answers from your documents and runs things on your machine, behind one governance gate that asks before anything that matters.
- **[agentic-toolkit](https://github.com/hamr0/agentic-toolkit)** — Agent kits (ten specialist agents, 12 commands and skills) for Claude, Opencode, Ampcode and Droid, plus a Linux terminal dev environment.
- **[flightlog](https://github.com/hamr0/flightlog) + [pulselog](https://github.com/hamr0/pulselog)** — a lightweight, self-hosted **server-log suite**; same zero-dep JSONL dialect, read with `tail`/`jq`.
  - **flightlog** — in-process error capture: uncaught exceptions, rejections, and errors you hand it, written as one JSON line each to a local file. Zero deps, about 130 lines. The local alternative to **Sentry**.
  - **pulselog** — external watcher: health, SSL, disk and backup checks, a weekly stats digest, and rotated backups. Silent when green, emails on trouble. Replaces **hosted analytics + uptime monitoring**.

---

<details open><summary>🗄️ <b>Archive & lineage</b> — earlier experiments, kept for the trail</summary>

- **[aurora](https://github.com/hamr0/aurora)** → code-aware memory and code intelligence for AI coding assistants (archived); memory became **litectx**, orchestration became **bareagent**, guard rails became **bareguard**.
- **[mcprune](https://github.com/hamr0/mcprune)** → MCP proxy that cut Playwright page snapshots by 75-95% for LLM agents; pruning folded into **barebrowse** and **baremobile**.
- **[mcp-gov](https://github.com/hamr0/mcp-gov)** → permission control and audit logging between an AI assistant and MCP servers; the idea folded into **bareguard**.
- **[terribic](https://github.com/hamr0/terribic)** — a single revocable token that lets AI assistants reach your apps, instead of sharing each API key. Its access-control idea went on to **mcp-gov**.
- **[AgenticAI](https://github.com/hamr0/AgenticAI)** — local training exercises for MCP, vector DBs, RAG and memory on JSON.
- **[bareapp](https://github.com/hamr0/bareapp)** — turns an old Android phone into a sensor station (decibel, GPS, environment) with Termux, no root.
- **[polarized](https://github.com/hamr0/polarized)** — browser popup showing who owns a site, which data brokers load on it, and the political money behind them (POC).

</details>

---

### How I work

- **POC first** — validate the idea before committing to a design
- **Simple over clever** — fewer deps, less config, readable code
- **Borrow and improve** — every past project is a library of lessons
- **Local-first, open-source only** — your data stays on your machine
- **Connect the dots** — cognitive science, product ops, AI research — patterns transfer
- **[Full dev standards](AGENT_RULES.md)** — the rules every project follows
- **[Building a JS library](LIBRARY_CONVENTIONS.md)** — the fixed ref for publishable JS libs: types (JSDoc → `.d.ts` → CI, no drift), the adopter `context.md`, doc set
- **[Shipping a container / MCP server](CONTAINER_MCP_CONVENTIONS.md)** — container-correctness foot-guns (signal forwarding, healthcheck-through-the-forwarder, loud-open defaults, pin+sha256) and MCP tool-surface discipline

---

### Background

Each project started as a question I couldn't stop thinking about, and most
fell out of the one before. I'm just a guy with an agent, working on my own
problems: privacy, bloated products, over-engineering — pockets of resistance,
one small solution at a time. The write-up covers how I started, the privacy
line, the products and the libraries behind them, and how the way I work with
AI changed — from spoon-feeding to workflows that verify themselves.

**[How I build](ai-native-building.md)**

---

[![Email](https://img.shields.io/badge/-Email-EA4335?style=flat-square&logo=gmail&logoColor=white)](mailto:avoidaccess@msn.com)
[![GitHub](https://img.shields.io/badge/-Follow-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/hamr0)
[![LinkedIn](https://img.shields.io/badge/-LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/hamr)
