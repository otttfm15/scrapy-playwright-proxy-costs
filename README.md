# scrapy playwright proxy: The Working Setup, the Auth Trap That Breaks Most Tutorials, and What JS Rendering Actually Costs

Combining Scrapy, Playwright, and a proxy solves one specific problem: pages that only render with JavaScript, sitting behind anti-bot systems that block your IP after a few requests. But the three pieces don't bolt together the way most tutorials imply. There's an authentication conflict that silently kills the most commonly copied proxy config, and once JavaScript rendering is involved, your per-request cost can jump by a factor of 10 to 75. Both of those things are verifiable before you spend a cent, and this guide covers how.

Here's the short version. With ScraperAPI, the proxy port route doesn't work inside Playwright — the API endpoint route does. Your real cost per request depends on the target domain and the parameters you send: a plain HTML page costs 1 credit, the same page with `render=true` costs 10, and a Google SERP costs 25 before any rendering add-on. Plans start at $49/month for 100,000 credits, with a free tier of 1,000 credits and a 7-day trial of 5,000 credits that needs no credit card.

> **Quick picks**
> - Testing whether your spider works against a protected target: start free — 1,000 credits on signup plus a 7-day trial with 5,000 credits, no credit card required.
> - Small ongoing projects: Hobby at **$49/month** ($44.10/month billed annually), 100,000 credits, 20 concurrent threads.
> - Production crawls with geo-targeting beyond US/EU: Business ($299/month) or higher.
>
> 👉 [Check ScraperAPI's full plan list and current prices](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Why the standard proxy config fails with scrapy-playwright

The `proxy` key inside `PLAYWRIGHT_LAUNCH_OPTIONS` — with `server`, `username`, and `password` — is the snippet you'll find in nearly every scrapy-playwright proxy tutorial. It works fine with ordinary proxy providers. It fails with ScraperAPI, and the failure is structural rather than a bug.

ScraperAPI's proxy mode (`proxy-server.scraperapi.com:8001`) authenticates by passing the API key as a URL query parameter. Playwright's proxy configuration, on the other hand, only understands Basic Auth (username plus password) or IP authentication. It has no concept of query-string auth for proxies. Force the two together and you get `Proxy Authentication Required` on every request. ScraperAPI's own integration guide documents this explicitly and labels proxy mode "Not Recommended" for Playwright for exactly this reason.

The supported method is simpler anyway: point your browser at ScraperAPI instead of the target site.

javascript
const { chromium } = require('playwright');
const SCRAPERAPI_KEY = process.env.SCRAPERAPI_KEY;
const targetUrl = 'https://httpbin.org/ip';
const scraperApiUrl = `https://api.scraperapi.com?api_key=${SCRAPERAPI_KEY}&url=${encodeURIComponent(targetUrl)}`;

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto(scraperApiUrl, { waitUntil: 'domcontentloaded' });
  console.log(await page.content());
})();


ScraperAPI fetches and renders the target page on its side, then returns the final HTML. Your Playwright instance never touches a proxy, so there's no authentication conflict, and everything you'd normally do afterward — `page.$$eval()`, `page.locator()`, screenshots, network interception — works as usual. A quick sanity check: run this against `httpbin.org/ip` and the terminal prints the ScraperAPI exit IP, not yours.

Two practical rules from the official docs are worth following from day one. Keep the API key in a `.env` file with no quotes around it — quoted keys are a common first-run failure. And leave Playwright's own proxy settings empty when using ScraperAPI; the two authentication models don't mix.

Inside a Scrapy spider, the same principle applies through scrapy-playwright's selective rendering: each request opts into the browser path via a `meta` flag, while everything else stays as fast plain-HTTP requests. For requests carrying that flag, route through the ScraperAPI endpoint instead of configuring a proxy at all; for everything else, keep them out of the browser path entirely — a plain request costs 1 credit, while the same page through Playwright rendering costs 10. You don't need every page rendered, and the flag exists precisely so you don't.

## ScraperAPI plans and prices

All plans include JS rendering, premium proxies, CAPTCHA handling, auto-parsing, custom sessions, and unlimited bandwidth. The free tier gives you 1,000 credits at signup with a maximum of 5 concurrent connections — and if you need more credits for testing, ScraperAPI says you can contact support for additional test allowance. Annual billing takes 10% off every paid plan.

