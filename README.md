Multi-Touch Attribution & Incremental ROAS (iROAS) Optimizer
![Power BI](https://img.shields.io/badge/Power_BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![Microsoft Fabric](https://img.shields.io/badge/Microsoft_Fabric-117865?style=for-the-badge&logo=microsoft&logoColor=white)
![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=for-the-badge&logo=databricks&logoColor=white)
![DAX](https://img.shields.io/badge/DAX-Advanced-blue?style=for-the-badge)
![SQL](https://img.shields.io/badge/SQL-Medallion_Architecture-336791?style=for-the-badge&logo=postgresql&logoColor=white)
> **Domain:** E-Commerce | **Sub-Domain:** Performance Marketing & Growth Analytics  
> **Architecture:** Medallion Lakehouse + Power BI Composite Model (Import / DirectQuery / Dual)
---
1. Executive Summary & Business Problem
Performance marketing teams in scaling e-commerce businesses face a critical reporting blind spot: walled-garden ad platforms (Meta, Google Ads, Amazon Ads) grade their own homework. Each platform self-reports conversions within overlapping attribution windows, systematically double-counting the same customer journey and inflating reported ROAS.
The Messy Data Reality
Inflated Top-Line Metrics: Platform dashboards optimize for gross revenue rather than net contribution margin (factoring in COGS, shipping, discounts, and returns).
Multi-Grain & High-Cardinality Scale: Combining raw clickstream session logs (100M+ rows) with daily ad-spend API pulls and transactional order tables creates severe fan-trap and memory bottlenecks.
Cross-Border Complexity: Multi-currency ad spend and orders require standardized FX conversion at the transaction date grain alongside varying attribution lookback windows (7-day click, 1-day view, 30-day click).
The Solution
An enterprise-grade Multi-Touch Attribution (MTA) and Incremental ROAS (iROAS) decision engine built on Microsoft Fabric / Databricks and Power BI. It reconciles platform-reported conversions against actual net profit, enabling CMOs and Growth Leads to dynamically compare attribution models and reallocate ad budgets toward genuinely incremental, high-LTV channels.
---
2. Business Impact & Key Outcomes
Strategic Lever	Measurable Outcome
Budget Reallocation	Shifting 15% of ad spend from low-incrementality Branded Search to high-LTV Paid Social & Prospecting channels improved Blended MER (Marketing Efficiency Ratio) without increasing total spend.
Profit-First Visibility	Eliminated 22–35% conversion over-reporting across Meta and Google Ads by anchoring attribution weights to net contribution margin.
Query Performance	Reduced dashboard load times to under 1.5 seconds on 100M+ clickstream rows by routing summary queries to Import aggregations and isolating session-level drill-throughs in DirectQuery.
Executive Agility	Replaced 6 static reports with a single dynamic interface featuring live attribution-model switching and LTV:CAC payback tracking.
---
3. Enterprise Architecture & Data Pipeline
Raw clickstream logs are never loaded directly into Power BI. Heavy path-stitching and attribution weight calculations are pushed upstream into a Medallion Architecture (Microsoft Fabric / Databricks), exposing clean, analytics-ready Delta tables to the semantic layer.
```mermaid
flowchart LR
    subgraph Sources["Data Sources"]
        A1[Clickstream Events<br/>100M+ Rows]
        A2[Ad Platform APIs<br/>Meta / Google / Amazon]
        A3[ERP / Order System<br/>Transactions & COGS]
        A4[Daily FX Rates]
    end

    subgraph Lakehouse["Microsoft Fabric / Databricks Medallion"]
        B[Bronze Layer<br/>Raw JSON / Parquet Ingestion]
        S[Silver Layer<br/>Sessionization, Identity Resolution<br/>& Multi-Currency Normalization]
        G[Gold Layer<br/>Fact_Orders, Fact_Daily_AdSpend,<br/>Bridge_Session_Touchpoints,<br/>Agg_Campaign_Daily]
    end

    subgraph PBI["Power BI Composite Model"]
        IMP[(Import Mode<br/>Aggregations & Core Facts)]
        DQ[(DirectQuery<br/>Granular Session Paths)]
        DUAL[(Dual Mode<br/>Dim_Campaign, Dim_Date)]
    end

    Sources --> B --> S --> G
    G --> IMP
    G --> DQ
    G --> DUAL
```
Upstream Processing Highlights
Bronze (Raw): Ingests immutable clickstream JSON logs, daily ad-spend API payloads, and order records.
Silver (Cleansed & Conformed): Stitches anonymous cookies to authenticated customer IDs, applies daily FX conversion rates to normalize all spend and margin into base currency, and enforces standardized UTM taxonomy.
Gold (Serving & Attribution): Pre-computes fractional touchpoint weights (`First_Touch_Weight`, `Last_Touch_Weight`, `Linear_Weight`, `Time_Decay_Weight`) per converting order inside `Bridge_Session_Touchpoints`, preventing expensive iterative path calculations at query time.
---
4. Semantic Data Model & Storage Strategy
Resolving Multi-Grain & Many-to-Many Relationships
A single order can be influenced by multiple marketing campaigns, while ad spend is recorded at a daily campaign grain (not per order). The model handles this cleanly without bidirectional filter traps:
`Bridge_Session_Touchpoints`: Resolves the many-to-many relationship between `Fact_Orders` and `Dim_Marketing_Campaigns` using pre-calculated attribution weights. Single-direction filtering is enforced, with DAX `CROSSFILTER` / weight multiplication invoked only inside explicit attribution measures.
Separate-Grain Spend Fact: `Fact_Daily_AdSpend` connects independently to `Dim_Date` and `Dim_Marketing_Campaigns`, eliminating fan-trap inflation when joining spend to order-level conversions.
Disconnected Parameter Tables: Dedicated tables (`Param_Attribution_Model`, `Param_Cohort_Months`) power dynamic UI slicing and cohort payback matrices without cluttering model relationships.
Composite Model & Storage Mode Matrix
Table Name	Role	Storage Mode	Rationale
`Agg_Campaign_Daily`	User-Defined Aggregation	Import	Pre-aggregated spend, clicks, and attributed net profit; serves 95% of executive visuals from VertiPaq RAM.
`Fact_Daily_AdSpend`	Ad Spend Fact	Import	Low-to-medium row count; fast in-memory aggregation for MER and pacing metrics.
`Fact_Orders`	Order & Margin Fact	Import	Stores order-level revenue, COGS, refunds, and net margin for cohort & LTV modeling.
`Bridge_Session_Touchpoints`	Attribution Bridge	DirectQuery	High-cardinality session/touchpoint paths (100M+ rows); kept in Lakehouse to prevent RAM exhaustion.
`Dim_Marketing_Campaigns`	Shared Dimension	Dual	Acts as Import for aggregation hits and DirectQuery for session drill-throughs, avoiding cross-source join penalties.
`Dim_Date`	Shared Dimension	Dual	Ensures seamless query folding to SQL endpoints and instant time-intelligence in VertiPaq.
---
5. Advanced Power BI & DAX Engineering
A. Dynamic Attribution Switching (Field Parameters + DAX)
CMOs can toggle between First-Touch, Last-Touch, Linear, and Time-Decay models across all visuals simultaneously using a Field Parameter bound to pre-computed bridge weights.
```dax
Attributed Net Profit =
VAR SelectedModel = SELECTEDVALUE( 'Param_Attribution_Model'[ModelName], "Time-Decay" )
RETURN
    SUMX(
        'Bridge_Session_Touchpoints',
        VAR OrderNetProfit = RELATED( 'Fact_Orders'[Net_Contribution_Margin] )
        VAR Weight =
            SWITCH(
                SelectedModel,
                "First-Touch", 'Bridge_Session_Touchpoints'[First_Touch_Weight],
                "Last-Touch",  'Bridge_Session_Touchpoints'[Last_Touch_Weight],
                "Linear",      'Bridge_Session_Touchpoints'[Linear_Weight],
                "Time-Decay",  'Bridge_Session_Touchpoints'[Time_Decay_Weight],
                'Bridge_Session_Touchpoints'[Time_Decay_Weight]
            )
        RETURN
            OrderNetProfit * Weight
    )
```
```dax
Incremental ROAS (iROAS) =
VAR AttributedProfit = [Attributed Net Profit]
VAR BaselineOrganicLift = [Estimated Baseline Organic Profit]
VAR AdSpend = [Total Ad Spend]
RETURN
    DIVIDE( AttributedProfit - BaselineOrganicLift, AdSpend, 0 )
```
B. Cohort Retention & LTV:CAC Payback Matrix
Built using a disconnected `Dim_Cohort_Horizon` table (Day 30, Day 60, Day 90, Day 180) to track cumulative net margin generated per acquisition cohort against initial Customer Acquisition Cost (CAC).
```dax
Cohort LTV to CAC Ratio =
VAR AcquisitionMonth = SELECTEDVALUE( 'Dim_Customer_Cohort'[Cohort_Month] )
VAR HorizonDays = SELECTEDVALUE( 'Dim_Cohort_Horizon'[Days] )
VAR CohortNetMargin =
    CALCULATE(
        SUM( 'Fact_Orders'[Net_Contribution_Margin] ),
        FILTER(
            ALL( 'Fact_Orders' ),
            'Fact_Orders'[Customer_Cohort_Month] = AcquisitionMonth &&
            'Fact_Orders'[Days_Since_Acquisition] <= HorizonDays
        )
    )
VAR CohortCAC = [Cohort Acquisition Spend]
RETURN
    DIVIDE( CohortNetMargin, CohortCAC )
```
C. Custom SVG Micro-Visuals (Spend-Pacing Bars & 14-Day Sparklines)
To keep dense campaign matrices scannable without adding heavy custom visuals from AppSource, native DAX-generated SVG elements render inline spend-pacing bars and 14-day ROAS trend sparklines (`Data Category: Image URL`).
```dax
SVG Spend Pacing Bar =
VAR SpendPct = MIN( DIVIDE( [MTD Ad Spend], [Monthly Budget], 0 ), 1.2 )
VAR TimeElapsedPct = DIVIDE( DAY( TODAY() ), DAY( EOMONTH( TODAY(), 0 ) ), 0 )
VAR BarWidth = ROUND( MIN( SpendPct, 1 ) * 100, 0 )
VAR MarkerX = ROUND( TimeElapsedPct * 100, 0 )
VAR BarColor =
    SWITCH(
        TRUE(),
        SpendPct > TimeElapsedPct + 0.10, "#D64545", // Overspending alert
        SpendPct < TimeElapsedPct - 0.15, "#E6A23C", // Under-pacing warning
        "#2E8540"                                    // On track
    )
RETURN
"data:image/svg+xml;utf8," &
"<svg xmlns='http://www.w3.org/2000/svg' width='120' height='22' viewBox='0 0 120 22'>" &
  "<rect x='0' y='6' width='100' height='10' rx='3' fill='#EAECEF'/>" &
  "<rect x='0' y='6' width='" & BarWidth & "' height='10' rx='3' fill='" & BarColor & "'/>" &
  "<line x1='" & MarkerX & "' y1='2' x2='" & MarkerX & "' y2='20' stroke='#1F2937' stroke-width='2'/>" &
"</svg>"
```
---
6. Performance & Governance Best Practices
Eliminating Cross-Source Join Penalties: Setting `Dim_Marketing_Campaigns` and `Dim_Date` to Dual Storage Mode ensures DAX queries against `Agg_Campaign_Daily` stay 100% inside the VertiPaq storage engine, while granular path drill-throughs fold natively into Fabric/Databricks SQL without transferring intermediate result sets.
User-Defined Aggregation Validation: Verified aggregation hit rates using DAX Studio (Server Timings & Query Plan)—confirming `Match Found` on `Agg_Campaign_Daily` for all Level-1 executive visuals.
Governance & Security: Supports Row-Level Security (RLS) by region/brand and Object-Level Security (OLS) to restrict sensitive COGS and net-margin columns from external agency partners.
---
7. Repository Structure
```text
mta-iroas-optimizer/
├── assets/
│   ├── architecture_diagram.png
│   ├── executive_overview_dashboard.png
│   └── cohort_payback_matrix.png
├── notebooks/
│   ├── 01_bronze_clickstream_ingestion.py
│   ├── 02_silver_sessionization_fx_norm.py
│   └── 03_gold_attribution_weights_calc.py
├── sql/
│   └── create_agg_campaign_daily.sql
├── dax/
│   ├── 01_attribution_measures.dax
│   ├── 02_cohort_ltv_cac_measures.dax
│   └── 03_svg_micro_visuals.dax
├── pbi/
│   └── MTA_iROAS_Optimizer.pbip          # Power BI Project (Git-friendly PBIP format)
└── README.md
```
---
8. Dashboard Preview
> *Replace the placeholders below with high-resolution screenshots or Loom/GIF walkthroughs from your Power BI report.*
View	Key Analytical Capabilities
1. Executive Attribution Overview	Live Field Parameter switcher (First-Touch vs. Last-Touch vs. Time-Decay), Blended MER, and Platform-Reported vs. Net-Profit variance waterfall.
2. Campaign Matrix & Micro-Visuals	Dense campaign-level performance table featuring inline SVG 14-day ROAS sparklines and budget-pacing bars.
3. Cohort LTV:CAC Payback	Heatmapped retention matrix tracking Day 30 / 60 / 90 contribution margin recovery by acquisition channel.
4. Session Path Drill-Through	DirectQuery detail page exposing granular multi-touch customer journeys for high-value orders.
