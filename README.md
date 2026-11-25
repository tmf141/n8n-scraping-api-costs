# n8n Web Scraping: Two Build Paths, the Point Where Core Nodes Fail, and What the API Behind Them Really Costs

n8n can scrape a static page with two nodes and no code: an HTTP Request node fetches the HTML, an HTML Extract node pulls the fields out with a CSS selector. That covers product listings, article indexes, directories, and any page whose HTML already contains the data you want. It stops working the moment the page builds its content with JavaScript, or the site decides your datacenter IP looks like a bot.

So there are two paths. Path one is core nodes only, free, and enough for a large share of n8n web scraping jobs. Path two swaps the raw fetch for a scraping API that handles rendering, proxies, and blocks, then keeps the parsing inside n8n. This article walks both, then covers the parts the usual tutorials skip: what an API request actually costs in credits, how pagination and deduplication hold up at scale, and which settings to check before you pay for anything.

The skeleton for either path is the same five steps: pick a URL, fetch the HTML, choose the data you want, run the extraction, export something readable.

> **Quick pick:** start on ScraperAPI's free tier. It includes **1,000 API credits and up to 5 concurrent connections**, which is enough to confirm your target site renders and returns the fields you need before you spend anything. A standard page costs 1 credit, though harder sites cost more (more on that below). 👉 [Check the free plan and current credit allowances](https://www.scraperapi.com/pricing?fp_ref=coupons)

## ScraperAPI plans at a glance

ScraperAPI's pricing page carries the current dollar figure and credit allowance for each tier. What the plan names don't tell you is what happens when you run out mid-month, and that rule decides whether a cheap plan stays cheap.

