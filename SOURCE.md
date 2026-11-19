# SOURCE.md — Data Sources

Students: Nevo Alani, Ori Mor

## 1. Approved source — summary

| # | Name | Address | Format | Key / user name | License / terms | Sampling frequency |
|---|---|---|---|---|---|---|
| 1A | Shufersal PriceFull, branch 113 (Deal Rishon LeZion, Golda Meir 1) | `https://prices.shufersal.co.il/FileObject/UpdateCategory?catID=2&storeId=113` | gzipped XML | None | Published by law (Promotion of Competition in the Food Sector Law, 2014) for public access. No license or terms page on the site | Once a day, 05:35 UTC |
| 1B | Shufersal PromoFull, branch 113 | Same, with `catID=4` | gzipped XML | None | Same as 1A | Once a day, 05:35 UTC |
| 1C | Rami Levy PriceFull, branch 043 (Rishon LeZion, Cinema complex) | `https://url.publishedprices.co.il` (file `PriceFull7290058140886-001-043-*.gz`) | gzipped XML | Public user name `RamiLevi`, empty password | Published by law, as 1A. The portal links to no terms of use | Once a day, 05:35 UTC |
| 1D | Rami Levy PromoFull, branch 043 | Same portal (file `PromoFull7290058140886-001-043-*.gz`) | gzipped XML | Same as 1C | Same as 1C | Once a day, 05:35 UTC |
| 1E | Bank of Israel USD/ILS representative rate | `https://edge.boi.gov.il/FusionEdgeServer/sdmx/v2/data/dataflow/BOI.STATISTICS/EXR/1.0/RER_USD_ILS?...&format=csv` | CSV | None | Bank of Israel terms of use: the information is the Bank's property, and copying or publishing it requires prior written consent. We store one public daily value for academic, non-commercial use only | Once a day, 05:35 UTC |

## 2. What we collect

Every day we save the full price file and the full promotions file of one large Shufersal branch
and one large Rami Levy branch in Rishon LeZion, and the Bank of Israel USD/ILS rate. The price files
list every item with its regular price. The promotion files list every active promotion with its
dates, its items and the promotion price. All five files of one run are packed, unchanged, into one
archive: `raw/YYYY-mm-dd/HHMM.tar` (UTC). Details and real samples of each source are in the
source details (1A–1E) below.

## 3. Schema of `data/dataset.csv`

One CSV file with a header row. **One row = one item, in one chain, on one sample date, in one promotion.**
An item with no promotion that day gets one row with empty promotion columns. An item in several
promotions gets one row per promotion. Only items whose barcode appears in **both** branches on that
date are included.

| Column | Meaning | Content / units |
|---|---|---|
| `sample_date` | Date of the sample (name of the `raw/` folder) | `YYYY-mm-dd`, UTC |
| `chain` | Chain | `shufersal` or `ramilevy` |
| `store_id` | Branch number in the chain's files | `113` or `043` (text) |
| `item_code` | Item barcode, the join key between the chains | Digits, kept as text |
| `regular_price` | Regular shelf price | ILS, 2 decimals |
| `price_update_time` | Last time the chain changed the regular price | `YYYY-mm-ddTHH:MMZ`, UTC (converted from Israel time) |
| `promotion_id` | Promotion ID in the chain's file | Text; empty if no promotion |
| `group_id` | Group inside the promotion (some promotions have two groups of items) | Text; empty if no promotion |
| `reward_type` | Type of promotion | `2` percent off, `3` fixed price, `10` multi-buy / gift; empty if no promotion |
| `promo_price` | Price in the promotion | ILS, as published (for `10`, the price of `min_qty` items) |
| `discount_rate` | Discount stated by the chain | Percent, as published |
| `min_qty` | Minimum quantity to get the promotion | Number of units |
| `promo_start` | Start of the promotion | `YYYY-mm-ddTHH:MMZ`, UTC (converted from Israel time) |
| `promo_end` | End of the promotion | `YYYY-mm-ddTHH:MMZ`, UTC (converted from Israel time) |
| `club_id` | Who can use the promotion | `0` = all customers; other numbers = a club |
| `is_coupon` | Whether the promotion needs a coupon | `1` or `0` |
| `usd_ils` | Bank of Israel representative rate on the latest business day up to `sample_date` | ILS per 1 USD, 3 decimals |

