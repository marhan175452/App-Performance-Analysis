# App Performance Campaign Analysis — SQL KPIs

This repository contains an Oracle-SQL workflow that identifies engagement opportunities around a “five-year app anniversary” theme for cardholders. The logic builds clean base populations, derives channel registration/usage cohorts, and surfaces targeted counts you can lift directly into dashboards or campaign lists.

---

## What it does

- **Builds base populations**
  - Open, active card accounts
  - Digitally registered customers (WEB or APP)
  - Valid contact-detail cohorts (email present & emailable, mobile present)
  - Channel cohorts (APP customers; WEB-only customers)
  - Product-active cohorts (card transactions in the last 12 months)
  - Recent channel activity (WEB events in the last 90 days)

- **Computes campaign-ready counts**
  - Product-active + emailable + valid email **but never registered digitally**
  - **WEB-only** product-active + emailable + valid email
  - **WEB-only** product-active + emailable **with no valid email** but **WEB-active in 90 days**
  - For each of the above, the subset **missing a valid mobile number**

- **Keeps outputs simple**
  - Final queries return single-row counts per metric that can be dropped into a reporting layer.

---

## Entities & staging tables

> All names are generic and warehouse-agnostic; adjust schema prefixes to your environment.

| Table | Purpose |
|------|---------|
| `CUST_OPEN_CARD` | Open/active cardholders enriched with application source metadata. |
| `DIGITALLY_REGISTERED` | Customers observed on **WEB** or **APP** at least once. |
| `CUST_WITH_VALID_EMAIL` | Customers with a syntactically valid, non-placeholder email address. |
| `CUST_EMAIL_OPT_IN` | Customers flagged as emailable for marketing. |
| `CUST_WITH_VALID_MOBILE` | Customers with a mobile (or home) number matching mobile format. |
| `APP_CUSTOMERS` | Customers active on mobile channel (non-WEB/OHI). |
| `WEB_ONLY_CUSTOMERS` | Customers seen on WEB but not in APP cohort. |
| `CARD_TXN_12M` | Card accounts with transactions in the last 12 months. |
| `ACTIVE_OPEN_CARD_12M` | Open cardholders mapped to `CARD_TXN_12M` (product-active). |
| `WEB_ACTIVE_90D` | Customers with WEB events in the last 90 days. |

---

## KPI logic (human-readable)

### A) Product-active, emailable, valid email, **never registered digitally**
- Join: `CUST_WITH_VALID_EMAIL` → `CUST_EMAIL_OPT_IN` → `ACTIVE_OPEN_CARD_12M`
- Exclude: any `CUSTOMER_ID` present in `DIGITALLY_REGISTERED`
- Output: single count
- **Variant:** of the above, exclude those present in `CUST_WITH_VALID_MOBILE` to get “missing mobile” count

### B) **WEB-only** product-active, emailable, valid email
- Join: `WEB_ONLY_CUSTOMERS` → `CUST_EMAIL_OPT_IN` → `CUST_WITH_VALID_EMAIL` → `ACTIVE_OPEN_CARD_12M`
- Output: single count
- **Variant:** filter out `CUST_WITH_VALID_MOBILE` for “missing mobile” count

### C) **WEB-only**, **no valid email**, WEB-active (90d), emailable, product-active (12m)
- Join: `WEB_ONLY_CUSTOMERS` → `CUST_EMAIL_OPT_IN` → `WEB_ACTIVE_90D` → `ACTIVE_OPEN_CARD_12M`
- Exclude: those present in `CUST_WITH_VALID_EMAIL`
- Output: single count
- **Variant:** additionally exclude `CUST_WITH_VALID_MOBILE` for “missing mobile” count

---

## Contact & activity rules

- **Valid email**: `EMAIL_ADDR` is non-null, matches `%_@__%.__%`, and is not a placeholder like `no.email@address.com`.
- **Mobile present**: `MOBILE_PHONE` is non-null **or** `HOME_PHONE` starts with `07%` (tweak to local numbering).
- **WEB-only**: observed in WEB logon data and **not** present in APP events.
- **Product-active 12m**: card transactions observed within a trailing 12-month window.
- **WEB-active 90d**: WEB events observed in a trailing 90-day window.

---

## Outputs

The repository ships with **count queries** that return single rows per metric. These are intended to be lifted into a BI layer, KPI tile, or campaign sizing slide. If you prefer a table sink, mirror prior patterns (e.g., `METRIC_RESULTS(METRIC, FIGURES)`) and wrap each `SELECT` with an `INSERT`.

---

## Assumptions & notes

- **Dates:** Windows use `SYSDATE`, `NEXT_DAY`, and `ADD_MONTHS` with week-ending Saturday anchors where relevant.
- **Joins:** Keys are on `CUSTOMER_ID` (and `CARD_ACCT_ID` for card facts).
- **Channels:** `WEB` and `APP` are logical channel flags inferred from logon/activity sources.
- **Field hygiene:** Names are intentionally generic; adapt to your warehouse dictionary if needed.

---

## Extending the analysis

- Add **regional** or **product family** filters to refine campaign cohorts.
- Split counts by **offer type**, lifecycle band, or tenure to prioritize outreach.
- Persist cohorts to a **contactable universe** table for downstream activation.
- Pair with KPI deltas (week-over-week, month-over-month) for trend insights.


