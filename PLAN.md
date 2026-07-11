# AI Vibe Tester — Product Plan

**One-liner:** "You vibe-coded it. We make sure it actually works." A consumer-grade launch-readiness platform that throws AI test bots at your app, probes your security and database, and tells you in plain English whether you can ship — with copy-paste fixes for the AI tool you built it with.

**Status:** Planning / pre-validation
**Last updated:** 2026-07-11

---

## 1. The Problem

Millions of people are building apps with Lovable, Bolt, Replit, v0, Cursor, and Claude Code without knowing engineering principles, system design, security, or databases. Their apps *seem* to work, but the builders have no way to answer:

- **Does it actually work?** Nobody wants to manually test a thousand inputs across every button and form.
- **Is it secure?** Exposed API keys, missing auth checks, open databases.
- **Is the database sound?** Will data leak, corrupt, or fall over?
- **Can it handle real users?** Nobody has load-tested anything.
- **Can I release this?** No trusted "you're good to go" signal exists for non-engineers.

The builder population without engineering skills is growing far faster than the population of people who can audit their work.

## 2. Honest Market Assessment

**The original assumption — "no product exists for this" — is wrong. The refined version is right: no product does the whole job, for this audience, in one place.**

The market has three fragmented clusters, and each fails the vibe coder in a different way:

### Cluster A: AI QA/testing agents (built for engineers)
TestSprite, QA.tech, Momentic, Shiplight, testRigor, Mabl, Autify, Checksum, TestBooster.
- These are genuinely good at agentic E2E testing ("vibe testing" is now an established category).
- **Gap:** They target dev teams and QA orgs. They live in IDEs, MCP servers, CI pipelines, and YAML files. Pricing, onboarding, and language assume you know what a test suite is. A Lovable user bounces off in minutes.

### Cluster B: Point security scanners for vibe-coded apps (cheap, shallow)
VibeShield ($9/scan), Vibe App Scanner ($5–19), CheckVibe, IsItSecure.ai (free).
- Proof of demand: these exist *because* vibe coders are paying for reassurance. Documented carnage backs them up (170+ Lovable apps with Supabase tables readable via the public anon key; ~70% of Lovable apps with RLS misconfigured; API keys shipped in JS bundles).
- **Gap:** Security-only, mostly static/surface-level URL scans. No functional testing, no load testing, no "does the app actually work" answer.

### Cluster C: Checklists and human audit services
LaunchReadyCode (free scan + $499 audit), agency code audits, blog checklists.
- LaunchReadyCode is the closest competitor to this idea: scores security/reliability/performance/monitoring out of 100 in founder-friendly language, with copy-paste fixes. Take it seriously.
- **Gap:** Wraps existing static tools (Semgrep, gitleaks, Trivy, k6). It does not *use* the app. Nobody in this cluster sends bots through your actual flows — signing up, submitting garbage inputs, racing two carts for the last item in stock.

