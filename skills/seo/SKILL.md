---
name: seo
description: SEO and AI-search autopilot for any site. Argument is "<site> <weekly|monthly>" (sites live in sites/<site>.md). Three runs a week, one focus each: measure and fix, content, CTR and AI visibility. Crawls, runs Lighthouse, reads Search Console, fixes within guardrails, pushes, logs and notifies. Covers classic search and AI search (ChatGPT, Claude, Perplexity, Gemini, Google AI Overviews).
---

# SEO autopilot (shared method)

One method for every site. Everything that differs (repo, stack, markets, guardrails, notify target) lives in `sites/<site>.md`. If a rule in the site file conflicts with this file, the site file wins.

**Arguments:** `<site> <mode>`. `site` is a file name in `sites/`. `mode` is `weekly` or `monthly`. With no mode (how the per-site scheduled tasks call it, three times a week), the run's focus (A, B or C) comes from the Cadence section, and the monthly pass rides on the first run A of a calendar month.

**Step 0, always:** read `sites/<site>.md` in full. It defines REPO, DIR (state and log), the Search Console property, markets, content rules, the NEVER list, the notify target and which tools exist. If it says `status: not-ready`, do its Bootstrap section first, mark it `status: ready`, create DIR with an empty `seo-state.json` and `SEO - Log.md`, then continue.

State file: `DIR/seo-state.json`
```json
{"last_weekly": null, "last_monthly": null, "runs": [], "changes": [], "gsc_snapshots": [], "ai_visibility": [], "long_queries": [], "killer_page": null}
```
`changes` entries: `{date, url, field, before, after, reason, query, baseline: {impr, clicks, ctr, pos}}`. Log: `DIR/SEO - Log.md`, newest run on top.

## Tool map

| Job | How it runs |
| --- | --- |
| Crawl | `npm run seo:crawl` in REPO, writes `reports/seo/crawl-<date>.json` (any crawler that emits JSON works) |
| On-page lint | a script chained after `build`; it **fails the build on hard errors**. Never weaken it to make a build pass. |
| PageSpeed | local Lighthouse (mobile) plus Search Console Core Web Vitals |
| Search Console | a real signed-in browser; property URL from the site file |
| Rich Results Test | `https://search.google.com/test/rich-results?url=<url>` (monthly) |
| Google Trends | `https://trends.google.com/trends/explore?geo=<geo>&q=<a>,<b>` |
| Question mining | Google autocomplete `https://suggestqueries.google.com/complete/search?client=firefox&hl=<lang>&gl=<country>&q=<seed>` (plus a-z and question words, 1 request a second) and People Also Ask |
| AI crawler access | fetch `<site>/robots.txt`; `curl -A "<bot UA>" -sI <url>` must return 200 (a WAF or CDN bot rule can block them silently) |
| AI referral traffic | GA4 sessions whose source matches `chatgpt.com`, `chat.openai.com`, `perplexity`, `claude.ai`, `gemini.google.com`, `copilot.microsoft.com`, `you.com` |
| AI citation probe | monthly: ask each assistant the site's target questions and record whether the domain is cited |
| Bing | Bing Webmaster Tools and IndexNow, only if the property already exists. Never create or verify it. |

## AI search (GEO and AEO): appear in ChatGPT, Claude, Perplexity, Gemini and AI Overviews

Same pages, written so a model can lift a clean, attributable answer. No tricks, no hidden text, no fake statistics.

