# Data quality log: Olist e-commerce dataset

*Last updated: 18 October 2026*

This log records every data problem found while profiling the 9 Olist tables, the evidence for it, and the decision taken. No rows were deleted: faulty rows were flagged and kept, and every table keeps its raw row count (the translation lookup grows by 3 rows by design).

**Status:** all 12 issues are decided and applied in the cleaned tables. Open follow-up work is listed in section 10.

## 1. Raw tables

| Table | Rows | Columns |
|---|---|---|
| customers | 99,441 | 5 |
| sellers | 3,095 | 4 |
| order_reviews | 99,224 | 7 |
| order_items | 112,650 | 7 |
| products | 32,951 | 9 |
| geolocations | 1,000,163 | 5 |
| category_name_translations | 71 | 2 |
| orders | 99,441 | 8 |
| order_payments | 103,886 | 5 |

## 2. Null profile (null % = nulls / total rows of the table)

| Table | Column | Nulls | Null % |
|---|---|---|---|
| order_reviews | review_comment_title | 87,656 | 88.34 |
| order_reviews | review_comment_message | 58,247 | 58.70 |
| products | product_category_name, product_name_lenght, product_description_lenght, product_photos_qty | 610 each | 1.85 |
| products | product_weight_g, product_length_cm, product_height_cm, product_width_cm | 2 each | 0.006 |
| orders | order_approved_at | 160 | 0.16 |
| orders | order_delivered_carrier_date | 1,783 | 1.79 |
| orders | order_delivered_customer_date | 2,965 | 2.98 |

All other columns have no nulls.

## 3. Issues and decisions