| Plan | Who it fits | Auto-upgrade when credits run out | Pay-as-you-go past the limit | Get started |
| --- | --- | --- | --- | --- |
| Free | Testing one target site end to end | — | — | [Free credits](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | Occasional small batches | Yes, opt-in | — | [Hobby pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | Recurring jobs on standard pages | Yes, opt-in | — | [Startup pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | Higher volume, still standard pages | Yes, opt-in | — | [Business pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | Spiky workloads | Yes, opt-in | Yes, up to a monthly cap you set | [Scaling pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | High volume plus overage headroom | Yes, opt-in | Yes, up to a monthly cap | [Professional pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | Large, unpredictable volume | — | Yes, up to a monthly cap | [Advanced pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom volume | — | Yes, under custom pricing | [Enterprise details](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Two behaviours are worth understanding before you pick a row. By default, scraping **pauses** until your credits renew. Auto-upgrade is the alternative: when credits hit zero, your account moves to the next plan automatically. It's opt-in, so if you want a hard stop rather than a bigger bill, leave it off. Annual plans can also renew early for a fresh batch of credits, which matters if you burn through an allowance in three weeks instead of four.

## Path 1: scraping with n8n's core nodes

Start with an **HTTP Request** node. Set Request Method to GET, paste the URL, set **Response Format** to *String*, and give the response a **Property Name** such as `data`. String format matters: if n8n parses the response as JSON, the HTML arrives mangled or empty.

Add an **HTML Extract** node next. Point Source Data at `data`, give the output key a name like `books`, enter a CSS selector for the repeating block, set Return Value to *HTML*, and turn on **Return Array**. On the standard books.toscrape demo catalogue, `.row > li` grabs every product card; on the individual listing pages, `article.product_pod` does the same job. You can get the selector by right-clicking the element in your browser and inspecting it.

A **Split Out** node follows, with Fields To Split Out set to `books`. Each product becomes its own item, which is what makes per-record extraction and filtering possible later.

Then a second HTML Extract node pulls the actual fields. For the books demo, this is the whole mapping:

| Key | CSS Selector | Return Value |
| --- | --- | --- |
| `name` | `h3 a` | Attribute → `title` |
| `url` | `h3 a` | Attribute → `href` |
| `price` | `.price_color` | Text |
| `availability` | `.instock.availability` | Text |
| `rating` | `p.star-rating` | Attribute → `class` |
| `image` | `.image_container img` | Attribute → `src` |

Two cleanups usually follow. Relative links come back as `../media/...` and need a base URL prefixed, and the rating arrives as a class string like `star-rating Three`, so a **Code** node (Mode: Run Once for All Items) strips the prefix and keeps the last word. Both are small JavaScript, and both are the kind of thing that silently produces broken CSVs if you skip them. Finish with a **Convert to File** node set to CSV.

For output, Google Sheets with *Append Row* and automatic column mapping works without extra setup. Microsoft Excel needs Azure credentials configured first, so most people start with Sheets or a local CSV.

One trap applies to every API-calling node you'll build from here: when you use **Import cURL**, n8n often converts booleans like `true` into the string `"true"`. APIs that expect a real boolean will reject it or ignore the flag. Click the gear icon next to the value, add an expression, and set it to `{{true}}`.

## Path 2: what a scraping API looks like inside n8n

The generic shape is always the same. An HTTP Request node set to POST, authentication via Generic Credential Type → Basic Auth, Send Body on, Body Content Type JSON, and the target URL plus options in the body. Many APIs accept `parse: true`, which returns structured data and makes the HTML Extract node unnecessary; others keep the HTML and let you parse in n8n as before.

Import cURL saves a lot of typing here. Paste the API's example command and n8n rebuilds the whole node, including headers and body, which is faster and less error-prone than filling fields by hand.

When you compare providers, the differences that actually affect your workflow are a short list:

- **JavaScript rendering.** Off by default on most APIs because it's slower and costlier. Turn it on only for pages that need it.
- **Proxy country.** Two-letter country codes for geo-specific results.
- **Proxy tier.** Standard, premium residential, and ultra-premium bypass mechanisms behave differently on Cloudflare, Datadome, or PerimeterX-protected sites.
- **Wait time or rendering instructions.** For pages that load content after an event, a click, or a delay.
- **Extraction schema or autoparse.** Some APIs return structured JSON for supported sites without any selector work.
- **Output format.** HTML, Markdown, plain text, JSON, and CSV are not equally available on every provider.

Geekflare's node, for example, exposes 7 output formats and 12 optional parameters including Render JavaScript, Proxy Country, Stealth Mode, Wait Time, and an Extraction Schema in JSON. That's a useful benchmark for what a mature n8n integration looks like, and it shows why the HTTP Request-only approach gets tedious once you need more than two of those switches.

## ScraperAPI has an official n8n node

This is where most n8n web scraping tutorials stop short: ScraperAPI ships its own n8n node, so you don't have to hand-build the HTTP Request. Open the node panel, search ScraperAPI, add the node, select the **API** resource, paste your URL, and execute. Proxy and user-agent rotation, CAPTCHA and bot-blocker bypass, and rendering are handled on ScraperAPI's side.

The parameters that matter in practice:

| Parameter | What it does | Watch out for |
| --- | --- | --- |
| `render` | JavaScript rendering | Default `false`; it increases processing time, so enable only for pages that need it |
| `countryCode` | Two-letter ISO code for geo-targeted scraping | Needed for localized SERPs and pricing pages |
| `outputFormat` | `markdown`, `text`, `csv`, or `json` | CSV and JSON only work on autoparse-supported sites |
| `autoparse` | Returns parsed JSON for selected sites | Check the supported-site list before relying on it |
| `premium` / `ultraPremium` | Stronger proxy pools and bypass mechanisms | Mutually incompatible with each other |
| `sessionNumber` | Reuses the same proxy session | Not compatible with `premium` or `ultraPremium` |
| `instructionSet` | JSON array of browser instructions (click, wait for event) run before returning the page | Implies `render` |
| `zipCode` | US ZIP for location-specific Amazon results | Amazon US only |

There's a real trade-off buried in `customHeaders`: passing your own headers switches on `keepHeaders`, and overriding ScraperAPI's header management usually lowers success rates. Only do it when the target genuinely requires a specific header. Custom headers are also discarded when `ultraPremium` is enabled.

The node also works as a tool input to an AI Agent node, which lets an agent decide when to fetch a page rather than following a fixed workflow. If your n8n web scraping job is "summarize whatever is on this page today," that's a shorter build than wiring the HTTP Request manually.

## What a request actually costs

This is the part that decides your bill, and it's the part plan tables hide. ScraperAPI prices by credit, and one credit is a standard page request. Harder sites cost more:

- Standard page: **1 credit**
- Amazon: **5 credits**
- Google and Bing, including subdomains: **25 credits**
- LinkedIn: **30 credits**
- Sites behind Cloudflare, Datadome, or PerimeterX: **+10 credits** per request when the bypass runs

Run the free tier's 1,000 credits against those numbers and the same allowance buys very different amounts of work: about 1,000 plain pages, 200 Amazon pages, 40 Google SERPs, or roughly 66 requests to a bot-protected Amazon page at 15 credits each. A workflow that looks cheap on a blog looks nothing like cheap on a search results page.

Two tools keep this predictable. The **Domain Cost Estimator** in the dashboard gives the exact cost for any URL before you commit, and the `max_cost` parameter caps what a single scrape is allowed to spend. Set it when you're pointing a loop at an unknown list of domains. 👉 [Get 1,000 free credits and price your target URL](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Running past one page without breaking things

An official n8n template shows what a multi-page job looks like in practice. It starts from a country list, opens each country page, paginates every page inside it, extracts the records, writes them to MongoDB, and repeats. Three techniques in it are worth copying into your own workflows:

**Pagination state.** `getWorkflowStaticData('global')` stores the next page number, so a workflow that stops halfway can resume from where it left off instead of starting over.

**Local caching.** Visited pages get saved to n8n's local disk. On a relaunch, the workflow checks for a cache file and skips requests it has already made. For a job that touches thousands of pages, this alone can cut the run in half.

**Deduplication.** Before inserting into MongoDB, the workflow checks whether the primary key already exists. Inserting twice is the most common way a clean dataset turns into a mess, and a uniqueness check is cheaper than cleaning up afterwards.

For freshness, a Cron node re-checks the source weekly. For IP health, route every request through a proxy with rotation; a single IP hammering one domain gets blocked fast, and the template author recommends a proxy for all requests rather than as a rescue measure.

The same author draws a boundary worth repeating. The template suits small data requirements. Dynamic pages need a headless browser or a scraping API. Huge lists of URLs are better handled by a dedicated framework like Scrapy. n8n is good at scheduling, connecting systems, and keeping the logic visible; it isn't a crawler engine.

## Compliance and rate limits

Check the target site's terms and privacy policy before you build anything. Where scraping isn't mentioned, look at `robots.txt`, but treat the terms of service as the binding document and `robots.txt` as a courtesy guideline. Scraping that violates a site's terms can get you banned, and commercial use of the data raises the stakes further.

Rate is the other lever. Rapid-fire requests strain servers, and sites respond by blocking you. Space requests out, use pagination with realistic delays, and don't treat a 200 response as permission to send a thousand more.

## Which path to build

- **Static HTML, a few hundred to a few thousand pages, no blocking:** stay on core nodes. HTTP Request, HTML Extract, Split Out, Convert to File. No subscription needed.
- **JavaScript-rendered pages or repeated IP blocks:** move to an API. Keep the parsing in n8n and let the API handle the fetch.
- **Recurring batches with pagination and a database:** add static data for page state, a disk cache, and a uniqueness check before insert. Set a Cron trigger and walk away.
- **Google, Amazon, or bot-protected targets:** price the domain first. The credit multiplier, not the subscription tier, determines whether your budget survives the month.

Before you pay, confirm three things: the Domain Cost Estimator's number for your actual target URLs, whether the site you need is covered by autoparse (CSV and JSON output depend on it), and whether auto-upgrade is on or off on your account. Those three decide whether your first invoice matches your estimate.

👉 [Compare ScraperAPI plans and start on the free tier](https://www.scraperapi.com/pricing?fp_ref=coupons)