## 4. Why these columns

- `regular_price` and `price_update_time` give the price history needed to check if a promotion is
  really cheaper than the price in the ~14 days before it, and if the price rose before it started.
- The promotion columns (`reward_type`, `promo_price`, `discount_rate`, `min_qty`, `promo_start`,
  `promo_end`, `club_id`, `is_coupon`) are what we need to count promotions and measure their depth.
- We keep only `reward_type` 2, 3 and 10. Type 0 rows are mostly coupons whose "promotion price" equals
  the regular price, and types 1 and 12 are rare. At Shufersal, type 10 is also used for gift coupons
  (for example `DiscountedPrice` 0.00), so `is_coupon` and `promo_price` let the analysis separate them.
- Multi-buy promotions such as "2 for 25" are `reward_type` 10 with `min_qty` 2 and `promo_price` 25. The price is for the whole bundle, so the price per unit is `promo_price / min_qty`, and we compare that with `regular_price`. We do not rely on `discount_rate` for multi-buy promotions: Shufersal usually leaves it empty, and at Rami Levy it matched our calculation in only 93% of the rows we checked on 2026-11-16.
- Most multi-buy promotions apply to a group of items ("any 2 of these 11"), and the items in a group can have different regular prices. All the rows of one promotion share `promotion_id`, and `group_id` separates the groups of a promotion, so the group can be rebuilt. Comparing each item alone with the bundle price is therefore an approximation.
- The chains publish times in Israel time. Following the project rule, all times in the dataset are converted to UTC (Israel is UTC+2 in winter), for example `2026-11-01T00:00` in Israel becomes `2026-10-31T22:00Z`.
- `usd_ils` is repeated on every row so the exchange rate can be compared with promotions and regular
  prices, including with a time lag, without a second file.
- We drop `ItemName`, `ManufactureName`, `UnitOfMeasure`, `UnitOfMeasurePrice`, `AllowDiscount`,
  `LastSaleDateTime`, `PromotionDescription` and similar fields. Names differ between chains (we join
  on the barcode), and the other fields do not help answer our questions. Dropping them keeps the file
  under GitHub's 100 MB file limit. They all stay in `raw/`, so we can add them later.
- Only shared barcodes are kept, so both chains are compared on the same products. On 2026-11-16 there
  were 3,513 shared barcodes. The files have no category field, so per-category questions would need
  an extra mapping.

## 5. Translation table (source field → schema column)

| Schema column | Shufersal (1A / 1B) | Rami Levy (1C / 1D) | Bank of Israel (1E) |
|---|---|---|---|
| `sample_date` | `raw/` folder name | `raw/` folder name | — |
| `chain` | constant `shufersal` | constant `ramilevy` | — |
| `store_id` | `StoreID` | `StoreID` | — |
| `item_code` | `ItemCode` | `ItemCode` | — |
| `regular_price` | `ItemPrice` | `ItemPrice` | — |
| `price_update_time` | `PriceUpdateTime` (Israel time → UTC) | `PriceUpdateTime` (Israel time → UTC) | — |
| `promotion_id` | `PromotionID` | `PromotionID` | — |
| `group_id` | `GroupID` | `GroupID` | — |
| `reward_type` | `RewardType` | `RewardType` | — |
| `promo_price` | `DiscountedPrice` | `DiscountedPrice` | — |
| `discount_rate` | `DiscountRate` | `DiscountRate` | — |
| `min_qty` | `MinQty` | `MinQty` | — |
| `promo_start` | `PromotionStartDateTime` (Israel time → UTC) | `PromotionStartDateTime` (Israel time → UTC) | — |
| `promo_end` | `PromotionEndDateTime` (Israel time → UTC) | `PromotionEndDateTime` (Israel time → UTC) | — |
| `club_id` | `ClubID` (number before ` - `, e.g. `0 - כלל הלקוחות` → `0`) | `ClubID` | — |
| `is_coupon` | `AdditionalIsCoupon` | `AdditionalIsCoupon` | — |
| `usd_ils` | — | — | `OBS_VALUE` of the latest `TIME_PERIOD` up to `sample_date` |