### The open lane
**Nobody combines agentic functional testing (Cluster A's core skill) with security/database/load checks (B and C) in a product a non-engineer can run from a URL and understand.** The wedge is the part vibe coders feel most viscerally and no consumer product delivers: *"I don't want to test 1000 inputs on every button — do it for me."* Security scans are table stakes to bundle in, not the wedge, because Cluster B has already commoditized them at $5–29.

## 3. Target User

**Primary:** The non-technical builder — solo founder, indie hacker, small-business owner, hobbyist — who built something real on Lovable / Bolt / Replit / v0 and is about to put it in front of customers or has just started charging money. They cannot read a Semgrep finding. They *can* paste a fix prompt into their AI tool.

**Secondary (later):** Agencies and freelancers shipping vibe-coded apps for clients who need a credible "tested" report to hand over. This segment pays more and churns less.

**Explicitly not the target (at first):** Engineering teams with QA processes. Cluster A owns them and competing there is a losing fight.

## 4. Product Definition

### The core loop
1. **Point us at your app** — paste a URL. Optionally connect the repo (GitHub) and database (Supabase first — it dominates this ecosystem) for deeper checks.
2. **The swarm runs** — AI agents explore the app like real users while scanners probe underneath. The user can literally watch bots click through their app (this is the demo, the marketing, and the trust-builder in one).
3. **Get a verdict** — a Launch Readiness Score with plain-English findings: what's broken, what it means, how bad it is, screenshot/video proof.
4. **Fix it with your own AI** — every finding ships as a copy-paste prompt tailored to the user's tool ("Paste this into Lovable"). We don't make them learn anything; we speak to the AI that wrote the bug.
5. **Re-run and watch the score climb** — the gamified retest loop is the retention mechanic and the upsell into continuous monitoring.

### Test dimensions
| Dimension | What runs | Examples |
|---|---|---|
| Functional | Browser agents explore and exercise every flow | Dead buttons, broken forms, unreachable pages, error states |
| Input abuse | Fuzzing every field the agents find | Emoji, 10k-char strings, SQLi/XSS payloads, negative numbers, malformed uploads |
| Security | Surface + auth probing | Exposed keys in bundles, missing headers, open Supabase/Firebase tables, broken RLS, injection |
| Database | Behavior under real actions | Orphaned records, missing validation, data that saves wrong, leaks across users |
| Resilience | Concurrent + spike load | Does it survive going viral, slow-network behavior, race conditions |

## 5. MVP — sharpest first cut

Do **not** build all dimensions on day one. Fastest path to a "wow" demo and revenue:

- **Input:** a live URL only. No repo/DB connect required — lowest friction, works
  for any vibe-coded app. Repo + Supabase connect is a fast-follow for depth.
- **Functional swarm:** a browser-driving agent (Playwright + an LLM planner) that
  autonomously explores, clicks every control, and fuzzes every field, recording
  each break with a screenshot/video.
- **Security surface scan (deterministic):** exposed secrets in the JS bundle,
  missing headers, open Supabase tables. Cheap (no LLM cost), instant, credible —
  the perfect free-tier hook.
- **Output:** a shareable Launch Readiness report — score, P0–P3 findings with
  proof, and a copy-paste fix prompt per finding.

Rationale: the autonomous swarm is the demo no consumer competitor delivers; the
deterministic scan is free-tier bait with zero token cost; URL-only removes all
onboarding friction.

## 6. Technical architecture (MVP)

```
  URL ─▶ Dashboard ─▶ Orchestrator (verifies domain ownership, budgets tokens,
                          fans out, aggregates)
                          │
              ┌───────────┴───────────┐
     Explorer workers            Security scanner
     Playwright + LLM planner    (deterministic:
     crawl / click / fuzz         secrets, headers,
     detect JS errors, 4xx/5xx    open tables)
              └───────────┬───────────┘
                   Scoring + report builder
                   (P0–P3, 0–100 score, fix-prompt generation, media proof)
```

- **Explorer:** headless Chromium; LLM plans the next action from DOM/screenshot;
  detects console errors, 4xx/5xx, unhandled states, broken navigation.
- **Security scanner:** static/dynamic pattern checks; no LLM needed; fast + cheap.
- **Scoring/report:** normalize to P0–P3, compute score, LLM writes the plain-English
  explanation and tool-specific fix prompt.
- **Cost control:** free = deterministic scan + shallow capped exploration; deep
  swarm and load testing are paid and hard-budgeted per job.

## 7. Business model

Freemium, matching where the market sits:
- **Free:** deterministic surface scan + shallow exploration → hook + viral report.
- **Deep Swarm (one-off):** full exploration + expanded security, ~$9–29.
- **Continuous monitoring (subscription):** re-run on every deploy, ~$29/mo.
- **Done-for-you audit (high tier):** branded report + ranked fix plan, ~$499.
- **Agencies (secondary segment):** higher-priced, lower-churn, credible handoff report.

## 8. Key risks & mitigations

1. **Authorization / abuse — the #1 issue.** "Point bots at any URL" is functionally
   a pentest + load-test service. **Require domain-ownership verification** (DNS TXT
   or file upload) before any deep, security, or load test. Rate-limit and sandbox.
   Without this it's a scan-and-DDoS-anyone tool — legally and ethically unacceptable.
2. **Crowded, cheap incumbents.** Differentiate on swarm + unified verdict + fix
   loop, never on a single check a $5 tool already commoditized.
3. **LLM cost per run.** Free tier must be deterministic; gate LLM-heavy exploration
   behind payment and hard token budgets.
4. **False positives erode trust.** Non-technical users can't triage noise. Bias to
   high-confidence findings; every finding needs proof (screenshot/recording).

## 9. Roadmap

- **Phase 0 (validate):** landing page + free deterministic security scan. Measure
  demand and conversion before building the swarm.
- **Phase 1 (MVP):** URL-only functional swarm + security surface scan + report.
- **Phase 2:** domain verification, paid deep swarm, continuous monitoring.
- **Phase 3:** repo + Supabase connect for data-integrity/DB checks; load testing.
- **Phase 4:** builder integrations (MCP for Cursor/Claude Code/Lovable) so testing
  runs *during* development, not just before launch.

## 10. Open questions

- Domain-ownership verification flow — DNS TXT vs. file upload vs. host OAuth?
- URL-only vs. repo-connect for the very first version (friction vs. depth).
- Which builder to optimize fix-prompt output for first (Lovable? Bolt?).
- Free-tier cost ceiling that keeps unit economics positive.