1. **Access (uncapped, check every weekly run).** `robots.txt` must not disallow the retrieval and search bots: `OAI-SearchBot`, `ChatGPT-User`, `GPTBot`, `Claude-SearchBot`, `Claude-User`, `ClaudeBot`, `PerplexityBot`, `Perplexity-User`, `Google-Extended`, `Applebot-Extended`, `CCBot`, `Bingbot`, `Googlebot`. If the site file marks one as a deliberate block, leave it and report it. Also confirm: full content in the raw HTML (no JavaScript-only rendering), no `noindex` / `nosnippet` / `max-snippet:0` on content pages, a sitemap listing every page, and no bot blocking in headers or the CDN firewall (probe with the bot user agents).
2. **Answer-first pages.** The question a page answers is an H2 or H3 phrased the way people ask it, followed by a direct 40-60 word answer in the first sentences, then the detail. Definitions, short comparison tables, numbered steps and a real FAQ are the passages models quote. One topic per page, one consistent name for the primary entity.
3. **Entity and trust signals.** Keep business details identical everywhere they appear online and keep the About page honest. Schema (`Organization`, `Person`, `Article`, `FAQPage`, `BreadcrumbList`) only where the page qualifies for a rich result or the markup is already correct. It is not an AI-citation shortcut: a large matched test saw no meaningful jump in citations from adding schema. AI answers lean on third-party sites (forums, video, industry publications), so list gaps and wrong details found there rather than faking mentions.
4. **Citable substance.** Primary-source citations with links, original facts only the owner has, concrete numbers with dates, a visible "Last updated" date. Refresh stale facts rather than just the date.
5. **No `llms.txt`.** Search engines say special AI files are not needed and models mostly ignore them. There is no special trick for AI Overviews: crawlable, indexed, snippet-eligible pages with the normal SEO basics.
6. **Measure.** Weekly: AI referral sessions from GA4 and the bot-access check. Monthly: the citation probe. Record `{date, bots_allowed:[...], referrals:{source:sessions}, probes:[{q, assistant, cited, competitors}]}` under `ai_visibility`.

## Long-question filter (Search Console regex)

Long, specific queries are the questions people type into Google before they find a site, and the same kind they ask an assistant. Pages that already rank for them but earn few clicks are the cheapest wins: answer the question at the top of the page that already ranks. (Technique popularised by AskOkara.)

**The filter:** Performance, Search results, Add filter, Query, **Custom (regex)**: `^(?:\S+\s+){7,}\S+$` (queries of 8 or more words). Add Clicks, Impressions, CTR and Position, sort by impressions.

**Unattended method:** open `https://search.google.com/u/<authuser>/search-console/performance/search-analytics?resource_id=<encoded property>&num_of_months=3&query_operator=REGEX&query=~<url-encoded regex>`. The `~` before the regex is essential: without it the filter is silently ignored. Add `&breakdown=page` for the ranking pages. Wait about 9 seconds, then read the table with a script (`document.querySelectorAll('table tbody tr')`, cells are query, clicks, impressions, CTR, position) and sort by impressions yourself, because plain text extraction can return a stale unfiltered table. The regex encodes as `%5E(%3F%3A%5CS%2B%5Cs%2B)%7B7%2C%7D%5CS%2B%24` for 8+ words (`%7B4%2C%7D` is 5+, `%7B3%2C%7D` is 4+).

**Thresholds:** use the strictest threshold that still returns about 20 or more queries. Big sites: 8+ words. Mid sites: 5+ (`{4,}`). Small sites: 4+ (`{3,}`). Under about 1,000 impressions in the window, say so and use `{3,}`.

**What to do with the list (max 5 pages a week, inside the existing caps):**
1. Keep rows with high impressions and low clicks. Find which page ranks for each query.
2. Group queries by ranking page. Per page add the question as an H2 or H3 with a direct 40-60 word answer high on the page, as part of that page's single pass.
3. At most 3 question blocks per page, only questions the page can answer from verified facts. Queries that need a different page type go to the log's Queued list as new-article candidates.
4. Feed the questions into the keyword map with source tag `gsc-long`, use them as FAQ entries and as citation-probe questions.
5. Record date, query, page and baseline in `changes`, plus `long_filter_runs: [{date, threshold, queries, pages_touched}]`. Measure after 28 days.

Run A pulls the list and stores the top candidates in `long_queries`; run B applies them first; run C counts them in "Measured".

## Content: refresh and new articles