## 6. Expected size after one semester

Based on the files of 2026-11-16 and about 74 sample days (late November 2026 to the end of January 2027):

- **`raw/`:** about 2.0 MB a day (Shufersal 0.51 MB, Rami Levy 1.49 MB, exchange rate under 1 KB),
  so about **150 MB**.
- **`data/dataset.csv`:** about 8,500 rows a day (shared barcodes, plus one row per extra promotion),
  so about **630,000 rows** and roughly **70 MB**.
- Together about 220 MB, below the 500 MB limit. If it grows too much, the fallback plan below
  (a basket of 50–100 barcodes) gives about 15,000 rows and under 2 MB.

## Proposal 1 (chosen): Do Shufersal and Rami Levy run their promotions differently?

Following the feedback, we compare two chains instead of one: **Shufersal** (a mainstream chain) and **Rami Levy** (a discount chain). 

### Research questions
1. **Fake promotions.** For each chain, what share of fixed-price (`RewardType=3`), percent-off
   (`RewardType=2`) and multi-buy (`RewardType=10`, e.g. "2 for 25") promotions are really cheaper
   than the item's regular price over the previous ~14 days? Does the regular price rise shortly
   before a promotion starts, or after it ends?
2. **Volume and depth.** Which chain runs more promotions, per day and per category, and how deep
   is the average discount?
3. **Holidays.** Does promotion behavior change around Hanukkah (evening of 2026-12-04 to 2026-12-12),
   the only Israeli holiday that falls inside the semester? Civil holidays such as Black Friday
   (2026-11-27) and New Year's Eve are not expected to bring promotions in Israel, but it will be
   interesting to check.
4. **Economics.** Is there a relation between USD/ILS movements and waves of promotions? Does the
   exchange rate also affect the **regular (base) price** of products and, if so, after how long
   (a lag analysis)?
5. **Interpretation.** A difference between the chains may point to a different business strategy,
   or to a different price sensitivity of their customers.

To answer question 1 we need a price history, so we collect **both** the full price file
(PriceFull) and the full promotions file (PromoFull) of each branch, every day.

### Why we must collect it ourselves
By the Food Price Transparency Law, every chain publishes its current price and promotion files
per branch. Only the **current** files are online. Shufersal shows only the latest file, and the
Rami Levy portal keeps about one day of files. There is no public archive, so a missed day is lost.

### Branches
We compare two large-format branches in the same city, so that the stores have a similar size
and serve a similar area:

| Chain | Store ID | Branch | Address | Items (2026-11-16) | Promotions (2026-11-16) |
|---|---|---|---|---|---|
| Shufersal (sub-chain Shufersal Deal) | `113` | דיל ראשל"צ- גולדה מאיר | Golda Meir 1, Rishon LeZion | 7,201 | 1,408 |
| Rami Levy | `043` | ראשון לציון | Moshe Dayan Blvd (Cinema complex), Rishon LeZion | 13,549 | 2,553 |

We chose them from each chain's `Stores` file. Small city-center branches (for example the Tel
Aviv branches) carry far fewer items, so we used large branches in the center of the country.

### Collection time
Once a day at **05:35 UTC** (07:35 Israel winter time). This is after Shufersal's daily file
(about 03:40 Israel time) and Rami Levy's 06:00 file, and inside the visit window that
www.shufersal.co.il asks for (04:00–08:45 UTC). All times in `raw/` are UTC.

### 1A: Shufersal PriceFull (regular prices)
- **What:** Every item in the branch, with its regular price (`ItemPrice`), unit price and the time of its last price update.
- **URL:** `https://prices.shufersal.co.il/FileObject/UpdateCategory?catID=2&storeId=113`
- **Updates:** Daily, at about 03:40 Israel time.
- **Size (2026-11-16):** 364 KB gzipped (6.2 MB XML), 7,201 items.
- **Key:** No.
- **robots / terms:** prices.shufersal.co.il has no robots.txt (404) and no terms page. The main site
  (www.shufersal.co.il) asks for `Crawl-delay: 10` and visits between 04:00 and 08:45 UTC. We make
  one sample a day inside that window.
