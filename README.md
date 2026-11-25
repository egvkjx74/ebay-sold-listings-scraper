# eBay sold listings scraper: what a pricing-ready export contains, which numbers wreck your median, and how to run one yourself (with per-page credit math)

A sold-listing record you can price from needs eight fields: final price, currency, sale date, condition, shipping cost, listing type, bid count for auctions, and the seller's feedback score. Everything else on the card is context.

Three of those numbers can be wrong in ways eBay's own search page doesn't flag. Best Offer sales display the asking price, not the accepted offer the buyer actually paid. Results for private sellers on ebay.co.uk, ebay.fr and ebay.it carry a buyer protection fee inside the displayed price. Thin searches get padded with "results matching fewer words" that belong to a different model entirely. Below: what a clean export looks like, what ScraperAPI charges to fetch those pages, and how to build a run that ends with a median you can price against.

> **Quick take.** If you have a handful of keywords and want a spreadsheet, per-result scrapers in this niche run $2.50–$3.50 per 1,000 sold listings and handle the parsing for you. If you run many keywords on a schedule and want the rows in your own schema, ScraperAPI bills an eBay request at 5 credits, its free tier is 1,000 credits a month at 5 concurrent threads, and the top fixed plan listed is 21,500,000 credits at $1,975/month. 👉 [Check ScraperAPI's current plans and credit allowances](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan currently on the pricing page

| Plan | Credits per month / concurrent threads | Price | Billing | Get started |
| --- | --- | --- | --- | --- |
| Free | 1,000 / 5 | $0, no card required | Monthly | [Start on the free tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 / 20 | Shown on the pricing page | Monthly or annual | [See the Hobby tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 / 50 | Shown on the pricing page | Monthly or annual | [See the Startup tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 / 100 | Shown on the pricing page | Monthly or annual | [See the Business tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 / 200 | $475/mo, or $427.50/mo billed annually ($5,130/yr) | Monthly or annual | [See the Scaling tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 / 300 (limited offer adds 250K credits) | $975/mo, or $877.50/mo billed annually ($10,530/yr) | Monthly or annual | [See the Professional tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 / 500 (limited offer adds 500K credits) | $1,975/mo, or $1,777.50/mo billed annually ($21,330/yr) | Monthly or annual | [See the Advanced tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ / 500+ | Custom | Contract | [Ask about Enterprise volume](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Two things in that table matter more than the headline price. Annual billing is a straight 10% off on the three tiers with fixed prices ($475 → $427.50, $975 → $877.50, $1,975 → $1,777.50). And geotargeting is listed as "US & EU" up to Startup, then "Country-level" from Business upward. If you scrape ebay.co.uk, that gap matters, since the UK isn't in the EU.

## What one eBay search page actually costs you

ScraperAPI's credit table prices a standard page at 1 credit and puts eBay in the e-commerce tier at **5 credits per request**. Extras stack on top: JS rendering (`render=true`) adds 10 credits, premium proxies add 10, and the table lists premium proxies plus JS rendering together at 25 credits per request.

Run the arithmetic for a comps job:

- 1,000 free credits ÷ 5 = **200 eBay search pages** before you hit the wall. That's a test, not a batch.
- Hobby's 100,000 credits = **20,000 eBay pages**. At 10 pages per keyword, that's 2,000 keywords a month.
- If every page needs rendering plus a premium proxy at the combined 25-credit rate, the same 1,000 credits buy 40 pages.

At the Scaling rate of $475 for 5,000,000 credits, one credit costs about $0.000095. A page of 60 sold listings costs 5 credits, so 1,000 sold listings (about 17 pages) lands under a cent in request credits. Per-result services charge $2.50–$3.50 for the same thousand rows. The difference is that you're paying for pages whether or not they contain usable sales, and you own the parsing.

## Where eBay sold data goes wrong

eBay shows the sale price on the card, so scrapers pass it through. The problems sit in what that price means.

**Best Offer prices are asking prices.** When a seller accepts an offer, eBay strikes through the original price and prints "Best offer accepted." The accepted amount is never published, not in search results and not on the item page. So `soldPrice` for those records is the seller's original ask, which is higher than what the buyer paid. If you average them in, your median drifts upward. To even detect them you need `LH_Complete=1` in the URL; with `LH_Complete=0` eBay strips the strikethrough styling and the caption, and those sales look identical to ordinary Buy It Now. The results are still genuinely sold either way, because `LH_Sold=1` is always applied. Mark them, then exclude them from every price statistic.

**UK, French and Italian results hide a fee inside the number.** On those three marketplaces eBay adds a Buyer Protection fee to private sellers' prices in search results. A Nintendo Switch OLED in one verified sample shows a displayed price of 128.62 and a fee of 5.62, meaning the item itself sold for 123 GBP. Pull the displayed number as your comp and you overpay by roughly 4.5% on that class of listing. The fix is to identify private sellers with a second, filtered search and split the fee into its own field. ScraperAPI won't do that for you on an eBay search URL; you get the HTML.

**eBay pads thin searches and confuses models.** When exact matches run low, the page fills with "results matching fewer words." eBay's matching is also loose in a way that bites resellers directly: a search for "playstation 5 slim" returns PlayStation 4 sales, and "iphone 14" brings back iPhone 14 Pro and iPhone 12. A title-level filter after the fetch is not optional. Exclude graded-card slabs (psa, bgs, cgc), "box only", "for parts", and the model string you didn't ask for.

**Shipping is often blank on UK and EU sales.** Most domestic sold listings there show "Postage not specified" (or "Keine Angaben zum Versand", "Livraison non spécifiée"), so `shippingPrice` and `totalPrice` come back null. Free shipping is a real 0, which is a different value. Never fill the gap with an estimate; a null shipping column is honest, a guessed one quietly corrupts your landed-cost math.

**Dates stop at the day.** Sold cards print a date, and scrapers parse it to midnight UTC. There's no sale timestamp. More importantly, eBay's search URL accepts no sold-date range parameter, so "last 30 days" can only be approximated: sort by `_sop=13` (ended recently) and stop paginating once items fall outside your window.

## The URL parameters that do the filtering work

Most of your filtering can happen on eBay's side, which saves credits. The ones worth knowing:

- `LH_Sold=1` for sold items, plus `LH_Complete=1` for Best Offer detection
- `_sop=13` for recently ended, `_sop=10` for newly listed, `_sop=15`/`16` for price plus postage
- `_sacat` for category, `_udlo`/`_udhi` for price bounds
- `LH_PrefLoc=1` for domestic only (US only on ebay.com), `2` for worldwide
- `LH_ItemCondition=1000` new, `3000` used, `1500` open box, `2010` refurbished, `7000` for parts
- `_ipg` for page size (60 default, 120 and 240 available depending on the implementation) and `_pgn` for paging

## Build it yourself, or pay per result

The build path is genuinely more work than the URL parameters suggest. Aggressive paging from one IP triggers eBay's `splashui/challenge` interstitial, which usually clears in seconds, or a "SORRY / Something went wrong" page, which needs a new IP or a wait. Space requests 2–4 seconds apart. ebay.co.uk and ebay.de can refuse IPs from unrelated regions outright, so a UK or German proxy is part of the setup, not an optimization.

Session handling is the part people get wrong. Don't parallelize pages of the same keyword inside one session; eBay's anti-scrape layer tracks sessions, not just IPs. For volume, spread keywords across separate sessions with their own fingerprints and rate budgets. Write results to JSONL after every page so a mid-run block resumes from the last completed page instead of restarting the keyword. Use `_ipg=240` for exhaustive collections to cut request count fourfold, and 60 for spot checks. When you do get blocked, fail loudly. A run that returns an empty dataset and calls it success is worse than an error.

The buy path removes all of that. Per-result services charge $2.50–$3.50 per 1,000 listings, handle the Buyer Protection fee split, drop other models, and refund or skip blocked runs. The trade is that you don't control the schema, the marketplace list, or the keyword batching, and the per-row price is 300–400× the raw request cost.

ScraperAPI sits between the two. It's a general fetching API, not a sold-listings product, so you bring the eBay search URLs and the parsing. What it handles is proxies, retries, and rendering. Its documented eBay endpoint is a Product Page API that returns JSON for a single `/itm/` page: title, seller, price, images, item specifics, and a sold count like "1,866 sold" on a live listing. That's useful for enriching a specific item you're considering, not for pulling a set of completed sales. For comps, you send search URLs and parse the cards yourself.

## Marketplaces, currencies and the seller-type label

Implementations differ here, so check before you commit. One widely used open implementation covers 8 marketplaces: .com, .co.uk, .de, .fr, .it, .es, .ca, .com.au. A more recent per-result product covers 17, adding NL, AT, CH, IE, PL, HK, MY, PH and SG, and prices every result in the marketplace's own currency regardless of where the proxy sits.

That last point is the one that quietly ruins cross-market comparisons. A £123 sale and a €140 sale are not comparable rows in the same column. Keep the currency field, and convert at a rate you record alongside the row, or keep separate sheets per marketplace.

The `sellerType` label (private vs business) only exists on ebay.de, ebay.fr, ebay.it and ebay.es. Non-EU sites return null. On European sites you can also pass seller type as a filter, which limits results to sellers registered in that country.

## A minimum viable run

1. Build one eBay search URL per keyword with `LH_Sold=1`, `LH_Complete=1`, `_sop=13`, and your condition and price filters.
2. Fetch through ScraperAPI, spacing requests 2–4 seconds apart, waiting for `li.s-card` to be visible before parsing.
3. Write each page to JSONL as it completes so a block doesn't cost you the run.
4. Stop when `soldAt` falls outside your window or when eBay stops returning pages.
5. Drop Best Offer records from price math, and drop titles that fail your model and exclude-word filter.
6. Compute median, p25 and p75 per keyword rather than a mean, since one outlier auction can drag an average far off.

## Which route fits you

Occasional checks on a few models: use a per-result scraper. At $2.50–$3.50 per 1,000 rows it's cheaper than the afternoon you'd spend on pagination and proxy rotation, and the fee split and model filtering are already done.

Daily runs across dozens of keywords, with output feeding your own pricing sheet or model: the API route wins on cost and control. Start on the free 1,000 credits to confirm your parser handles eBay's card markup, then size the plan from real page counts. A 50-keyword sweep at 10 pages each is 500 pages, or 2,500 credits, which Hobby's 100,000 covers many times over.

Before you order, confirm two things on the live page: the current credit allowance and thread count for the tier you're picking (prices and limited-time bonus credits change), and whether blocked or failed requests are billed. 👉 [Compare ScraperAPI's plans and start with the free credits](https://www.scraperapi.com/pricing?fp_ref=coupons)
