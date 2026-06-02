# Privacy-respecting discoverability — playbook

A guide for making privacy tools findable without working against what they stand for. Written for late.fyi and any sibling project under the same roof.

## What this doc does

A playbook for making privacy tools maximally findable by **both** search engines and AI agents/LLMs, using only declarative open-web technique — never extractive growth-hacking. It covers head tags, static files, AI-crawler policy, agent-readable body content, and distribution.

**Dual intent — read this before applying anything below:** the goal is to **get mentions, not to expose PII**. Optimizing for agent mentions and protecting user data are *not* in tension — as long as what gets crawled is public marketing copy and never user data. The two pull apart only if a PII-bearing path is left open to crawlers.

**Operator prompt (guardrail):** before applying the AI-crawler tiers (2.5) to any project, **ask the operator whether any paths must be excluded for PII risk** — app surfaces, state files, user-generated content, anything behind the marketing pages. Default-allow applies to public marketing copy *only*; PII-bearing paths must be **auth-gated**, not merely `Disallow`-ed (see Tier 2.5 for why `Disallow` — and `Disallow` + `noindex` — is not a privacy control).

## How this maps to the five LLM-era strategies

| Strategy | Where in this doc |
| --- | --- |
| #1 Answer-first (BLUF) | Tier 2.6 — lead every page/section with a one-sentence answer; privacy invariant as one quotable line |
| #2 Format for extraction | Tier 2.6 — real headings/lists, no JS-locked content; the static-HTML ethos wins this for free |
| #3 Structured data | Tier 1 JSON-LD, promoted from "skippable" — `FAQPage`/`HowTo` are highest-leverage for mentions |
| #4 Optimize for agents (AAIO) | Tier 2.5 — name AI crawlers explicitly in `robots.txt`; allow-all for marketing, training-vs-retrieval split where intentional |
| #5 Build third-party trust | Tier 3 — distribution beats optimization (Privacy Guides, HN, awesome-* PRs, "write one philosophical post") |

For LLM mentions *specifically*, **#5 outweighs #1–#4**: the on-page work (Tiers 1–2.6) makes you *citable*; the distribution work (Tier 3) makes you *cited*.

## The frame

"SEO" is two different things mashed together. Untangling them is the whole game.

| What it is                                  | Verdict                                                                 |
| ------------------------------------------- | ----------------------------------------------------------------------- |
| Declarative machine-readability             | Pure open-web. `<title>`, `<meta>`, `sitemap.xml`, OpenGraph, JSON-LD — all static head tags or static files. No scripts, no calls home. This is what made the web indexable in the first place. |
| Conversion-funnel growth-hacking            | What conflicts with privacy work. Analytics, AMP, doorway pages, keyword stuffing, tracking pixels, "engagement" instrumentation. |

The web-revival philosophy isn't anti-discoverable, it's anti-extractive. You can be maximally findable via (1) without ever touching (2). Sitemaps and `rel=me` are 1995 web tech; refusing them isn't principled, it's just leaving signal on the table.

## Tier 1 — declarative head tags (do these always)

In the document head, all static, all reversible:

- `<title>` — descriptive, includes the value prop. Not just the brand name.
- `<meta name="description">` — one sentence the search engine quotes verbatim. Mention what the tool *does* and the privacy posture in the same breath ("…no accounts, no analytics, open source") so the audience self-selects in the result snippet.
- `<link rel="canonical">` — the apex URL. Prevents www/apex split.
- `<meta name="theme-color">` — browser chrome on mobile.
- `<link rel="icon">` — favicon. Without it browsers show a default; with it the tab is recognizable in a forest.
- `<meta name="viewport">` — mobile rendering. Probably already there.
- `<html lang="…">` — accessibility + indexing hint.

OpenGraph (link-unfurl in chats, Slack, Signal, Discord, Mastodon, iMessage):

