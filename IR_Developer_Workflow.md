# IR (Instant Recall) Developer Workflow Guide

> Practical, step-by-step reference for any developer (human or AI) working on the Instant Recall codebase.
> Repository: `rjmjr1962831/ir`

---

## 1. Environment Setup

### Prerequisites

- **Git** (any recent version)
- **Node.js** (v18+) and npm
- **Supabase CLI** (`npm install -g supabase` or `npx supabase`)
- **Vercel CLI** (optional, for manual deploys: `npm install -g vercel`)

### Clone and Branch

```bash
git clone https://github.com/rjmjr1962831/ir.git
cd ir
git checkout staging
npm install
```

### Environment Variables

Create a `.env` file in the project root with these keys (never commit actual values):

| Variable | Purpose |
|---|---|
| `SUPABASE_URL` | Supabase project URL (`https://dewbyvlbmkersxjrcknm.supabase.co`) |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key (used by edge functions and API routes) |
| `SUPABASE_ACCESS_TOKEN` | Personal access token for Supabase CLI deployments |
| `VERCEL_TOKEN` | Vercel API token (used for force-redeploy commands) |

---

## 2. Architecture Overview

### Request Flow

```
Browser/Bot
    |
    v
Vercel Edge (vercel.json rewrites)
    |  e.g. /solution -> /api/html?path=%2Fsolution
    v
api/html.js  (Vercel Edge Function -- thin proxy)
    |  Fetches from Supabase, forwards path via x-forwarded-path header
    v
Supabase Edge Function: serve-html/index.ts  (Deno runtime)
    |  Matches path in ROUTES map, calls page renderer
    v
Page renderer (e.g. pages/solution.ts)
    |  Returns clean-room HTML string
    v
shared/layout.ts  (wraps body in full HTML document with header, footer, JSON-LD, meta tags)
    |
    v
Response flows back through api/html.js which sets Content-Type: text/html
```

**Why the proxy?** Supabase gateway forces `Content-Type: text/plain` on edge function responses. The Vercel proxy (`api/html.js`) re-wraps the response with `Content-Type: text/html; charset=utf-8` and passes through upstream `Cache-Control` headers.

### Directory Structure

```
ir/
├── api/                          # Vercel Edge Functions (proxy + AI discovery endpoints)
│   ├── html.js                   # Main proxy: forwards to Supabase serve-html
│   ├── sitemap.js                # /sitemap.xml
│   ├── llms.js                   # /llms.txt
│   ├── llms-full.js              # /llms-full.txt
│   ├── ai-content-index.js       # /ai-content-index.json
│   ├── robots.js                 # /robots.txt
│   ├── for-ai.js                 # /for-ai, /for-ai.txt
│   ├── mcp.js                    # /.well-known/mcp.json
│   ├── contact-submit.js         # Contact form handler
│   └── nightly-refresh.js        # Cron job (runs daily at 5 AM UTC)
│
├── supabase/functions/serve-html/ # The main edge function (Deno)
│   ├── index.ts                  # Router: imports pages, maps paths, bot detection, logCrawl
│   ├── shared/
│   │   ├── layout.ts             # renderPage() -- wraps body in full HTML document
│   │   ├── styles.ts             # CSS constant -- shared styles for all pages
│   │   ├── freshness.ts          # FreshnessData cache from site_freshness table
│   │   └── citations.ts          # Citation blocks + CSS for AI-facing data sections
│   └── pages/                    # One file per page
│       ├── home.ts
│       ├── solution.ts
│       ├── contact.ts
│       ├── about.ts
│       ├── research-index.ts
│       ├── ... (28 page files total)
│       └── work-log.ts
│
├── vercel.json                   # Rewrites, headers, cron config
├── public/                       # Static assets (images, favicon)
├── admin/                        # Admin panel static files
└── CLAUDE.md                     # Project SSoT and execution rules
```

### Shared Modules

