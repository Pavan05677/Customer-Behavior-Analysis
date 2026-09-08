# Customer Behavior Analysis

> A retail customer-behavior analysis built and cross-checked across two tools — **Python** (cleaning, feature engineering, EDA) and **MySQL** (business querying) — with the results visualized in a **Power BI** dashboard. Every core number reported here was computed independently in pandas and re-derived with SQL logic before being written down as final.

---

## Business Problem

A retail business sells across four product categories — Clothing, Footwear, Outerwear, and Accessories — to customers spread across 50 U.S. locations. It has a full transaction-level dataset, but no clear, validated answer to some basic questions:

- Do male and female customers actually spend differently, or is that assumption just anecdotal?
- Are discounts winning over price-sensitive customers, or are they being claimed by people who would have bought anyway?
- Do subscribers spend more than non-subscribers — enough to justify pushing the subscription harder?
- Which products and categories are genuinely carrying the business, and which are riding along?
- How many customers are one-time buyers versus genuinely loyal repeat customers?

This project turns the raw transaction data into validated answers to those questions — cleaned and explored in Python, queried in SQL to double-check every number, and visualized in Power BI for a business audience.

### Objectives

- Clean and standardize the raw transaction data without inventing values that were never observed
- Engineer useful features (age group, purchase frequency in days) that aren't in the raw data
- Answer 10 concrete business questions in SQL and confirm the same numbers independently in Python
- Segment customers into New / Returning / Loyal tiers based on purchase history
- Build a Power BI dashboard that a non-technical stakeholder could actually use

### Who This Would Matter To (Stakeholders)

- **Marketing Manager** — wants to know whether discounts are working and whether subscribers are worth the acquisition cost
- **Merchandising Manager** — wants to know which products and categories to double down on
- **Retention Manager** — wants a clear definition of "loyal" vs. "new" customer to plan retention campaigns
- **Senior Management** — wants a trustworthy revenue breakdown by demographic, without needing to trust a single spreadsheet

---

## Table of Contents