- **Refresh first, up to 5 pages a week, shared across the three runs:** pages with impressions but low CTR, or position 8-20. Update stale facts, add the answer-first block and FAQ, add internal links, improve title and meta, fix thin sections, add primary-source links. Keep the author's voice and opinions.
- **Keyword coverage pass before editing a page:** take the cluster assigned to the page in the keyword map (primary query, sub-terms, questions). If there is none, build one from Search Console queries for that URL plus autocomplete. Compare against the title, headings and body, list the covered and skipped terms, and rewrite to cover what genuinely fits. If two pages claim the same cluster, log a cannibalization decision instead of rewriting. Record per page: cluster, terms added, terms deliberately left out.
- **Free keyword sources, per refreshed page:** competitor headings (terms recurring in 2 or more of the top 5 results are "expected coverage"), People Also Ask and related searches (stop on any CAPTCHA, never solve it), autocomplete order (earlier suggestions are usually more searched), Google Trends as a tiebreaker only, and Keyword Planner or Bing data only if the site's state file says the account exists. Write the merged cluster back to the keyword map with a source tag per term.
- **Deep rewrite mode (only if the site file allows it):** restructure sections, rewrite weak passages, add sections competitors cover and the page skips, merge duplicates, better headings. Run B must ship at least one body rewrite per site per week, or the log says exactly why not.
- **New article, at most 1 a week per site:** from a real gap (queries with impressions and no dedicated page). One topic, one search intent, 900-1300 words, answer-first, sourced, linked from 2 related pages and linking to the money page.
- **Quality gate before publishing:** every factual claim checked against a primary source; anything unverifiable is cut. No invented experiences, clients, numbers, quotes, credentials or reviews. No duplicate of an existing page. If no gap clears the bar, publish nothing and say so. Quality over cadence.

## Conversion-first and attribution rules

Traffic that does not convert does not count.

1. **Each site has a conversion** (site file): purchase, contact form, call click, affiliate click, newsletter. If it is not tracked as a named GA4 event, say so on the warning line every run. Do not change GA settings yourself.
2. **Killer page (run A):** join Search Console pages with GA4 landing-page conversions and pick ONE page that already converts or sits close to the money and ranks low enough that most searchers never see it (about position 8-25 with real impressions). Store it as `killer_page` with the main query and a one-line reason. Pages with big impressions and no conversion potential are traps: flag them, do not prioritise them.
3. **SERP intent check before rewriting or writing:** open the live results for the page's main query. If the top results are a different page type than yours, the fix is the right page type, not tuning. Verdict: keep, keep after fixing one named condition, or drop.
4. **Read the competitors in full (run B):** the top 5 organic results, compared with your page. Every claim in the log names its source URL, and missing data is stated as missing.
5. **One pass per page at a time.** A pass bundles title, meta, body, FAQ and links on one page in one commit and is measured as a whole. A page with a full pass in the last 28 days is measured, not changed again. A meta-only edit does not start that clock for body work.
6. **Judge honestly ("Measured"):** use search and business together. Climbed in search with no extra conversions is a miss. Flat rankings with better conversion is a win. Ignore ranking wiggles under two weeks. AI-citation changes cannot be pinned on one edit, so record them as observations, not wins.
7. **Data cautions:** Search Console lags 2-3 days, so exclude the last 3 days; page-and-query breakdowns drop rows; AI Overview appearances can flatter average position; pull by day and keep the history.
8. **Check access and speed before content:** URL Inspection for the killer page (crawlable, indexed, snippet-eligible, real text in raw HTML, status 200) and mobile PageSpeed failures that hurt a real visitor.
9. **Version the method:** stamp `skill_version` on every `runs` entry so a later reader knows which instructions produced a result. Do not edit the shared skill mid-test unless something is broken.

## Full autonomy

These runs never stop to ask. They decide, act, log and report afterwards.

1. **Do the whole job on partial access.** Missing Search Console data does not cancel the run: still do repo work, technical fixes and content refreshes from the last snapshot. Retry a failed browser step once.
2. **Never hand a decision back.** Apply the conservative default and log the reason. Factual conflicts about the business are left unchanged and listed under "needs the owner" without blocking. Truly impossible things (a login, a password, a physical action) go on the warning line as one concrete request each.
3. **Backlog over cap:** write remaining ready-to-apply items into the log's Queued list (page, new title and description, reason) so the next run applies them first.
4. **A run is finished only when:** the run stamp is written, the log entry is on top, the run record is appended, tabs are closed, and the final report lists changes, shipped URLs and warnings.

