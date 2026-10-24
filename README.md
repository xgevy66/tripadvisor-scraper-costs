# tripadvisor scraper: How to Pull Hotel and Restaurant Data Without Getting Blocked (Plus What It Actually Costs Per Request)

Anyone searching for a tripadvisor scraper runs into the same wall within the first request: TripAdvisor sits behind DataDome, and a plain Python `requests` call to a hotel review page gets an HTTP 403 back with a 775-byte CAPTCHA page instead of content. One guide published in mid-2026 ran the same requests + BeautifulSoup scraper against three major travel sites and confirmed this: Airbnb returned HTTP 200 and parsed cleanly, Booking.com returned a 202 JS challenge, and TripAdvisor returned 403 on the very first request. So the real question behind "tripadvisor scraper" is usually two questions: how do I get past the block, and what will it cost me to do it at scale?

This article covers both. We'll walk through the anti-bot situation, show a working approach using ScraperAPI (a proxy API service that handles the blocking for you), break down its credit system so you can estimate your real per-request cost, and compare the alternatives fairly — including TripAdvisor's own official API, which most scraper tutorials skip.

> **Quick take:** For scraping TripAdvisor hotels, restaurants, and reviews, ScraperAPI's **Hobby plan at $49/month** ($44.10/month on annual billing, 100,000 credits/month) covers most solo projects, and its anti-bot bypass handles the DataDome wall automatically. Every plan includes JavaScript rendering and residential IPs. There's a 7-day trial with 5,000 credits and no credit card required — **👉 start the ScraperAPI free trial here** to test it against your target pages before paying anything.

## Why a plain scraper fails on TripAdvisor

TripAdvisor blocks most scrapers on the first request, and the blocking is layered. Based on documented tests and scraper engineering notes, the defenses include:

- **DataDome anti-bot protection** — returns a 403 with a CAPTCHA script instead of the page
- **Aggressive datacenter IP blocking** — requests from cloud servers get flagged fast, which is why residential proxies are effectively mandatory
- **Dynamically loaded reviews** — review text loads via JavaScript, so even if you get the HTML, the reviews aren't in it without rendering
- **Interactive pagination** — review and search pagination needs extra handling (more on the exact URL trick below)

ScrapFly's open-source tripadvisor scraper notes that TripAdvisor requires its anti-scraping-protection bypass feature turned on and US proxy locations, and even warns that GraphQL payloads with values outside expected ranges can get a scraper blocked. This is not a site where "add a User-Agent header" gets you far.

## ScraperAPI plans and pricing (all current plans)

ScraperAPI prices by API credits and concurrency. Here's the full plan lineup from its pricing page:

| Plan | Price (monthly / annual) | Credits/mo | Concurrency | Geo targeting | Get it |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | 1,000 | 5 | — | [ Sign up free](https://dashboard.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | $49 / $44.10 | 100,000 | 20 | US & EU only | [ See Hobby plan pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149 / $134.10 | 1,000,000 | 50 | US & EU only | [ See Startup plan pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299 / $269.10 | 3,000,000 | 100 | Country-level | [ See Business plan pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | $475 / $427.50 | 5,000,000 | 200 | Country-level | [ See Scaling plan pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Professional | $975 / $877.50 | 10,500,000 | 300 | Country-level | [ See Professional plan pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975 / $1,777.50 | 21,500,000 | 500 | Country-level | [ See Advanced plan pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom | 22,000,000+ | 500+ | Country-level | [ Contact for quote](https://scraperapi.com/pricing?fp_ref=coupons) |

Annual billing takes 10% off every plan. All paid plans include JavaScript rendering, premium residential and mobile IPs, CAPTCHA and anti-bot bypass, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. Failed requests get their credits refunded, which matters more than it sounds once you're running thousands of requests a day. The Hobby, Startup, Business, Scaling, and Professional plans support auto-upgrade when you run out of credits; Scaling and above add pay-as-you-go overage up to a monthly spending cap you set. Hobby through Business plans have no overage option at all — when credits run out, scraping pauses until the next billing cycle.

Two plan-limit details that don't show up in the headline pricing: the two cheapest paid plans are restricted to **US and EU proxy locations** and only 30 days of usage analytics. That's fine for tripadvisor.com, but if you're scraping regional TripAdvisor domains (say, tripadvisor.jp or .com.br) and need local IPs from those countries, Business is the first tier with country-level geo-targeting. Also worth knowing: **credits do not roll over** — unused credits expire at the end of each billing cycle, so buying a bigger plan "for later" buys you nothing.

## What a TripAdvisor request actually costs in credits

This is the part most tutorials gloss over, and it changes your budget math significantly. ScraperAPI's pricing isn't flat: a standard page costs 1 credit, but protected sites and optional features cost more.

The official credit cost breakdown:

| Request type | Credits per request |
| --- | --- |
| Standard page | 1 |
| Amazon | 5 |
| Google / Bing (including subdomains) | 25 |
| LinkedIn | 30 |
| Sites behind Cloudflare, DataDome, or PerimeterX (when bypassed) | **+10 on top of base** |

Feature flags add their own multipliers. According to ScraperAPI's technical documentation, `render=true` (JavaScript rendering) adds 10 credits, `premium=true` (premium proxies) adds 10, and `ultra_premium=true` adds 30 — and these stack non-linearly rather than additively, so always verify with the estimator rather than doing mental math.

Here's why this matters for TripAdvisor specifically. TripAdvisor sits behind DataDome, so a successful request carries the +10 protection-bypass surcharge on top of the base cost. If you follow the code pattern below, which uses `render=true` because review text is dynamically loaded, a back-of-envelope estimate lands around **21 credits per request** (1 base + 10 rendering + 10 bypass). Treat that as an arithmetic estimate, not an official number — run your actual TripAdvisor URLs through the **Domain Cost Estimator** in your dashboard first, and set the `max_cost` parameter per request so no single scrape can burn more than you intend.

With that estimate, the budget math looks roughly like this: the free plan's 1,000 credits translates to around 45–50 rendered protected requests rather than 1,000 pages, and the Hobby plan's 100,000 credits covers somewhere near 4,500. Independent pricing analyses that ran the same credit math put ScraperAPI's real cost at roughly **$4.90 per 1,000 JS-rendered pages** on the Hobby plan, dropping to about $1.00 per 1,000 on Business — a per-credit price difference worth knowing before you commit to a tier.

If you want to verify the real cost against your specific URLs before committing money, the 7-day trial's 5,000 credits is enough for a couple hundred protected, rendered requests. **👉 Check current plan prices and credit rates on ScraperAPI's pricing page.**

## How to scrape TripAdvisor reviews with ScraperAPI

The mechanics are straightforward: you send your target URL through ScraperAPI's endpoint, and it returns the rendered HTML. A documented working pattern for hotel reviews looks like this:

python
import requests
from bs4 import BeautifulSoup
import time

API_KEY = "YOUR_SCRAPERAPI_KEY"

def scrape_tripadvisor_hotel(hotel_url, review_pages=3):
    hotel_data = {"reviews": []}
    response = requests.get("https://api.scraperapi.com", params={
        "api_key": API_KEY,
        "url": hotel_url,
        "render": "true"
    })
    soup = BeautifulSoup(response.text, "html.parser")
    name = soup.select_one('[data-test-target="top-info-header"]')
    rating = soup.select_one('[class*="overallRating"]')
    hotel_data["name"] = name.text.strip() if name else "N/A"
    hotel_data["rating"] = rating.text.strip() if rating else "N/A"

    for page in range(review_pages):
        offset = page * 10
        review_url = hotel_url.replace("-Reviews-", f"-Reviews-or{offset}-")
        response = requests.get("https://api.scraperapi.com", params={
            "api_key": API_KEY,
            "url": review_url,
            "render": "true"
        })
        soup = BeautifulSoup(response.text, "html.parser")
        reviews = soup.select('[data-test-target="reviews-tab"] [class*="review-container"]')
        for review in reviews:
            title_el = review.select_one('[data-test-target="review-title"]')
            text_el = review.select_one('[class*="reviewText"]')
            bubble = review.select_one('[class*="bubble_rating"]')
            hotel_data["reviews"].append({
                "title": title_el.text.strip() if title_el else "N/A",
                "text": text_el.text.strip() if text_el else "N/A",
                "rating": bubble.get("class", [""])[1] if bubble else "N/A"
            })
        time.sleep(3)
    return hotel_data


Three details in this pattern are worth understanding rather than copy-pasting blindly:

- **The review pagination trick.** TripAdvisor review pages use `-Reviews-or{offset}-` in the URL, where offset equals page number × 10. Page 1 has no suffix, page 2 is `-or10-`, page 3 is `-or20-`. You don't need to hunt for "next page" buttons.
- **The key selectors.** Hotel name lives in `[data-test-target="top-info-header"]`, the overall rating in `[class*="overallRating"]`, review containers under `[data-test-target="reviews-tab"]`, review text in `[class*="reviewText"]`, and the rating bubble in `[class*="bubble_rating"]`.
- **Rate limiting still applies on your end.** Keep 3–5 second gaps between requests. Also, TripAdvisor updates its HTML structure frequently, so check selectors if output suddenly goes empty.

One gotcha from search-result scraping (different from reviews): ScrapFly's scraper code warns that TripAdvisor's pagination step must be read from the site's own next-page link, because sponsored listings inflate the number of cards on a page — counting parsed cards to compute your next offset will drift you off course.

## TripAdvisor scraper alternatives: what else is out there

ScraperAPI isn't the only route, and depending on what data you need, something else might fit better. Here's the honest comparison — no links, just the facts:

**TripAdvisor's official Content API.** This is the option most scraper tutorials never mention. TripAdvisor offers an official API with 5,000 free calls per month. The catch: it caps you at **5 reviews per location**. If you need hotel names, ratings, and a small sample of reviews, this may be all you need — and it's free. If you need full review histories, it won't cut it, which is when a scraper API becomes relevant.

**SerpApi.** A search-results API rather than a full-page scraper: you query for a destination and get structured results (title, rating, review count, description, location) for restaurants, hotels, attractions, and more. It offers 250 free searches per month, supports different TripAdvisor country domains, GPS coordinate searches, and pagination up to 100 results per page. Best fit: you only need structured search listings, not individual review text, and you'd rather have pre-parsed JSON than parse HTML yourself.

**ScrapFly.** Publishes a full open-source TripAdvisor scraper and handles the anti-bot bypass and US proxy routing automatically. A good option if you want a battle-tested TripAdvisor-specific reference implementation.

**ScrapingAnt.** Uses headless Chrome with residential proxies; a documented comparison rated it roughly equal to ScraperAPI for TripAdvisor restaurant data extraction. Both were "strongly recommended over attempting raw proxy rotation yourself."

The main advantage ScraperAPI has in this lineup is the pricing model: flat credit-based plans with transparent per-domain costs, the `max_cost` budget guard, credit refunds on failed requests, and credit counts that scale up to 21.5 million per month without moving to custom enterprise quotes until you exceed that.

## Is scraping TripAdvisor legal?

Short version: scraping publicly available data is generally permitted under U.S. law, but the fine print matters. In *hiQ Labs v. LinkedIn*, the Ninth Circuit ruled that scraping public data doesn't violate the Computer Fraud and Abuse Act, and the Supreme Court's *Van Buren* decision narrowed CFAA further. However, hiQ still ended up paying LinkedIn $500,000 for breaching the user agreement — so contract terms carry real consequences even where criminal law doesn't apply, and TripAdvisor's own terms prohibit automated access and data collection.

Practical low-risk rules, per legal analysis of the travel-site scraping landscape: stay on public pages and never script logins or booking flows; avoid collecting personal data like reviewer names and profiles (GDPR exposure in Europe); and check robots.txt — TripAdvisor allows crawling of its search and review pages but lists AI crawlers like GPTBot and ClaudeBot in restricted groups of their own.

## FAQ

**How many reviews can I scrape per TripAdvisor page?**
Each review page shows 10 reviews, accessed via the `-or{offset}-` URL pattern where offset is the page number × 10. A 3-page scrape with the code pattern above yields roughly 30 reviews per hotel.

**Can I scrape TripAdvisor for free?**
Partly. TripAdvisor's official Content API gives 5,000 free calls monthly but only 5 reviews per location. On the scraper side, ScraperAPI offers 1,000 free credits on signup plus a 7-day trial with 5,000 credits — enough to measure your real per-request credit cost on DataDome-protected pages before spending anything.

**Do I need JavaScript rendering for TripAdvisor reviews?**
Yes. Review content is loaded dynamically, so a static request returns HTML without the review text even if the anti-bot wall is handled. All ScraperAPI plans include rendering via the `render=true` parameter — just remember that rendering adds credits per request.

**Why does my scraper work on Airbnb but fail on TripAdvisor?**
Airbnb exposes much of its listing data in embedded JSON-LD that a plain request can read; TripAdvisor wraps everything behind DataDome and requires rendered, proxied requests. They're just built differently.

## Which route should you take?

If you need full review histories from hotels or restaurants — the use case the official API can't cover — ScraperAPI's Hobby plan at $49/month ($44.10 on annual billing) is the natural starting point, with the trial letting you validate your target pages first. If you only need structured search listings for a market analysis, SerpApi's free 250 searches per month might honestly be enough, and the official Content API covers basic location data at zero cost. Business and above are for when you need geo-targeting outside the US and EU, or you're pushing past 100,000 credits a month.

Before you buy anything, do one thing: run your actual TripAdvisor URLs through the trial and check the Domain Cost Estimator to confirm the per-request credit cost — that single number determines whether your project fits in the Hobby tier or needs the Startup plan, and whether the per-1,000-page price justifies annual billing's 10% discount for your expected volume. **👉 Start with ScraperAPI's 7-day trial and check your real costs before committing to a plan.**