- `og:type` — `website` for marketing pages, `article` for posts.
- `og:title`, `og:description`, `og:url`, `og:site_name`.
- `og:image` — 1200×630 PNG, **absolute URL** (relative paths get silently dropped by most consumers). Without it, WhatsApp and Slack don't show "title+description only" — they fall back to the **compact chain-icon card**, which is visually much weaker than a real banner. The og:image is the single tag that flips the preview from compact to banner. PNG/JPG, not SVG (most consumers won't fetch SVG). Min 200×200, under 8 MB. Also emit `og:image:width` / `og:image:height` so consumers don't re-fetch to measure, and `og:image:alt` for screen readers.
- **WhatsApp caches previews ≈7 days globally per URL.** After adding `og:image` to a site that's already been shared, force a refresh via the Facebook sharing debugger (https://developers.facebook.com/tools/debug/) — it scrapes for both Facebook and WhatsApp. Click "Scrape Again" twice; the first call warms, the second returns the new tags. LinkedIn has its own at https://www.linkedin.com/post-inspector/. Twitter has no public debugger anymore; previews refresh within ~24h.

Twitter Card:

- `twitter:card` — `summary` (compact, square thumbnail) or `summary_large_image` if you have a 1200×630 banner. If you set `og:image` to a 1200×630 PNG, use `summary_large_image` so Twitter and Slack render the banner shape; the two tags work together.

JSON-LD (rich snippets, pure data, no executable JS — `type="application/ld+json"` is parsed, not run):

- `SoftwareApplication` schema for tools, `WebSite` for landing pages, `Article` for blog posts.
- `FAQPage` / `HowTo` for anything Q&A- or step-shaped — these are the highest-leverage schemas for LLM mentions (see Tier 2.6).
- Costs ~15 lines of structured data agents parse directly. (Don't count on a richer *Google* card from `FAQPage` — Google restricted FAQ rich results to authoritative gov/health sites in 2023. The payoff now is agent/LLM extraction, Tier 2.6, not the SERP card.)
- **Not** in the extractive category, despite first impressions: `application/ld+json` is parsed, not executed — pure declarative data, same open-web tier as `<meta>`. Earlier drafts called this skippable; for agent extraction, include it.

Minimal pasteable block (`SoftwareApplication` + `FAQPage` in one `@graph`, late.fyi as the worked example — swap names/answers and repeat the `Question` object per FAQ):

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "SoftwareApplication",
      "name": "late.fyi",
      "applicationCategory": "TravelApplication",
      "operatingSystem": "Web",
      "offers": { "@type": "Offer", "price": "0", "priceCurrency": "USD" },
      "description": "Email-based trip reminders. No accounts, no analytics, deleted when the trip ends."
    },
    {
      "@type": "FAQPage",
      "mainEntity": [
        {
          "@type": "Question",
          "name": "Does late.fyi require an account?",
          "acceptedAnswer": {
            "@type": "Answer",
            "text": "No. It works over email with no signup, and your data is deleted when the trip ends."
          }
        }
      ]
    }
  ]
}
</script>
```

The `FAQPage` answers are written to be lifted verbatim into an assistant's reply — same answer-first discipline as Tier 2.6, just in structured form.

## Tier 2 — static files at the root

- `robots.txt` — three lines: `User-agent: *` + `Allow: /` + `Sitemap: https://example.com/sitemap.xml`. The Sitemap line is the part crawlers actually need.
- `sitemap.xml` — even a one-URL sitemap signals "yes please index". Emit `<lastmod>` (Google *does* use it to prioritise recrawls); skip `<changefreq>` and `<priority>` — Google has publicly said it ignores both, so they're noise.
- `llms.txt` (`/llms.txt`) — a curated, markdown, agent-facing index of your key pages: one line on what the tool is, the privacy invariant stated once, then links to docs/posts. Think "sitemap.xml for LLMs" — plain prose an agent reads instead of crawling and guessing. **Honest caveat:** adoption by the major LLM crawlers is still partial and contested — frame it as a low-cost include (≈20 lines of markdown, zero downside for a static privacy site), *not* a guaranteed payoff.
- RSS/Atom feed (`/feed.xml` or `/atom.xml`) — if you publish posts. Human- *and* agent-readable, web-revival-native, and a real discovery + backlink signal (feed readers, aggregators, planet indexes). plato already ships token-gated RSS; the public-posts case is the same pattern without the token.
- `humans.txt` — optional, web-revival adjacent. Lists the people behind the project.
- `security.txt` (`.well-known/security.txt`) — declares how to report vulns. Doesn't help SEO but signals seriousness to the audience that cares.