| Module | What it provides |
|---|---|
| `shared/layout.ts` | `renderPage({ title, description, path, jsonLd?, body })` -- wraps any page body in a full HTML document with header, footer, canonical URL, Open Graph tags, BreadcrumbList JSON-LD, WebPage JSON-LD with freshness dates, and HubSpot script |
| `shared/styles.ts` | `CSS` constant -- all shared CSS (header, hero, cards, features, footer, responsive breakpoints) inlined into every page via `<style>` tag |
| `shared/freshness.ts` | `getFreshness()` / `setCurrentFreshness()` / `getCurrentFreshness()` -- fetches and caches `site_freshness` table data (last content update, page count, latest FDA recall, etc.) |
| `shared/citations.ts` | `homeCitationBlock()`, `methodologyBlock()`, `CITATION_CSS` -- pre-formatted, citable data blocks with Schema.org Dataset JSON-LD for AI systems to quote |

---

## 3. How to Add a New Page

### Step 1: Create the page file

Create `supabase/functions/serve-html/pages/my-new-page.ts`:

```typescript
import { renderPage } from "../shared/layout.ts";

const JSON_LD = JSON.stringify({
  "@context": "https://schema.org",
  "@type": "WebPage",
  name: "My New Page Title",
  description: "Description of the page for structured data.",
  url: "https://www.instantrecall.com/my-new-page",
});

export function renderMyNewPage(): string {
  return renderPage({
    title: "My New Page Title | Instant Recall",
    description: "Meta description for search engines and AI systems (under 160 chars).",
    path: "/my-new-page",
    jsonLd: JSON_LD,
    body: `
      <section class="hero" style="background-image:url('/images/your-hero.jpg')">
        <div class="hero-overlay" style="background:rgba(0,0,0,0.65)"></div>
        <div class="hero-content">
          <h1>My New Page Heading</h1>
          <p>Subheading text here.</p>
        </div>
      </section>

      <div class="container">
        <h2 class="section-title">Section Title</h2>
        <p class="section-subtitle">Section description.</p>

        <div class="cards">
          <article class="card">
            <h3>Card Title</h3>
            <p>Card content with semantic HTML.</p>
          </article>
        </div>
      </div>
    `,
  });
}
```

### Step 2: Add import and route in index.ts

Open `supabase/functions/serve-html/index.ts`:

**Add the import** (alphabetical with existing imports):

```typescript
import { renderMyNewPage } from "./pages/my-new-page.ts";
```

**Add the route** to the `ROUTES` map:

```typescript
const ROUTES: Record<string, () => string> = {
  // ... existing routes ...
  "/my-new-page": renderMyNewPage,
};
```

### Step 3: Add vercel.json rewrite

Open `vercel.json` and add to the `rewrites` array:

```json
{ "source": "/my-new-page", "destination": "/api/html?path=%2Fmy-new-page" }
```

Note: The `path` query parameter value must be URL-encoded (`/` becomes `%2F`). For nested paths like `/research/something`, encode as `%2Fresearch%2Fsomething`.

### Step 4: Add to sitemap

Open `api/sitemap.js` and add to the `urls` array:

```javascript
{ loc: "/my-new-page", priority: "0.7", changefreq: "monthly" },
```

### Step 5: Add to llms.txt

Open `api/llms.js` and add a line to the text body:

```
- [My New Page Title](https://www.instantrecall.com/my-new-page): Brief description of the page content.
```

Also update `api/llms-full.js` if the page has substantive content worth including in the full version.

### Step 6: Add to ai-content-index.json

Open `api/ai-content-index.js` and add to the `pages` array:

```javascript
{
  url: "https://www.instantrecall.com/my-new-page",
  title: "My New Page Title",
  description: "Brief description.",
  content_type: "informational",  // or "research", "service", "legal"
  last_updated: lastUpdated,
},
```

### Step 7: Add breadcrumb name (for SEO)

Open `shared/layout.ts` and add to the `BREADCRUMB_NAMES` map:

