# Delivery Performance and Customer Satisfaction Analysis

*Olist Brazilian e-commerce (about 99K orders, 2016-2018). Python, SQL (DuckDB) and Power BI.*

## Summary

Orders that arrive after the estimated delivery date are far more likely to receive a bad review, and the effect grows with the length of the delay. Before reaching that result, I profiled and cleaned nine relational tables, logged every data-quality decision with its evidence and reasoning, and checked each step against row counts.

**Headline result:** late orders are **6.7x more likely to get a bad review** (62.4% unhappy vs 9.3% for on-time orders).

![Delivery and reviews dashboard](https://github.com/user-attachments/assets/3fc3a62d-5bf5-42a8-a906-4d5e14cbff98)

## Key findings

"Unhappy" means a review score of 2 or less. "Late" means delivered after the estimated date, compared by calendar day.

### 1. Late delivery and unhappy reviews

| Group | Reviews | Unhappy share |
|---|---|---|
| On time | 89,949 | 9.3% |
| Late | 6,410 | 62.4% |
| No delivery date | 2,865 | 77.7% |

- Late orders are 6.5% of all reviews but about 27% of all unhappy reviews.
- The no-delivery-date group is mostly orders that never reached the customer (canceled, unavailable, still in transit), so it is reported separately and not treated as "late".

### 2. The unhappy share rises with the delay

| Delay band | Reviews | Unhappy | Unhappy share |
|---|---|---|---|
| On time | 89,949 | 8,350 | 9.3% |
| 1 to 3 days late | 1,856 | 597 | 32.2% |
| 4 to 7 days late | 1,756 | 1,188 | 67.7% |
| More than 7 days late | 2,798 | 2,215 | 79.2% |

- Compared with on-time orders, the unhappy share is about 3.5x higher at 1 to 3 days late, 7.3x at 4 to 7 days, and 8.5x beyond 7 days. The increase slows at the top.
- Orders more than 7 days late are 2,798 reviews but 2,215 of the 4,000 unhappy reviews on late orders (about 55%).
- Of delivered orders, 93.2% arrived on time and 3.0% were more than 7 days late.

### 3. Late share by review score

| Review score | Share of that score's reviews that were on a late order |
|---|---|
| 1 | 30.1% |
| 2 | 17.6% |
| 3 | 8.5% |
| 4 | 3.4% |
| 5 | 1.8% |

The late share falls steadily as the score rises. The denominators include orders with no delivery date (which cannot be late), so these shares are slightly understated.

## Limitations (please read)

- **Association, not proof of cause.** This analysis shows that late delivery and bad reviews go together. It does not show that lateness caused every bad review. Other factors (product quality, seller behaviour, price) are not controlled for.
- **Only customers who left a review are counted.** Review behaviour is not random, so the percentages describe reviewers, not all customers.
- **"Late" uses calendar days.** Time of day is ignored. A timestamp-based rule would give different counts for some orders.
- **"Unhappy" is my definition** (score of 2 or less). A different cutoff would change the percentages, though not the pattern.
- **Some orders have more than one review.** Counts are reviews, not orders. Distinct `review_id` values (98,410) are also fewer than review rows (99,224), so some review IDs repeat.
- **Small groups.** Some order statuses (for example `approved`, 2 orders) have too few reviews to read anything into their percentages.
- **The estimated delivery date is set by the platform.** A late order is late relative to that promise, not relative to a fixed standard.
- **Not everything was analysed.** A freight-cost analysis (freight vs price, weight and distance) was started but is not finished and is **not part of the findings above**.

## Data

Source: [Brazilian E-Commerce Public Dataset by Olist (Kaggle)](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce). Nine tables:

| Table | Rows (raw) | Columns (raw) |
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

Note: `customers` has one row per order, so state and city percentages are shares of orders, not of people. About 96.6% of rows are distinct `customer_unique_id` values (about 96,096 people), so repeat purchasing is low in this data.

## Method

### Data-quality work

I profiled null counts for every column, then investigated each anomaly and logged it with evidence, a decision and a reason. The full log is in [`docs/data_quality_log.md`](docs/data_quality_log.md). The principle throughout: **flag and keep real records; never invent values.**

| # | Issue | Rows | Decision |
|---|---|---|---|
| 1 | Missing delivery date on orders not yet delivered (shipped, canceled, etc.) | 2,957 | Expected; left as is |
| 2 | Status `delivered` but no delivery date | 8 | Flagged and kept; no date invented |
| 3 | Status `canceled` but a delivery date exists | 6 | Flagged and kept (no cancellation date exists to settle the order of events) |
| 4 | Review title and message mostly null (optional fields) | 88.3% / 58.7% | Labelled `no_title` / `no_comment` |
| 5 | 610 products missing category, name length, description length and photo count | 610 | Kept (they carry 1.32% of item revenue); category set to `unknown`; flagged |
| 6 | Misspelled source columns (`..._lenght`) | 2 | Renamed |
| 7 | Products missing weight and all dimensions | 2 | Left null (physical facts); flagged |
| 8 | Zip prefixes with more than one city spelling | 39 prefixes, 476 rows (0.48%) | Left as is and documented; a wrong merge would be worse than the small effect |
| 9 | Customer delivery timestamp earlier than carrier pickup | 23 | Flagged and kept |
| 10 | Review text that is only whitespace | 9 caught by a first pass, 18 more by a strip check | Converted to `no_comment` |
| 11 | Categories with no English translation | 623 products | Added two missing translations plus `unknown`; lookup is 74 rows |
| 12 | Customer delivery date present but carrier date missing | 1 | Flagged and kept as its own flag |

### Flag columns

Orders: `canceled_with_delivery_date` (6), `delivered_missing_delivery_date` (8), `delivered_before_carrier_pickup` (23), `delivered_missing_carrier_date` (1).
Products: `missing_product_details` (610), `missing_dimensions` (2; one product has both flags).
Flags are stored as `yes` / `no`. The carrier-pickup flag compares full timestamps; a date-only comparison would give 20 instead of 23.

### Checks and mistakes caught along the way

- Null percentage first divided by non-null counts instead of total rows. Fixed to divide by `len(table)`.
- A join check returned 37,325 rows instead of 32,951 because the lookup table had duplicate rows after a cell was re-run. Fixed by dropping duplicates on the Portuguese category name and adding a row-count check.
- The first whitespace fix only caught empty strings and 1 to 2 spaces. A strip-based check found 18 more blank messages, now handled (58,274 `no_comment` messages in total).
- Cleaned versions were not automatically in the table dictionary used for saving. Fixed by writing them back before export.

### Cleaned table sizes

| Table | Rows | Columns |
|---|---|---|
| orders | 99,441 | 13 |
| products | 32,951 | 11 |
| order_reviews | 99,224 | 7 |
| category_name_translations | 74 | 2 |
| customers, sellers, order_items, order_payments, geolocations | unchanged | unchanged |

### Dashboard

Built in Power BI from the cleaned tables.

- **Page 1:** where orders, customers and sellers are concentrated (top 10 states and cities).
- **Page 2:** unhappy share by delivery status, unhappy share by days late, and review counts per delay band.

Every number on page 2 was cross-checked against the SQL results above (for example 9.28%, 62.40%, 32.17%, 67.65% and 79.16%). The charts on page 2 exclude the 2,865 reviews with no delivery date.

To keep the file small enough to upload, the geolocation table and some unused columns and tables were removed from the Power BI file. They are not used by any visual.

## Repository contents

```
notebooks/   cleaning and analysis notebook
docs/        data quality log
dashboard/   Power BI file and screenshots
cleaned_data/ cleaned tables (geolocation not included because of its size)
```

## How to reproduce

1. Open the notebook on Kaggle with the Olist dataset attached (the file paths in the notebook point to `/kaggle/input/...`).
2. Run all cells top to bottom (Restart & Run All).
3. The last cells copy the cleaned tables into a dictionary and save them as CSV files.

Requirements: Python 3, pandas, NumPy, DuckDB.

## What I would do next

- Finish the freight analysis: separate the effects of product price, weight and distance on freight cost, and test how far sellers are located from customers by state.
- Break late delivery down by seller state and product category to see where the problem sits.
- Estimate the effect of improving the worst delays, clearly labelled as an estimate.
- Replace calendar-day lateness with a timestamp-based version and compare.

## Skills shown

Data profiling and cleaning (pandas), SQL joins and aggregation (DuckDB), documenting decisions with evidence, reconciling counts across steps, and dashboarding with DAX in Power BI.