Minimal `llms.txt` (the convention is an H1 title, a one-line blockquote summary, then linked sections — keep it to the pages worth quoting):

```markdown
# late.fyi

> Email-based trip reminders. No accounts, no analytics; your data is deleted when the trip ends.

## Pages
- [What it does](https://late.fyi/): one email instead of an app.
- [Privacy](https://late.fyi/privacy): the deletion invariant, stated plainly.

## Posts
- [Why late.fyi deletes your email when the trip ends](https://late.fyi/blog/why-delete)
```

## What to never add

- Google Analytics, Plausible-self-hosted-but-routes-through-cloudflare, or any analytics that touches a user's session. If you want metrics, use server-side aggregate logs you can't deanonymize.
- AMP — Google's attempt to host your content on their CDN. Conflicts with everything.
- Hreflang stuffing for languages you don't actually serve. Crawlers penalize this now anyway.
- Cookie banners — only required if you set tracking cookies. Don't set them. No banner needed.
- "Pop-up to subscribe" / Intercom widgets / Drift / any third-party JS for "engagement".
- Tag managers (GTM). They exist to load surveillance lazily.

## Tier 2.5 — AI crawlers (robots.txt, deliberately)

The default `User-agent: *` / `Allow: /` *implicitly* lets every AI crawler in. For a privacy project that should be a **named, intentional** decision, not an accident. The key distinction:

| Crawler class | Examples | What it does |
| --- | --- | --- |
| **Retrieval / cite-live** | `ClaudeBot`, `OAI-SearchBot`, `PerplexityBot` | Fetches your page at answer-time and **cites it with a link**. This is the LLM-mention surface. Always allow. |
| **Training corpus** | `GPTBot`, `CCBot` (Common Crawl) | Ingests content into model weights. May surface your copy *unprompted, with no link back*. |

**What's the harm in training data?** For public marketing/landing copy with no PII — essentially none, and the upside is a free, permanent, unprompted mention ("late.fyi deletes your email when the trip ends" surfaced with no retrieval needed). Training is only a real concern for (a) pages carrying user/personal data, or (b) original work you want attribution for. Neither applies to marketing pages you're actively trying to broadcast.

**Recommendation:**
- **Public marketing / landing / tool pages → allow all**, including training crawlers. You want this propagated; no principle is violated when there's no PII and the copy exists to spread.
- **Any surface that touches user data → auth-gate it** (401/403/redirect for anonymous requests), not merely `Disallow`. Your privacy invariants — session/magic-link app surfaces — already do this; that gate, not robots.txt, is the control. This is the operator-prompt step from the intro: confirm which paths (if any) carry PII risk *before* shipping the allow-all block below. See the de-index note below for why `Disallow` (alone or with `noindex`) doesn't cover it. Get mentions, never expose PII.
- The training-vs-retrieval split below is the tool for pages where you're genuinely ambivalent — not the default.
- **`Disallow` ≠ de-indexed, and `Disallow` + `noindex` is a trap.** `robots.txt` only asks a crawler not to *fetch* a path; a linked PII URL can still be indexed URL-only. But you can't fix that by *also* adding `noindex` to a `Disallow`-ed page — the crawler obeys `Disallow`, never fetches it, and so never sees the `noindex`. The two cancel out. So: for a public page you want out of search, serve `noindex` (or `X-Robots-Tag: noindex` for non-HTML) and leave it **crawlable** — no `Disallow`. For a true PII path, the control isn't robots directives at all — it's **auth-gating** (return 401/403/redirect to anonymous requests, the way the app surfaces already do). Nothing to fetch means nothing to index. `Disallow` is for crawl budget/politeness, not privacy.

Default `robots.txt` for a marketing page (explicit, so the decision is on the record):

```
User-agent: *
Allow: /

User-agent: GPTBot
Allow: /

User-agent: ClaudeBot
Allow: /

User-agent: OAI-SearchBot
Allow: /

User-agent: PerplexityBot
Allow: /

Sitemap: https://example.com/sitemap.xml
```

