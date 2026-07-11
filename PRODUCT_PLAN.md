# AI Vibe Tester — Product Plan

> An automated "is my app ready for real users?" service for people who build apps
> and websites without an engineering background. Point it at your app, a swarm of
> bots exercises it like real (and hostile) users, and you get a single readiness
> score with plain-English findings and copy-paste fixes.

---

## 1. The thesis

Millions of people now build real, deployable apps with tools like Lovable, Bolt,
Replit, v0, Cursor, and Claude Code — without knowing engineering principles, system
design, architecture, security, or databases. They can't answer the questions that
matter before launch:

- Does every button and flow actually work, for inputs I didn't think to try?
- Can a stranger steal my users' data?
- Is my database doing the right thing?
- Will it fall over when traffic spikes?

Nobody wants to manually test a thousand inputs on every button. There is a real,
growing market for a product that answers "is this safe to show real people?" in one
click. The number of non-technical builders only goes up from here.

## 2. Market reality (as of 2026)

The space exists but is **fragmented into point solutions**. Three clusters:

1. **Security scanners for vibe apps** — VibeShield ($9/scan, $29/mo),
   Vibe App Scanner ($5–19), IsItSecure.ai (free), CheckVibe. Catch exposed API
   keys, broken Supabase Row-Level-Security (RLS), missing headers, injection.
2. **AI QA / test agents** — TestSprite, QA.tech, Shiplight, Mabl, testRigor,
   Checksum. Generate/run functional tests; mostly aimed at teams and QA engineers,
   often integrated into IDEs via MCP.
3. **Launch-readiness scorers** — Launch Ready Code is the closest analog: a free
   scan scoring an app 0–100 across security, reliability, performance, monitoring,
   wrapping pro tools (Semgrep, gitleaks, Trivy, k6) in founder-friendly language
   with copy-paste fixes.

**The gap to own:** no one has nailed the *consumer, one-click, "throw a swarm of
bots at my live app and tell me exactly what breaks"* experience for people who don't
know what RLS or a load test is. Each incumbent covers one pillar. The unmet job is a
**single unified verdict** plus a **fix-it loop** ("paste this into Lovable").

Documented, high-signal failure modes the market already proves are common:
- Supabase tables without RLS, queryable via the public anon key (one incident
  exposed 170+ Lovable apps).
- API keys placed directly in generated files / JS bundles (`sk-`, `AKIA`, `AIzaSy`).
- ~70% of Lovable apps with RLS disabled or partial.
- Nearly universal absence of error monitoring and structured logging.

## 3. Product concept — the bot swarm

The differentiated wedge is the **swarm**: point it at a URL (or connect the repo),
and autonomous agents *use* the app like thousands of impatient, confused, and
malicious users would — then return a plain-English report, one readiness score, and
paste-ready fixes.

### Four pillars, one score

| Pillar | What the bots do | Non-technical translation |
|---|---|---|
| **Functionality** | Crawl every page, click every control, fuzz every input, follow flows | "Does every button actually work?" |
| **Security** | Test auth boundaries, exposed secrets, DB/RLS access, injection, headers | "Can a stranger steal your users' data?" |
| **Data integrity** | Observe DB behavior under real actions; orphaned records, missing validation | "Is your database saving the right things?" |
| **Resilience / load** | Concurrent bots, spike traffic, slow-network simulation | "Will it survive going viral?" |

### Report is the product
- Single readiness score (0–100) with a clear go / not-yet verdict.
- Findings ranked P0–P3, each with a screenshot or recording of the break.
- A copy-paste fix prompt per finding, written for the user's builder tool.
- Shareable link (social proof + viral loop).

## 4. MVP — the sharpest first cut

Do **not** build all four pillars day one. Fastest path to a "wow" demo and revenue:

- **Input:** a live URL. No repo access required — lowest friction, works for any
  vibe-coded app. (Repo/GitHub connect is a fast-follow for deeper checks.)
- **Pillar 1 (Functionality):** a browser-driving agent (Playwright + an LLM planner)
  that autonomously explores the app, clicks controls, and fuzzes inputs; records
  each failure with a screenshot.
- **Slice of Pillar 2 (Security):** fast, deterministic, high-signal surface scan —
  exposed secrets in the JS bundle, missing security headers, open/unprotected
  Supabase tables. These are proven, cheap, and already validated by incumbents.
- **Output:** the shareable readiness report described above.

