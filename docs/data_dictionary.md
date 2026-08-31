# Data Dictionary — Price Point, STRATIFY 2.0
**Project:** Price Point  
**Phase:** 1 — Data Ingestion & Cleaning  
**Owner:** M1 — Tushar Dhingra  
**Audience:** M2 (Manu) — promo & inventory join; M3 (Paras) — EDA visualisation  
**Last updated:** Phase 1 completion

---

## transactions_clean.parquet

**Location:** `data/processed/transactions_clean.parquet`  
**Source:** `data/raw/transactions.csv`  
**Shape:** 53,704 rows × 9 columns  
**Unique key:** `(SKU_ID, Date)`

| Column | Type | Description |
|---|---|---|
| `Date` | `string` | Transaction date in ISO 8601 format `YYYY-MM-DD` |
| `SKU_ID` | `string` | Product identifier e.g. `SKU0001` |
| `Category` | `string` | Product category e.g. `Electronics` |
| `Listed_Price_INR` | `float64` | Original listed price before discount (INR) |
| `Discount_Pct` | `float64` | Discount applied as decimal e.g. `0.10` = 10% |
| `Effective_Price_INR` | `float64` | Actual transaction price paid (INR) |
| `Quantity_Sold` | `int64` | Units sold in this transaction |
| `Base_Cost_INR` | `float64` | Cost price joined from product master (INR) |
| `price_violation` | `bool` | `True` if `Effective_Price_INR < Base_Cost_INR` |

**Notes:**
- 0 null values in any column
- 0 fully duplicate rows
- 53,655 `(SKU_ID, Date)` pairs are expected to repeat — same SKU appearing across multiple dates; this is not a data error
- **3,417 rows have `price_violation = True`** — these rows are **retained** in this file but **must be excluded from elasticity model fitting in Phase 3**. Do not drop them; downstream phases need the flag to filter correctly
- All dates standardised to `YYYY-MM-DD` using `format='ISO8601'` parser; no mixed formats remain
- Join key for competitor prices: `(SKU_ID, Date)`

---

## product_master_clean.parquet

**Location:** `data/processed/product_master_clean.parquet`  
**Source:** `data/raw/product_master.csv`  
**Shape:** 49 rows × 6 columns  
**Unique key:** `SKU_ID` (one row per SKU — verified)

| Column | Type | Description |
|---|---|---|
| `SKU_ID` | `string` | Product identifier — primary join key |
| `Product_Name` | `string` | Human-readable product name |
| `Category` | `string` | Product category |
| `Base_Cost_INR` | `float64` | Cost price in INR — non-null, critical for Phase 4 optimisation |
| `margin_pct` | `float64` | Target margin as decimal e.g. `0.18` = 18% — non-null, critical for Phase 4 optimisation |
| `Shelf_Life_Days` | `float64` | Shelf life in days — **may contain nulls**, non-critical for pricing model |

**Notes:**
- `margin_pct` was renamed from `Margin_Target_Pct` in the raw file for consistency across all phases — use `margin_pct` everywhere downstream
- `Base_Cost_INR` and `margin_pct` are guaranteed non-null; any SKU missing either cannot participate in the Phase 4 price optimisation constraint: `(p − c) / p ≥ m*`
- `Shelf_Life_Days` nulls are acceptable and do not block modelling
- Join key into `transactions_clean.parquet`: `SKU_ID`

---

## competitor_prices_clean.parquet *(pending — fill after Step 6)*

**Location:** `data/processed/competitor_prices_clean.parquet`  
**Source:** `data/raw/competitor_prices.csv`  
**Shape:** *[fill after profiling]*  
**Unique key:** `(SKU_ID, Date)` *(confirm after profiling)*

| Column | Type | Description |
|---|---|---|
| `SKU_ID` | `string` | Product identifier — join key; verify exact column name from raw file |
| `Date` | `string` | Observation date in ISO 8601 format `YYYY-MM-DD` |
| *[other columns]* | *[fill]* | *[fill after Step 3 profiling]* |

**Coverage gaps (fill after Step 4):**
- SKUs in competitor data but NOT in product master: *[fill]*
- SKUs in product master but NOT in competitor data: *[fill]*
- SKUs in both: *[fill]*

**Notes:**
- In Phase 4, the optimiser enforces `p ≤ 1.10 × p_comp`. If `p_comp` is missing for a SKU, a fallback rule is required — coverage gaps documented above feed directly into M2's fallback design
- All dates standardised to `YYYY-MM-DD` using `format='ISO8601'` parser to ensure the `(SKU_ID, Date)` join with `transactions_clean.parquet` does not silently fail

---

## Cleaning Decisions Log

| Decision | Rationale |
|---|---|
| Price violation rows retained, not dropped | 3,417 rows where `Effective_Price_INR < Base_Cost_INR` are flagged via `price_violation = True` and kept; Phase 3 filters them out at model-fit time |
| Dates standardised to `YYYY-MM-DD` | Consistent format required for `(SKU_ID, Date)` joins across all three files; mixed formats cause silent zero-row join failures |
| `Margin_Target_Pct` renamed to `margin_pct` | Aligns with column name used throughout the project plan; prevents M2/M3 from having to track two names for the same concept |
| Parquet instead of CSV for outputs | Preserves exact dtypes (`bool`, `float64`, `int64`) so downstream phases do not re-infer types; loads significantly faster than CSV at this row count |
| Notebook outputs cleared before commit | Raw transaction rows and price data in notebook outputs would violate the "data stays private" principle from Section 1.2 |

---

*This document is the interface contract between M1 (Tushar) and M2 (Manu). Manu must confirm column names match his promo join requirements before the Phase 1 PR is approved. Do not start Phase 2 EDA until schema review meeting is complete.*
