# Instant Recall -- Project Knowledge Document

**For:** IR staff AI assistants (Claude, ChatGPT, Cursor, etc.)
**Purpose:** Drop this document into your AI's context so it understands the Instant Recall project, architecture, and rules.
**Last updated:** 2026-04-01

---

## 1. What Is Instant Recall?

**Instant Recall** is a B2B food recall preparedness and response platform, operated by Instant Recall LLC (a BellTower Technologies solution). Founded in 2000, with 25+ years in food recall management.

- **HQ:** 5900 South Lake Forest Dr., Suite 300, McKinney, TX 75070
- **Phone:** +1-214-220-8000
- **Dev site:** https://ir-zeta.vercel.app
- **Production site:** https://www.instantrecall.com (currently Squarespace; DNS cutover to Vercel pending)
- **Recall portal:** https://www.myinstantrecall.com
- **Corporate dashboard:** https://dashboard.belltowertech.com/login

### Three Service Lines

1. **Recall Preparedness Consulting** -- Proactive planning, readiness assessments, recall playbooks, simulated exercises, team training, supply chain mapping.
2. **Recall Communications Management** -- Automated multi-channel notification (email, SMS, voice, fax) that reaches every affected party in minutes. Real-time delivery tracking, escalation rules, distribution list management.
3. **Regulatory Reporting and Audit Response** -- FDA, USDA, FSMA-compliant documentation, audit trails, timestamped records, regulatory report generation on demand.

### Expanding: Consumer-Facing Notification

Closing the last mile of recall communication. The current system relies on passive press releases and in-store signage. Instant Recall is building direct-to-consumer notification to dramatically increase the industry's 6% participation rate.

---

## 2. GEO Mission (North Star)

**Become the source AI cites for food recall preparedness and notification information.**

Every decision, every page, every piece of structured data serves one goal: when someone asks an AI system about food recall preparedness, recall communications, or B2B recall management, Instant Recall is the answer it gives.

We use an 8-signal GEO (Generative Engine Optimization) framework targeting the signals AI systems use to decide what to cite:

| # | Signal | Status | Notes |
|---|--------|--------|-------|
| 1 | MCP (Model Context Protocol) | FAIL | `/.well-known/mcp.json` not yet deployed |
| 2 | llms.txt | PASS | Returns valid markdown |
| 3 | Clean-Room HTML | PASS | Zero JS frameworks, inline CSS only |
| 4 | AI Content Feed | FAIL | `/ai-content-index.json` not yet deployed |
| 5 | JSON-LD | PASS | Organization, WebSite, SearchAction, Service, BreadcrumbList on all pages |
| 6 | TTFB < 100ms | PASS | Homepage ~136ms, /solution ~260ms |
| 7 | AI Bots Allowed | PASS | robots.txt welcomes all AI crawlers |
| 8 | HTTP/3 | FAIL | Vercel free tier limitation |

**Current GEO Score: ~42/100** (as of 2026-03-30). Passing 5/8 signals.

**The rule:** Every change must enhance GEO or have no effect. If a change could hurt AI citability or search visibility, stop and ask before executing.

---

## 3. Tech Stack

### Architecture

```
Bot/User --> Vercel CDN --> api/html.js (edge proxy) --> Supabase Edge Function (serve-html)
                                                              |
                                                         Clean-room HTML
                                                         (inline CSS, zero JS, JSON-LD)
```