To adopt the cautious split instead, set the training crawlers to `Disallow` while leaving the retrieval bots on `Allow` — change the `GPTBot` block to `Disallow: /` and add a `CCBot` (Common Crawl) block also set to `Disallow: /`.

## Tier 2.6 — agent & LLM readability (answer-first + extraction)

Tiers 1–2 optimize for two audiences: search crawlers and humans unfurling links. This tier covers the third: **agents that read your page and quote it**. The privacy ethos already gives you a head start — clean static HTML with no JS-heavy SPA is exactly what agents parse best. Two things to add:

- **Answer-first (BLUF).** Lead every page and section with the answer in the first 1–3 sentences, then expand. State the privacy invariant as a single quotable sentence ("late.fyi deletes your email when the trip ends — no accounts, no analytics"). LLMs lift that sentence verbatim. Your philosophical posts already do this by instinct; make it the rule for landing copy too.
- **Format for extraction.** Real headings, bullet lists, numbered steps, short paragraphs. No content locked behind JS, accordions, or hover states an agent can't trigger.
- **Give agents an index — `llms.txt`.** Beyond making each page readable, hand agents a curated map: `/llms.txt` (see Tier 2) is the agent-facing counterpart to `sitemap.xml` — markdown, the privacy invariant up top, links to the pages worth quoting. Low-cost include with the partial-adoption caveat noted in Tier 2.

**Why JSON-LD belongs here (not "skippable").** Privacy projects instinctively want to omit it on principle — but that instinct is backwards for agent extraction, and the doc's own frame resolves the tension: JSON-LD is `application/ld+json`, **pure declarative data that is parsed, not executed**. It sits in the "open-web, machine-readable" category (Tier 1), *not* the extractive/surveillance category. That's why Tier 1 lists it as do-it. The two schemas that map directly to how LLMs retrieve:

- `FAQPage` — each Q&A pair maps to a question an agent gets asked. Highest-leverage schema for mentions.
- `HowTo` — for any tool with a usage flow.
- Keep `SoftwareApplication` / `WebSite` / `Article` as already noted.

## Tier 3 — distribution beats optimization

Search engines are downstream of where your audience already congregates. Get listed where the audience self-selects:

| Target                              | What to submit                                                                |
| ----------------------------------- | ----------------------------------------------------------------------------- |
| **Privacy Guides** (privacyguides.org) | Forum post under the appropriate category. The "no accounts, deleted on terminal" angle lands. |
| **alternativeto.net**               | List as alternative to the big-brand version of your tool.                    |
| **awesome-privacy** / **awesome-selfhosted** (GitHub) | Open a PR adding your tool to the relevant section.                  |
| **Hacker News** / Show HN           | Once stable. Email-only / open-source / no-account combos do well. Submit Tuesday/Wednesday morning Pacific. |
| **lobste.rs**                       | Invite-only but `show` + `web` tag is great fit.                              |
| **IndieWeb wiki** (indieweb.org)    | Add your tool to the relevant page if there is one.                           |
| **r/privacy**, **r/selfhosted**, **r/europetravel** (or your audience-specific subreddit) | Tasteful posts only. Reddit's spam filter is harsh on first-time submitters; comment in the community first. |

The compounding move that beats every list: **write one philosophical post**. Examples for late.fyi:
- "Why late.fyi deletes your email when the trip ends" — make the privacy invariant a story.
- "An email instead of an app" — the web-revival argument.
- "Why no accounts" — the user-experience case for stateless tools.

Privacy communities link to *posts*, not landing pages. The post lives forever, ranks naturally, and seeds backlinks to the tool. One good post outranks any sitemap trick over 12 months.

Ship those posts with an **RSS/Atom feed** (see Tier 2). Feed readers, aggregators, and planet-style indexes pick it up automatically — seeding backlinks and repeat discovery the post wouldn't earn on its own, and giving agents a structured, dateable record of what you've published.

## Audit checklist (per-site, run quarterly)