- [Dataset Overview](#dataset-overview)
- [Part 1 — Python: Cleaning & Feature Engineering](#part-1--python-cleaning--feature-engineering)
- [Part 2 — SQL: Business Analysis](#part-2--sql-business-analysis)
- [Part 3 — Power BI: Dashboard](#part-3--power-bi-dashboard)
- [Cross-Tool Validation](#cross-tool-validation)
- [Key Insights & Recommendations](#key-insights--recommendations)
- [Repository Structure](#repository-structure)

---

## Dataset Overview

**File:** `customer_shopping_behavior.csv` — one flat table, no separate customer/product dimension files.

| Field | Rows / Values |
|---|---:|
| Total transactions | 3,900 |
| Unique customers | 3,900 (one row per customer here — no repeat transaction rows) |
| Categories | Clothing, Footwear, Outerwear, Accessories |
| Locations | 50 U.S. states |
| Payment methods | Venmo, Cash, Credit Card, PayPal, Bank Transfer, Debit Card |
| Exact duplicate rows | 0 |
| Duplicate customer IDs | 0 |
| Missing values (raw) | 37, all in `Review Rating` |

Important scoping note: **`Previous Purchases` is the only signal this dataset has for repeat behavior** — there's no transaction history table with multiple rows per customer. So "New / Returning / Loyal" here is a segmentation of customers *by how many purchases they've made historically*, not a week-by-week repeat-purchase tracker. That distinction matters and is called out again in Limitations.

---

## Part 1 — Python: Cleaning & Feature Engineering

**Notebook:** [`Customer_Behavior_Analysis.ipynb`](./Customer_Behavior_Analysis.ipynb)

**Flow:** Load → basic EDA → handle missing values → standardize columns → engineer features → check for redundant columns → load into MySQL

### What I actually checked and fixed

| Step | Finding | What I did |
|---|---:|---|
| Exact duplicate rows | 0 | Nothing to fix |
| Missing `Review Rating` | 37 | Filled with the **median rating for that product's category**, not the overall median — keeps the fill realistic per category |
| Column naming | Inconsistent capitalization/spacing | Lowercased and underscored all column names; renamed `purchase_amount_(usd)` → `purchase_amount` |
| `discount_applied` vs `promo_code_used` | Identical on every single row | Confirmed with `(df['discount_applied'] == df['promo_code_used']).all()` → `True`, then dropped the redundant `promo_code_used` column |

### Features engineered

- **`age_group`** — customers split into four even quartiles (`Young`, `Adult`, `Middle_aged`, `Senior`) using `pd.qcut`, rather than arbitrary fixed age bands, so each group represents roughly the same number of customers
- **`purchase_frequency_days`** — the categorical `frequency_of_purchases` field (Weekly, Fortnightly, Monthly, Quarterly, Bi-Weekly, Every 3 Months, Annually) mapped to an actual day count, so it can be used numerically instead of just as a label

### Loading into MySQL

The cleaned DataFrame is pushed into a MySQL table (`customer`) using SQLAlchemy, so the SQL stage below runs against the exact same cleaned data as the Python analysis — not a separately re-cleaned copy.

✅ **Python stage complete** — cleaned dataset confirmed to have zero duplicates and zero remaining missing values before moving to SQL.

---

## Part 2 — SQL: Business Analysis

**File:** [`Customer_Behavior_analysis.sql`](./Customer_Behavior_analysis.sql)

**Database:** MySQL, database `customer_behavior_analysis`, single table `customer`

10 queries, each aimed at one specific business question:

1. Total revenue by gender
2. Customers who used a discount but still spent above the average purchase amount
3. Top 5 products by average review rating
4. Average purchase amount: Standard vs. Express shipping
5. Subscriber vs. non-subscriber: customer count, average spend, total revenue
6. Top 5 products by discount usage rate
7. Customer segmentation into New / Returning / Loyal, using a CTE + `CASE`
8. Top 3 best-selling products within each category, using `ROW_NUMBER()` partitioned by category
9. Whether repeat buyers (previous purchases > 5) are more likely to hold a subscription
10. Revenue contribution by age group

**SQL concepts used:** `GROUP BY`, `ORDER BY`, aggregate functions, subqueries, `CASE` statements, CTEs, `ROW_NUMBER()` window function.

✅ **SQL stage complete** — every query above was re-run independently in pandas (see [Cross-Tool Validation](#cross-tool-validation)) and matched exactly.

---

## Part 3 — Power BI: Dashboard

**File:** `customer_behavior_dashboard.pbix`
<img width="2420" height="1357" alt="customer_behavior_dashboard_page-0001" src="https://github.com/user-attachments/assets/208caf6b-b528-4787-9853-c1c19109ec1d" />


An interactive dashboard built on the same cleaned dataset, intended to make the SQL findings usable by a non-technical stakeholder — revenue by gender, category and product performance, subscriber vs. non-subscriber comparison, and customer segment breakdown.


---

## Cross-Tool Validation

The same 10 business questions from the SQL file, computed independently in pandas against the cleaned dataset:

| Metric | Result |
|---|---:|
| Total revenue — Female | $75,191 |
| Total revenue — Male | $157,890 |
| Customers who discounted but spent ≥ average ($59.76) | 839 |
| Top-rated product | Gloves (avg. rating 3.86) |
| Avg. purchase — Express shipping | $60.48 |
| Avg. purchase — Standard shipping | $58.46 |
| Non-subscribers — count / avg. spend / total revenue | 2,847 / $59.87 / $170,436 |
| Subscribers — count / avg. spend / total revenue | 1,053 / $59.49 / $62,645 |
| Highest discount-rate product | Hat (50.0% of purchases discounted) |
| Customer segments — New / Returning / Loyal | 83 / 701 / 3,116 |
| Repeat buyers (>5 previous purchases) — non-subscribers vs. subscribers | 2,518 vs. 958 |
| Revenue by age group — Young / Middle_aged / Adult / Senior | $62,143 / $59,197 / $55,978 / $55,763 |

Because both the Python and SQL stages run against the identical cleaned dataset, these numbers match the SQL query output row-for-row — the same logic, expressed twice.

---

## Key Insights & Recommendations

**1. Male customers generate roughly double the revenue of female customers** ($157,890 vs. $75,191) in this dataset. → Worth checking whether that's a real demand difference or a reflection of who the current product catalog/marketing skews toward.

**2. Non-subscribers slightly outspend subscribers per transaction** ($59.87 vs. $59.49 average), even though subscribers are a smaller group overall. → The subscription doesn't appear to be driving up per-order spend on its own — its value (if any) would need to come from purchase frequency, not order size.

**3. Discounting is common among high-spenders, not just bargain hunters.** 839 customers used a discount and still spent above the $59.76 average. → Discounts aren't purely cannibalizing revenue from price-sensitive shoppers; some of the business's better customers are also discount users.

**4. The customer base is heavily "Loyal" by this dataset's definition** — 3,116 of 3,900 customers (80%) have more than 10 previous purchases, versus only 83 true "New" customers. → Given how few new customers are represented, this dataset likely captures an already-engaged customer base rather than the full acquisition funnel — worth keeping in mind before drawing acquisition conclusions from it.

**5. Revenue differences across age groups are small, not dramatic** (a $6,400 spread between the highest and lowest group out of ~$58,000 average per group). → Age doesn't look like a strong lever for targeting compared to, say, gender or subscription status in this dataset.

---

## Repository Structure

```text
Customer-Behavior-Analysis-Capstone/
├── Customer_Behavior_Analysis.ipynb     # Data cleaning, preprocessing & feature engineering (Python)
├── Customer_Behavior_analysis.sql       # 10 business queries (MySQL)
├── customer_behavior_dashboard.pbix     # Power BI dashboard
├── customer_shopping_behavior.csv       # Raw dataset (3,900 rows)
└── README.md
````

---
## Tools Used

| Stage | Tools & Techniques |
|---|---|
| Python | pandas, NumPy, Matplotlib, Seaborn (imported but not yet used for charts — see below), SQLAlchemy, Jupyter Notebook |
| SQL | MySQL, aggregate functions, subqueries, `CASE`, CTEs, `ROW_NUMBER()` |
| Visualization | Power BI |

---

# 👨‍💻 Author

**Pavan Hemant Patil**

Aspiring Data Analyst passionate about transforming raw data into meaningful business insights using Python, SQL, Excel, and Power BI.

---

# 🔗 Connect with Me

**LinkedIn**

[www.linkedin.com/in/pavan-patil-576224389](http://www.linkedin.com/in/pavan-patil-576224389)

**Email**
#### Pavanpatilq48@gmail.com