- **Note:** The download links are signed and expire after about 30 minutes, so a sample needs one curl
  for the list page and one for the file:
```bash
U=$(curl -s "https://prices.shufersal.co.il/FileObject/UpdateCategory?catID=2&storeId=113" | grep -o 'https://pricesprodpublic[^"]*' | head -1 | sed 's/&amp;/\&/g'); curl -s "$U" -o pricefull.gz
```
- **Sample (2026-11-16, file `PriceFull7290027600007-002-113-20261116-034226.gz`; some fields omitted):**
```xml
<Item>
  <PriceUpdateTime>2025-06-06T05:43:00</PriceUpdateTime>
  <ItemCode>3600524166687</ItemCode>
  <LastSaleDateTime>2026-11-05T14:32:33</LastSaleDateTime>
  <ItemName>אלביב מרכך הידרה הילורונ</ItemName>
  <ManufactureName>לוריאל-אלביב</ManufactureName>
  <Quantity>500.00</Quantity>
  <UnitOfMeasure>100 מיליליטר</UnitOfMeasure>
  <ItemPrice>19.90</ItemPrice>
  <UnitOfMeasurePrice>3.98</UnitOfMeasurePrice>
  <AllowDiscount>1</AllowDiscount>
</Item>
```

Sample data points (2026-11-16, Shufersal branch 113):

| Field | Example value | Meaning |
|---|---|---|
| `ItemCode` | `3600524166687` | Item barcode (join key between chains) |
| `ItemName` | `אלביב מרכך הידרה הילורונ` | Item name |
| `ItemPrice` | `19.90` | Regular price in ILS |
| `UnitOfMeasurePrice` | `3.98` | Price per 100 ml |
| `PriceUpdateTime` | `2025-06-06T05:43:00` | Last change of the price |
| `LastSaleDateTime` | `2026-11-05T14:32:33` | Last time the item was sold |

### 1B: Shufersal PromoFull (promotions)
- **What:** Active promotions: ID, description, start and end dates, club, items and the discounted price or rate.
- **URL:** `https://prices.shufersal.co.il/FileObject/UpdateCategory?catID=4&storeId=113`
- **Updates:** Daily, with the same file time as 1A.
- **Size (2026-11-16):** 145 KB gzipped (5.6 MB XML), 1,408 promotions (1,297 of them started in November).
- **Key:** No.
- **robots / terms:** Same as 1A. It uses the same two-step curl with `catID=4`.
- **Note:** Many coupon rows (`RewardType=0`) list a "discounted price" equal to the regular price, so
  we do not count them as real discounts. Item rows by `RewardType` on 2026-11-16:
  0: 3,604 · 1: 191 · 2: 1,335 · 3: 693 · 10: 2,599 · 12: 263.
- **Sample (2026-11-16, file `PromoFull7290027600007-002-113-20261116-034226.gz`; some fields omitted):**
```xml
<Promotion>
  <PromotionID>4663671</PromotionID>
  <PromotionDescription>17.90 שוקולד טורינו חלב/מריר/לבן 100</PromotionDescription>
  <PromotionStartDateTime>2026-11-01T00:00:00.000</PromotionStartDateTime>
  <PromotionEndDateTime>2026-11-28T02:59:00.000</PromotionEndDateTime>
  <ClubID>0 - כלל הלקוחות</ClubID>
  <PromotionItem>
    <ItemCode>7610008641506</ItemCode>
    <RewardType>3</RewardType>
    <MinQty>1.00</MinQty>
    <DiscountRate>18</DiscountRate>
    <DiscountedPrice>17.90</DiscountedPrice>
  </PromotionItem>
</Promotion>
```

Sample data points (2026-11-16, Shufersal branch 113):

