# Delivery Performance and Customer Satisfaction Analysis

*Olist Brazilian e-commerce (about 99K orders, roughly 2016 to 2018). Python, SQL (DuckDB) and Power BI.*

## Summary

Orders that arrive after the estimated delivery date are far more likely to get a bad review, and the effect grows with the length of the delay. Before reaching that result, I profiled and cleaned nine related tables, logged every data-quality decision with its evidence and reasoning, and checked each step against row counts.

**Headline result:** late orders are **6.7 times as likely to get a bad review** (62.4% unhappy against 9.3% for on-time orders).

![Delivery dashboard](https://github.com/user-attachments/assets/7dfadf04-011b-4ee5-b1f6-09530d3b7130)

## Key findings

"Unhappy" means a review score of 2 or less. "Late" means delivered after the estimated date, compared by calendar day.

### 1. Late delivery and unhappy reviews

| Group | Reviews | Unhappy share |
|---|---|---|
| On time | 89,949 | 9.3% |
| Late | 6,410 | 62.4% |
| No delivery date | 2,865 | 77.7% |

- Late orders are 6.5% of all reviews but about 27% of all unhappy reviews.
- Of the 2,965 orders with no delivery date, 2,957 have a status other than `delivered` (shipped, canceled, unavailable, invoiced, processing, created, approved). This group is reported separately and is not treated as "late".
- The overall unhappy share is 14.7% (14,575 of 99,224 reviews).

### 2. The unhappy share rises with the delay

| Delay band | Reviews | Unhappy | Unhappy share |
|---|---|---|---|
| On time | 89,949 | 8,350 | 9.3% |
| 1 to 3 days late | 1,856 | 597 | 32.2% |
| 4 to 7 days late | 1,756 | 1,188 | 67.7% |
| More than 7 days late | 2,798 | 2,215 | 79.2% |

- Compared with on-time orders, the unhappy share is about 3.5 times as high at 1 to 3 days late, 7.3 times at 4 to 7 days, and 8.5 times beyond 7 days. The increase slows at the top.
- Orders more than 7 days late are 2,798 reviews but 2,215 of the 4,000 unhappy reviews on late orders (about 55%).
- Of delivered orders, 93.2% arrived on time and 3.0% were more than 7 days late. Orders with no delivery date are not part of this table.

### 3. Late share by review score

| Review score | Share of that score's reviews on a late order |
|---|---|
| 1 | 30.1% |
| 2 | 17.6% |
| 3 | 8.5% |
| 4 | 3.4% |
| 5 | 1.8% |

The late share falls steadily as the score rises. The denominators include orders with no delivery date, which cannot be late, so these shares are slightly understated.

### Context: who buys and who sells

These shares count orders and sellers, not people.

- About 42% of orders come from the state of São Paulo, and the top 5 states make up 77%. São Paulo city alone is 15.6% of orders.
- Of the 3,095 sellers, 59.7% are in São Paulo state and the top 6 states hold 94.7%.
- About 96.6% of customer rows have a different `customer_unique_id`, so only about 3.4% of rows are repeat purchases.

## Limitations (please read)

- **Association, not proof of cause.** Late delivery and bad reviews go together, but this analysis does not show that lateness caused every bad review. Product quality, seller behaviour and price are not controlled for.
- **Only customers who left a review are counted.** Review behaviour is not random, so the percentages describe reviewers, not all customers.
- **"Late" uses calendar days.** Time of day is ignored; an order delivered on the estimated day counts as on time.
- **"Unhappy" is my definition** (score of 2 or less). A different cutoff would change the percentages, not the pattern.
- **Duplicate reviews are not resolved.** A Power BI card showed about 98.4K distinct `review_id` values against 99,224 review rows, so some review IDs probably repeat and some orders may have more than one review. I have not yet confirmed this with SQL, so all counts are review rows as they are.
- **Small groups.** Some order statuses (for example `approved`, 2 orders) have too few records to read anything into their percentages.
- **The estimated delivery date is set by the platform.** A late order is late relative to that promise.
- **City names keep their spelling variants** (see issue 8 below).
- **Not everything was analysed.** A freight-cost analysis was started but is not finished and is **not part of the findings above**.

## Data

Source: [Brazilian E-Commerce Public Dataset by Olist (Kaggle)](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce). See the dataset page for its license. The raw files are not stored in this repository.

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

`customers` has one row per order, so state and city percentages are shares of orders, not of people.

## Method

### Data-quality work

I profiled null counts for every column, then investigated each anomaly and logged it with evidence, a decision and a reason. The full log is in [data_quality_log.md](data_quality_log.md). The principle throughout: **flag and keep real records; never invent values.**

| # | Issue | Rows | Decision |
|---|---|---|---|
| 1 | Missing delivery date on orders not yet delivered (shipped, canceled, etc.) | 2,957 | Expected; left as is |
| 2 | Status `delivered` but no delivery date | 8 | Flagged and kept; no date invented |
| 3 | Status `canceled` but a delivery date exists | 6 | Flagged and kept (no cancellation date exists to settle the order of events) |
| 4 | Review title and message mostly null (optional fields) | 88.3% / 58.7% | Labelled `no_title` / `no_comment` |
| 5 | Products missing category, name length, description length and photo count | 610 | Kept (they carry 1.32% of item revenue); category set to `unknown`; flagged |
| 6 | Misspelled source columns (`..._lenght`) | 2 | Renamed |
| 7 | Products missing weight and all dimensions | 2 | Left null (physical facts); flagged |
| 8 | Zip prefixes with more than one city spelling | 39 prefixes, 476 rows (0.48%) | Left as is and documented; a wrong merge would be worse than the small effect |
| 9 | Customer delivery timestamp earlier than carrier pickup | 23 | Flagged and kept |
| 10 | Review text that is only whitespace | 9 caught by a first pass, 18 more by a strip check | Converted to `no_comment` |
| 11 | Categories with no English translation | 13 products (plus the 610 with no category) | Added two missing translations plus `unknown`; the lookup is 74 rows |
| 12 | Customer delivery date present but carrier date missing | 1 | Flagged and kept as its own flag |

### Flag columns

Orders: `canceled_with_delivery_date` (6), `delivered_missing_delivery_date` (8), `delivered_before_carrier_pickup` (23), `delivered_missing_carrier_date` (1).
Products: `missing_product_details` (610), `missing_dimensions` (2; one product has both flags).
Flags are stored as `yes` / `no`. The carrier-pickup flag compares full timestamps; a date-only comparison would give 20 instead of 23.

### Checks and mistakes caught along the way

- The null percentage first divided by non-null counts instead of total rows, which gave 757% for review titles. Fixed to divide by the number of rows.
- A join check returned 37,325 rows instead of 32,951 because the lookup table had duplicate rows after a cell was re-run. Fixed by dropping duplicates on the Portuguese category name.
- The first whitespace fix only caught empty strings and one or two spaces. A strip-based check found 18 more blank messages (58,274 `no_comment` messages in total).
- The dictionary used for saving still held the raw tables. The cleaned versions are now written back before export.

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
- **Page 2:** unhappy share by delivery status, unhappy share by days late, and review counts per delay band (the screenshot above).

The percentages on page 2 (9.28%, 62.40%, 32.17%, 67.65% and 79.16%) and the review counts per band were compared with the SQL results above and match.

To keep the file small enough to upload, the geolocation table and some unused columns and tables were removed from the Power BI file. No visual uses them.

## Repository contents

| File | What it is |
|---|---|
| `README.md` | This file |
| `data_quality_log.md` | Every data problem, with evidence, decision and checks |
| `delivery-performance-and-customer-satisfaction-ana.ipynb` | Cleaning and analysis notebook |
| `Delivery Performance and Customer Satisfaction Analysis-main_file_compressed.pbix` | Power BI dashboard |
| `delivery_dashboard.png` | Screenshot of dashboard page 2 |
| `.gitignore` | Files kept out of the repository |

## How to reproduce

1. Open the notebook on Kaggle with the Olist dataset attached. The file paths in the notebook point to `/kaggle/input/...`; to run it elsewhere, change them to where you saved the data.
2. Run all cells from top to bottom (Restart and Run All). The printed row counts should match the tables in this README and in the data quality log.
3. The last cells save the cleaned tables as CSV files. They are not included in this repository.

Requirements: Python 3, pandas, NumPy, DuckDB.

## About this project

I wrote the code and the analysis myself. An AI assistant was used as a guide and reviewer: it explained concepts, checked my results, pointed out bugs and wrong conclusions, and helped draft this README and the log. It did not write the project code.

## What I would do next

- Check duplicated review IDs and orders with more than one review, and repeat the findings with one review per order.
- Break down the order status of the 23 orders with delivery before pickup and of the reviews on orders with no delivery date.
- Break late delivery down by seller state and product category to see where the problem sits.
- Finish the freight analysis: price, weight and distance against freight cost.
- Put the cleaning steps into one script with automatic row-count checks.

## Skills shown

Data profiling and cleaning (pandas), SQL joins and aggregation (DuckDB), documenting decisions with evidence, reconciling counts across steps, and dashboarding in Power BI.