```typescript
"/my-new-page": "My New Page Title",
```

### Step 8: Add dashboard card (if applicable)

If the page should appear on the internal dev dashboard, update `pages/dashboard.ts` to include a card for it.

### Checklist

- [ ] Page file created in `pages/`
- [ ] Import added to `index.ts`
- [ ] Route added to `ROUTES` map in `index.ts`
- [ ] Rewrite added to `vercel.json`
- [ ] Breadcrumb name added to `layout.ts`
- [ ] Entry added to `api/sitemap.js`
- [ ] Entry added to `api/llms.js` (and `llms-full.js` if appropriate)
- [ ] Entry added to `api/ai-content-index.js`
- [ ] JSON-LD structured data included in the page
- [ ] Page uses semantic HTML, inline CSS, no client-side JS rendering

---

## 4. How to Modify an Existing Page

### Find the page file

The naming convention maps URL paths to filenames:

| URL | File |
|---|---|
| `/` | `pages/home.ts` |
| `/solution` | `pages/solution.ts` |
| `/contact-instant-recall` | `pages/contact.ts` (`renderContact`) |
| `/contact` | `pages/contact.ts` (`renderContactDirect`) |
| `/research` | `pages/research-index.ts` |
| `/research/industry-survey` | `pages/research-industry.ts` |
| `/who-trusts-us` | `pages/press.ts` |

Check the `ROUTES` map in `index.ts` to find the exact render function and trace it to its import.

### Edit the content

All page content is TypeScript template literals returning HTML strings. Edit the HTML directly. Use the shared CSS classes from `styles.ts` (`.hero`, `.container`, `.cards`, `.card`, `.section-title`, `.section-subtitle`, `.features`, `.feature`).

### Deploy and verify

After editing, deploy the edge function (see Section 6) and verify with curl:

```bash
curl -s -H "Cache-Control: no-cache" "https://ir-zeta.vercel.app/my-page" | head -50
```

---

## 5. Git Workflow (Critical)

### The Rules

1. **ALWAYS create a feature branch from staging before writing code.**
2. **NEVER commit directly to staging or main.**
3. **ALWAYS run code through Qodo review before merging to staging.**
4. **Branch flow is one-way: staging -> main. Never merge main into staging.**

### Step-by-Step

```bash
# 1. Start from staging
cd C:\Users\rober\ir
git checkout staging
git pull origin staging

# 2. Create a feature branch
git checkout -b feature/my-change

# 3. Make your changes
# ... edit files ...

# 4. Commit
git add <specific-files>
git commit -m "Add my-new-page with JSON-LD and citation blocks"

# 5. Push feature branch
git push origin feature/my-change

# 6. Create a PR to staging
gh pr create --base staging --title "Add my-new-page" --body "Description of changes"

# 7. WAIT FOR QODO REVIEW (mandatory gate)
# Poll for Qodo comments -- do not ask the user to paste them
# Fix all issues Qodo raises
# Push fixes to the feature branch

# 8. After Qodo approval, merge PR to staging
gh pr merge <PR-number> --merge

# 9. Deploy (see Section 6)
```

### Special Commands (only when instructed by Robert)

| Command | What it does |
|---|---|
| `pts` | Push to staging: `git add . && git commit && git push origin staging`, then deploy edge function, then force-redeploy Vercel |
| `ptm` | Push to main: runs `npm run merge-to-main` only. **Never touch main without explicit `ptm` instruction.** |

---

## 6. Deploying Edge Functions

### When to deploy

Any time you change files under `supabase/functions/serve-html/` (including `index.ts`, any page in `pages/`, or any shared module).

### Deploy command

```bash
cd C:\Users\rober\ir
SUPABASE_ACCESS_TOKEN=$SUPABASE_ACCESS_TOKEN npx supabase functions deploy serve-html --no-verify-jwt --project-ref dewbyvlbmkersxjrcknm
```

