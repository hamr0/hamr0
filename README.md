# Hey, I'm Amr 👋

📍 **NL** · <!-- STATS -->⭐ 156 stars | 🔱 16 forks | 📦 46 repos<!-- /STATS -->

> AI-Native Builder. Local-first **AI agents** and **privacy-first products** — no accounts, no tracking, open rails over rented infrastructure.

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

*Two throughlines, one principle: local-first **AI agents**, and **privacy-first products & primitives** — no accounts, no custody, no rented infrastructure.*

---

### 🤖 AI & agents

Local-first rails for AI agents, plus tools for building with AI. My most active work.

**baresuite — agent infrastructure** · light enough to read in an afternoon, zero/low deps

*Core — the brain, the gate, the memory*

- ⚡ **[bareagent](https://github.com/hamr0/bareagent)** — Give an agent a goal, get coordinated actions — or hand it a hard one and it decomposes, fans out, verifies, and synthesizes (RLM). Replaces LangChain, CrewAI, AutoGen.
- 🚦 **[bareguard](https://github.com/hamr0/bareguard)** — One gate on every action an agent takes: allow / deny / ask-a-human — plus an opt-in harness that checks outputs against intent.
- 🕸️ **[litectx](https://github.com/hamr0/litectx)** — Tree-sitter memory with activation decay + context-engineering verbs (write/select/compress/isolate). Ranked recall + impact over SQLite/FTS5, no LSP.

*Optional reach — give the agent hands*

- 🌐 **[barebrowse](https://github.com/hamr0/barebrowse)** — Let agents browse the web like a human. Replaces Playwright, Selenium, Puppeteer.
- 📱 **[baremobile](https://github.com/hamr0/baremobile)** — Let agents drive Android + iOS devices. Replaces Appium, Espresso, XCUITest.
- 📨 **[beeperbox](https://github.com/hamr0/beeperbox)** — Reach 50+ messengers (WhatsApp, iMessage, Signal, Telegram, Slack, …) through one MCP server — full Docker appliance (headless Beeper bundled) or a lite `npx` MCP layer over your own Beeper.

**Building with AI**

- 🤖 **[liteagents](https://github.com/hamr0/liteagents)** — Skills toolkit for AI coding assistants; learns from past sessions to work better over time.
- 💬 **[multis](https://github.com/hamr0/multis)** — An agent that lives in your chat apps: controls your machine, remembers conversations, searches documents. Built on the rails above.
- 🧰 **[agentic-toolkit](https://github.com/hamr0/agentic-toolkit)** — Agent workflows, automation, prompt engineering.
- 🧪 **[edgelms](https://github.com/hamr0/edgelms)** — Where does a frozen local embedder earn its keep? One model, four endpoint jobs (log/command anomaly, routing, entity resolution), measured on real data with pre-registered verdicts: it wins at narrowing/recall, loses at judgment. Zero training.
- 🦙 **[coding-assistant](https://github.com/hamr0/coding-assistant)** — Run small language models locally for code assistance. No cloud required.
- 🔬 **[relayfact](https://github.com/hamr0/relayfact)** — An autonomous senior-dev runner assembled from the bare suite: grounds the loop on executable verification (checks that can fail) and narrates itself as an event stream. Builds no primitives — it either graduates or gets archived. [WIP]
- 🧬 **[adaptlearn](https://github.com/hamr0/adaptlearn)** — Can an agent's *harness* — not its plan or code — be an emergent artifact that improves across runs? Assembled from the bare suite; a dumb outer loop holds the grounded close, the agent authors its own workflow and inherits what worked (verdict-gated). Sibling of relayfact — graduates or gets archived.
- 🔁 **[bareloop](https://github.com/hamr0/bareloop)** — Workflows that earn their own design, with receipts: describe a repeated job and its checkpoints, an agent authors the scaffolding, runs execute under an un-gameable gate, and the scaffolding improves across runs (verdict-gated inheritance). Graduated from the adaptlearn experiment. [WIP]

<sub>Lineage: **[aurora](https://github.com/hamr0/aurora)** (archived) pioneered this — its memory became litectx, its orchestration became bareagent.</sub>

---

### 🛡️ Privacy-first

The same local-first approach as the rails above — shipped as products people use, and as primitives to build on. No accounts, no tracking, no central trust.

**Privacy products** · use them today

- 📍 **[addypin](https://addypin.com)** — Turn a GPS coordinate into a short, memorable link (addypin.com/HOUSE1) or email `HOUSE1@addypin.com`. 12 map-app buttons, no accounts. · [repo](https://github.com/hamr0/addypin)
- 📨 **[signedreply](https://signedreply.com)** — Coordinate multi-party actions over email. Every reply DKIM-verified + OpenTimestamped and committed to a per-event git repo — proofs verify offline even if the service dies. · [repo](https://github.com/hamr0/gitdone)
- 🏛️ **[plato](https://ownsub.com)** — Self-hosted forum: Reddit-shaped, 2002-operated, one program + one data file, plain-text posts. Live at ownsub.com. · [repo](https://github.com/hamr0/plato)
- 🚆 **[late.fyi](https://late.fyi)** — Email a train number → real-time platform, delay, and cancellation alerts back. Your inbox is the protocol. · [repo](https://github.com/hamr0/latefyi)
- 💬 **[ama](https://github.com/hamr0/ama)** — Ask any website anything using your existing AI subscription. Researches whole sites, translates foreign content, answers in English.
- 🔒 **[privpn](https://github.com/hamr0/privpn)** — Your own WireGuard VPN on a VPS. Your server, your keys, no third-party provider.
- ☁️ **[privcloud](https://github.com/hamr0/privcloud)** — Self-hosted home server: photo backup, music streaming, files, remote access. One script from fresh Fedora to fully running.
- 🛡️ **[wearehere](https://github.com/hamr0/wearehere)** — All-in-one browser privacy audit: cookies, trackers, fingerprinting, dark patterns, ToS toxicity — one scan. Also ships an MCP server for AI agents.

<details><summary>↳ ten earlier scanners, now folded into wearehere</summary>

- 🍪 **[wearecooked](https://github.com/hamr0/wearecooked)** — cookie + tracking-pixel scanner. [ARCHIVED]
- 📡 **[wearebaked](https://github.com/hamr0/wearebaked)** — network + data-broker dashboard. [ARCHIVED]
- 🔍 **[weareleaking](https://github.com/hamr0/weareleaking)** — local tracking-storage flagger. [ARCHIVED]
- 🔗 **[wearelinked](https://github.com/hamr0/wearelinked)** — redirect-chain + tracking-param stripper. [ARCHIVED]
- 👁️ **[wearewatched](https://github.com/hamr0/wearewatched)** — fingerprinting + silent-permission detector. [ARCHIVED]
- 🎭 **[weareplayed](https://github.com/hamr0/weareplayed)** — dark-pattern scorer. [ARCHIVED]
- 📜 **[wearetosed](https://github.com/hamr0/wearetosed)** — privacy-policy / ToS scorer. [ARCHIVED]
- 🤫 **[wearesilent](https://github.com/hamr0/wearesilent)** — keystroke-exfiltration detector. [ARCHIVED]
- 🏷️ **[wearesold](https://github.com/hamr0/wearesold)** — data-broker detector. [ARCHIVED]
- 👁️ **[wearecounted](https://github.com/hamr0/wearecounted)** — tracking-pixel + beacon detector. [ARCHIVED]

</details>

**Privacy primitives** · build on them · local-first, no telemetry

- 🔑 **[knowless](https://github.com/hamr0/knowless)** — Full-stack passwordless auth for Node.js: magic-link sign-in, no passwords, no profile stored. The auth layer under everything I ship.
- 🗂️ **[flightlog](https://github.com/hamr0/flightlog) + [pulselog](https://github.com/hamr0/pulselog)** — a lightweight, self-hosted **server-log suite**; same zero-dep JSONL dialect, read with `tail`/`jq`.
  - ✈️ **flightlog** — in-process error capture: uncaught exceptions, rejections, and errors you hand it. The local alternative to **Sentry**.
  - 🩺 **pulselog** — external watcher: health/SSL/disk/backup checks, a weekly stats digest, rotated backups. Replaces **hosted analytics + uptime monitoring**.
- 🔞 **[8een](https://github.com/hamr0/8een)** — One-bit, unlinkable, stateless age verification — the ZK verifier the EU didn't ship. Proof in → true/false out. Replaces ID uploads and face scans. Built on `google/longfellow-zk`. [WIP]

---

### 🎧 Also

- 🔊 **[sawt](https://github.com/hamr0/sawt)** — Turn any book into a multi-voice Arabic audiobook. File in, narrated audio out.
- 🧾 **[mailproof](https://github.com/hamr0/mailproof)** — Email-native coordination kernel: verify a reply (DKIM/DMARC), commit it to a tamper-evident git ledger, sequence the workflow, trigger the next email. The engine extracted from signedreply. Zero deps.

---

<details open><summary>🗄️ <b>Archive & lineage</b> — earlier experiments, kept for the trail</summary>

- 🧠 **[aurora](https://github.com/hamr0/aurora)** → memory became **litectx**, orchestration became **bareagent**.
- ✂️ **[mcprune](https://github.com/hamr0/mcprune)** → snapshot-pruning folded into **barebrowse**.
- 🛡️ **[mcp-gov](https://github.com/hamr0/mcp-gov)** → MCP access-control idea folded into **bareguard**.
- 🔑 **[terribic](https://github.com/hamr0/terribic)** — one token for permissioned AI access to your apps and data.
- 📚 **[AgenticAI](https://github.com/hamr0/AgenticAI)** — MCP / vector-DB / RAG / memory training exercises.
- 🧪 **[bareapp](https://github.com/hamr0/bareapp)** — sensor + webapp automation experiments.
- 🎯 **[polarized](https://github.com/hamr0/polarized)** — data-selling / ownership / political-money overlay (POC).

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

Each project started as a question I couldn't stop thinking about — from NLP
for Arabic, to map tech for developing nations, to AI frameworks that give
agents memory and reasoning. I build things to understand them, and share
what works.

**[Read the full story: doing more with less — a year of building privacy primitives and AI tooling](AI_BUILDING.md)**

---

[![Email](https://img.shields.io/badge/-Email-EA4335?style=flat-square&logo=gmail&logoColor=white)](mailto:avoidaccess@msn.com)
[![GitHub](https://img.shields.io/badge/-Follow-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/hamr0)
[![LinkedIn](https://img.shields.io/badge/-LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/hamr)