### Why this cut
- The **autonomous browser swarm** is the demo nobody else nails for consumers.
- The **deterministic security scan** is cheap (no LLM cost) and delivers instant,
  credible findings — perfect for a free tier hook.
- A URL-only flow removes all onboarding friction.

## 5. Technical architecture (MVP)

```
             ┌──────────────┐
  URL ─────▶ │  Web app /    │   submit target, view report
             │  dashboard    │
             └──────┬───────┘
                    │ enqueue job
             ┌──────▼───────┐
             │ Orchestrator  │   verifies domain ownership, budgets tokens,
             │ (job queue)   │   fans out to workers, aggregates results
             └──┬────────┬──┘
     ┌──────────▼──┐  ┌──▼───────────────┐
     │ Explorer     │  │ Security scanner  │
     │ workers      │  │ (deterministic)   │
     │ Playwright + │  │ secrets, headers, │
     │ LLM planner  │  │ RLS/open tables   │
     └──────┬───────┘  └────────┬──────────┘
            └──────────┬─────────┘
                ┌──────▼───────┐
                │ Scoring +     │  P0–P3 ranking, 0–100 score,
                │ report builder│  fix-prompt generation, screenshots
                └──────────────┘
```

- **Explorer worker:** headless Chromium via Playwright; an LLM plans the next action
  from the current DOM/screenshot (crawl, click, fill with fuzzed/edge-case inputs),
  detects errors (JS console errors, 4xx/5xx, unhandled states, broken navigations).
- **Security scanner:** static/dynamic checks — grep bundles for secret patterns,
  probe security headers, attempt anon reads against detected Supabase/Firebase
  endpoints. No LLM needed; runs fast and cheap.
- **Scoring + report:** normalize findings into P0–P3, compute the score, use an LLM
  to write the human explanation and the tool-specific fix prompt.
- **Cost control:** free tier = deterministic scan + a shallow, capped exploration.
  Deep swarm (long LLM-driven exploration, load testing) is paid and budgeted.

## 6. Positioning & differentiation

- **Consumer-first, not QA-engineer-first.** Zero jargon. One score, one verdict.
- **The swarm experience** — "we sent 1,000 bots at your app" — is visceral and
  demo-able, versus a static scan report.
- **The fix loop** — every finding ends in a paste-ready prompt for the user's
  builder, closing the gap between "found a problem" and "fixed it."
- **Unified verdict** across pillars, versus buying four separate point tools.

## 7. Business model

Freemium, mirroring where incumbents have landed:
- **Free:** deterministic surface scan + shallow exploration. Hook + viral report.
- **Deep Swarm (one-off):** full exploration + expanded security, ~$9–29 range.
- **Continuous monitoring (subscription):** re-run on every deploy, ~$29/mo.
- **Done-for-you audit (high tier):** branded report, ranked fix plan, ~$499.

## 8. Key risks & how to design around them

1. **Abuse / authorization (the #1 issue).** "Throw bots at any URL" is functionally
   a pentesting + load-testing service. **Require domain-ownership verification**
   (DNS TXT record or a file upload) before any deep, security, or load test.
   Rate-limit and sandbox. Without this, the product is a scan-and-DDoS-anyone tool —
   legally and ethically unacceptable.
2. **Crowded, cheap incumbents.** Differentiate on swarm + unified verdict + fix
   loop, never on a single check that a $5 tool already does.
3. **LLM cost per run.** Free tier must be deterministic; gate LLM-heavy exploration
   behind payment and hard token budgets per job.
4. **False positives erode trust.** A non-technical user can't triage noise. Bias
   toward high-confidence findings; every finding needs proof (screenshot/recording).

## 9. Roadmap sketch

- **Phase 0 (validate):** landing page + free deterministic security scan only.
  Measure demand and conversion intent before building the swarm.
- **Phase 1 (MVP):** URL-only functional swarm + security surface scan + report.
- **Phase 2:** domain verification, paid deep swarm, continuous monitoring.
- **Phase 3:** repo/GitHub connect for data-integrity + deeper DB checks; load testing.
- **Phase 4:** builder integrations (MCP for Cursor/Claude Code/Lovable) so testing
  runs *during* development, not just before launch.

## 10. Open questions to resolve next

- Domain-ownership verification flow — DNS TXT vs. file upload vs. OAuth into the host?
- URL-only vs. repo-connect for the very first version (friction vs. depth).
- Which builder(s) to optimize the fix-prompt output for first (Lovable? Bolt?).
- Free-tier cost ceiling that keeps unit economics positive.