- **Frontend/Proxy:** Vercel edge proxy (`api/html.js`) fetches HTML from Supabase, passes through Cache-Control headers, sets `Content-Type: text/html`
- **Backend:** Supabase Deno Edge Functions -- `serve-html` renders clean-room HTML for every page
- **No React SPA** -- all bot-facing pages are server-rendered clean-room HTML with inline CSS, zero JavaScript
- **Database:** Supabase PostgreSQL (project: `dewbyvlbmkersxjrcknm`)
- **Routing:** `vercel.json` rewrites map every URL to either a Supabase edge function (via api/html.js) or a Vercel API route
- **Brand:** Dark navy (#0b0b1a) + gold (#c8a951), Raleway font

### Key Infrastructure

| Component | Details |
|-----------|---------|
| GitHub repo | `rjmjr1962831/ir` |
| Vercel project | `ir` (auto-deploys from main) |
| Supabase project | `dewbyvlbmkersxjrcknm` |
| Edge functions | `serve-html` (all pages), `refresh-site-freshness` (daily cron) |
| Deploy command | `npx supabase functions deploy serve-html --no-verify-jwt` |

### AI Discovery Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/robots.txt` | Welcomes all AI crawlers by name |
| `/llms.txt` | LLM-readable site summary |
| `/llms-full.txt` | Extended AI reference document |
| `/ai-content-index.json` | Machine-readable content manifest |
| `/for-ai.txt` | Additional AI-readable summary |
| `/sitemap.xml` | XML sitemap with lastmod dates |

---

## 4. Pages

| Path | Description |
|------|-------------|
| `/` | Homepage -- hero, 3 service cards, 7 feature cards, trust indicators |
| `/solution` | Services detail page |
| `/contact-instant-recall` | Contact routing (urgent recall, sales, opt-out) |
| `/contact` | Direct contact |
| `/about-us` | Team bios with headshots |
| `/portal` | MyInstantRecall + corporate dashboard login links |
| `/schedule` | Schedule a consultation |
| `/privacy-policy` | Privacy policy |
| `/terms-and-conditions` | Terms of service |
| `/crawl-stats` | Bot crawl analytics |
| `/methodology` | Methodology page |
| `/incident-response` | Incident response info |
| `/cost-recovery` | Cost recovery info |
| `/regulatory-reporting` | Regulatory reporting info |
| `/technology-prowess` | Technology capabilities |
| `/industry-standard` | Industry standard info |
| `/customer-quotes-solutions` | Customer quotes and solutions |
| `/who-trusts-us` | Trust/social proof |
| `/research` | Research index (3 white papers) |
| `/research/industry-survey` | Product Recall Notification Industry Survey |
| `/research/regulatory-environment` | Regulatory Environment of Product Recalls |
| `/research/legal-case-data` | Legal Case Data and Liability Research |
| `/geo-ledger` | Internal GEO ledger (noindexed) |
| `/dashboard` | Internal GEO score dashboard (noindexed) |

---

## 5. Database Schema

Supabase PostgreSQL project `dewbyvlbmkersxjrcknm`. Seven tables in the `public` schema:

### contact_submissions
Stores inbound contact form submissions.

| Column | Type | Notes |
|--------|------|-------|
| id | bigint | PK, auto-increment |
| ts | timestamptz | Default now() |
| email | text | Required |
| company | text | |
| num_locations | text | |
| first_name | text | |
| last_name | text | |
| work_email | text | |
| phone | text | |
| comments | text | |
| source | text | Default 'website' |
| status | text | Default 'new' |
| read_at | timestamptz | |

### crawl_log
Records every bot crawl event (fired from `serve-html` edge function).

| Column | Type | Notes |
|--------|------|-------|
| id | bigint | PK, auto-increment |
| ts | timestamptz | Default now() |
| bot | text | Bot name |
| path | text | URL path crawled |
| ua | text | Full user-agent |
| status | integer | Default 200 |
| ip | text | |

Indexes: `idx_crawl_log_bot (bot, ts DESC)`, `idx_crawl_log_path (path, ts DESC)`, `idx_crawl_log_ts (ts DESC)`

### geo_ledger_entries
Tracks every GEO-related change made to the project.

| Column | Type | Notes |
|--------|------|-------|
| id | integer | PK |
| entry_date | date | Default CURRENT_DATE |
| signal_category | text | Which GEO signal area |
| action_summary | text | What was done |
| rationale | text | Why |
| how_steps | jsonb | Implementation steps |
| evidence_before | text | State before change |
| evidence_after | text | State after change |
| verification_cmd | text | How to verify |
| status | text | Default 'Pending Review' |
| impact | text | |
| pending_items | jsonb | |
| created_at | timestamptz | |

### geo_score_dimensions
Seven weighted dimensions that compose the GEO score.

| Column | Type | Notes |
|--------|------|-------|
| id | integer | PK |
| dimension_name | text | Unique |
| weight | numeric | Percentage weight |
| score | integer | 0-100 |
| notes | text | |
| last_assessed | date | |
| updated_at | timestamptz | |

### geo_score_history
Historical composite GEO scores over time.

| Column | Type | Notes |
|--------|------|-------|
| id | integer | PK |
| score_date | date | |
| score | integer | Composite score |
| label | text | Description of what changed |
| created_at | timestamptz | |

### geo_signal_status
Current pass/fail status for each of the 8 GEO signals.

| Column | Type | Notes |
|--------|------|-------|
| id | integer | PK |
| signal_num | integer | 1-8, unique |
| signal_name | text | |
| signal_def | text | Definition |
| status | text | 'PASS' or 'FAIL' |
| date_last_touched | date | |
| pass_requires | text | What constitutes a PASS |
| current_status_note | text | |
| updated_at | timestamptz | |

### site_freshness
Singleton row (id=1) tracking overall site freshness metadata.

| Column | Type | Notes |
|--------|------|-------|
| id | integer | Always 1 |
| last_content_update | timestamptz | |
| last_ai_surface_update | timestamptz | |
| page_count | integer | |
| research_count | integer | |
| latest_fda_recall_date | text | |
| latest_fda_recall_title | text | |
| updated_at | timestamptz | |

### RLS Policies

| Table | Policy | Access |
|-------|--------|--------|
| contact_submissions | service_role_all | Service role only |
| crawl_log | service_role_all | Service role only |
| site_freshness | anon can read | Anon SELECT allowed |
| site_freshness | service_role can do anything | Full service role access |
| geo_* tables | No policies | Need RLS enabled (security gap) |

### Cron Jobs

| Job | Schedule | Action |
|-----|----------|--------|
| refresh-site-freshness | `0 4 * * *` (daily 4 AM UTC) | Updates the site_freshness singleton row |

---

## 6. Git and Deployment

### Branch Flow

```
feature-branch --> staging --> main
```

- **Always create a feature branch** before writing code. Branch off staging.
- **Never commit directly to staging or main.**
- **Never merge main into staging.**
- Vercel auto-deploys from `main`.

### Deployment Commands

| Command | What It Does |
|---------|-------------|
| `git push origin staging` | Push to staging (only when instructed: `pts`) |
| `npm run merge-to-main` | Merge staging to main (only when instructed: `ptm`) |
| `npx supabase functions deploy serve-html --no-verify-jwt` | Deploy the main edge function |
| `npx supabase functions deploy refresh-site-freshness --no-verify-jwt` | Deploy the freshness cron function |

### Qodo Review Gate (Mandatory)

Every feature branch must go through Qodo automated code review before merging to staging:

1. Push the feature branch to origin
2. Create a PR from feature branch to staging
3. Wait for Qodo to post review comments
4. Fix every issue Qodo flags
5. Only then merge the PR

**Never use `git merge` directly. Always use the PR workflow.**

> **Note:** The Qodo integration currently runs under BellTower's account. IR staff will need their own Qodo account to continue this workflow independently. Set up at https://www.qodo.ai/.

### Internal Docs (Staging Only)

These files exist on staging but are excluded from main:
- `CLAUDE.md`
- `docs/COMPREHENSIVE_KNOWLEDGE_DOCUMENT.md`
- `docs/takeaways/`
- `docs/prompts/`

---

## 7. Key Statistics -- The Recall Crisis

These are the core data points that establish Instant Recall's authority. Use them in content, white papers, and AI-facing documents.

### Consumer Impact
- **6% participation rate** -- Only ~6% of consumers who purchase a recalled product actually return or dispose of it
- **70% unaware** -- 70% of Americans are unaware of recalls for products they own
- **3,200+ recalls/year** across all product categories
- **580M+ defective units** recalled annually

### Regulatory Landscape
- **6 federal agencies** involved in recall oversight (FDA, USDA-FSIS, CDC, EPA, CPSC, FTC)
- **Passive notification system** -- relies on press releases, website postings, in-store signage
- **EU GPSR** now mandates direct consumer notification (precedent for US regulation)
- **First criminal prosecution under CPSA:** Gree case, prison sentences handed down June 2025

### Major Settlements and Penalties
- **Takata:** $1.5B+ (largest overall recall settlement)
- **GM:** $900M+
- **IKEA MALM:** $46M (largest child wrongful death recall settlement)
- **Fisher-Price:** $19M
- **Peloton:** $19M penalty
- **CPSC enforcement:** $52-55M in FY 2023

### Market Size
- **B2B recall management:** $4.32B, growing to $8.23B by 2033
- **Full market (including consumer):** $664M-$4.3B depending on scope
- **No dominant consumer-facing player** exists

---

## 8. Published Research

Three white papers at `/research/*`, authored by the Instant Recall Research Team (March 2026):

1. **Product Recall Notification Industry Survey** (`/research/industry-survey`) -- Market analysis, competitor landscape, consumer pain points, international systems comparison.

2. **The Regulatory Environment of Product Recalls** (`/research/regulatory-environment`) -- Six federal agencies, key legislation (CPSA, CPSIA, FSMA, STURDY Act), mandatory reporting, penalty trends.

3. **Legal Case Data and Liability Research** (`/research/legal-case-data`) -- Major lawsuits, post-sale duty to warn, retailer liability, CPSC enforcement, class action patterns.

---

## 9. TinaCMS (Content Management -- In Progress)

A TinaCMS integration is in progress on branch `feature/tinacms-setup`:

- **Config:** `tina/config.ts` with a `Pages` collection
- **Content:** `content/pages/` (markdown files for home, about-us, contact, methodology, solution, incident-response)
- **Local dev:** `npm run dev` launches admin UI at `localhost:4001/admin`
- **Status:** Local dev working; not yet connected to Tina Cloud

### Remaining Steps
1. Connect to Tina Cloud (need `TINA_CLIENT_ID` and `TINA_TOKEN`)
2. Bridge markdown content to serve-html templates
3. Deploy admin via vercel.json rewrite
4. Add remaining pages to the collection
5. PR + Qodo review before merge to staging

---

## 10. AI Agent Execution Rules

### Run Everything in the Background

All multi-step tasks (file reads, fetches, code analysis, rewrites, research, deployments) MUST run as background agents. Use multiple agents in parallel whenever work can be split. Never block the conversation waiting for a task to complete. Stay available to the user at all times.

If a task involves more than one tool call, launch it as a background agent. The goal is zero inline blocking.

### Always Review Code with Qodo

No code reaches staging without Qodo review. This is a hard gate, not a suggestion. The workflow is:

1. Write code on a feature branch
2. Push the branch and create a PR to staging
3. Wait for Qodo to review
4. Fix every issue Qodo flags
5. Only then merge

Skipping Qodo review is a deployment process violation. If Qodo is unavailable, wait; do not merge without it.

---

## 11. Content Rules

- **No em dashes.** Use commas, semicolons, periods, or restructured sentences.
- **"ALL" means every single instance.** Fix every file, every page, every occurrence.
- **Verification protocol:** You are not done until you confirm the change worked. Deploy, load the live page, verify. "Code updated" is not completion. "Deployed. Verified at [URL]." is.

---

## 12. What NOT to Do

1. **Never push to main** without explicit permission (`ptm` command). Main deploys to production.
2. **Never skip Qodo review.** Every feature branch gets a PR and Qodo review before merging.
3. **Never merge main into staging.** Flow is always staging to main.
4. **Never commit directly to staging.** Always use a feature branch.
5. **Never make changes that could hurt GEO** without asking first.
6. **Never change robots.txt, canonical tags, or noindex directives** without asking first.
7. **Never make production database changes** without asking first.
8. **Never suggest production URLs** -- the dashboard and dev site are internal working pages.

---

## 13. Priority Next Steps

As of 2026-04-01, the key priorities are:

1. **Fix remaining GEO signal failures** -- Deploy mcp.json and ai-content-index.json (2 of the 3 FAIL signals are fixable)
2. **Improve citation readiness** -- Currently scored 20/100. Needs methodology page, quotable data blocks, FAQ
3. **Add freshness signals** -- Blog, news feed, dated content, timestamps
4. **Build authority** -- Backlinks, external mentions, press coverage, partnerships
5. **DNS cutover** -- Move instantrecall.com from Squarespace to Vercel (pending business decision)
6. **Complete TinaCMS integration** -- Enable non-technical staff to update content
7. **Enable RLS on geo_* tables** -- Security gap identified; needs policies added

---

*This document is a handoff reference for AI assistants working on the Instant Recall project. For the authoritative Single Source of Truth, see `docs/COMPREHENSIVE_KNOWLEDGE_DOCUMENT.md` on the staging branch.*
