# Low-ROAS-Diagnostic-Analysis
### Why are certain campaigns generating low return on ad spend?

----
## Objective

Marketing teams often know that some campaigns underperform, but rarely know exactly why. This project takes a diagnostic approach to identify the root cause of low ROAS.

The goal is to determine whether low ROAS is caused by targeting decisions (wrong country, industry, platform, or campaign type) or by execution inefficiencies deeper in the funnel.

---

## Business Question

> **Why are certain campaigns generating low ROAS, and what funnel factors are responsible?**

---

## Sub-Questions

| # | Sub-Question | Purpose |
|---|---|---|
| Q1 | Is low ROAS influenced by countries? 
| Q2 | Is low ROAS influenced by industries? 
| Q3 | Is low ROAS influenced by specific platforms?
| Q4 | Is low ROAS influenced by specific campaign types? 
| Q5 | What funnel metrics differ between High and Low ROAS campaigns? 
| Q6 | Where exactly does the funnel break down CTR, CPC, conversion, CPA? 
| Q7 | Does the pattern hold when combining country + industry + platform? 

---

## Dataset

| Field | Detail |
|---|---|
| Table | `global_ads2` |
| Platforms | Google Ads, Meta Ads, TikTok Ads |
| Countries | USA, UK, Canada, Germany, India, UAE, Australia |
| Industries | E-commerce, EdTech, Fintech, Healthcare, SaaS |
| Campaign Types | Search, Video, Shopping, Display |
| Key Columns | impressions, clicks, ad_spend, conversions, revenue, CTR, CPC, CPA, ROAS, conversion_rate |

Metrics were pre-calculated in Excel Power Query, then independently recreated from raw columns in SQL to validate accuracy.

---

### Defining Low ROAS
 The 25th percentile of ROAS values across all campaigns was calculated that came out to **3.07**. Campaigns at or below 3.07 were classified as Low ROAS.

```sql
WITH campaign_yearly AS (
    SELECT
        platform, campaign_type, industry, country,
        SUM(revenue) / NULLIF(SUM(ad_spend), 0) AS roas_yearly
    FROM global_ads2
    GROUP BY platform, campaign_type, industry, country
),
ordered AS (
    SELECT roas_yearly,
           ROW_NUMBER() OVER (ORDER BY roas_yearly) AS rn,
           COUNT(*) OVER () AS total_rows
    FROM campaign_yearly
)
SELECT roas_yearly FROM ordered
WHERE rn = FLOOR(total_rows * 0.25);
-- Result: 3.07
```

### SQL View for Reuse
A base view was created so all segmentation queries reuse the same logic without repetition:

```sql
CREATE VIEW delivery_analysis AS
SELECT
    platform, campaign_type, industry, country, date,
    impressions, clicks, ad_spend, conversions, revenue,
    ctr_calculated, cpc_calculated, cpa_calculated,
    roas_calculated, conversions_rate,
    CASE WHEN roas_calculated <= 3.07 THEN 'Low ROAS' ELSE 'High ROAS' END AS roas_segment
FROM global_ads2;
```

---

## Key Findings

### Overall Funnel Comparison

| Metric | High ROAS | Low ROAS | Gap |
|---|---|---|---|
| Avg CTR | 0.0386 | 0.0382 | ≈ Equal |
| Avg CPC | $1.33 | $2.01 | **+51%** |
| Avg Conversion Rate | 5.24% | 3.25% | **−38%** |
| Avg CPA | $28.27 | $79.85 | **+182%** |

**The funnel tells a clear story:**
- CTR is virtually identical ads attract clicks equally well. Ad creative and targeting are NOT the problem.
- CPC is 51% higher in Low ROAS traffic is significantly more expensive.
- Conversion rate is 38% lower clicks are not converting.
- The combined effect drives CPA up 182% Low ROAS campaigns cost 2.8x more per customer acquired.

---

### Platform-Level Breakdown

| Platform | High ROAS CPC | Low ROAS CPC | High ROAS Conv. | Low ROAS Conv. | CPA High | CPA Low |
|---|---|---|---|---|---|---|
| Google Ads | $1.90 | $2.42 | 5.6% | 3.3% | $34.00 | $71.52 |
| Meta Ads | $1.19 | $1.64 | 5.1% | 3.2% | $23.20 | $50.51 |
| TikTok Ads | $0.96 | $1.26 | 5.2% | 2.7% | $18.61 | $46.42 |

- **Google Ads** — biggest CPC gap
- **TikTok Ads** — sharpest conversion rate drop
- **Meta Ads** — moderate gap in both

---

### Segmentation :

- No country is exclusively Low ROAS
- No industry is exclusively Low ROAS
- No platform or campaign type is exclusively Low ROAS
- Both High and Low ROAS exist within every segment

**This confirms: the problem is execution level, not targeting level.**

---

### Geographic Validation

The CPC and conversion rate gap holds consistently across all 7 countries:

| Country | High ROAS CPC | Low ROAS CPC | High ROAS Conv. | Low ROAS Conv. |
|---|---|---|---|---|
| Australia | $1.32 | $2.36 | 5.5% | 3.2% |
| Canada | $1.32 | $2.09 | 5.3% | 2.9% |
| Germany | $1.34 | $1.97 | 5.1% | 3.3% |
| India | $1.23 | $1.91 | 5.2% | 3.1% |
| UAE | $1.32 | $1.98 | 5.2% | 3.2% |
| UK | $1.30 | $1.96 | 5.5% | 3.2% |
| USA | $1.33 | $2.00 | 5.3% | 3.5% |

Notable: Canada Healthcare shows the most extreme CPA gap $21.27 High ROAS vs $110.95 Low ROAS (5x difference).

---

## Conclusion

> Low ROAS is not a targeting problem. It is a campaign execution problem campaigns paying too much per click and failing to convert those clicks. This pattern holds without exception across every country, industry, platform, and campaign type in the dataset.


## Dashboard

Power BI dashboard 2 pages:

- **Page 1 Overview:** Funnel comparison visual, CPC by platform (High vs Low), Conversion rate by platform (High vs Low), KPI cards
- **Page 2 Diagnosis:** CPC by country, Conversion rate by country, CPA by industry, Segmentation elimination panel

---

## Tools Used

| Tool | Purpose |
|---|---|
| Excel Power Query | Data cleaning, metric calculation |
| MySQL | Threshold calculation, view creation, segmentation queries |
| Power BI | Interactive 2-page diagnostic dashboard |

---