(Load `SUPABASE_ACCESS_TOKEN` from `.env`)

### Verify deployment

```bash
# Test a specific page through the full proxy chain
curl -s -H "Cache-Control: no-cache" "https://ir-zeta.vercel.app/solution" | head -20

# Test the edge function directly (returns text/plain due to Supabase gateway)
curl -s "https://dewbyvlbmkersxjrcknm.supabase.co/functions/v1/serve-html" \
  -H "x-forwarded-path: /solution" | head -20

# Verify Content-Type through Vercel proxy
curl -sI "https://ir-zeta.vercel.app/solution" | grep -i content-type
# Should show: Content-Type: text/html; charset=utf-8
```

---

## 7. Deploying to Vercel

### Automatic deployment

Vercel auto-deploys from the **main** branch. Changes to main trigger a production build automatically.

### Manual force-redeploy (clears edge cache)

```bash
curl -s -X POST "https://api.vercel.com/v13/deployments?teamId=team_7PGzhT9wecJMLFkOWDGGxptE&forceNew=1" \
  -H "Authorization: Bearer $VERCEL_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"ir","project":"prj_htbgIKYkHEwa0WNdPKVQ4sVy3iiE","target":"production","gitSource":{"type":"github","org":"rjmjr1962831","repo":"ir","ref":"staging"}}'
```

### When vercel.json changes take effect

Changes to `vercel.json` (rewrites, headers, crons) only take effect after a push to the deployed branch and a Vercel redeploy. If you add a new rewrite, it will not work until that deploy completes.

---

## 8. Adding AI Discovery Content

### llms.txt and llms-full.txt

These files are generated by `api/llms.js` and `api/llms-full.js` (Vercel Edge Functions). They are NOT static files.

- **`api/llms.js`**: Concise summary of the site for LLMs. Add new page entries as markdown list items.
- **`api/llms-full.js`**: Extended version with deeper content. Add full sections for substantive pages.

Both pull freshness data from the `site_freshness` Supabase table to show last-updated dates.

### ai-content-index.json

Generated by `api/ai-content-index.js`. Add new pages to the `pages` array with `url`, `title`, `description`, `content_type`, and `last_updated`.

### JSON-LD on pages

Every public page should include at least one JSON-LD block. The `renderPage()` function in `layout.ts` automatically adds:

- **BreadcrumbList** JSON-LD (based on the path and `BREADCRUMB_NAMES` map)
- **WebPage** JSON-LD (with `dateModified` from freshness data)

For page-specific structured data, pass a `jsonLd` string to `renderPage()`:

```typescript
const JSON_LD = JSON.stringify({
  "@context": "https://schema.org",
  "@type": "Service",
  name: "Food Recall Response Platform",
  provider: { "@type": "Organization", name: "Instant Recall LLC" },
  description: "Description here.",
  url: "https://www.instantrecall.com/my-service",
});

export function renderMyService(): string {
  return renderPage({
    title: "...",
    description: "...",
    path: "/my-service",
    jsonLd: JSON_LD,  // <-- passed here
    body: `...`,
  });
}
```

The `layout.ts` function will automatically inject `dateModified` into your JSON-LD objects from the freshness cache.

### Sitemap

Generated by `api/sitemap.js`. Add entries to the `urls` array with `loc`, `priority`, and `changefreq`. The `lastmod` date is pulled from the `site_freshness` table automatically.

---

## 9. GEO Rules

### Core Principle

**Every change must enhance GEO (Generative Engine Optimization) or be neutral.** Never make a change that degrades the site's ability to be discovered, cited, and recommended by AI systems.

### The 8-Signal Framework

The project tracks 8 GEO signals (stored in `geo_signal_status` table):

