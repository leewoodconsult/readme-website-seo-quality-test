# SEO & AEO QA Scanner

Enter a live URL and get a plain-English report on four things:

1. **Structured data**: is the JSON-LD valid, typed, linked together and consistent with the sitemap?
2. **Crawl and indexing basics**: robots.txt, sitemap, status codes, titles, descriptions, canonicals, redirects, broken links, soft 404s. Also checks drawn from Google's spam policies: hidden text and links (white-on-white, off-screen, zero-size, a link hidden on one punctuation mark, found from the page's own markup), a cloaking comparison (is the home page the same for a browser, Googlebot, a phone and a visitor arriving from Google?) and informational repetition signals (heavily repeated phrases, long lists of place phrases, phone-number lists, near-copy pages, keyword-string titles).
3. **Speed**: server response time, compression, page and image weight, render-blocking scripts, image dimensions and caching. These are measured from the scanner's server in one location, without a browser, so they point to likely causes and don't replace real-visitor data (Search Console's Core Web Vitals, PageSpeed Insights). The caching check scores only files served from the site's own domain; when everything comes from a CDN it reports "Not measured" instead of a pass.
4. **AI readiness (AEO/GEO)**: can AI tools reach and read the site? It checks that Google, Bing and AI search crawlers are allowed in robots.txt, that a firewall isn't refusing AI crawlers, that snippets aren't blocked, and (unscored) that the text is in the raw HTML and the business is clearly described. It does not predict whether AI tools will cite you.

Every finding says what was found, why it matters, how to fix it, and, where the evidence is thin, an **evidence note**. For example, no AI provider has confirmed that it reads `llms.txt`, so that check is informational and never scored.

The rules come from a personal website-build playbook covering structured data, crawlability and AI-search access, turned into automated checks.

## Run it

```bash
pip install -r requirements.txt
python app.py            # http://localhost:5000
python -m unittest discover -s tests -v
```

## Deploy

`render.yaml` and the `Dockerfile` deploy to Render (free plan) or any container host. Run **one** worker: scan jobs are held in memory.

| Variable | Default | Meaning |
|---|---|---|
| `QA_RATE_LIMIT` | 5 | scans per visitor per hour |
| `QA_DAILY_LIMIT` | 20 | scans per visitor per day |
| `QA_DOMAIN_LIMIT` | 3 | scans of one site per hour, from any visitor |
| `QA_GLOBAL_DAILY` | 300 | scans per day for the whole server |
| `QA_MAX_RUNNING` | 4 | simultaneous scans |
| `QA_CACHE_SECONDS` | 600 | a finished report for the same page is reused, not rescanned |
| `QA_PROXY_HOPS` | 1 | trusted proxies in front of the app (0 when none); controls which `X-Forwarded-For` entry is believed |
| `QA_ALLOW_PRIVATE` | unset | **tests only**: allows scanning localhost |

Limits are held in memory and reset when the server restarts.

### How AI readiness is scored

Each AI readiness check carries an evidence label. **Documented** (a vendor says so in its own guidance) and **Tested** (the scanner observed it) count toward the score. **Best practice** and **Info** items are shown but never scored, because nobody publishes how answer engines choose what to cite.

The firewall test requests the home page once per AI crawler, sending that crawler's published User-Agent string, and compares the result with a normal visit. It cannot use the crawlers' verified IP addresses, so a refusal means "may be blocked".

The cloaking comparison requests the home page as a desktop browser (twice, to measure how much the page changes by itself), Googlebot, Googlebot smartphone, a phone, and a browser arriving from google.com, then compares the final address, title, headings, noindex, canonical and text similarity. A page that changes on its own is downgraded to information, and a refused request is reported as refused, never as cloaking. A clean result only means no difference was found for those requests. The hidden-text check reads inline styles and `<style>` rules only (no browser), and skips accordions, tabs, screen-reader text and animated reveals, which Google says are fine. The repetition signals are plain statistics. Google publishes no thresholds, so they are information only and never scored.

## How it works

`fetch.py` is the security boundary: http/https only, standard ports, public IPs only (re-checked on every redirect), 2 MB body cap, timeouts. `crawl.py` collects a small snapshot (home page, robots.txt, up to 4 sitemaps (the first 2,000 URLs of each are read, and the report says "2,000+" when that limit is reached), up to 12 pages, 20 link checks, 40 s for the page sample and 70 s overall). `checks_*.py` are deterministic rules, with no AI model in the loop, and `report.py` scores them (error = 3, warning = 2, info = 0) per category.

## Honest limits

- Reads at most about 12 pages, so a score describes a sample.
- Schema **types** are checked against the full schema.org type list (bundled in `qa/data/schema_types.json`, currently v30.1, refreshed with `scripts/build_schema_data.py`). Schema **properties** are not checked, so confirm with the Schema Markup Validator and Google's Rich Results Test.
- It cannot see rendered (JavaScript) output or real rankings, and it is not a substitute for Search Console.
- Answer engines don't publish how they cite pages. Items marked as best practice are exactly that.

## Roadmap

Share-preview checks (og:image size and weight), full schema.org vocabulary, optional Claude-written explanations, saved report history.

## Credits

The schema.org type list in `qa/data/schema_types.json` is derived from the [schema.org](https://schema.org) vocabulary (version recorded in the file), which is available under the [Creative Commons Attribution-ShareAlike 3.0 license](https://creativecommons.org/licenses/by-sa/3.0/). The crawler User-Agent strings used by the firewall test imitate the formats published by OpenAI, Anthropic and Perplexity, for testing only.