| # | Issue | Rows | Evidence | Decision |
|---|---|---|---|---|
| 1 | Missing delivery date on orders that have not reached the customer | 2,957 | By status: shipped 1,107, canceled 619, unavailable 609, invoiced 314, processing 301, created 5, approved 2. Together with issue 2 they make the 2,965 nulls | Expected. Left as is |
| 2 | Orders with status `delivered` but no delivery date | 8 | 7 have a carrier pickup date, 1 has none. All 8 have a review with a score (seven 5-star, one 1-star whose comment says a different brand arrived). The reviews indicate the orders arrived and the timestamp is missing (indicated, not proven) | Flag `delivered_missing_delivery_date` and keep. Dropping would lose valid orders and reviews. No made-up dates |
| 3 | Orders with status `canceled` but a delivery date | 6 | Five of the six were bought in Oct 2016. No cancellation date exists, so the order of events cannot be known | Flag `canceled_with_delivery_date` and keep |
| 4 | Review comment title and message mostly null | title 88.34%, message 58.70% | Combinations before cleaning (add up to 99,224): both present 9,839; title only 1,729; message only 31,138; neither 56,518. All four cases occur, so the fields are optional | Fill with a clear label, not fake text: `no_title`, `no_comment`. All rows kept |
| 5 | Four product columns with exactly 610 nulls | 610 products | The same 610 products miss category, name length, description length and photo count. All 610 were sold: 1,603 order lines and 179,535.28 item revenue (1.32% of all item revenue) | Keep. Fill category with `unknown` and flag `missing_product_details`. Dropping would delete real sales |
| 6 | Misspelled source column names | 2 columns | `product_name_lenght`, `product_description_lenght` come from the original file | Renamed to `product_name_length`, `product_description_length` |
| 7 | Products with missing weight and size | 2 products | `5eb564652db742ff8f28759cd8d2652a` has every column null except id (it is one of the 610). `09ff539a621711667c43eba6a3bd8466` has a category and descriptions but no weight or sizes. Both were sold | Weight and size are physical facts and are not invented: left null and flagged `missing_dimensions`. One product carries both product flags |
| 8 | Zip prefixes with more than one city name in `customers` | 39 prefixes, 476 rows (0.48%) | Mostly spelling variants (arraial d ajuda / arraial d'ajuda, bataipora / bataypora, brasopolis / brazopolis, embu / embu das artes, piumhii). A few are different but overlapping places (brasilia / ceilandia; abrantes / camacari may be a district of Camacari, not verified). Only the first 20 distinct pairs were reviewed by hand | Left as is and documented. No city share can change by more than about 0.5 percentage points. Fixing needs a hand-checked lookup, and a wrong merge would silently damage the data |
| 9 | Customer delivery date earlier than carrier pickup date | 23 (0.02%) | Full timestamp comparison. Gap sizes in calendar days: 0 to 1 day 10 rows, 2 to 3 days 5 rows, 5 to 16 days 8 rows; median 2. A calendar-date comparison finds only 20, because 3 rows are on the same day. At least one date is wrong in each row | Flag `delivered_before_carrier_pickup` and keep. The orders have real items and payments. No overlap with the canceled flag |
| 10 | Review texts that look empty but are not null (only whitespace) | 27 messages, 2 titles | 9 messages had one or two spaces; a strip-based check found 18 more (whitespace the first check did not cover, such as three or more spaces, tabs or line breaks; the exact characters were not itemised) | Treated as missing and given the same labels as issue 4. After cleaning, the strip check returns 0 for both columns |
| 11 | Categories that become `None` after joining the translation table | 13 products plus the 610 | `pc_gamer` (3 products) and `portateis_cozinha_e_preparadores_de_alimentos` (10) have no English translation. Products have 74 distinct categories including `unknown` (73 real) against 71 in the translation table | Added the two translations by hand and `unknown` -> `unknown` to make a 74-row lookup (duplicates dropped on the Portuguese name). The join gives exactly 32,951 rows and no product is left without an English category |
| 12 | Delivery date present but no carrier pickup date | 1 | Found when the carrier condition also caught missing carrier dates: 24 combined minus 23 | Flag `delivered_missing_carrier_date` and keep |

## 4. Columns added in the cleaned tables

All flags are stored as `yes` / `no` text, one column per problem, so a row can carry more than one flag.

| Table | Column | Rows flagged |
|---|---|---|
| orders | `canceled_with_delivery_date` | 6 |
| orders | `delivered_missing_delivery_date` | 8 |
| orders | `delivered_before_carrier_pickup` | 23 |
| orders | `delivered_missing_carrier_date` | 1 |
| orders | `delivery_status` (calendar-day rule) | not late 89,941; late 6,535; data not available 2,965 |
| products | `missing_product_details` | 610 |
| products | `missing_dimensions` | 2 (one product also has `missing_product_details`) |

Label counts in `order_reviews`: `no_title` 87,658 (87,656 missing + 2 blank); `no_comment` 58,274 (58,247 missing + 27 blank).

Order of steps matters: flags were built before missing categories were filled, so the flag conditions still match. Date columns were converted to datetime before any date comparison.

## 5. Reconciliation: row counts raw vs cleaned

| Table | Raw rows | Cleaned rows | Raw columns | Cleaned columns |
|---|---|---|---|---|
| customers | 99,441 | 99,441 | 5 | 5 |
| sellers | 3,095 | 3,095 | 4 | 4 |
| order_reviews | 99,224 | 99,224 | 7 | 7 |
| order_items | 112,650 | 112,650 | 7 | 7 |
| products | 32,951 | 32,951 | 9 | 11 |
| geolocations | 1,000,163 | 1,000,163 | 5 | 5 |
| category_name_translations | 71 | 74 | 2 | 2 |
| orders | 99,441 | 99,441 | 8 | 13 |
| order_payments | 103,886 | 103,886 | 5 | 5 |

The shapes above were printed from the save step and match the expected values. `geolocations` is unchanged and is not used in any finding.

## 6. Findings

Definitions: **late** = delivery date after the estimated date, compared by calendar day. **Unhappy** = review score 2 or less. Counts are reviews, not orders.

### Finding 1: late orders get far more bad reviews

| Group | Reviews | Unhappy share |
|---|---|---|
| On time | 89,949 | 9.3% |
| Late | 6,410 | 62.4% |
| No delivery date | 2,865 | 77.7% |

- The groups add up to 99,224 reviews. Unhappy reviews: 8,350 + 4,000 + 2,225 = 14,575, so the overall unhappy share is 14.7% (matches 11,424 one-star + 3,151 two-star reviews).
- Late orders are about **6.7 times** as likely to get a bad review as on-time orders (62.4 / 9.28).
- Late orders are 6.5% of reviews but about 27% of unhappy reviews.
- Cross-check from the other side: the share of each score's reviews that sit on a late order falls from 30.1% (1 star) to 17.6%, 8.5%, 3.4% and 1.8% (5 stars). These shares are slightly understated because the denominators include orders with no delivery date, which cannot be late.
- The 2,865 reviews with no delivery date are on orders with no recorded delivery date. Their `order_status` has not been broken down yet (see section 10); 8 of them are on orders marked `delivered` (issue 2).

### Finding 2: the longer the delay, the worse

| Band | Reviews | Unhappy | Unhappy share |
|---|---|---|---|
| On time | 89,949 | 8,350 | 9.3% |
| 1 to 3 days late | 1,856 | 597 | 32.2% |
| 4 to 7 days late | 1,756 | 1,188 | 67.7% |
| More than 7 days late | 2,798 | 2,215 | 79.2% |

- Even 1 to 3 days late is 3.5 times the on-time rate; the increase slows beyond 7 days.
- About 44% of late reviews are more than 7 days late, and they account for about 55% of the unhappy reviews on late orders.
- Orders with no delivery date are excluded. The four bands add up to 96,359 reviews (99,224 minus 2,865).
- Among delivered orders (96,476): 93.2% on time, 1.9% 1 to 3 days late, 1.9% 4 to 7 days, 3.0% more than 7 days.

### Geography (shares of orders or sellers, not of people)

- `customers` has one row per order. Distinct `customer_unique_id` values are 96.64% of rows (about 96,096 people), so only about 3.4% of rows are repeat purchases. The data covers roughly 2016 to 2018, so this says little about long-term loyalty.
- Customers by state (share of orders): SP 42.0%, RJ 12.9%, MG 11.7%, RS 5.5%, PR 5.1%. Top 3 = 66.6%, top 5 = 77.2%.
- Customers by city: sao paulo 15.6%, rio de janeiro 6.9%, belo horizonte 2.8%. Top 10 cities = about 35%.
- Zip prefixes: about 14,994 distinct. The top prefix (22790, Rio) has 142 rows, about 21 times an even split. Sao Paulo city is spread over 2,843 prefixes (largest 36 rows); Rio over 373 (largest 142).
- Sellers (3,095 rows): SP 59.7%, PR 11.3%, MG 7.9%, SC 6.1%, RJ 5.5%, RS 4.2%. Sao Paulo city holds about 22% of sellers, roughly 5.5 times the next city. These seller figures were calculated with SQL on the `sellers` table (share of sellers by state and by city).

## 7. Dashboard cross-check (Power BI vs SQL)

The Power BI report was built on the cleaned tables and its numbers were compared with sections 5 and 6.

| Measure | Power BI | This log |
|---|---|---|
| Orders (count) | 99,441 | 99,441 |
| Reviews (count) | 99,224 | 99,224 |
| Unhappy %, on time | 9.28% | 9.3% |
| Unhappy %, late | 62.40% | 62.4% |
| Unhappy %, no delivery date | 77.66% | 77.7% |
| Unhappy %, 1 to 3 days late | 32.17% | 32.2% |
| Unhappy %, 4 to 7 days late | 67.65% | 67.7% |
| Unhappy %, more than 7 days late | 79.16% | 79.2% |
| Reviews per band (on time / 1-3 / 4-7 / over 7 / no date) | 89,949 / 1,856 / 1,756 / 2,798 / 2,865 | same |

The delay-band chart and the late-vs-not-late chart on the dashboard exclude the 2,865 reviews with no delivery date, and the page says so. The dashboard also shows an average review score of 4.09 (not checked against SQL).

## 8. Limits

- The findings show association, not cause. A late order may differ from an on-time one in other ways (product, seller, route).
- Only customers who left a review are counted, and review behaviour is not random.
- `review_id` is not unique: a Power BI card showed about 98.4K distinct values against 99,224 rows. This suggests duplicated review IDs and possibly orders with more than one review, but it has not been checked with SQL, so the findings count review rows as they are.
- Calendar-day comparison: an order delivered on the estimated day counts as on time, and time of day is ignored.
- "Unhappy" (score 2 or less) is a chosen cutoff; another cutoff would change the percentages.
- Small groups (for example the 2 `approved` orders) give percentages that mean little.
- Geography shares count orders (or sellers), not unique customers.
- City names keep their spelling variants (issue 8).

## 9. Corrections made along the way

- The null percentage first divided by non-null rows, which gave values above 100% for review titles. Fixed to divide by all rows of the table.
- The carrier flag first gave 20 (date-only comparison), then 24 (a second condition was mixed in). Final: 23 and 1, as separate columns using full timestamps.
- Joining products to the translation lookup gave 37,325 rows instead of 32,951 after a cell was run repeatedly and duplicated the added rows. Duplicates are now dropped on the Portuguese name and the join gives 32,951.
- Blank review messages: 9 found first with a narrow check, 27 after a strip-based check. The first fix also wrote into a misspelled extra column (`review_comment_messgae`), which was removed.
- One check cell compared the stripped text with `'yes'` instead of an empty string, so it always returned 0. It was replaced with the correct check, which showed 18 blank messages still present.
- The `tables` dictionary used for saving still held the raw versions (orders 12 columns, products 9, lookup 71 rows) because the cleaned tables lived in separate variables. They were written back before export.
- Flag columns were first named `canceled_but_has_date`, `delivered_but_has_nodate`, `carrier_date_before_delivered_date` and `missed_category_name`. They were renamed to the names in section 4, partly because the carrier name read as the opposite of what it flags.
- Power BI cards first showed 98.67K orders and 98.41K reviews because they used distinct counts. Switching to Count gave 99,441 and 99,224.
- An early freight query filtered statuses with `!= 'canceled' OR != 'unavailable'`. A row cannot equal both, so the filter removed nothing; it was changed to `NOT IN ('canceled', 'unavailable')`.
- An early claim that "no city dominates" was wrong: Sao Paulo city is 15.6% of orders.

## 10. Open items

- Break down `order_status` for the 23 orders in issue 9 and for the 2,865 reviews on orders with no delivery date.
- Count duplicated `review_id` values and orders with more than one review, and check whether the findings change when each order counts once.
- Freight analysis (not finished): the notebook query now filters statuses with `NOT IN ('canceled', 'unavailable')`. Next, add weight and volume, compare same-state and cross-state routes, and measure the share of each state's order lines served by in-state sellers.
- Late share by seller state and by product category.
- Put the cleaning steps into one script with automated row-count and flag-count checks.
- A PostgreSQL star schema is planned but not built.
