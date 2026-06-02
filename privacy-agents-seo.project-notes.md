# Privacy-agents-SEO — per-project notes

Project-specific implementation records. Split out of `privacy-agents-seo.md` so that doc stays generic (it's handed to an agent to follow as-is). The playbook itself lives in `privacy-agents-seo.md`; this file is the status ledger.

## late.fyi

Implemented 2026-05-04 (commit TBD): tier 1 head tags + robots.txt + sitemap.xml. Skipped JSON-LD on principle for now. og-card.png deferred — design task.

Note (post-2026-05-10): a WhatsApp test surfaced that "no og:image" doesn't render as title+description, it renders as a compact chain-icon card. Visually weaker than expected. Worth promoting og-card.png from "deferred" to a real ticket; the design lift is small (a logo on the brand colour, 1200×630 PNG) and the payoff is the entire link-preview surface across WhatsApp, Slack, Discord, iMessage. — **Done:** `web/og.png` (1200×630) shipped 2026-05-10 and is wired in the head with `og:image:width/height/alt` + `twitter:card: summary_large_image`.

Completed 2026-06-02 (uncommitted): the remaining playbook gaps closed in `web/` — **JSON-LD** (`SoftwareApplication` + `FAQPage`, 4 Q&As lifted verbatim from "What we don't do"), **AI-crawler `robots.txt`** (retrieval bots `Claude-User`/`Claude-SearchBot`/`OAI-SearchBot`/`ChatGPT-User`/`PerplexityBot` + training `ClaudeBot`/`GPTBot`, all `Allow` — marketing copy, no PII; the app surface is email, not a web path), **`sitemap.xml`** switched to `<lastmod>` (dropped `changefreq`/`priority`), and **`llms.txt`** (curated index, privacy invariant up top, real URLs only — apex + GitHub, since it's a single-page site). 299 tests still green. FAQ/llms claims mirror CLAUDE.md's privacy invariant — keep in sync. Deploys on `git push` (Pages auto-deploy); not yet pushed.

The privacy-claim invariant is in `CLAUDE.md` (state/active/<msgid>.json deleted on terminal, no archive). The landing page section "What we don't do" must stay in sync; if retention changes, update the page first per CLAUDE.md.

## plato (terribic.com)

Shipped 2026-05-10 (commit `bb3cdcc`): `og:image` pointing to a static 1200×630 PNG of plato's three-dot mark on `--bg`, served at `/static/og.png`. `twitter:card` upgraded `summary` → `summary_large_image`. The PRD locks the og.png as **project identity, not fork-rebrandable** — the link-preview surface is where someone who has never seen plato first encounters the project, and consistency there pays for the recognition signal. Forks that want a different mark fork the repo; no `branding.*` knob.

The same pattern is the recommendation for sibling projects: ship a static PNG of the project mark on the brand bg colour, served from the app (no CDN — keeps single-binary integrity). The image is identity, not theming.

## (other projects)

Add sections per project as they ship.