## Cadence: three runs a week, one focus each

Each site runs on three weekdays (for example Monday, Wednesday, Friday). Work is split so no run does everything. **Weekly caps are per ISO week and shared by the three runs.**

- At the start of a run, count the `runs` entries in the current ISO week: 0 is **run A**, 1 is **run B**, 2 is **run C**, 3 or more means stop (an extra manual run). If the last run started under 45 minutes ago, stop (double fire). **Stamp `{date, focus}` at the START.**
- **Run A: measure and fix.** Pick or refresh the `killer_page`, run the access and speed check on it, then weekly steps 1, 2, 3.A (technical, uncapped), 3.D (performance), the AI-bot access check, ship, log. On the first A of a month, also do the monthly research.
- **Run B: content.** Apply stored `long_queries` first, then the SERP intent check and competitor reading for the killer page and any new-article topic, then 3.F (refreshes with the keyword coverage pass, plus a new article) and 3.E answer-first rewrites on the pages you touch, ship, log.
- **Run C: CTR, links and AI.** Steps 1 and 2, 3.B (CTR rewrites), 3.C (striking-distance links), 3.E for the top-impression pages, the "Measured" pass on changes now 28 days old or more, ship, log, and the weekly notification.
- Skip any part whose cap is used up. With nothing to do, log "nothing to do".

## Weekly mode

### 1. Sync and measure
```bash
cd REPO && git pull --ff-only && npm ci --silent
npm run build && npm run seo:crawl && npm run seo:lighthouse
```
The build gate must pass on a clean main; if not, fixing it is task one. If `git pull` fails or the tree is dirty with changes that are not yours, **stop and report**. Never `git add -A`; add named files only.

### 2. Search Console (your own browser tab; close it at the end)
Read with page text extraction:
1. **Performance, Search results**, 28 days against the previous 28: totals, then Queries and Pages (top 50 by impressions). Append a snapshot to `gsc_snapshots`.
2. **Indexing, Pages:** indexed and not indexed, each reason with examples.
3. **Sitemaps:** status Success, discovered count close to the sitemap's URL count; submit if never submitted.
4. **Core Web Vitals** (mobile and desktop): poor and needs-improvement groups.
5. **Long-question filter** (run A): save the top candidates to `long_queries`.

### 3. Decide fixes (in order; stop adding when caps are hit)
- **A. Technical, uncapped:** every crawl or lint ERROR; broken links; links to redirects (point at the final URL); missing alt text (an honest description); hreflang and canonical problems; descriptions outside 110-160 characters; titles over 65 characters (shorten the tail, keyword first); Lighthouse SEO and accessibility failures; "not indexed" reasons you cause.
- **B. CTR rewrites, max 3 pages a week:** pages with at least 100 impressions in 28 days, average position 15 or better, and CTR clearly below typical (positions 1-3 under 10%, 4-7 under 4%, 8-15 under 1.5%). Rewrite title and meta around the query bringing the impressions. Skip any page changed in the last 28 days.
- **C. Striking distance, max 5 new links a week:** queries at position 5-20 with at least 50 impressions get a natural in-text link (descriptive anchor) from 1-2 related posts to the ranking page. At most one new link per post per week.
- **D. Performance:** if mobile Lighthouse performance is under 80 or LCP over 2.5 s on a money page, fix the cause in your own code. No new dependencies.
- **E. AI-search readiness, max 3 pages a week:** fix any AI-crawler blocking first (uncapped), then apply answer-first and entity fixes to the pages with the most impressions.
- **F. Content:** refreshes first, then at most one new article if a real gap exists.

Universal rules: no invented clients, numbers, credentials or experiences; mirror every change across language versions; never change URLs or slugs (unless adding a 301 for a broken one), delete pages, weaken the lint gate, edit CI workflows or force-push.

