# 🛍️ Customer Behavior Analysis

> A full-stack data analytics project combining **Python (Pandas)**, **PostgreSQL**, and **Power BI** to uncover purchasing patterns, customer segmentation, and revenue drivers from retail shopping data.

---

## 📌 Table of Contents

- [Project Overview](#-project-overview)
- [Tech Stack](#-tech-stack)
- [Dataset](#-dataset)
- [Project Workflow](#-project-workflow)
- [Data Cleaning & Feature Engineering](#-data-cleaning--feature-engineering)
- [SQL Analysis — Key Questions & Findings](#-sql-analysis--key-questions--findings)
- [Power BI Dashboard](#-power-bi-dashboard)
- [Key Business Insights](#-key-business-insights)
- [Repository Structure](#-repository-structure)
- [How to Run](#-how-to-run)

---

## 📖 Project Overview

This project performs an end-to-end analysis of retail customer shopping behavior. Starting from raw CSV data, the pipeline covers:

1. **Data Wrangling** in Python/Pandas (cleaning, feature engineering)
2. **Database Loading** into PostgreSQL via SQLAlchemy
3. **SQL Analysis** answering 10 targeted business questions
4. **Visual Storytelling** through a Power BI dashboard (`.pbix`)

The goal is to answer real business questions: *Who spends more? Do subscribers behave differently? Which products drive the most revenue? How does discount strategy affect purchases?*

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| **Python 3 / Pandas** | Data loading, cleaning, feature engineering |
| **SQLAlchemy + psycopg2** | Connecting Python to PostgreSQL |
| **PostgreSQL** | Structured querying and analysis |
| **Power BI** | Interactive dashboard and visualizations |
| **Jupyter Notebook** | Exploratory analysis and pipeline orchestration |

---

## 📂 Dataset

**Source:** `customer_shopping_behavior.csv`

The dataset contains individual customer purchase records with the following fields (after cleaning):

| Column | Description |
|---|---|
| `customer_id` | Unique customer identifier |
| `age` | Customer age |
| `age_group` | Derived segment: Young Adults / Adults / Middle Aged / Senior |
| `gender` | Male / Female |
| `item_purchased` | Product name |
| `category` | Product category (Clothing, Footwear, Accessories, etc.) |
| `purchase_amount` | Transaction value in USD |
| `review_rating` | Rating given by the customer (1–5) |
| `subscription_status` | Yes / No |
| `shipping_type` | Standard, Express, Free Shipping, etc. |
| `discount_applied` | Whether a discount was used (Yes / No) |
| `previous_purchases` | Number of past transactions |
| `frequency_of_purchases` | Textual frequency (Weekly, Monthly, etc.) |
| `purchases_frequency_days` | Derived numeric equivalent in days |

---

## 🔄 Project Workflow

```
Raw CSV
   │
   ▼
Python (Pandas) ──► Data Cleaning & Feature Engineering
   │
   ▼
SQLAlchemy ──► PostgreSQL Database (table: customer)
   │
   ▼
SQL Queries ──► 10 Business Questions Answered
   │
   ▼
Power BI ──► Interactive Dashboard (.pbix)
```

---

## 🧹 Data Cleaning & Feature Engineering

All preprocessing was done in `Customer_Behavior_Analysis.ipynb`.

### Null Value Handling
Missing `Review Rating` values were imputed with the **category-level median** to preserve distribution integrity:

```python
df['Review Rating'] = df.groupby('Category')['Review Rating'].transform(
    lambda x: x.fillna(x.median())
)
```

### Column Standardization
```python
df.columns = df.columns.str.lower().str.replace(' ', '_')
df = df.rename(columns={'purchase_amount_(usd)': 'purchase_amount'})
```

### Feature: Age Group
Customers were divided into four equal-frequency quartile buckets:

```python
labels = ['Young Adults', 'Adults', 'Middle Aged', 'Senior']
df['age_group'] = pd.qcut(df['age'], q=4, labels=labels)
```

### Feature: Purchase Frequency (Days)
The text-based purchase frequency was mapped to numeric day equivalents:

```python
frequency_mapping = {
    'Weekly': 7, 'Fortnightly': 14, 'Monthly': 30,
    'Quarterly': 90, 'Annually': 365, 'Every_3_months': 90
}
df['purchases_frequency_days'] = df['frequency_of_purchases'].map(frequency_mapping)
```

### Redundant Column Removal
`promo_code_used` was found to be a perfect duplicate of `discount_applied` and was dropped:

```python
(df['discount_applied'] == df['promo_code_used']).all()  # → True
df = df.drop('promo_code_used', axis=1)
```

### Loading to PostgreSQL
```python
from sqlalchemy import create_engine
engine = create_engine("postgresql+psycopg2://user:password@localhost:5432/customer_behaviour")
df.to_sql('customer', engine, if_exists='replace', index=False)
```

---

## 🔍 SQL Analysis — Key Questions & Findings

### Q1 — Revenue by Gender

```sql
SELECT gender, SUM(purchase_amount) AS revenue
FROM customer
GROUP BY gender;
```

**Finding:** Both male and female customers contribute significantly to overall revenue. Male customers tend to generate slightly higher total revenue due to a larger transaction count.

---

### Q2 — High-Spending Discount Users

```sql
SELECT customer_id, purchase_amount
FROM customer
WHERE discount_applied = 'Yes'
  AND purchase_amount >= (SELECT AVG(purchase_amount) FROM customer);
```

**Finding:** A substantial portion of discount users still spend above the average purchase amount, indicating that discounts attract quality buyers — not just bargain hunters.

---

### Q3 — Top 5 Products by Average Review Rating

```sql
SELECT item_purchased,
       ROUND(AVG(review_rating::numeric), 2) AS "Average Product Rating"
FROM customer
GROUP BY item_purchased
ORDER BY avg(review_rating) DESC
LIMIT 5;
```

**Finding:** The top-rated products cluster around specific clothing and accessory items, suggesting strong satisfaction in those sub-categories.

---

### Q4 — Standard vs. Express Shipping Spend

```sql
SELECT shipping_type, ROUND(AVG(purchase_amount), 2)
FROM customer
WHERE shipping_type IN ('Standard', 'Express')
GROUP BY shipping_type;
```

**Finding:** Express shipping customers show a comparable or slightly higher average purchase amount, indicating they are willing to pay more overall — both for goods and faster delivery.

---

### Q5 — Subscriber vs. Non-Subscriber Spend

```sql
SELECT subscription_status,
       COUNT(customer_id) AS total_customers,
       ROUND(AVG(purchase_amount), 2) AS avg_spend,
       ROUND(SUM(purchase_amount), 2) AS total_revenue
FROM customer
GROUP BY subscription_status
ORDER BY total_revenue, avg_spend DESC;
```

**Finding:** Subscribed customers contribute disproportionately high total revenue despite being a smaller group, validating the value of investing in subscription/loyalty programs.

---

### Q6 — Products with Highest Discount Rates

```sql
SELECT item_purchased,
       ROUND(100.0 * SUM(CASE WHEN discount_applied = 'Yes' THEN 1 ELSE 0 END) / COUNT(*), 2) AS discount_rate
FROM customer
GROUP BY item_purchased
ORDER BY discount_rate DESC
LIMIT 5;
```

**Finding:** Certain products have discount rates exceeding 50%, signaling potential over-reliance on promotional pricing. These items may benefit from a revised pricing strategy.

---

### Q7 — Customer Segmentation: New / Returning / Loyal

```sql
WITH customer_type AS (
    SELECT customer_id, previous_purchases,
           CASE
               WHEN previous_purchases = 1 THEN 'New'
               WHEN previous_purchases BETWEEN 2 AND 10 THEN 'Returning'
               ELSE 'Loyal'
           END AS customer_segment
    FROM customer
)
SELECT customer_segment, COUNT(*) AS "Number of Customers"
FROM customer_type
GROUP BY customer_segment;
```

**Finding:** The majority of the customer base falls in the **Returning** segment, with a healthy cohort of **Loyal** customers — a positive sign for long-term retention.

```
Segment Distribution (approximate)
─────────────────────────────────
  Loyal     ████████████████  ~40%
  Returning ████████████████████ ~50%
  New       ████  ~10%
```

---

### Q8 — Top 3 Products per Category

```sql
WITH item_counts AS (
    SELECT category, item_purchased,
           COUNT(customer_id) AS total_orders,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY COUNT(customer_id) DESC) AS item_rank
    FROM customer
    GROUP BY category, item_purchased
)
SELECT item_rank, category, item_purchased, total_orders
FROM item_counts
WHERE item_rank <= 3;
```

**Finding:** Clothing dominates by transaction volume. Within each category, a small number of items drive the majority of orders — a classic long-tail distribution.

---

### Q9 — Repeat Buyers and Subscription Likelihood

```sql
SELECT subscription_status, COUNT(customer_id) AS repeat_buyers
FROM customer
WHERE previous_purchases > 5
GROUP BY subscription_status;
```

**Finding:** Customers with more than 5 previous purchases skew heavily toward being **subscribed**, confirming that loyalty and subscription are strongly correlated.

---

### Q10 — Revenue by Age Group

```sql
SELECT age_group, SUM(purchase_amount) AS total_revenue
FROM customer
GROUP BY age_group
ORDER BY total_revenue DESC;
```

**Finding:** **Middle Aged** and **Adult** customers are the top revenue contributors. **Senior** customers, while fewer, show high per-transaction values. Marketing efforts should be prioritized toward the Adult and Middle Aged brackets.

```
Revenue by Age Group (relative)
────────────────────────────────────────
  Middle Aged  ████████████████████ Highest
  Adults       ███████████████████
  Senior       █████████████
  Young Adults ███████████          Lowest
```

---

## 📊 Power BI Dashboard

The `Customer_behavior.pbix` file contains an interactive dashboard with the following views:

| Dashboard Page | Description |
|---|---|
| **Revenue Overview** | Total revenue split by gender, age group, and subscription status |
| **Product Performance** | Top products by orders and average rating per category |
| **Customer Segments** | New / Returning / Loyal breakdown with purchase behavior |
| **Discount Analysis** | Discount rate by product and its impact on average spend |
| **Shipping Insights** | Average order value by shipping type |
| **Subscription Impact** | Revenue comparison: subscribers vs. non-subscribers |

> 📁 Open `Customer_behavior.pbix` in **Power BI Desktop** to explore filters, slicers, and drill-throughs.

---

## 💡 Key Business Insights

| # | Insight | Recommendation |
|---|---|---|
| 1 | Subscribed customers deliver higher revenue per head | Expand subscription perks and retention campaigns |
| 2 | Repeat buyers (5+ purchases) are mostly subscribers | Use purchase-count milestones to trigger subscription upsells |
| 3 | Discount users often spend above average | Strategic discounts can attract quality buyers, not just deal seekers |
| 4 | Some products have >50% discount dependency | Review pricing strategy for heavily discounted items |
| 5 | Middle Aged & Adult segments drive the most revenue | Focus ad spend and personalization on these cohorts |
| 6 | Express shipping correlates with higher order values | Offer Express as a premium bundle to increase AOV |
| 7 | Most customers are Returning (not New or Loyal) | Focus on loyalty conversion: Returning → Loyal |

---

## 📁 Repository Structure

```
customer-behavior-analysis/
│
├── Customer_Behavior_Analysis.ipynb   # Python data pipeline (cleaning + DB load)
├── customer_behavior.sql              # 10 SQL business questions with queries
├── Customer_behavior.pbix             # Power BI dashboard
└── README.md                          # This file
```

---

## ▶️ How to Run

### 1. Python Pipeline (Jupyter Notebook)

```bash
pip install pandas sqlalchemy psycopg2-binary
jupyter notebook Customer_Behavior_Analysis.ipynb
```

Update the database credentials in the notebook before running:

```python
username = "your_username"
password = "your_password"
host     = "localhost"
port     = "5432"
database = "customer_behaviour"
```

### 2. SQL Queries

Once the data is loaded into PostgreSQL, run queries from `customer_behavior.sql` using:

- **pgAdmin** (GUI)
- **psql** CLI: `psql -U your_username -d customer_behaviour -f customer_behavior.sql`

### 3. Power BI Dashboard

Open `Customer_behavior.pbix` in [Power BI Desktop](https://powerbi.microsoft.com/desktop/) (Windows only). Refresh the data connection if required.

---

## 🙌 Acknowledgements

Dataset sourced from publicly available retail shopping behavior data. Analysis, feature engineering, and visualizations are original work.

---

*Built with Python · PostgreSQL · Power BI*

*Built by Ayush Ranjan

