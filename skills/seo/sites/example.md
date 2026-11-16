---
site: example
status: not-ready
---
> Copy this file to `sites/<your-site>.md`. The shared skill reads it first and it wins over any conflicting rule.

# Site: example.com

- REPO = `~/Projects/example-site` (Astro, Next.js, Shopify theme: anything with a build step)
- DIR = `~/seo/example` (holds `seo-state.json` and `SEO - Log.md`)
- Schedule: Mon, Wed, Fri 07:30, three runs a week (A, B, C per the shared skill; the monthly pass rides the first run A of a month)
- Search Console property: `sc-domain:example.com` (open it at `https://search.google.com/u/<authuser>/search-console?resource_id=sc-domain%3Aexample.com`)
- GA4 property: `<id>`; conversion event: `<name>` (if none is tracked, say so on the warning line every run)

## Bootstrap (do on the first run, then set `status: ready`)
1. Add `scripts/seo/{crawl,lighthouse,lint-dist}` to the repo and npm scripts `seo:crawl` and `seo:lighthouse`; chain the lint after `build` so hard errors fail the build.
2. Create DIR with an empty `seo-state.json` (schema in `SKILL.md`) and `SEO - Log.md`.
3. Commit the scripts as their own `seo(bootstrap)` commit, naming files explicitly.

## Markets and language
English only. Trends geo: `GB`. Autocomplete `hl=en&gl=gb`.

## Content rules
- Article length 900-1300 words. Voice: first person, plain, no hype.
- No private information in published text.
- No invented experiences, clients, numbers or credentials.
- The author is a `<role>`: never position them as anything else.

## NEVER
Change preserved URLs or their redirects (except adding one), `git add -A`, edit CI workflows, weaken the lint gate, force-push, touch `<!-- TODO -->` comments.

## Deploy rule
Push to `main` after the build passes. (Or: open a pull request per run if you want a human review.)

## Deep rewrite autonomy
Optional. Add this section only if refreshes may be full body rewrites (up to 5 pages a week). Without it the site stays light-touch.

## Notify
`<channel or none>`. With none, write the summary in the log and report it in the session output.

## AI search (this site)
`robots.txt` must allow the AI bots listed in the shared skill. Probe questions: `<5-8 questions a customer would ask an assistant>`.