| Plan | Monthly price (annual) | Credits | Concurrent threads | Geo targeting | Link |
| --- | --- | --- | --- | --- | --- |
| Hobby | $49 / $44.10 | 100,000 | 20 | US & EU | [Start with Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 / $134.10 | 1,000,000 | 50 | US & EU | [Start with Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 / $269.10 | 3,000,000 | 100 | Country-level | [Start with Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling (Most popular) | $475 / $427.50 | 5,000,000 | 200 | Country-level | [Start with Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 / $877.50 | 10,500,000 | 300 | Country-level | [Start with Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 / $1,777.50 | 21,500,000 | 500 | Country-level | [Start with Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | 22,000,000+ | 500+ | Country-level | [Contact sales for Enterprise](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

Two plan-level limits don't show up in the headline numbers. Hobby and Startup can only target US and EU IPs — if your targets gate content by geography and you need, say, a Japanese or Brazilian exit IP, country-level targeting starts at Business. The analytics dashboard on the lower two plans also only keeps the last 30 days of data.

When credits run out, scraping pauses by default until the next cycle. Hobby through Professional can opt into auto-upgrade, which moves you to the next plan automatically. Scaling and above can keep scraping past the limit with pay-as-you-go at a fixed per-credit rate, capped at a monthly spending limit you set — that cap is worth setting, because it's the only thing standing between you and a runaway loop draining the account. Annual plans can also renew early for a fresh batch of credits.

## What a JS-rendered request actually costs

This is the part most scrapy-playwright tutorials skip, and it changes which plan you should buy. ScraperAPI charges per request in credits, and the price of one request depends on both the domain and the parameters.

Base rates per request:

- Standard page: **1 credit**
- Amazon, Walmart, eBay: **5**
- Google and Bing (all subdomains): **25**
- LinkedIn: **30**
- Cloudflare, Turnstile, Datadome, PerimeterX/Human, Akamai bypass: **+10**

Parameter add-ons:

- `render=true`: **+10**
- `premium=true`: **+10**
- `screenshot=true`: **+10**
- `ultra_premium=true`: **+30** (paid plans only)
- `premium=true` + `render=true`: **25 combined**
- `ultra_premium=true` + `render=true`: **75 combined**

That last line is the trap. Stacking is non-linear: render plus premium costs 25, not the 20 you'd get from adding the parts, and ultra premium plus render costs 75, not 40. ScraperAPI's own documentation marks both combos separately, and it's the single most consistently mis-budgeted part of the pricing.

A concrete example of how this mis-budgets real projects: suppose you plan 10,000 rendered requests against a Cloudflare-protected site and budget 20 credits each (`render=true` 10 + bypass 10). If a colleague then adds `premium=true` to handle the blocks, the actual combination — premium + render + Cloudflare bypass — runs higher than your budget, and a batch planned at 200,000 credits comes in well past that. The combinations are priced as fixed bundles in the official docs, not as sums, so check the combo table before the batch, not during it.

Free parameters, for contrast: `wait_for_selector`, `country_code`, `session_number`, `device_type`, `output_format`, `keep_headers`, and `autoparse` all add nothing. So pinning a proxy session across a pagination crawl costs nothing extra, which is convenient since session stickiness is exactly what you want when combining scrapy-playwright with rotating proxies.

The practical math on the Hobby plan's 100,000 credits: 100,000 requests against plain HTML blogs, roughly 6,600 Amazon product pages with rendering (5 base + 10 render = 15 per request), about 4,000 plain Google SERP requests (25 each), or roughly 2,900 SERP pages with rendering (25 + 10 = 35 each). Same plan, four completely different capacities. Test against your actual targets before committing to a tier.

One detail that softens the billing: ScraperAPI only charges for successful requests — status codes 200 and 404 — plus requests you cancel yourself within its 70-second window. Everything else (timeouts, 403s, blocks) costs nothing. And every response includes an `sa-credit-cost` header showing exactly what that request consumed; log it during development so you know your real cost curve before scaling up.

If you'd rather know the price before making the request, there are three ways: the Domain Cost Estimator in the dashboard, the API Playground for testing a target URL and seeing results, or a plain API call to `https://api.scraperapi.com/account/urlcost?api_key=API_KEY&url=...&render=true` that accepts all the same parameters and returns the exact cost. You can also set a `max_cost` per scrape so no single request can blow past your limit.

👉 [Test your target URLs in the API Playground with free trial credits](https://www.scraperapi.com/signup?fp_ref=coupons)

## Parameters that matter for scrapy-playwright proxy setups

Most of the work happens in a handful of query parameters appended to the ScraperAPI URL:

- **`render=true`** — enables JavaScript rendering via headless Chromium. The parameter you'll use most, and the one to use sparingly: it turns a 1-credit request into 10.
- **`country_code=us`** — routes through a US IP (or any supported country). Free.
- **`session_number=123`** — pins consecutive requests to the same proxy IP. Free. Essential for login flows and pagination, where appearing as one consistent visitor keeps you under anti-bot thresholds.
- **`premium=true`** — routes through the premium residential pool for harder targets.
- **`ultra_premium=true`** — the highest tier, reserved for the most aggressive anti-bot defenses. Paid plans only.
- **`wait_for_selector=.product-title`** — waits for a specific selector before returning the page. Free, but requires `render=true`.

A working "JS-rendered, US-targeted, session-sticky" request URL combines them: `https://api.scraperapi.com?api_key=KEY&url=ENCODED_TARGET&render=true&country_code=us&session_number=123`.

The escalating logic is straightforward: start with nothing extra, add `render=true` when the content genuinely needs JS, add `premium=true` when you start seeing blocks, and only reach for `ultra_premium=true` when the target defeats the premium pool. Jumping straight to the 75-credit combo on day one is how people burn through a trial in an afternoon. If your target renders everything server-side — many news sites and documentation sites do — leave rendering off entirely and your credits last ten times longer.

It's also worth being honest about when this whole setup is overkill. If your targets are static HTML with no meaningful bot protection, plain Scrapy with a standard rotating proxy handles them faster and cheaper, and you don't need Playwright at all. The ScraperAPI-plus-Playwright combination earns its cost against JS-heavy, protected targets — that's the scenario it's built for.

## Which plan fits which scraping project

Since geo-targeting is the real dividing line between tiers, here's the decision rule.

You're running a personal project or prototype against mostly unprotected pages → Hobby at $49 is the natural fit, and its 100,000 credits cover a lot of ground when most requests cost 1 credit. Just run the multiplier math first if Amazon, Google, or LinkedIn are on the target list — those rates hit an entry-level plan hard.

You need consistent sessions, moderate volume, and can live with US/EU-only IPs → Startup at $149 doubles the concurrency to 50 and gives you a million credits — roughly 66,000 rendered Amazon pages or 28,000 plain SERP requests per month.

Your targets are spread across countries, or you're feeding a production pipeline → Business at $299 unlocks country-level targeting and 100 concurrent threads. Going over the concurrency cap returns a 429 and drops the request rather than queueing it, so the thread count matters more than it looks — a spider with 200 parallel requests against a 100-thread plan silently loses half its throughput.

Large-scale operations → Scaling ($475, 5M credits, 200 threads) is ScraperAPI's most popular tier, and Professional or Advanced make sense once you're consistently exceeding its monthly allowance; upgrading mid-cycle generally lands a better per-credit rate than paying overage.

One thing to verify before any of this: your targets. The free tier exists precisely so you can check real credit costs and block rates against your actual domains before paying, and the `urlcost` endpoint makes that a two-minute check per URL. If the numbers come back ugly — say, 35 credits per rendered SERP page — it's better to know while the trial still has 5,000 credits on it.

👉 [Start the free 7-day trial — 5,000 credits, no credit card](https://www.scraperapi.com/signup?fp_ref=coupons)

Before you buy, two last checks are worth thirty seconds each: confirm your target domains' actual per-request cost through the cost estimator (the Amazon-at-5-credits surprises people every time), and if you're on annual billing, remember the 10% discount is already reflected in the annual column of the table above — the monthly figures are the pay-monthly rate. From there, whether you land on 👉 [Hobby's $44.10 annual rate](https://www.scraperapi.com/pricing/?fp_ref=coupons) or a country-targeting plan depends mostly on where your targets live, and now you have the numbers to decide rather than guess.
