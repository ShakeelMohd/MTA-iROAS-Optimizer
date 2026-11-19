# 📊 Multi-Touch Attribution & Incremental ROAS (iROAS) Optimizer

![Power BI](https://img.shields.io/badge/Power_BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=for-the-badge&logo=databricks&logoColor=white)
![Microsoft Fabric](https://img.shields.io/badge/Microsoft_Fabric-117865?style=for-the-badge&logo=microsoft&logoColor=white)
![DAX](https://img.shields.io/badge/DAX-Advanced-blue?style=for-the-badge)
![SQL](https://img.shields.io/badge/SQL-Medallion_Architecture-lightgrey?style=for-the-badge)

> **An enterprise-grade marketing analytics solution built to eliminate ad-platform double-counting, model multi-touch customer journeys at scale, and optimize marketing spend around true Net Margin & LTV:CAC payback.**

---

## 📌 Executive Summary & Business Problem

Ad platforms (Meta, Google Ads, Amazon Ads) operate in walled gardens and routinely self-report inflated conversions by claiming 100% credit for overlapping customer journeys. Relying on platform-reported top-line ROAS leads to misallocated budgets—over-funding low-incrementality channels (like Branded Search) while starving true upper-funnel growth drivers.

### Key Data Challenges Solved
* **De-duplicating Conversion Credit:** Reconciled multi-channel journeys across hundreds of millions of raw clickstream events against actual ERP/order ledger transactions.
* **Profitability Over Top-Line Revenue:** Shifted optimization metrics from gross revenue to **Net Contribution Margin** (accounting for COGS, discounts, shipping, and refunds) and **Blended MER** (Marketing Efficiency Ratio).
* **Cross-Grain & Cross-Currency Integration:** Unified daily ad-spend API pulls (USD/EUR/INR) with order-level transactions and session-level clickstreams across varying attribution windows (1-day view, 7-day click, 28-day click).

---

## 🏗️ Enterprise Architecture & Data Modeling

To handle 100M+ clickstream rows without sacrificing sub-second dashboard interactivity or blowing up Power BI capacity RAM, this solution implements a **Medallion Architecture** paired with a **Power BI Composite Model**.

```text
[Raw Clickstream Logs] ──┐
[Ad Platform APIs]     ──┼──► [Bronze/Silver] ──► [Gold Layer (Fabric/Databricks)] ──► [Power BI Semantic Model]
[ERP Order Ledger]     ──┘     (Deduplication &     (Pre-aggregated Session Paths &      (Hybrid Import + DirectQuery
                                Currency FX)         Attribution Weights)                 with User Aggregations)
