# SOURCE.md — Data Source Proposals

Students: Nevo Alani, Ori Mor

## Proposal 1 (primary): Are Shufersal "deals" really cheaper?

**Question:** When a product goes on promotion at Shufersal, is the promo price really lower than
the regular price before it? Does the regular price rise just before a promotion starts or after it ends?

**Why collect ourselves:** By the Food Price Transparency Law, Shufersal publishes daily price and
promotion files per branch, but only the **current** file is online. There is no public archive, so a missed week is lost.

### 1A: Shufersal PriceFull (regular prices)
- **What:** Every item in a branch, with its regular price (`ItemPrice`) and the time of its last price update.
- **URL:** `https://prices.shufersal.co.il/FileObject/UpdateCategory?catID=2&storeId=1`
- **Updates:** Daily, at about 03:00 Israel time.
- **Size:** About 336 KB gzipped (5.6 MB XML), about 6,500 items.
- **Key:** No.
- **robots / terms:** The prices site has no robots.txt or terms page. The main site asks for
  `Crawl-delay: 10` and visits between 04:00–08:45 UTC, so we take one sample a day at about 07:00.
- **Note:** The download links are signed and expire after about 30 minutes, so a sample needs one curl for the list page and one for the file:
```bash
U=$(curl -s "https://prices.shufersal.co.il/FileObject/UpdateCategory?catID=2&storeId=1" | grep -o 'https://pricesprodpublic[^"]*' | head -1 | sed 's/&amp;/\&/g'); curl -s "$U" -o pricefull.gz
```
- **Sample (2026-11-05):**
```xml
<Item>
  <PriceUpdateTime>2025-12-28T14:15:00</PriceUpdateTime>
  <ItemCode>10900302814</ItemCode>
  <ItemName>ניילון נצמד 30ס"מ30* מטר</ItemName>
  <ManufactureName>ריינולדס</ManufactureName>
  <Quantity>30.00</Quantity>
  <ItemPrice>10.00</ItemPrice>
  <UnitOfMeasurePrice>0.33</UnitOfMeasurePrice>
</Item>
```

### 1B: Shufersal PromoFull (promotions)
- **What:** Active promotions: ID, description, start and end dates, club, items and discounted price or rate.
- **URL:** `https://prices.shufersal.co.il/FileObject/UpdateCategory?catID=4&storeId=1`
- **Updates:** Daily.
- **Size:** About 224 KB gzipped (10.7 MB XML), 1,278 promotions (1,150 of them started in November).
- **Key:** No.
- **robots / terms:** Same as 1A, and it uses the same two-step curl with `catID=4`.
- **Note:** Many coupon rows (`RewardType=0`) list a "discounted price" equal to the regular price.
  We will compare only fixed-price (`RewardType=3`) and percent-off (`RewardType=2`) promotions.
- **Sample (2026-11-05):**
```xml
<Promotion>
  <PromotionID>667573</PromotionID>
  <PromotionDescription>קופון דבש לחיץ 500 גרם - מתנה</PromotionDescription>
  <PromotionStartDateTime>2014-07-16T00:00:00.000</PromotionStartDateTime>
  <PromotionEndDateTime>2031-01-01T02:59:00.000</PromotionEndDateTime>
  <ClubID>0 - כלל הלקוחות</ClubID>
  <PromotionItem>
    <ItemCode>7296073173045</ItemCode>
    <RewardType>2</RewardType>
    <DiscountRate>100</DiscountRate>
    <DiscountedPrice>0.00</DiscountedPrice>
  </PromotionItem>
</Promotion>
```

**Storage:** About 560 KB a day per branch, which is about 17 MB a month. That is far below 500 MB.

## Proposal 2 (secondary): Which airlines and hours at Ben Gurion are delayed most?

**Question:** Which airlines, destinations and hours of the day show the largest gap between
scheduled and actual times at TLV?

**Why collect ourselves:** The board shows only about 1 day back. Final statuses disappear soon after a flight leaves or lands.

### 2A: Israel Airports Authority flight board (data.gov.il)
- **What:** All TLV flights: airline, flight number, scheduled time (`CHSTOL`), updated time (`CHPTOL`), destination and status.
- **URL:** `https://data.gov.il/api/3/action/datastore_search?resource_id=e83f763b-b7d7-479e-b172-ae981ddc6de5&limit=5000`
- **Updates:** Several times an hour.
- **Size:** About 670 KB JSON, about 1,880 flights.
- **Key:** No.
- **robots / terms:** robots.txt has `Disallow: /api/`, and the CSV link is blocked to curl (403). The CKAN API is the
  portal's documented access method. We would poll it once every 30–60 minutes and flag this as a risk.
- **Sample (2026-11-05):**
```json
{"CHOPER":"IZ","CHFLTN":"417","CHOPERD":"ARKIA  ISRAELI  AIRLINES",
 "CHSTOL":"2026-11-04T15:20:00","CHPTOL":"2026-11-04T15:53:00","CHAORD":"D",
 "CHLOC1D":"TBILISI","CHTERM":1,"CHRMINE":"DEPARTED"}
```

### 2B: OpenSky Network (live ADS-B)
- **What:** Live aircraft positions in a box around TLV. `on_ground`, altitude and speed show the real takeoff and landing.
- **URL:** `https://opensky-network.org/api/states/all?lamin=31.9&lomin=34.8&lamax=32.1&lomax=35.0`
- **Updates:** Every 10 seconds.
- **Size:** About 1 KB per request.
- **Key:** No (anonymous daily quota).
- **robots / terms:** robots.txt returns 403. The terms allow non-commercial and research use. History requires a research account.
- **Sample (2026-11-05):**
```json
{"time":1793887057,"states":[
 ["39ceb2","TVF45QT ","France",1793887036,1793887040,34.8877,32.0034,22.86,false,61.32,300.22,-0.98,null,60.96,"3746",false,0]]}
```

## Ranking
1. **Shufersal (Proposal 1):** It has no archive at all, so it fits the lost-week rule best. Both files join on `ItemCode`, they need no key, and they are small. Its risk is the two-step download.
2. **Ben Gurion (Proposal 2):** A clean keyless JSON source, but data.gov.il's robots.txt disallows `/api/` and OpenSky's quota is limited.

## AI Use Disclosure
- **Tool:** Claude (Anthropic), in the Claude desktop app (Claude Code).
- **How it was used:**
  1. It suggested candidate sources. We chose the topics and the sources.
  2. It ran test `curl` requests to check each source: whether it works, robots.txt, size and whether an archive exists.
  3. It wrote the `curl` commands. We ran them on our own VM to collect the samples.
  4. It helped format and organize this document (SOURCE.md) in Markdown. We reviewed and approved the content.
