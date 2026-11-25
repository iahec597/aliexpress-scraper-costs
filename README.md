# AliExpress Scraper: Do You Need a Headless Browser, Which Fields Decide Your Margin, and What 1,000 Products Really Cost (Plan Comparison Inside)

AliExpress no longer prints product prices into the HTML it serves. The price, SKU list and shipping options are fetched by JavaScript after the document loads, which is why a plain `requests` + BeautifulSoup script that worked a few years ago now comes back with a title, a gallery and an empty price field. ScraperAPI's own AliExpress tutorial puts it plainly: the product information is injected through AJAX, the page has to render first, and without that you get a blank page.

Two kinds of scraper survive that change. You can run a headless browser yourself and keep it alive, or you can send the URL to a managed scraping API that renders the page and absorbs the bot checks for you. The decision comes down to whether you'd rather maintain a Chrome process or pay a monthly bill.

> **Quick verdict.** A one-off pull of a few hundred products, or a parser you want to own outright: run the open-source Node library and accept the upkeep. Recurring runs across several storefronts, where rendering and anti-bot work should not be your problem: use a managed API. ScraperAPI renders the AliExpress page and returns the HTML, so the parsing code you already wrote keeps working while someone else tracks Chrome versions and stealth plugins. 👉 [See ScraperAPI's current plans and credit allowances](https://www.scraperapi.com/pricing?fp_ref=coupons)

## ScraperAPI's plans, side by side

ScraperAPI sells API credits, so the plan you need is decided by what an AliExpress page costs in credits, not by how many product rows you want. The plan names below are the ones that appear in the company's own billing documentation, alongside their confirmed differences.

| Plan | Confirmed allowance and differences | When credits run out | Price and billing | Get started |
| --- | --- | --- | --- | --- |
| Free | 1,000 API credits on signup, up to 5 concurrent connections | Scraping pauses until credits renew | $0 to start | [Free plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | Entry paid tier; upgrading to a higher plan brings a larger allowance and usually a better price per credit | Auto-upgrade to the next plan can be switched on | Monthly or annual, USD, current rate on the pricing page | [Hobby plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | Larger allowance than Hobby | Auto-upgrade available | Monthly or annual, USD | [Startup plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | Larger allowance again | Auto-upgrade available | Monthly or annual, USD | [Business plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | Volume tier | Auto-upgrade, or pay-as-you-go at a fixed per-credit rate up to a monthly cap you set | Monthly or annual, USD | [Scaling plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | Volume tier with the same overage options as Scaling | Auto-upgrade or pay-as-you-go | Monthly or annual, USD | [Professional plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | Top self-serve tier | Pay-as-you-go | Monthly or annual, USD | [Advanced plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Highest tier listed | Pay-as-you-go | Quoted on the pricing page | [Enterprise plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |

The price column is left to the live page on purpose. ScraperAPI quotes a monthly rate and a discounted annual rate, both in USD, and those numbers move; a stale figure copied into an article is worse than a link. Annual plans can also renew early for a fresh batch of credits, which matters if you burn through an allowance mid-project.

What doesn't move is how credits are counted, and this is where AliExpress budgets go wrong. A standard page costs **1 credit**. Amazon costs 5, Google and Bing 25, LinkedIn 30. Sites sitting behind Cloudflare, DataDome or PerimeterX **add 10 credits per request** whenever ScraperAPI has to bypass that protection. AliExpress is a site that actively fights scrapers, so a 1,000-credit plan is not automatically 1,000 AliExpress pages. Check the real number for an AliExpress URL in the Domain Cost Estimator in your dashboard before sizing anything, and set `max_cost` on a request so a single expensive page can't wreck the budget.

Running out doesn't produce a surprise invoice either. Requests beyond your concurrency limit come back as a 429 and are refused rather than queued, and by default scraping pauses until credits renew. Auto-upgrade is opt-in from Hobby upward; pay-as-you-go at a fixed per-credit rate, capped monthly, starts at the Scaling tier.

## What a usable AliExpress record contains, and the three fields that decide your margin

Field count is a weak way to judge a scraper. One Apify actor (silentflow's AliExpress scraper, page dated 29 September 2026) publishes a 29-field schema, and in its own prefilled test run 24 of those fields were filled on every row. The structure is worth copying as a checklist for whatever tool you pick:

- **Identity:** `id`, `url`, `title`
- **Classification:** `categoryId`, `isChoice`, `isSponsored`, `tags`, `skuId`
- **Numbers:** `price`, `originalPrice`, `discountPercent`, `sellerOfferPrice`, `rating`, `soldCount`, `soldText`, `currency`, `formattedPrice`
- **Logistics:** `shipFrom`, `deliveryDaysMin`, `deliveryDaysMax`
- **Time:** `listedAt`
- **Media:** `imageUrl`, `images`
- **Search context:** `keyword`, `page`, `position`, `totalResults`, `searchUrl`, `scrapedAt`

Three of those fields are where sourcing decisions actually get made or ruined.

**`soldCount` against the "5,000+ sold" label.** AliExpress rounds sales into buckets for shoppers. The same actor's sample row shows a product whose card reads "5,000+ sold" with an exact `soldCount` of **5,343**. If you rank candidates by the label, you're sorting products that all say "10,000+ sold" into an arbitrary order. With the integer you can tell a 10,200-unit listing from a 44,000-unit one.

**`price` is often a first-order price.** The sample row carries a card price of **0.33 USD**, a struck-through `originalPrice` of 3.45, a 90% discount badge, and a `sellerOfferPrice` of **3.45 USD**. That 0.33 is a new-shopper deal. Your margin after the first order comes from the seller price, and any sourcing sheet built on card prices will overstate margin by an order of magnitude on exactly the listings that look most attractive. The same trap exists at the page level: AliExpress serves different prices by region and currency, which is why the Python example scraper in Scrapfly's repository pins `country="US"` and sends a cookie carrying `c_tp=USD&region=US&b_locale=en_US`. Before comparing two sellers, confirm both prices came from the same storefront and currency.

**`listedAt` and `totalResults` for niche sizing.** `listedAt` is an RFC 3339 date for when the product went live, so you can filter for listings under three months old that already have real sales, which is a much better signal than raw sales volume on a five-year-old listing. `totalResults` reports how many products AliExpress says exist for the search: **14,107** for `wireless earbuds` on 29 September 2026. That number tells you how crowded the shelf is before you spend an afternoon on supplier outreach.

Two more fields earn their place in a supplier sheet: `isChoice` (the Choice badge changes buyer trust and delivery expectations) and `deliveryDaysMin`/`deliveryDaysMax` with `shipFrom`, since a 6–14 day window from China and a 30-day window are different products commercially.

## Self-hosted vs managed: what the money and the maintenance actually are

### The open-source route

The `aliexpress-product-scraper` package on npm is the reference implementation most tutorials point at, and its version history is a good map of the problem. Version 4.0.0 exists because AliExpress switched to client-side rendering; the library moved from reading SSR `runParams` to intercepting API responses, added `puppeteer-extra-plugin-stealth` to get past bot detection, and raised the Node requirement from 22 to **Node.js >= 24.0.0**. The API stayed the same, so your code doesn't change, but your runtime does.

Its documented failure modes are the maintenance contract in miniature: `Cannot find module` means Node is below 24; Puppeteer Chrome not found means running `npx puppeteer browsers install chrome`; timeouts get fixed by passing `{ timeout: 90000 }` instead of the 60-second default; empty data means the product is unavailable or you were blocked. Reviews default to 20 per product, and each one you add is another page to load.

None of that is hard. All of it recurs every time AliExpress redesigns its front end, and the redesigns don't wait for your sprint planning.

### The managed route

Managed scrapers carry the browser, the proxies and the anti-bot logic. Scrapfly's example AliExpress scraper shows the shape: Python 3.10+, a single API key, `unblocker=True` for anti-bot bypass, a US proxy country and a currency cookie, with separate functions for search pages, product pages and reviews. Its README notes that the example runs and tests scrape the live site and therefore consume credits.

For a price anchor on structured product data, one Apify actor (logical_scrapers, page dated 17 March 2025) publishes **$0.00399 per product** on the free tier and **$0.00299 per product** on Gold and above, plus Apify's platform fee:

| Products | At $0.00399 each | At $0.00299 each |
| --- | --- | --- |
| 1,000 | $3.99 | $2.99 |
| 5,000 | $19.95 | $14.95 |
| 20,000 | $79.80 | $59.80 |

That is the market rate for clean product records, and it sets a useful expectation: if a managed quote lands far above it for the same volume, you're paying for something else, usually rendering quality or coverage of regional storefronts. Two cost details from the same page are easy to miss. Turning on full details costs one extra request per product, so the price doesn't change but the run time does; the page estimates about 90 seconds for 100 products with details enabled.

ScraperAPI's billing model uses credits per page rather than dollars per product, so the comparison isn't one to one. Pages are the input; structured records are the output. Estimate your credit cost per AliExpress page first, then divide your monthly budget by that number to see how many rows you can realistically pull.

👉 [Check the current per-plan credit allowances before you estimate](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Pagination and quota traps

AliExpress search results carry **60 products per page**. How deep a single search can go depends on which tool you read: the newer Apify page (29 September 2026) caps a keyword at 60 pages, which puts a hard ceiling of **3,600 products per keyword**, while the older page (17 March 2025) says up to 100 pages per search. Plan against 3,600 and treat anything beyond it as a bonus.

The `maxItems` parameter is where most budget mistakes happen, because two tools with the same parameter name mean different things. On the newer actor, `maxItems` defaults to 50 and is shared evenly across every keyword and search link in the run. On the older one, `maxItems` applies per start URL with a default of 10, so five URLs at 100 each returns up to 500 rows. Start from the number of final rows you need, then work backwards to keywords, pages and credits.

One more detail worth knowing before you filter: `minPrice` and `maxPrice` apply to the price shown on the card, which is frequently the new-shopper price. A range of 2–10 USD can therefore include listings whose seller price sits above 10 USD.

## The official API doesn't cover search pages

AliExpress's open platform API is available to approved affiliates and sellers. It covers the affiliate catalogue, not arbitrary search or category pages, and it requires an approved developer account. If your requirement is "every result for this keyword, sorted by orders", no official API will do it.

That leaves scrapers reading public pages, and the responsibility for staying inside AliExpress's terms of service and your own jurisdiction's law rests with you. Practical guardrails: don't touch logged-in data, keep request rates modest, cache what you already pulled, and treat a 429 as a signal to slow down rather than a bug to retry through.

## FAQ

**Can I scrape AliExpress without a browser?**
Not reliably. The product data arrives via AJAX after the document loads, so the first response is a shell. You either render the page (a headless browser locally, or a `render` parameter on a managed API) or intercept the underlying API responses the way the npm library's version 4 does.

**Why is the scraped price lower than what I see at checkout?**
The card price is often a new-shopper deal. The sample row shows 0.33 USD on the card against a 3.45 USD `sellerOfferPrice`. Region and currency also change the number, so compare prices only when both came from the same storefront settings.

**Is there a cap on reviews?**
Yes, and it's usually yours to set. The npm library defaults to 20 reviews per product; Scrapfly's review function takes `max_scrape_pages` explicitly and paginates until you stop it, with each page spending credits.

**Why did a product page come back empty?**
Three common causes: the listing is unavailable, you were blocked, or the destination isn't reachable. The npm library's sample response makes the third case explicit, returning a shipping entry with `fees: 0` and `unreachable: true` for a China-to-India route. A row with data but no shippable option is still a useful row; it just isn't a sourcing candidate for that market.

## Which one to pick

- **A few dozen SKUs, one-off research:** the open-source Node library on Node 24, Chrome installed through Puppeteer, timeout raised to 90 seconds. Cheap, and you only pay in setup time.
- **Daily runs of thousands of rows with shipping cost and seller price:** a managed API. The $2.99–$3.99 per 1,000 products range is roughly what structured records cost in this market, and it buys back the afternoons you'd spend on stealth plugins.
- **Product pages plus search pages plus reviews in one pipeline:** check whether your tool returns all three from one input type. Mixing a scraper for listings and another for reviews doubles the maintenance.
- **Multi-region price comparison:** confirm the tool lets you fix the proxy country and currency before you trust any price column.

Two things to confirm before you pay anyone: whether the plan counts requests or credits, and what one AliExpress page costs in that unit once bot protection kicks in; and whether the storefront country and currency you need are supported. Get those two answers and the rest is arithmetic.

👉 [Compare ScraperAPI's plans and start with the free 1,000 credits](https://www.scraperapi.com/pricing?fp_ref=coupons)
