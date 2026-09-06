# Retail Revenue Growth & Profitability Diagnostic

A Power BI case study analyzing the quality and drivers of retail revenue growth through profitability analysis, Price–Volume–Mix decomposition, product-mix diagnostics, market performance, and channel concentration.

> **Dataset:** Starttrain – The Next Analyst Challenge Season 1  
> **Period:** 2016–2018  
> **Records:** 72,743  
> **Tools:** Power BI, Power Query, DAX

---

## Business Objective

The objective of this project was to move beyond descriptive sales reporting and answer four management-level questions:

1. **How healthy is revenue growth?**
2. **What is actually driving incremental revenue?**
3. **Which products are growing with strong or weak growth quality?**
4. **Which markets and channels are driving or concentrating business performance?**

The final dashboard therefore focuses on explaining **what changed, why it changed, where the impact came from, and what management should investigate next**.

---

## Dataset Overview

The dataset contains 72,743 retail sales records from 2016 to 2018.

Key dimensions include:

- **14 countries**
- **5 product lines**
- **19 product types**
- **6 sales channels / order methods**

The source dataset was already clean, so no material data-cleaning operations were required. Power Query was used as part of the data-loading and model-preparation workflow.

---

## Data Model

The report uses a star-schema-style model with `Sales` as the central fact table.

Main dimensions include:

- Calendar
- Product
- Region
- Sales Manager / Territory

Relationships follow a:

- **1-to-many cardinality**
- **Single-direction filtering from dimension to fact**

Additional disconnected tables and field parameters were used for:

- Revenue growth bridge
- Prior-year vs current-year comparison
- Dynamic growth-driver selection

This structure keeps the analytical logic reusable across products, markets, channels, and time periods.

---

## Core KPIs

The report tracks both business scale and growth quality.

### Performance

- Revenue
- Revenue YoY %
- Revenue Change
- Gross Profit
- Gross Profit YoY %
- Gross Margin %
- Gross Margin Change
- Quantity Growth
- Average Selling Price (ASP)
- ASP Growth

### Diagnostic Measures

- Revenue Volume Effect
- Within-Product ASP Effect
- Product Mix Effect
- Growth Contribution %
- Mix Offset %
- Quantity Mix Shift
- Channel Revenue Share
- Channel Mix Shift
- Market Growth Contribution

---

# Analytical Methodology

## 1. Price–Volume–Mix Analysis

Revenue was decomposed into:

**Revenue Change = Volume Effect + Within-Product ASP Effect + Product Mix Effect**

A midpoint decomposition approach was used to separate ASP changes from product-mix changes while avoiding order dependency.

The decomposition was validated with a reconciliation measure:

**Revenue PVM Residual = $0**

This ensured that the analytical bridge fully reconciled to actual revenue growth.

---

## 2. Product-Mix Diagnostic

Aggregate ASP changes were further decomposed into:

**ASP Change = Within-Product ASP Effect + Product Mix Effect**

This made it possible to distinguish between:

- changes in selling value within existing product types, and
- changes caused by selling a different mix of high- and low-value products.

---

## 3. Growth Contribution

Instead of ranking segments only by total revenue, the analysis focuses on:

**Contribution to incremental revenue**

This identifies which products, countries, and channels actually created or diluted year-over-year growth.

---

# Key Findings

## 1. 2018 growth shifted from value-led to volume-led

Revenue reached approximately **$424.4M**, growing **17.4% YoY**, while unit volume grew faster at **20.9%**.

At the same time:

- ASP declined **2.9%**
- Gross Profit increased **17.6%**
- Gross Margin remained resilient at **42.25%**, improving approximately **0.07 percentage points**

This indicates that 2018 growth depended increasingly on selling more units rather than generating higher revenue per unit.

---

## 2. Volume generated $74.5M of potential growth, but ASP and mix offset $11.7M

The 2018 revenue bridge showed:

| Revenue Driver | Impact |
| --- | ---: |
| Volume Effect | **+$74.5M** |
| Within-Product ASP Effect | **-$2.6M** |
| Product Mix Effect | **-$9.0M** |
| Net Revenue Change | **+$62.8M** |

Although volume expansion generated more growth than the final revenue increase, approximately **$11.7M** was offset by ASP and product-mix effects.

---

## 3. Approximately 77% of the ASP decline was mix-driven

Company ASP declined by approximately **$2.52 per unit**.

The decomposition showed:

- Within-product ASP effect: **-$0.57**
- Product-mix effect: **-$1.95**