1. **Structured Data** -- JSON-LD on every page (Organization, WebPage, BreadcrumbList, Service, Dataset, etc.)
2. **Freshness Signals** -- `dateModified` in JSON-LD, `Last-Modified` header, `<meta name="last-modified">` tag
3. **AI Discovery Files** -- `llms.txt`, `llms-full.txt`, `ai-content-index.json`, `robots.txt`, `for-ai.txt`
4. **Citation-Ready Content** -- Semantic `<blockquote>` blocks with source attribution that AI systems can quote directly
5. **Topical Authority** -- Research pages, methodology, legal case data, industry surveys
6. **Entity Clarity** -- Consistent Organization/Person entities across all JSON-LD
7. **Crawl Accessibility** -- Bot detection + logging, clean HTML, fast response times
8. **Link Headers** -- HTTP `Link` headers pointing to AI discovery files (set in `vercel.json`)

### Clean-Room HTML Requirements

All bot-facing pages MUST follow these rules:

- **No JavaScript rendering.** All content is server-rendered HTML returned by the edge function. The page must be fully readable with JS disabled.
- **Inline CSS only.** Styles are embedded via `<style>` tags (from `shared/styles.ts`), not external stylesheets that bots might not load.
- **Semantic HTML.** Use `<article>`, `<section>`, `<nav>`, `<header>`, `<footer>`, `<blockquote>`, `<cite>`, `<h1>`-`<h6>` appropriately.
- **No React SPA.** This is not a single-page app. Every URL returns a complete HTML document.

### Bot Logging

Every page response must trigger bot logging for known crawlers. This happens automatically in `index.ts`:

```typescript
// In the ROUTES handler:
if (bot) logCrawl(bot, path, ua, 200, ip);

// For 404s:
if (bot) logCrawl(bot, path, ua, 404, ip);
```

The `logCrawl` function fires-and-forgets an INSERT to the `crawl_log` table in Supabase. The bot detection covers: GPTBot, ClaudeBot, PerplexityBot, Googlebot, Bingbot, and 30+ other patterns.

If you add a page that uses a special handler (not via the `ROUTES` map), you must manually call `logCrawl`:

```typescript
if (path === "/my-special-page") {
  const resp = await handleMySpecialPage(req);
  if (bot) logCrawl(bot, path, ua, resp.status, ip);
  return resp;
}
```

---

## 10. Testing and Verification

### After deploying the edge function

```bash
# 1. Verify the page renders through the full chain
curl -s -H "Cache-Control: no-cache" "https://ir-zeta.vercel.app/my-page"

# 2. Verify Content-Type is text/html
curl -sI -H "Cache-Control: no-cache" "https://ir-zeta.vercel.app/my-page" | grep -i content-type
# Expected: content-type: text/html; charset=utf-8

# 3. Verify JSON-LD is present
curl -s "https://ir-zeta.vercel.app/my-page" | grep -o '<script type="application/ld+json">[^<]*</script>' | head -5

# 4. Verify Cache-Control headers
curl -sI "https://ir-zeta.vercel.app/my-page" | grep -i cache-control
# Public pages: public, max-age=60, s-maxage=86400, stale-while-revalidate=3600
# Internal pages: private, no-store

# 5. Test bot crawl logging (simulate a bot)
curl -s -H "User-Agent: GPTBot/1.0" "https://ir-zeta.vercel.app/my-page" > /dev/null
# Then check crawl_log table in Supabase for the entry

# 6. Verify AI discovery files include the new page
curl -s "https://ir-zeta.vercel.app/sitemap.xml" | grep "my-page"
curl -s "https://ir-zeta.vercel.app/llms.txt" | grep "my-page"
curl -s "https://ir-zeta.vercel.app/ai-content-index.json" | grep "my-page"
```

### Structured data validation

