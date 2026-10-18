# Customer Churn Analysis

SQL + Python (pandas) analysis of a small subscription-business dataset to understand **who churns, why, and what the early-warning signals look like.**

| | |
|---|---|
| **Data** | `customer_churn.db` (SQLite) – 3 tables: `db_customer`, `db_subscription`, `db_support` |
| **Notebook** | `Churn_analysis.ipynb` |
| **Stack** | Python, pandas, NumPy, Matplotlib, Seaborn, SQLite |
| **Sample size** | 21 customers, 9 support tickets (7 customers) |

> ⚠️ **Small-sample caveat:** with only 21 customers (6 churned), every percentage below moves ~5 points per customer. Treat findings as directional hypotheses, not statistically proven effects.

---

## 1. Headline Numbers

| Metric | Value |
|---|---|
| Churned customers | **6 of 21** |
| Churn rate | **28.6%** |
| Retention rate | **71.4%** |
| Monthly revenue lost to churn | **73.94** of 395.79 total (currency not specified in data) (18.7%) |
| CLTV of churned customers | 2,047 of 17,294 total (11.8%) |
| Average monthly charge (ARPU) | 18.85 |
| Avg. tenure – churned vs. retained | ~773 days vs. ~1,879 days |

Churned customers pay far less on average (12.32 vs 21.46 per month): the business is losing **low-value** customers, which is why revenue churn (18.7%) is lower than customer churn (28.6%).

---

## 2. Data Pipeline

1. Loaded all three tables from SQLite into DataFrames.
2. **Cleaning**
   - Renamed `name` → `customer_name`; dropped empty/unused columns (`interests`, `pincode`, `col_1`, `comment`).
   - Converted `dob` and date columns to `datetime`.
   - Standardised gender labels (`Men`/`Women` → `Male`/`Female`).
   - Imputed 3 missing `country` values by mapping from `state` (e.g. Kathmandu → Nepal).
3. **Feature engineering:** `churn_flag` (cancellation date present), `new_age`, `tenure_days`, `complaint_count`, `churn_risk` bucket (Low / Medium / High).
4. **Joining:** `db_support` has multiple tickets per customer, which duplicated rows on merge (23 rows instead of 21). Fixed by counting complaints per customer, then keeping the latest ticket, resulting in one row per customer (21).

---

## 3. Key Findings


### 3.1 Contract type is the strongest structural driver
| Contract | Customers | Churned | Churn rate |
|---|---|---|---|
| Monthly | 9 | 5 | **55.6%** |
| Annual | 12 | 1 | **8.3%** |

Monthly contracts churn ~7× more than annual ones. Within plans: Basic-Monthly 75%, Standard-Monthly 40%, while Standard-Annual and Basic-Annual had 0% churn.

### 3.2 Basic plan is the weakest tier
Basic churns at **60%** (3 of 5) vs. Standard 22% and Premium 14%. Basic is also almost entirely monthly (4 of 5), so plan and contract effects overlap and can't be separated with this sample size.

### 3.3 Referral-acquired customers churn heavily
| Channel | Customers | Churned | Churn rate |
|---|---|---|---|
| Referral | 6 | 5 | **83.3%** |
| Paid | 6 | 1 | 16.7% |
| Organic | 9 | 0 | **0%** |

5 of the 6 churners came through referral, and 0 of 9 organic customers churned. This is the most striking pattern in the data and worth validating on a larger sample, since referral incentives may be attracting low-intent users.

### 3.4 Support experience is a strong early-warning signal
- **Only customers who contacted support churned:** 6 of the 7 customers with a ticket churned (86%); none of the 14 customers without a ticket churned.
- Among customers with tickets, those with an **escalation** churned 5 of 5 times.
- Lowest CSAT scores belong to churners (10, 10, 20, 25, 30, 30); the one retained complainer scored 90.
- 5 of 9 tickets (56%) were escalated. Escalation and poor CSAT point to unresolved issues.

### 3.5 Why customers leave

Reasons are spread out (competitor switch ×2, too expensive, not enough content, poor streaming quality, forgot to cancel a trial), so there is no single dominant cause. Two are product/quality related and one is an involuntary/trial-conversion issue (**"Forgot to cancel trial"** customer churned immediately, 0 days after renewal, with a churn score of 99).

### 3.6 Geography
Karnataka (2/2) and Meghalaya (2/3) show the highest churn, but state-level groups have 1–4 customers each, which is too small to draw conclusions. Churn is also similar across genders (F 33%, M 22%).

---

## 4. Churn Score Model Check

All 6 churned customers had `churn_score` ≥ 76; all 15 retained customers scored ≤ 62. The existing score separates the groups perfectly, which is suspiciously clean. Either it's an excellent model or (more likely) it was computed **after** cancellation and leaks the outcome. Confirm how it's generated before using it for prediction.

**Retained customers to watch** (highest remaining scores): `0020-JDNXP` (62, Premium, 20.99/month), `0020-INWCK` (58, Standard), `0023-UYUPN` (47, Standard).

---

## 5. Recommendations

1. **Push monthly customers onto annual plans** with a discount or perk: annual customers churn at 8% vs. 56%.
2. **Review the Basic plan** value proposition and pricing, since 3 of 5 customers left.
3. **Audit the referral programme:** 83% churn suggests a quality-of-acquisition problem; consider rewarding retention (e.g. referral credit after 90 days).
4. **Trigger proactive outreach on support tickets**, especially escalations or CSAT ≤ 30, because these preceded nearly every cancellation.
5. **Add trial-end reminders** to reduce involuntary "forgot to cancel" churn.
6. **Scale the analysis:** repeat on a larger dataset before acting on segment-level (state, gender) differences.

---

## 6. Notebook Issues to Fix Before Publishing

While reviewing the notebook I found several bugs that affect reported numbers:

| Cell / metric | Issue | Fix |
|---|---|---|
| Revenue at Risk | Labelled "Rs 'K'" (73.94) but the data has no currency or thousands unit | Label as monthly revenue; state currency |
| ARPU | Printed with a `%` sign | Remove `%` |
| Escalation rate (19.05%) | Calculated on the latest ticket per customer divided by all 21 customers, so it's diluted | Use 4/7 = 57% of customers with tickets, or 5/9 = 56% of tickets |
| Avg complaint rate (42.86%) | It's 0.43 complaints per customer, not a percentage | Report as 0.43 complaints/customer |
| Escalation–churn correlation | Returns `NaN` because `escalations` is `Y`/`N` text; the commented-out encoding line is never run | Encode `Y`→1, `N`→0 first |
| Correlation heatmap | First version uses only `.head()` (5 rows); both versions silently drop `escalations` (still text) | Use full data and encode first |
| `churn_risk` bins | `churn_score > 70` leaves exactly 70 as `'unknown'` | Use `>= 70` |
| `tenure_days`, `new_age` | Use `pd.Timestamp.today()`, so results change on every run | Fix a reference date |
| Churn-by-state output | Filters to churned only, so no rates | Show rate + count per state |
| Stale renewal dates | 15 of 15 active customers have renewal dates already in the past | Confirm if data is a snapshot or needs refreshing |
| Typo in data | Subscription type `Refferal` | Standardise to `Referral` |

---

## 7. Repo Structure (suggested)

```
.
├── README.md
├── Churn_analysis.ipynb
├── customer_churn.db
└── exported_churn_data.csv   # generated by the notebook
```

## 8. Reproduce

```bash
pip install pandas numpy matplotlib seaborn jupyter
jupyter notebook Churn_analysis.ipynb
```