- [ ] `<title>` present, descriptive, includes value prop
- [ ] `<meta description>` present, one sentence, includes privacy stance
- [ ] `<link rel="canonical">` set to apex
- [ ] OpenGraph `og:title`, `og:description`, `og:url`, `og:type`, `og:site_name` present
- [ ] `og:image` present (absolute URL, 1200×630 PNG/JPG) OR explicitly skipped (knowing the cost: WhatsApp/Slack render a compact chain-icon card, not a banner)
- [ ] If `og:image` is set, `twitter:card` is `summary_large_image` and `og:image:width` / `og:image:height` are emitted
- [ ] `twitter:card` present
- [ ] `<link rel="icon">` present and renders
- [ ] `robots.txt` exists at root, references sitemap, and **names AI crawlers explicitly** (GPTBot/ClaudeBot/OAI-SearchBot/PerplexityBot) — allow-all for marketing pages, or the training-vs-retrieval split where intentional
- [ ] `sitemap.xml` exists at root, lists every public URL, emits `<lastmod>` (and omits `<changefreq>`/`<priority>` — Google ignores them)
- [ ] `llms.txt` present at root — curated markdown index, privacy invariant up top, links to quote-worthy pages (low-cost include; adoption still partial)
- [ ] RSS/Atom feed present if the site publishes posts
- [ ] Each page/section is **answer-first** — privacy invariant stated as one quotable sentence in the first 1–3 sentences
- [ ] Body content is extraction-friendly — real headings/lists, no content locked behind JS/accordions
- [ ] JSON-LD present for tools/posts — `SoftwareApplication`/`Article` plus `FAQPage`/`HowTo` where the content is Q&A- or step-shaped
- [ ] True PII / app paths are **auth-gated** (401/403/redirect for anonymous requests) — not just `Disallow`-ed. Public pages you want out of search use `noindex` **with crawling allowed**; never `Disallow` + `noindex` together (the crawler can't see a `noindex` on a blocked page)
- [ ] No analytics scripts in the page source (`grep -i 'analytics\|gtag\|plausible\|fathom\|umami'` returns clean)
- [ ] No tracking cookies (DevTools → Application → Cookies, empty for first-party)
- [ ] No third-party JS at all (DevTools → Network → filter by domain, only own domain visible)
- [ ] Privacy claim copy on the landing page matches what the code actually does (audit yearly; the privacy claim is a contract).
- [ ] **Outcome check — not just inputs.** Everything above is on-page *input*. Once per audit, verify the *output* the doc says matters most (#5, mentions): ask the major assistants (ChatGPT, Claude, Perplexity, Gemini) "alternatives to `<big-brand competitor>`" and "best `<category>` tool" and confirm you're actually mentioned/linked. No mention after the on-page work is done → the gap is distribution (Tier 3), not markup.

## Per-project notes

### late.fyi

Implemented 2026-05-04 (commit TBD): tier 1 head tags + robots.txt + sitemap.xml. Skipped JSON-LD on principle for now. og-card.png deferred — design task.

Note (post-2026-05-10): a WhatsApp test surfaced that "no og:image" doesn't render as title+description, it renders as a compact chain-icon card. Visually weaker than expected. Worth promoting og-card.png from "deferred" to a real ticket; the design lift is small (a logo on the brand colour, 1200×630 PNG) and the payoff is the entire link-preview surface across WhatsApp, Slack, Discord, iMessage.

The privacy-claim invariant is in `CLAUDE.md` (state/active/<msgid>.json deleted on terminal, no archive). The landing page section "What we don't do" must stay in sync; if retention changes, update the page first per CLAUDE.md.

### plato (terribic.com)

Shipped 2026-05-10 (commit `bb3cdcc`): `og:image` pointing to a static 1200×630 PNG of plato's three-dot mark on `--bg`, served at `/static/og.png`. `twitter:card` upgraded `summary` → `summary_large_image`. The PRD locks the og.png as **project identity, not fork-rebrandable** — the link-preview surface is where someone who has never seen plato first encounters the project, and consistency there pays for the recognition signal. Forks that want a different mark fork the repo; no `branding.*` knob.

The same pattern is the recommendation for sibling projects: ship a static PNG of the project mark on the brand bg colour, served from the app (no CDN — keeps single-binary integrity). The image is identity, not theming.

### (other projects)

Add sections per project as they ship.
