# 🧶 Knit Away – Pay Rate Deep Dive (DA-210)

> Investigative analysis of the **pay rate decline** in _Knit Away_, with a focus on identifying which user segments, versions, and behavioral changes are driving the drop, and ruling out issues related to IAP exposure.

---

## 🗂 1. Overview

Recently, the team observed that:

- Overall **pay rate is trending down**.
- On **2025-11-16**, **IAP count increased**, but both **revenue** and **unique purchasers decreased**.
- There was concern that:
  - A specific **version / country / platform** might be causing the decline.
  - Changes to **IAP configuration** or **IAP show logic** might be reducing conversion.

**Core business question**

> “Tại sao pay rate giảm? Nhóm user nào, phiên bản nào, yếu tố hành vi nào đang kéo pay rate xuống, và có phải do lỗi cấu hình IAP/show hay không?”

This repository contains the notebook **`DA-210.ipynb`**, which documents a step-by-step diagnostic analysis to answer that question.

---

## 🎯 2. Objectives & Key Questions

### 2.1 Main objective

Identify the **root drivers** of the pay rate decline by decomposing metrics across:

- Versions (`1.7.2`, `1.7.3`, …)  
- Platforms (`IOS`, `ANDROID`)  
- Country groups (`US` vs `Others`)  
- User types (**new vs existing users**)  
- IAP show and pack behavior  

### 2.2 Key questions

1. Is the decline **version-specific** or **global**?
2. Is it driven by **new users** (UA quality) or **existing users** (fatigue / churn / behavior change)?
3. Did **IAP show volume or pack composition** change significantly?
4. Are there changes in **engagement** (play time) that correlate with lower conversion?
5. Are there any signs of **configuration bugs** (e.g. missing key packs, wrong offers)?

---

## 🧱 3. Data & Sources

All data is queried from **BigQuery**.

### 3.1 Core tables

- **User activity / DAU**
  - `wool-away.knit_away_flatten.user_fact`  
  - Used for:
    - `dau` (Daily Active Users)
    - `platform` (`IOS`, `ANDROID`)
    - `version` (e.g. `1.7.2`, `1.7.3`)
    - `country`, `country_group` (`United States` vs `Others`)

- **In-app purchases**
  - `wool-away.knit_away_flatten.in_app_purchase`  
  - Metrics:
    - `iap_revenue` = `SUM(event_value_in_usd)`
    - `iap_count` = number of IAP events
    - `iap_users` = distinct purchasers
    - `pay_rate = iap_users / dau`

- **IAP show logs**
  - `wool-away.dev_staging.stg_iap_show`  
  - Metrics:
    - Total IAP show count per `event_date`, `version`, `country`, `pack_name`
    - Distribution of `pack_name` before vs after key dates

- **Engagement / play time**
  - `wool-away.dev_staging.stg_user_engagement_agg`  
  - Metrics:
    - `avg_play_time_minutes` per `event_date` and `version`

### 3.2 Time window

- High-level trend analysis: **2025-10-27 → 2025-11-18**
- Focus windows:
  - **2025-11-07 → 2025-11-14** for pre / post comparisons
  - New-user cohorts (e.g. **2025-11-03 → 2025-11-11**)

---

## 📓 4. Notebook Structure (`DA-210.ipynb`)

The notebook is organized as a diagnostic workflow:

1. **Setup & config**
   - Imports (`pandas`, `numpy`, `matplotlib`, etc.)
   - BigQuery client initialization
   - Date range and parameter configuration

2. **Overall metrics & version breakdown**
   - Query `user_fact` + `in_app_purchase`
   - Compute daily:
     - `dau`, `iap_revenue`, `iap_count`, `iap_users`, `pay_rate`
   - Visualization:
     - Line charts of pay rate by date, split by `platform` and `version`

3. **Country segmentation**
   - Group countries into:
     - `United States`
     - `Others`
   - Recompute pay rate by `event_date`, `platform`, `country_group`
   - Visualization:
     - Check whether the decline is US-driven, non-US, or global

4. **New vs existing users & cohorts**
   - Compute cohort metrics:
     - `login_date` as cohort
     - `cohort_days = event_date − login_date`
   - Metrics:
     - `iap_users` per cohort day
     - `new_users` per `login_date`
     - `cohort_pay_rate = iap_users / new_users`
   - Visualization:
     - Heatmaps of cohort pay rate
   - Split users:
     - `login_date` **before** vs **after** `2025-11-01`
     - Compare `iap_revenue`, `iap_count`, `iap_users` over time

5. **IAP show & pack composition**
   - Query `stg_iap_show` for:
     - Versions `1.7.2`, `1.7.3`
     - Focus market (e.g. `United State` as per tracking column)
   - Compare:
     - `total_show` per `pack_name` in **before vs after** windows
   - Visualization:
     - Bar charts comparing pack share and show volume

6. **Engagement analysis**
   - Query `stg_user_engagement_agg`
   - Compute `avg_play_time_minutes` per `event_date`, `version`
   - Check relationship between engagement and pay rate by version

7. **Summary & interpretation**
   - Combine insights from all above steps
   - List potential root causes and non-causes

---

## 📌 5. Key Findings (Short)

1. **Version-driven effect**
   - The decline is **concentrated in versions `1.7.2` and `1.7.3`**, not all versions.

2. **Existing users are the main driver**
   - **Old cohorts** (existing users) show a stronger drop in pay rate.
   - **New user cohorts** keep a relatively stable pay rate.
   - So this is **not primarily a UA quality problem** for the analyzed period.

3. **IAP exposure looks healthy**
   - IAP show volume and `pack_name` distribution are **stable** before vs after the change.
   - No evidence of a major **missing key pack** or **severe show drop**.

4. **Engagement needs deeper attention**
   - Engagement (play time) is monitored; any meaningful drop in `avg_play_time_minutes` on problem versions likely contributes to the pay rate decline.
   - Points towards **product / content / live-ops** follow-up rather than pure configuration bug.

---

## 🧪 6. How to Run the Notebook

### 6.1 Requirements

- **Python**: 3.9+  
- **Access to GCP & BigQuery** with:
  - `wool-away.knit_away_flatten`
  - `wool-away.dev_staging`
- Recommended Python packages:
  - `pandas`
  - `numpy`
  - `matplotlib`
  - `seaborn`
  - `google-cloud-bigquery`
  - `jupyter` / `jupyterlab`

### 6.2 Setup steps

1. Create and activate a virtual environment (optional but recommended):

   ```bash
   python -m venv .venv

   # macOS / Linux
   source .venv/bin/activate

   # Windows
   .venv\Scripts\activate