| Field | Example value | Meaning |
|---|---|---|
| `PromotionID` | `4663671` | Promotion ID |
| `PromotionDescription` | `17.90 שוקולד טורינו חלב/מריר/לבן 100` | Promotion text |
| `PromotionStartDateTime` | `2026-11-01T00:00:00.000` | Start of the promotion |
| `PromotionEndDateTime` | `2026-11-28T02:59:00.000` | End of the promotion |
| `ClubID` | `0 - כלל הלקוחות` | Who can use it (0 = all customers) |
| `ItemCode` | `7610008641506` | Barcode of the discounted item |
| `RewardType` | `3` | Type of discount (3 = fixed price) |
| `MinQty` | `1.00` | Minimum quantity |
| `DiscountRate` | `18` | Discount, in percent |
| `DiscountedPrice` | `17.90` | Promotion price in ILS |

### 1C: Rami Levy PriceFull (regular prices)
- **What:** The same kind of file as 1A, for Rami Levy branch 043.
- **Access:** Rami Levy publishes on the shared "PublishedPrices" portal (Cerberus FTP web client)
  at `https://url.publishedprices.co.il`. The portal requires a login, but the chain publishes a
  **public user name, `RamiLevi`, with an empty password**, so there is no secret to store. The steps are:
  1. `GET /login` returns a session cookie (`cftpSID`) and a `csrftoken`.
  2. `POST /login/user` with `username=RamiLevi`, an empty `password` and the `csrftoken`. The answer is a redirect to `/file`.
  3. `POST /file/json/dir` (with the new `csrftoken` from `/file`) returns the list of files as JSON.
  4. `GET /file/d/<file name>` downloads a file, for example `PriceFull7290058140886-001-043-20261116-180011.gz`.
- **Updates:** Four full files a day, at about 00:00, 06:00, 12:00 and 18:00 Israel time (the time
  in the file name is Israel time). We take one a day (the 06:00 file) to match Shufersal.
- **Size (2026-11-16):** 1.13 MB gzipped (11.8 MB XML), 13,549 items.
- **Key:** No (public user name, no password).
- **robots / terms:** The portal has no robots.txt (404). The login page links to no terms of use.
- **Note:** The file uses the same XML standard as Shufersal, with the same item fields. The
  chain's `Stores` file is UTF-16, but the price and promotion files are UTF-8.
- **Sample (2026-11-16, file `PriceFull7290058140886-001-043-20261116-180011.gz`; some fields omitted):**
```xml
<Item>
  <PriceUpdateTime>2025-11-09T12:16:02.000</PriceUpdateTime>
  <ItemCode>7290002057383</ItemCode>
  <LastSaleDateTime>2026-11-16T12:55:34.000</LastSaleDateTime>
  <ItemName>מלפפון יבנה במלח קטנ</ItemName>
  <ManufactureName>קבוצת יבנה</ManufactureName>
  <Quantity>320.00</Quantity>
  <UnitOfMeasure>100 גרם</UnitOfMeasure>
  <ItemPrice>12.90</ItemPrice>
  <UnitOfMeasurePrice>4.03</UnitOfMeasurePrice>
  <AllowDiscount>1</AllowDiscount>
</Item>
```

Sample data points (2026-11-16, Rami Levy branch 043):

| Field | Example value | Meaning |
|---|---|---|
| `ItemCode` | `7290002057383` | Item barcode (join key between chains) |
| `ItemName` | `מלפפון יבנה במלח קטנ` | Item name |
| `ItemPrice` | `12.90` | Regular price in ILS |
| `UnitOfMeasurePrice` | `4.03` | Price per 100 g |
| `PriceUpdateTime` | `2025-11-09T12:16:02.000` | Last change of the price |
| `LastSaleDateTime` | `2026-11-16T12:55:34.000` | Last time the item was sold |

