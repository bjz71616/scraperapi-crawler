# ScraperAPI Crawler: What It Does, How the URL Regex, Depth, and Crawl Budget Work, and What a Crawl Really Costs (Full Plan and Credit Table)

ScraperAPI Crawler starts at one seed URL, follows the links it finds, and scrapes every page that matches a regex you supply. It exists for a single situation: you need many pages from a site and you don't know the URLs in advance.

Four cases where it is the wrong tool:

- **You already have a URL list.** The Async API is cheaper and more predictable for a known set of URLs.
- **You need one page.** Send it to the standard endpoint at `api.scraperapi.com`.
- **The content is behind a login.** Crawler only reaches public pages.
- **You're on the free plan and need to go deeper than one hop.** Free accounts are capped at `max_depth: 1`.

The rest of this article covers what you need to run a job: the three endpoints, the named capture group your include regex must contain, how depth and budget differ, where credits go, and where the plan limits actually bite.

> **Quick pick:** start on the free plan (1,000 credits a month, 5 concurrent threads, depth capped at 1) to test your regex on a small section. Move to Hobby at $49/month for 100,000 credits once the match pattern is right. Those 100,000 credits buy 100,000 plain pages, 10,000 pages with JavaScript rendering, or 4,000 with premium proxies plus rendering.
>
> 👉 [Compare ScraperAPI plans and credit allowances](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## ScraperAPI Crawler pricing: plans, credits, and what a crawl costs

Crawler has no separate price list. A crawl costs the sum of the pages it scrapes, and each page is billed at the same credit rate as a direct Standard API call made with the same parameters. So the plan you pick sets the ceiling, and your `api_params` set the burn rate.

| Plan | Monthly | Annual (per month, 10% off) | Credits/month | Concurrent threads | Purchase |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 | [Check price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [Check price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [Check price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [Check price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [Check price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [Check price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [Check price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | [Check price](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

A few things the table doesn't show. New accounts get a 7-day trial with 5,000 credits and no card required. Geotargeting is US and EU only on Hobby and Startup; country-level targeting starts at Business. Credits reset at each renewal and do not roll over. Pay-as-you-go is available from Scaling upward, and auto-upgrade (opt-in) runs from Hobby through Professional. There is no credit rollover on any plan.

| Request type | Credits per request |
| --- | --- |
| Standard page | 1 |
| Amazon, Walmart, eBay | 5 |
| Google, Bing (including subdomains) | 25 |
| LinkedIn | 30 |
| Anti-bot bypass (Cloudflare, DataDome, PerimeterX, Akamai, Turnstile) | +10 on top of the base cost |
| `render=true` | 10 |
| `premium=true` | 10 |
| `premium=true` + `render=true` | 25 |
| `ultra_premium=true` | 30 (paid plans) |
| `ultra_premium=true` + `render=true` | 75 (paid plans) |

Country code, session, device type, output format, and autoparse add nothing.

The arithmetic matters more than the sticker price. A 5,000-page blog crawl on plain HTML pages costs 5,000 credits, which fits inside Hobby with room to spare. Turn on `render` for a JavaScript-heavy store and the same 5,000 pages cost 50,000 credits. Add premium proxies and rendering together and 5,000 pages cost 125,000 credits, more than Hobby's entire monthly allowance. That is the single most common way a crawl budget gets blown, and it happens before the crawl even starts.

👉 [See how far 100,000 credits stretch on Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## The three endpoints and the smallest request that works

Crawler runs on its own host, separate from `api.scraperapi.com`.

| Action | Method | URL |
| --- | --- | --- |
| Start a crawl | POST | `https://crawler.scraperapi.com/job` |
| Check status | GET | `https://crawler.scraperapi.com/job/<jobId>` |
| Cancel a crawl | DELETE | `https://crawler.scraperapi.com/job/<jobId>` |

Authentication differs by verb: `api_key` goes in the JSON body on POST, and as a query parameter on GET and DELETE. A required parameter sent the wrong way returns 401, not 400, which makes the mistake easy to misread.

The required fields are `api_key`, `start_url`, `url_regexp_include`, and at least one of `max_depth` or `crawl_budget`. Supply both and the crawl stops at whichever limit it hits first.

python
import os, time, requests

API_KEY = os.environ["SCRAPERAPI_API_KEY"]

job = requests.post(
    "https://crawler.scraperapi.com/job",
    json={
        "api_key": API_KEY,
        "start_url": "https://example.com/blog/",
        "url_regexp_include": r"(?<full_url>https?://example\.com/blog/.*)",
        "url_regexp_exclude": r".*\.(pdf|png|jpg|zip)",
        "max_depth": 3,
        "crawl_budget": 500,
        "api_params": {"country_code": "us", "render": False},
        "callback": {"type": "webhook", "url": "https://yourapp.com/crawler-results"},
    },
).json()

job_id = job["jobId"]


Polling works for short jobs. Stop on any terminal state, including `in delivery`, which follows `completed` and precedes `delivered`:

python
def wait_for_crawl(job_id, api_key, interval=10):
    url = f"https://crawler.scraperapi.com/job/{job_id}"
    while True:
        data = requests.get(url, params={"api_key": api_key}).json()
        if data.get("status") in {"completed", "failed", "cancelled", "delivered"}:
            return data
        time.sleep(interval)


The full state chain is `delayed` → `running` → `completed` / `failed` / `cancelled` → `in delivery` → `delivered`. A job sitting in `in delivery` is finished scraping and pushing results out, so don't treat it as still running.

## The include regex needs a named capture group

`url_regexp_include` is a standard regex with one hard requirement: it must contain a named capture group. Use `(?<full_url>...)` for absolute URLs and `(?<relative_url>...)` for relative paths. An include pattern without the capture group is rejected, which is the most common reason a first job returns 400.

`url_regexp_exclude` is a plain regex and needs no group.

| Goal | Pattern |
| --- | --- |
| Whole domain | `(?<full_url>https?://example\.com/.*)` |
| Blog section only | `(?<full_url>https?://example\.com/blog/.*)` |
| HTML pages only (no file extension) | `(?<full_url>https?://example\.com/[^.]*$)` |
| Relative paths | `(?<relative_url>/products/.*)` |

Exclusions are where you prevent waste. `.*\.(pdf|png|jpg|jpeg|gif|svg|zip|mp4)` keeps binary files out, and `.*(login|logout|signup|admin|auth).*` skips the pages that would return a redirect or a login wall.

On `.*` as a shorthand for "crawl everything": the UI documentation offers it as a way to crawl all pages on a site, and it does that. It also follows any external link on the page, which is why the API documentation warns that a pattern like `.*` will crawl the wider web. On a domain you control, or one you know links only to itself, `.*` is fine. Otherwise pair it with a tight exclude list, or start with a narrow include such as a single directory, run 20 or 30 pages, and check the returned URLs before widening it. A crawl that wanders off-domain still costs credits.

## max_depth vs crawl_budget: pick one, or use both

These two limits answer different questions. Depth asks "how far from the seed", budget asks "how much am I willing to spend".

| Use `max_depth` when | Use `crawl_budget` when |
| --- | --- |
| The site structure is known and bounded | Site size is unknown |
| You want one section, e.g. 3 hops from a category page | You want a hard cap on credit spend |
| The site is small | The site is large with unpredictable link density |

Depth semantics are literal. Depth 0 scrapes the start URL only. Depth 1 scrapes the start URL plus every page it links to. Depth 2 goes one level deeper, and so on. On a site with pagination and faceted navigation, depth 3 can mean thousands of near-duplicate pages, because every filter combination is a link.

The safe pattern for an unfamiliar site is both limits at once: `max_depth: 3` with `crawl_budget: 500`. You get a structural boundary and a spend ceiling, and the job ends on whichever fires first. Once you've seen the URL list a run produces, loosen whichever limit was binding.

For a known section of a known site, `max_depth` alone is simpler and gives reproducible job sizes. For a first look at a client's site you've never crawled, `crawl_budget` alone protects the account, at the cost of an arbitrary cutoff mid-section.

## Where the credits go, and how to cap them

`crawl_budget` is a hard ceiling, not a target. When it's reached the job stops gracefully rather than overshooting, and failed requests are not charged. That combination makes a first exploratory crawl low-risk: a 500-credit budget on a site that returns errors for half its pages costs less than 500.

Each webhook payload carries an `sa-credit-cost` header showing what that page cost. Log it on the first run and you have a real per-page number for that domain instead of an estimate from the table. Multiply by the URL count you expect, add 20% for retries, and you have a defensible budget for the production run.

`render: true` deserves its own warning. It multiplies the cost of every crawled page and only helps when the site actually loads content with JavaScript. Test without it first: if the HTML you get back already contains the product names or article text, rendering adds 10 credits per page for nothing. The same logic applies to `premium` and `ultra_premium`; both exist for sites that block plain requests, and both raise the per-page cost several times over.

`api_params` accepts the full standard parameter set, including `country_code`, `device_type`, and `output_format: "markdown"`. Markdown output is the practical choice when the crawl feeds an LLM or a training set; raw HTML stays the default and is what you want if you're parsing with your own selectors.

## Webhooks, job states, and the 7-day result window

For anything longer than a few minutes, use a webhook instead of polling. With `callback` set, ScraperAPI POSTs each page to your endpoint as it is scraped, not in one batch at the end. Every payload contains the crawled URL, the HTML or content, the credit cost of that request, and the current depth. A final summary payload follows when the crawl finishes, listing succeeded and failed URLs with the total cost.

Three operational details. The webhook URL must be publicly reachable. Delivery is retried up to 3 times. A failed webhook does not stop the crawl, so an endpoint that's down loses data without failing the job.

Crawled results are stored for up to 7 days. For scheduled crawls, each new run replaces the previously stored results, so a weekly crawl gives you one week of history, not a growing archive. If you need the data long-term, persist it on your side from the webhook payloads rather than re-fetching from the dashboard later.

The Crawler UI mirrors these fields: link depth, include path, exclude path, callback URL, maximum cost in credits, and crawling frequency. It also exposes request-level options that are easy to miss in the API docs, including disabling redirect-following, retrying 404 responses (useful on domains that throw fake 404s), advanced bypassing, residential or mobile IPs, session reuse, and desktop or mobile device emulation.

## Free plan limits and what scheduling requires

Free accounts can only run one-time crawls, and they're capped at `max_depth: 1`. That combination is enough to validate a regex against a single directory and see whether the pages you want are reachable. It is not enough to crawl a site three levels deep.

Scheduling requires a paid plan. Omit `schedule` and the job runs once. Add it to recur:

json
{"schedule": {"name": "weekly-blog-crawl", "interval": "weekly"}}


Accepted intervals are `once`, `hourly`, `daily`, `weekly`, and `monthly`. The UI adds a "scheduling disabled" option that saves the crawler configuration without running it, which is handy when you want to set a job up now and start it after a site migration.

Notification preferences in the UI are separate from scheduling: never, after every run, or a daily/weekly digest. For a crawl that runs hourly, the digest is the only sane setting.

👉 [Pick a paid plan if you need scheduled crawls](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Running Crawler from n8n

The official n8n community node (search "ScraperAPI" in the node panel, credentials are your dashboard API key) exposes three Crawler operations: initiate a job, get job status, and cancel a job. Parameters map to the API: start URL, max depth, crawl budget, include and exclude regex, per-page API parameters, an enabled toggle, and schedule name and interval.

One difference from the raw API: in the node, `callbackUrl` is marked required. Webhooks are optional when you call the API directly, but the node expects an endpoint to stream results to, and both successful and failed attempts go there. If you're wiring Crawler into an automation, plan for a receiving endpoint before you start.

## Error codes and what to do about them

| Code | Meaning | Fix |
| --- | --- | --- |
| 200 | Job created or status returned | Nothing |
| 400 | Malformed request | Check required fields and regex syntax, especially the named capture group |
| 401 | Invalid API key | Check the key and whether it's in the body (POST) or query string (GET/DELETE) |
| 403 | Credits exhausted, or free plan depth limit hit | Upgrade, lower `max_depth`, or lower `crawl_budget` |
| 429 | Too many concurrent jobs | Wait and retry, or raise your plan's thread count |
| 500 | Transient failure | Retry with backoff |

A 403 on the free plan almost always means `max_depth` is above 1, not that the account is out of credits. Those are different problems with the same status code.

## Which setup fits your job

Match the tool to what you already know:

- **Known URL list, any size:** Async API. It's cheaper per page than a crawl because you're not paying to discover links.
- **One page, or a handful:** the standard endpoint. Don't stand up a crawl job for five URLs.
- **An entire section you can't enumerate:** Crawler with a narrow include regex pointing at that directory, `crawl_budget` set to what you're willing to spend, and `render` off until you've confirmed the HTML is empty without it.
- **A site you've never touched:** `max_depth: 3` plus `crawl_budget: 500`, run once, read the URL list, then scale.
- **Login-gated content:** none of the above. Crawler only sees public pages.
- **Testing before committing:** free plan, one directory, `max_depth: 1`.

Two things to confirm before you pay. First, count your pages and multiply by the real credit cost for that domain, including any rendering or proxy surcharges; the plan table above is the multiplier, not the answer. Second, decide whether you need scheduling, because that alone moves you off the free plan, and whether you need country-level geotargeting, which starts at Business.

👉 [Open the ScraperAPI pricing page and match credits to your page count](https://www.scraperapi.com/pricing/?fp_ref=coupons)