Paste the page URL into Google's Rich Results Test (https://search.google.com/test/rich-results) or Schema.org's validator (https://validator.schema.org/) to verify JSON-LD.

### Verification protocol

From CLAUDE.md: **"You are not done until you confirm the change actually worked. Deploy, load the live page, verify. 'Code updated' is not completion. 'Deployed. Verified at [URL].' is."**

---

## 11. Common Gotchas

### Supabase gateway forces text/plain

Supabase Edge Functions always return `Content-Type: text/plain` regardless of what you set. That is why `api/html.js` exists as a proxy -- it re-wraps the response with `text/html`. If you call the Supabase function directly, you will get plain text.

### vercel.json rewrites must encode the path

The `destination` in vercel.json rewrites passes the path as a query parameter. Forward slashes must be encoded as `%2F`:

```json
// CORRECT:
{ "source": "/research/industry-survey", "destination": "/api/html?path=%2Fresearch%2Findustry-survey" }

// WRONG (will break):
{ "source": "/research/industry-survey", "destination": "/api/html?path=/research/industry-survey" }
```

### Routes must be added in TWO places

A new page will not work unless you add:
1. The route in `index.ts` (the `ROUTES` map)
2. The rewrite in `vercel.json`

Missing either one means a 404. The `index.ts` route handles the Supabase side; the `vercel.json` rewrite handles the Vercel side.

### Cache-Control: private vs public

- **Public pages** (home, solution, research, etc.): `public, max-age=300, s-maxage=3600` from the edge function; Vercel proxy upgrades to `public, max-age=60, s-maxage=86400, stale-while-revalidate=3600`
- **Internal/protected pages** (dashboard, crawl-stats, work-log, ai-best-practices, project-knowledge): `private, max-age=30` from the edge function; Vercel proxy converts to `private, no-store`

If your page requires authentication (token check), use `private` caching. If it is public content, use `public`.

### Pages with request access (special handlers)

Most pages use the simple `ROUTES` map pattern (function takes no arguments, returns HTML string). Some pages need access to the request object (for query params, auth tokens, etc.) and are handled separately before the `ROUTES` lookup:

```typescript
// These are handled as special cases in index.ts:
if (path === "/geo-ledger") return await handleGeoLedger(req);
if (path === "/dashboard") return await handleDashboard(req);
if (path === "/crawl-stats") return await handleCrawlStats(req);
```

If your new page needs request access, add it as a special-case `if` block before the `ROUTES` lookup, and make sure to call `logCrawl` manually.

### Internal docs stay on staging

Files like `CLAUDE.md`, `COMPREHENSIVE_KNOWLEDGE_DOCUMENT.md`, takeaways, and prompts must never be pushed to main. They live on the staging branch only.

### "ALL" means every single instance

From CLAUDE.md: when fixing something across the codebase, you must grep exhaustively and fix every occurrence. Do not fix one file and assume you are done.

### Never use web_fetch for live site checks

Always use `curl -s -H "Cache-Control: no-cache"` to check live pages. `web_fetch` returns stale/cached content.

### No em dashes

Project style rule: never use em dashes (--) in generated content, file edits, or responses. Use commas, semicolons, periods, or restructure the sentence.

---

## Quick Reference Card

| Task | Command/Action |
|---|---|
| Deploy edge function | `SUPABASE_ACCESS_TOKEN=$SUPABASE_ACCESS_TOKEN npx supabase functions deploy serve-html --no-verify-jwt --project-ref dewbyvlbmkersxjrcknm` |
| Force Vercel redeploy | See Section 7 curl command |
| Verify page renders | `curl -s -H "Cache-Control: no-cache" "https://ir-zeta.vercel.app/PAGE"` |
| Check Content-Type | `curl -sI "https://ir-zeta.vercel.app/PAGE" \| grep content-type` |
| Check crawl logs | Query `crawl_log` table in Supabase |
| Dev dashboard | `https://ir-zeta.vercel.app/dashboard?token=<TOKEN>` |
| Crawl stats | `https://ir-zeta.vercel.app/crawl-stats?token=<TOKEN>` |
| GEO ledger | `https://ir-zeta.vercel.app/geo-ledger` |

---

*Generated from the actual IR codebase on 2026-03-31.*