### 4. Apply, verify, ship
1. Edit, then build (the gate must pass) and run tests if the repo has them.
2. One commit: `seo(weekly <date>): <n> fixes - <short list>`; push per the site file's deploy rule.
3. Wait for the deploy (poll a changed page until the new text shows, max 5 minutes), re-crawl: ERROR count 0 (or the same known ones), changed URLs return 200.
4. If the deploy breaks anything, `git revert`, push, report.
5. In Search Console, URL Inspection then **Request indexing** for each changed page (max 5 a run). Open the property, find the "Inspect any URL" box, type the full URL, press Return, wait about 10 seconds, click "Request indexing", then wait up to about 30 seconds for the "Indexing requested" dialog. Deep links into the inspect view tend to 404; use the search box.

### 5. Log
`seo-state.json`: append each title, meta or link change to `changes` with its Search Console baseline. `SEO - Log.md`: a new section on top with the date, totals and delta, the top 5 rising and falling queries, the Lighthouse table, crawl error and warning counts, every change (URL, before, after, why), the commit hash, an **AI visibility** line (bots allowed, AI referral sessions, cited over asked from the monthly probe) and **Measured**: for every change now 28 days old or more, CTR and position before and after.

### 6. Notify
Only run C sends the weekly message (it summarises all three runs), plus an immediate message from any run that breaks something. Per the site file. Emoji-led, one fact per line:
```
📝 Weekly SEO · <domain>
🔎 GSC 28d: <clicks> clicks (<±%>) · <impr> impressions (<±%>) · avg position <x>
⚙️ <n> technical fixes · <n> titles rewritten · <n> internal links
🚀 Lighthouse mobile: <min-max> performance · SEO <x>
🤖 AI: bots allowed <ok/total> · assistant referrals <n> · <n> pages adjusted
📈 Best query: "<q>" position <a> to <b>
⚠️ <only if something needs the owner, else omit>
```

### 7. Run record (last step of every run, including no-op, stopped and failed runs)
Append exactly ONE line of compact JSON to a shared `runs.jsonl` with `>>` (never rewrite the file; other sites append to it too):
```json
{"ts":"2026-11-02T07:40","site":"example","run":"A","status":"ok","commits":["e3670b8"],"shipped":["homepage title shortened"],"gsc":{"clicks":22,"impressions":5860,"position":18.5,"window_days":28},"indexed":240,"not_indexed":279,"lighthouse_home":92,"needs_owner":["one short item per open request"]}
```
`status` is `ok` (something shipped), `no-op` (finished, nothing shipped) or `fail`. Use `null` for any field not measured this run, never a guess. Keep `needs_owner` wording identical to the previous record for the same issue so you can see how long it has waited. If `indexed` dropped more than 20% against the previous record, or the status is `fail`, also raise an alert for your monitoring (see [claude-agent-fleet-monitor](https://github.com/waltermitas/claude-agent-fleet-monitor)).

## Monthly mode

1. **Gate:** if `last_monthly` is in the current month, stop. Otherwise stamp it first.
2. **Research:** Search Console top 200 queries over 3 months (also run the long-question regex once and tag hits `gsc-long`), clustered by intent, with content gaps and cannibalization. Google Trends for the topic trio per market. Autocomplete expansion. People Also Ask for the top 3 gaps. Rich Results Test on 3 URLs, fixing schema errors. **AI citation probe:** 8-10 target questions phrased as a person would ask an assistant, asked in each assistant you can reach without logging in or solving CAPTCHAs, read-only, in new chats. Record whether the domain is cited and who is cited instead; turn each miss into a candidate fix.
3. **Update the keyword map** and calendar for the next 3 months.
4. **Publish one new post** if a real gap and a verified source exist, following the site file's post conventions.
5. **Ship, log, notify** with commit message `seo(monthly <month>): keyword map + post "<title>"`.

## Hard rules (all sites)

- One run is at most one push per mode (plus a revert if needed).
- Write only: the site's repo, its `seo-state.json` and log, one appended line in `runs.jsonl`, and one message to the target in the site file.
- In Search Console only read, submit the sitemap and request indexing. Never remove URLs, change settings, add users or verify properties.
- Never solve CAPTCHAs, create accounts or sign in with a password. Logged out means report it and do the rest.
- Close every browser tab you opened.
- One site per run. Never touch another site's repo, state or log.