Therefore, approximately **77% of the ASP decline was attributable to product-mix changes**, rather than broad-based deterioration in within-product ASP.

This distinction was important because the decline in aggregate ASP did not necessarily imply company-wide price erosion.

---

## 4. Computer showed the strongest product-mix dilution

Computer revenue still increased approximately **10.4%**, supported by strong volume growth of **21.9%**.

However:

- Volume Effect: **+$15.25M**
- Product Mix Effect: **-$7.81M**

The changing product mix therefore offset approximately **51% of the revenue uplift created by volume growth**.

Further drill-down showed that lower-value Computer Accessories gained unit share while higher-value Laptop and Desktop products lost share.

This explains why aggregate Computer ASP declined sharply despite relatively resilient within-product ASP performance.

---

## 5. Revenue growth was concentrated by product

Video Games generated approximately:

- **$28.4M** of incremental revenue
- **45.2%** of net company revenue growth

Mobile contributed another **27.6%**.

Together, **Video Games and Mobile generated approximately 73% of 2018 net incremental revenue**.

This indicates meaningful product-level growth concentration.

---

## 6. Geographic growth was more diversified

The United States was the largest geographic growth contributor:

- Incremental Revenue: **+$15.35M**
- Growth Contribution: **24.4%**

However, growth was distributed across multiple countries.

The top three markets accounted for approximately **43% of incremental revenue**, indicating substantially lower concentration than at the product or channel level.

---

## 7. Growth was highly concentrated in the Web channel

Web represented approximately:

- **88.8% of current revenue**
- **99.5% of net incremental revenue**

Its revenue share also increased by approximately **1.86 percentage points** compared with the prior year.

This suggests that business growth became increasingly dependent on a single order channel.

---

# Dashboard Structure

## Page 1 – Executive Performance

**Management question:** How did the company grow?

Highlights:

- Revenue, Gross Profit, Margin, Volume, and ASP KPIs
- Revenue Price–Volume–Mix bridge
- Product-line PVM comparison
- Dynamic growth-quality and management-priority insights

---

## Page 2 – Growth Drivers

**Management question:** Where did incremental revenue come from?

Highlights:

- Dynamic Growth Driver field parameter
- Revenue Change by Product / Region / Country / Channel
- Growth Contribution analysis
- Revenue Change decomposition tree
- Growth diagnostic matrix

The page focuses on **incremental growth**, rather than simply ranking segments by business size.

---

## Page 3 – Product Portfolio & Mix

**Management question:** Which products are generating high-quality growth, and why?

Highlights:

- Product Growth Contribution vs Mix Impact
- Product-type quantity-mix comparison
- Quantity Mix Shift vs ASP
- Product diagnostic matrix
- Mix Offset analysis

This page provides the deepest diagnostic drill-down in the report.

---

## Page 4 – Market & Channel Performance

**Management question:** Which markets and channels are driving business performance?

Highlights:

- Revenue Growth vs Gross Margin by country
- Incremental Revenue by market
- Prior-year vs current-year channel mix
- Territory performance matrix

> Manager results represent assigned territory performance and are not normalized for market potential, sales targets, or territory difficulty.

---

# Management Implications

Based on the analysis:

### Protect core growth engines

Video Games and Mobile generated the majority of incremental revenue with relatively limited adverse product-mix effects.

### Review Computer product mix

Computer remained a growth contributor, but its increasing exposure to lower-value products materially diluted the revenue benefit generated by higher volume.

### Monitor Home & Kitchen mix

Home & Kitchen delivered strong volume and revenue growth but also experienced meaningful mix-driven ASP dilution.

### Monitor channel concentration

Web already dominates total revenue and generated nearly all of 2018 net incremental growth, increasing dependency on a single order channel.

---

# Limitations

This analysis is based on historical transactional data and should not be interpreted as causal evidence.

Specific limitations include:

- No sales quotas or targets
- No commission or incentive data
- No customer acquisition or marketing cost data
- No territory-potential normalization
- ASP movements may still contain geographic, channel, or lower-level product-mix effects
- Gross Profit analysis does not represent full operating profitability

Accordingly, the analysis identifies performance patterns and diagnostic signals rather than proving causal relationships.

---

# Skills Demonstrated

- Financial performance analysis
- Revenue variance analysis
- Price–Volume–Mix decomposition
- Profitability analysis
- Product-mix analysis
- Growth-contribution analysis
- Data modeling
- DAX
- Power Query
- Dynamic field parameters
- Interactive Power BI dashboard design
- Management-oriented data storytelling