### 1D: Rami Levy PromoFull (promotions)
- **What:** The same kind of file as 1B, for Rami Levy branch 043.
- **Access:** Same portal and steps as 1C, with file names starting with `PromoFull`.
- **Updates:** Four times a day, like 1C. We take the 06:00 file.
- **Size (2026-11-16):** 364 KB gzipped (9.0 MB XML), 2,553 promotions (1,329 of them started in November).
- **Key:** No.
- **robots / terms:** Same as 1C.
- **Note:** Small differences from Shufersal: `ClubID` is a bare number (`0`) instead of
  `0 - כלל הלקוחות`, and empty fields are written as `<Field/>`. Item rows by `RewardType` on 2026-11-16:
  0: 767 · 1: 251 · 2: 647 · 3: 6,692 · 10: 5,302.
- **Sample (2026-11-16, file `PromoFull7290058140886-001-043-20261116-180102.gz`; some fields omitted):**
```xml
<Promotion>
  <PromotionID>0001431714</PromotionID>
  <PromotionDescription>חטיף דגני בוקר נסקוויק 6*25גרם ב5.90</PromotionDescription>
  <PromotionStartDateTime>2026-11-08T00:00:00.000</PromotionStartDateTime>
  <PromotionEndDateTime>2027-01-02T23:59:00.000</PromotionEndDateTime>
  <ClubID>0</ClubID>
  <PromotionItem>
    <ItemCode>5900020020741</ItemCode>
    <RewardType>3</RewardType>
    <MinQty>1</MinQty>
    <DiscountRate>62.89</DiscountRate>
    <DiscountedPrice>5.90</DiscountedPrice>
  </PromotionItem>
</Promotion>
```

Sample data points (2026-11-16, Rami Levy branch 043):

| Field | Example value | Meaning |
|---|---|---|
| `PromotionID` | `0001431714` | Promotion ID |
| `PromotionDescription` | `חטיף דגני בוקר נסקוויק 6*25גרם ב5.90` | Promotion text |
| `PromotionStartDateTime` | `2026-11-08T00:00:00.000` | Start of the promotion |
| `PromotionEndDateTime` | `2027-01-02T23:59:00.000` | End of the promotion |
| `ClubID` | `0` | Who can use it (0 = all customers) |
| `ItemCode` | `5900020020741` | Barcode of the discounted item |
| `RewardType` | `3` | Type of discount (3 = fixed price) |
| `MinQty` | `1` | Minimum quantity |
| `DiscountRate` | `62.89` | Discount, in percent |
| `DiscountedPrice` | `5.90` | Promotion price in ILS |

### 1E: Bank of Israel USD/ILS representative rate (supplementary context)
- **What:** The daily representative exchange rate of the US dollar in shekels (series `RER_USD_ILS`).
- **URL:** `https://edge.boi.gov.il/FusionEdgeServer/sdmx/v2/data/dataflow/BOI.STATISTICS/EXR/1.0/RER_USD_ILS?startPeriod=YYYY-MM-DD&endPeriod=YYYY-MM-DD&format=csv`
  (the older `.../ws/public/sdmxapi/rest/data/EXR/RER_USD_ILS` URL also worked on 2026-11-16, and returns SDMX-XML).
- **Updates:** Once a business day.
- **Size:** Under 1 KB per request.
- **Key:** No.
- **robots / terms:** edge.boi.gov.il has no robots.txt (404).
- **Note:** Unlike the supermarket files, this series **has a public history**, so it does not meet
  the "lost week" rule. It is only a context source for question 4 (promotions, and the lagged effect on regular prices). We collect it daily with the
  other files so that all our data comes from the same run.
- **Sample (2026-11-16):**
```csv
SERIES_CODE,FREQ,BASE_CURRENCY,COUNTER_CURRENCY,UNIT_MEASURE,DATA_TYPE,DATA_SOURCE,TIME_COLLECT,CONF_STATUS,PUB_WEBSITE,UNIT_MULT,COMMENTS,TIME_PERIOD,OBS_VALUE,RELEASE_STATUS
RER_USD_ILS,D,USD,ILS,ILS,OF00,BOI_MRKT,V,F,Y,0,,2026-11-13,3.199,YP
RER_USD_ILS,D,USD,ILS,ILS,OF00,BOI_MRKT,V,F,Y,0,,2026-11-16,3.223,YP
```

Sample data points (2026-11-16):

| Field | Example value | Meaning |
|---|---|---|
| `SERIES_CODE` | `RER_USD_ILS` | Series: representative USD/ILS rate |
| `TIME_PERIOD` | `2026-11-16` | Date of the rate |
| `OBS_VALUE` | `3.223` | ILS per 1 USD |
| `UNIT_MEASURE` | `ILS` | Unit |
| `RELEASE_STATUS` | `YP` | Release status flag from the Bank of Israel |

### Joining the two chains
The same product does not always have the same name in both chains. We measure the promotions
themselves rather than compare product prices, so this matters little. When we need to match
products, we join on the barcode (`ItemCode`). On 2026-11-16, 3,513 barcodes appeared in both branches.

### Fallback plan
Each file holds thousands of items. If the full analysis turns out too heavy, we will narrow it to
a fixed basket of about 50–100 barcodes that appear in both branches. We will still save the full
files in `raw/`, so we can widen the analysis again later.

### Storage
Measured on 2026-11-16: Shufersal about 0.51 MB a day and Rami Levy about 1.49 MB a day, gzipped
(the exchange rate is under 1 KB). That is about **2.0 MB a day**, or about **150 MB** from late
November 2026 to the end of January 2027. This is below the 500 MB limit. The files are saved
gzipped, as published.

## Proposal 2: Ben Gurion airport delays — approved, not pursued

- **Question:** Which airlines, destinations and hours of the day show the largest gap between scheduled and actual times at TLV?
- **2A:** Israel Airports Authority flight board on data.gov.il:
  `https://data.gov.il/api/3/action/datastore_search?resource_id=e83f763b-b7d7-479e-b172-ae981ddc6de5&limit=5000`
- **2B:** OpenSky Network live ADS-B: `https://opensky-network.org/api/states/all?lamin=31.9&lomin=34.8&lamax=32.1&lomax=35.0`
- **Risk:** data.gov.il's robots.txt has `Disallow: /api/`, and OpenSky's anonymous quota is limited.

## Ranking
1. **Proposal 1 (chosen): Shufersal vs Rami Levy promotions.** Neither chain keeps an archive, so it fits
   the lost-week rule best. Both chains use the same file standard and join on `ItemCode`. No key is needed.
   Its risks are Shufersal's two-step download and the Rami Levy portal login.
2. **Proposal 2: Ben Gurion delays.** Approved, but not pursued.

## AI Use Disclosure
- **Tool:** Claude (Anthropic), in the Claude desktop app (Claude Code).
- **How it was used:**
  1. It suggested candidate sources. We chose the topics and the sources.
  2. It ran test `curl` requests to check each source: whether it works, robots.txt, size and whether an archive exists.
  3. It wrote the `curl` commands. We ran them on our own VM to collect the samples.
  4. It helped format and organize this document (SOURCE.md) in Markdown. We reviewed and approved the content.
- **Update after the feedback (2026-11-16):**
  1. We chose the second chain (Rami Levy), the exchange rate source and the branch criteria
     (large branches in the same city in the center). Claude searched the chains' `Stores` files and
     measured candidate branches; we chose Rishon LeZion.
  2. Claude ran `curl` tests to verify access to the Rami Levy portal and the Bank of Israel API, and
     to measure file sizes, item and promotion counts, and robots.txt.
  3. Claude organized the text of this document and arranged it in a clean Markdown format. We reviewed and approved it.
- **Update for exercise 3 (2026-11-19):**
  1. We chose the schema of `dataset.csv`: shared barcodes only, and one row per item, chain, date and promotion.
     We decided to include the Bank of Israel rate.
  2. Claude measured row counts and sizes for each schema option on our 2026-11-16 files, checked the Bank of
     Israel terms of use, and organized the six new sections in Markdown. We reviewed and approved them.
- **Initial prompt(s):**
  > Organize this information into a clean Markdown format, following the base structure that already
  > exists in SOURCE.md. Run curl commands against the sources and put examples of the data points in
  > tables. Place each table in the appropriate location, after the written details about the data source.
