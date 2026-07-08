# From Chaos to Star Schema: An Elaborated Data Modeling Walkthrough

> **A behind-the-scenes look at how a 23-table "nightmare dataset" was rebuilt, table by table, into a clean, secure, production-style star schema in Power BI.**

This document is the companion to my main `README.md`. Where the README tells you *what* the project is, this file tells you *how* it was built — the reasoning, the trade-offs, the debugging moments, and the small tricks that separate a "working" data model from a *trustworthy* one.

---

## Table of Contents

1. [The Setup: A Deliberately Broken Data Model](#1-the-setup-a-deliberately-broken-data-model)
2. [The Five Ground Rules](#2-the-five-ground-rules)
3. [Phase 1 — Reconnaissance: Reading the Chaos](#3-phase-1--reconnaissance-reading-the-chaos)
4. [Workspace Strategy: Organizing Power Query Before Touching Anything](#4-workspace-strategy-organizing-power-query-before-touching-anything)
5. [Phase 2 — Building the Dimensions](#5-phase-2--building-the-dimensions)
6. [The Fact vs. Dimension Decision Framework](#6-the-fact-vs-dimension-decision-framework)
7. [Phase 3 — Building the Facts](#7-phase-3--building-the-facts)
8. [Phase 4 — Final Polish](#8-phase-4--final-polish)
9. [Row-Level Security](#9-row-level-security)
10. [The Transformation: Before vs. After](#10-the-transformation-before-vs-after)
11. [Techniques & Tricks — Quick Reference](#11-techniques--tricks--quick-reference)
12. [Lessons Carried Forward](#12-lessons-carried-forward)

---

## 1. The Setup: A Deliberately Broken Data Model

Every real Power BI project eventually collides with the same problem: a report looks fine until someone asks *"why don't these two numbers match?"* — and the honest answer is almost never "the visual is wrong." It's the **model** underneath it.

This project started from a dataset engineered to reproduce that exact pain: **23 raw tables**, no discernible star schema, many-to-many relationships firing in every direction, duplicate tables from bad imports, cryptic technical IDs, mixed naming conventions, and at least one outright "junk" table with a single meaningless column.

The goal wasn't to build a dashboard. It was to take that spider-web of tables and turn it into a **healthy, testable, secure star schema** — using the exact same discipline you'd want to see in a professional BI engagement.

---

## 2. The Five Ground Rules

Before a single table was touched, five standards were locked in. These weren't just style preferences — they were the constitution the entire rebuild had to obey, from the first dimension to the last measure.

| # | Rule | Why it matters |
|---|------|-----------------|
| 1 | **Star schema only.** Fact tables sit in the center, surrounded by dimensions. Facts are *never* connected directly to other facts. | Prevents ambiguous filter paths and circular logic — the #1 cause of "wrong numbers" bugs. |
| 2 | **Know the grain before touching a table.** Say out loud what one row represents. | Merging tables of different grains without knowing it is the single most common modeling mistake (see the fan-out incident below). |
| 3 | **Every column must earn its place.** If it's not needed for analysis, drop it. | Unused columns bloat file size, slow refreshes, and confuse report builders. |
| 4 | **Protect the numbers.** Know your key totals by heart; re-check them after every structural change. | A model can *look* correct and still silently duplicate or drop rows. Trust nothing — verify. |
| 5 | **Define standards up front, and follow them until the end.** | English only. `snake_case` for every table/column. Prefixes: `dim_` for dimensions, `fact_` for facts. Surrogate keys always end in `_key` (to distinguish them from natural/source IDs like `_id`). All table, column, and value names must be human-friendly — no raw technical labels from the source system. |

These five rules were re-applied, quite literally, at every single step from here on.

---

## 3. Phase 1 — Reconnaissance: Reading the Chaos

Before reshaping anything, every one of the 23 tables was opened in Table View just to build a mental map: *what is this business, and what role does each table play?*

No relationships were trusted at this stage — they'd all be rebuilt from scratch anyway. This phase was purely about pattern recognition.

### What the recon revealed

| Table | First impression | Verdict |
|---|---|---|
| `addresses` | Street/city info | Supporting context → folds into Customer |
| `campaign_log` | Daily spend, clicks, impressions per campaign | Fact candidate (event-level, numeric, dated) |
| `campaign_skus` | Campaign name + comma-separated list of product codes | Needs unpivoting — a bridge between two dimensions |
| `cities` / `regions` | Geography lookups | Supporting/dimension material |
| `customer_master` | Companies (B2B), segments, account managers, hash keys | **The** customer dimension anchor |
| `customer_contacts` | Multiple contacts per company | One-to-many — needs grain resolution before merging |
| `dimension_order` | A single column of unexplained IDs | Garbage — flagged for deletion |
| `exchange_rate` | Currency/rate/date | Interesting but ultimately unconnectable to the model |
| `inventory` | Product × month-columns of stock counts | Fact candidate, but shaped wrong (wide, not long) |
| `invoice_lines` / `invoices` | Classic header/detail split | Fact candidates, same pattern as orders |
| `order_line_items` / `orders_2025` / `orders_2026` | Split-by-year orders + a details table | The **biggest** fact-building challenge in the project |
| `payments` | Transaction-level payment events | Fact candidate |
| `products` | Rich product attributes | **The** product dimension anchor |
| `sales_targets` | Monthly planned revenue | Small standalone fact |
| `security` | Employee email → allowed region | Row-Level Security source |
| `sheet1` | Identical to `shipments` | Duplicate import — delete |
| `shipments` | Ship/delivery dates per order | Fact candidate |
| `subcategories` | Category\|Subcategory pipe-delimited pairs | Needs splitting before use |
| `user_details` | Credit limits, phone numbers, keyed like customers | Same entity as "customer," different name |

**Key early insight:** the dataset used two different names — *"customer"* and *"user"* — for the exact same business entity. This is a classic symptom of multiple source systems (or departments) feeding one warehouse, and it had to be resolved with a single naming decision (see [Section 5](#5-phase-2--building-the-dimensions)) rather than left ambiguous.

---

## 4. Workspace Strategy: Organizing Power Query Before Touching Anything

With ~23 raw queries about to multiply into dozens of intermediate and final tables, Power Query's query list would become unmanageable fast. So before any transformation work began, four query groups were created:

```
📁 1_stage        → all raw, untouched source queries land here
📁 2_dimensions   → clean dimension tables get built and stored here
📁 3_facts        → clean fact tables get built and stored here
📁 4_support      → tables that are neither fact nor dimension (e.g. security)
```

This is a small thing, but it pays off constantly: at any point in the build, you can jump straight to "what dimensions exist so far?" without hunting through 40+ queries in an "Other Queries" bucket. Every new query created during the build was immediately dragged into the right folder — no exceptions.

---

## 5. Phase 2 — Building the Dimensions

The workflow for every dimension followed the same loop:

```
Pick a topic → Gather all related raw tables → Pick the biggest/most complete as the anchor
→ Reference it (never edit the raw table directly) → Merge in supporting tables one at a time
→ Test row count after each merge → Drop unnecessary columns → Rename to standards
→ Disable load on all now-obsolete raw tables
```

### 5.1 Dimension: Customer (the big one)

Six raw tables were consolidated into `dim_customer`:
`customer_master`, `customer_contacts`, `user_details`, `addresses`, `cities`, `regions`.

**Step 1 — Anchor and clean.**
`customer_master` became the foundation via a *Reference* (not a duplicate — this preserves a live link to the raw stage table and keeps the Power Query dependency graph clean). A dummy test record (`customer_id = 999`) was filtered out immediately, locking the baseline at **60 real customers** — the number that had to survive every subsequent merge untouched.

**Step 2 — The grain trap.**
Merging `customer_contacts` directly caused customer count to balloon far past 60. The cause: `customer_contacts` has a *different grain* — one row per **contact**, not per **customer** (some companies had 4 contacts). Merging two tables of different grains without noticing is exactly how "duplicated facts" bugs are born.

Three fixes were considered:
- Group/aggregate contacts into a list per customer (grain conversion), or
- Split contacts into their own dimension with a relationship, or
- **Filter down to only the primary contact** (chosen — since the reporting need was simple, this kept everything inside one clean dimension without artificially inflating row counts).

After filtering to `is_primary = true`, the row count matched 60-for-60, and the merge became safe.

**Step 3 — The duplicate-check trick.**
Before trusting any merge key, group the *would-be* key column and count rows. If every count is exactly `1`, the key is unique and the merge is safe. If you see any `2`s, you have a fan-out waiting to happen. This single technique — check the cardinality *before* you merge, not after — became the backbone of every subsequent merge in the project.

**Step 4 — The naming decision.**
With `user_details` merged in, the model had two competing terms for the same entity: *customer* vs. *user*. Rather than trying to satisfy every department's preferred term, the decision rule applied was: **follow the majority.** Since most tables used "customer," the entire model standardized on `dim_customer` and the term "customer" everywhere — a deliberate, documented decision rather than an accidental inconsistency.

**Step 5 — Star, not snowflake.**
`addresses`, `cities`, and `regions` all *could* have become their own linked dimension (a snowflake pattern). Per Rule #1 (star schema only), they were instead folded directly into `dim_customer`. Before each merge, the **model-view cardinality check** was used as a second safety net: temporarily drawing a relationship between two tables in the Model view and reading Power BI's auto-detected cardinality (1:1 or 1:many was safe; anything suggesting many:many was a red flag) before committing to the merge in Power Query.

**Step 6 — Deduplication of information.**
`regions` was found to add *zero* new information — the region name already existed via the `cities` merge. Rather than merge it "just in case," it was left out entirely. This is Rule #3 in action: repeating the same fact in multiple places doesn't add value, it adds confusion and inconsistency risk.

**Step 7 — Housekeeping.**
Dropped: `address_id` (only needed for joining, not reporting), hash keys and source IDs (large string columns with zero analytical value — and a noted real-world case where removing these alone shrank a model's refresh time by ~20%), the "is primary" flag (now meaningless — it's always true for the filtered contact), and account manager (out of scope). Renamed every surviving column to `snake_case`. Cleaned up derived text like `cities.region_name` down to a simple `region`.

**Step 8 — Retirement.**
All six source tables had their **"Enable Load"** flag disabled — keeping them available for reference inside Power Query without cluttering the data model or wasting refresh time on tables nobody queries anymore.

### 5.2 Dimension: Product

A much shorter loop, but with its own instructive wrinkles:

- **Anchor:** `products` (60 real products after filtering a dummy "DO NOT USE" test row).
- **Duplicate check:** grouped by product code — clean, no duplicates.
- **The pipe-delimited trap:** `subcategories` stored `category|subcategory` as a single text value. This was split into two columns using **Split Column → By Delimiter**, then each new column renamed.
- **The silent join killer — case mismatch:** the subcategory values in one table were lowercase, while the corresponding key in the product table was capitalized ("Title Case"). A merge on mismatched casing returns *only nulls*, with no error thrown — a classic silent failure. Fixed with **Transform → Capitalize Each Word** before merging, applied consistently to the *reporting-facing* casing standard (nicer to look at in visuals than the raw lowercase).
- **No natural surrogate key:** unlike customers, the source system provided no numeric product ID — only a business key (`product_code`). An **Index Column** starting at 1 was added and renamed `product_key`, becoming the model's first fully model-generated surrogate key.
- **A judgment call on scope:** long free-text product descriptions were deliberately dropped. The reasoning: Power BI is an analytics tool, not a document viewer — long text fields belong in operational systems, not semantic models.
- **Column ordering as a courtesy:** columns were manually reordered so `product_key` leads, followed by `subcategory` then `category` — mirroring the natural drill-down hierarchy a report builder would expect (Product → Subcategory → Category).

---

## 6. The Fact vs. Dimension Decision Framework

With Customer and Product finished, every remaining raw table was some flavor of *event* — orders, invoices, payments, shipments, campaigns. Here the project had to formalize a rule that isn't obvious to newcomers:

> **A table isn't a fact just because it has dates and numbers. It's a fact if the data changes constantly and represents a transaction. If it's largely static and merely describes something, it's a dimension — no matter how "event-like" it looks on the surface.**

The clearest test case was the campaign data: `campaign_log` *looked* like a fact (it has a start date, an end date, a budget). But the start date, end date, and budget almost never change once a campaign exists — they're **descriptive** attributes of the campaign entity. Meanwhile the *daily* spend/impressions/clicks rows genuinely move every single day. So the table was split: the static, describing columns became `dim_campaign`; the daily transactional columns became `fact_campaign_spend`.

### The Header/Detail pattern (the deepest concept in this project)

Almost every transactional business system stores events as a **header** (one row per event — order, invoice, shipment) plus **details** (multiple rows per event — line items, products, quantities). This is standard OLTP design, and it appears constantly in Orders, Invoices, and similar tables here.

The classic beginner mistake: treat the header and the detail as *two separate fact tables*, then connect them to each other using the shared order ID. This technically "works" but is a landmine — connecting fact-to-fact breaks the star schema, produces ambiguous filter paths, tanks performance, and eventually produces wrong numbers in visuals that pull from both.

**The rule applied instead:** measures almost always live at the *lowest* grain — the details. So the **fact table is built from the detail rows**, and the header is used purely as a **source of dimensional context** (dates, customer, channel, status) that gets merged *into* the fact and/or split off into new dimensions. You can always aggregate detail-level facts back up to header-level totals — but you can never safely go the other direction.

This single principle — "facts are built from the finest grain, headers just supply context" — was reused for Orders, Invoices, and the later Order Fulfillment fact.

---

## 7. Phase 3 — Building the Facts

### 7.1 `fact_sales` — the flagship fact table

This was the most complex build in the project, assembled from `orders_2025`, `orders_2026`, and `order_line_items`.

**Step 1 — Stitch the split-year tables.**
Two identical-grain tables (`orders_2025`, `orders_2026`) were combined with **Append Queries → As New Query**. Appending revealed a subtle mismatch: 2025 carried a legacy reference column and a "gift message" column that 2026 didn't have — both were pure noise (all-null or unused for reporting) and were dropped so the two years aligned perfectly before continuing.

**Step 2 — Extract the junk dimension.**
The header table contained three unrelated flag-like columns (`order_channel`, `status`, `priority`) — each independently useful for filtering, but with no natural relationship to one another, and each too small to justify its own dimension table. The solution: bundle all three into a single **junk dimension** (`dim_order_flags`), built by referencing the header, keeping only those three columns, removing duplicate rows (down to just 14 unique combinations), and adding a surrogate `flag_key` index.

**Step 3 — Manual data enrichment (with an explicit warning label).**
`order_channel` stored only numeric codes (10, 20, 30, 40) with no built-in meaning. Since no lookup table existed in the source, a small mapping table was created manually inside Power BI (**Home → Enter Data**) translating each code to a friendly name (Online Store, Retail Partner, Wholesale, Field Sale), then merged into the junk dimension and the raw codes dropped.

> ⚠️ **A deliberately flagged risk:** manually-entered mapping tables are a liability if the source system starts emitting new codes later — the mapping silently goes stale and produces nulls in reports. The correct long-term fix is to push this mapping back into the source system (or a governed reference table) rather than leaving it as a hand-maintained artifact inside the semantic model. It was kept here only because the project scope didn't include pipeline ownership — but it's documented as technical debt, not a best practice.

**Step 4 — Build the fact from the details, merge in the header for context.**
`order_line_items` (the true transaction grain) was referenced as `fact_sales`. The header table was merged in on `order_id` (always **Left Outer Join** — Inner Join risks silently dropping fact rows whose header is missing, which is far more dangerous than a few unmatched nulls).

**Step 5 — The "protect the number" ritual.**
Before any merge that touches the fact table, a single trusted measure (`line_total`, summed on a report card visual) was pinned as a sentinel value. After every single subsequent merge, that card was re-checked. Any unexpected change meant *stop and investigate before continuing* — never push forward on a fact table that has drifted from its known-good total.

**Step 6 — The fan-out incident (a genuine debugging story).**
Merging in the product lookup — joined by **product name** instead of a stable ID, since names were the only shared field — caused the sentinel total to spike upward. Root cause, found by grouping the dimension by product name and filtering for count > 1: `dim_product` secretly contained **duplicate rows for two products**, with inconsistent completeness (one row had a `brand`/`price`, the identical-named other row didn't). Every sales line for those two products was now matching *two* dimension rows instead of one, silently duplicating the fact rows.

The fix: identify the "bad twin" using its `source_id`, and filter it out of `dim_product` at the step just after the raw load (not as an afterthought at the end — fixing it upstream keeps the dimension itself clean for every consumer, not just this one fact). Post-fix, the sentinel number returned to its original value — merge confirmed safe.

> This incident is a useful illustration of *why* Rule #4 ("protect the numbers") isn't a formality — it's the only thing that caught a real data-quality bug before it reached a report.

**Step 7 — Convert every text lookup into a real surrogate key.**
`customer_name` → `customer_id` (via lookup merge against `dim_customer`), `product_name` → `product_key`, and the three junk-dimension text fields → `flag_key` — each one tested against the sentinel total immediately after merging.

**Step 8 — Deduplicate repeated context, but recognize what's genuinely different.**
Customer city/region were dropped from the fact (already living in `dim_customer` — single source of truth). But **ship-to city** and **bill-to city** were *kept as transactional* fields, because they represent a per-order shipping decision, not a permanent customer attribute — a customer's *billing address* doesn't change order to order, but *where a specific order ships* can. This distinction — master data vs. transactional data that happens to share a name — is a recurring theme in real-world modeling and was called out explicitly rather than resolved by rote rule-following.

**Step 9 — Drop pre-aggregated values.**
The header's `order_total` was removed once the fact carried line-level totals — a rule of thumb applied throughout: never trust a pre-aggregated number from a source system when the same total can be derived from detail rows already present in the model. Aggregates go stale; detail-level sums don't.

**Step 10 — A second dimension built mid-fact, and a role-playing dimension.**
Realizing the ship-to/bill-to cities deserved their own lookup (to shrink the fact and allow future enrichment like country-level geography), a fresh `dim_geo` was built from the earlier-skipped `cities` table — proof that dimension discovery is iterative, not a one-shot upfront exercise. `dim_geo` was then connected **twice** to `fact_sales` (once for shipping, once for billing) — a **role-playing dimension**, with one relationship active and the second forced inactive (to avoid Power BI's single-active-relationship constraint), to be activated via DAX (`USERELATIONSHIP`) only when a report specifically needs the billing-city angle.

**Final shape:** IDs first, then dates, then measures last — a deliberate, consistent column ordering applied to every fact table in the model for report-builder ergonomics.

### 7.2 `fact_inventory` — the "easy win"

A deliberate pacing choice after the marathon of `fact_sales` — proof that not every fact is a slog.

- **The problem:** the raw table stored one row per product with **one column per month** — human-readable, but structurally unusable for reporting (you can't filter or aggregate "January" as a column name).
- **The fix — Unpivot:** select all month columns → **Transform → Unpivot Columns**. In one operation, 12 wide columns become two tall columns (`month`, `units`) — turning a spreadsheet-style table into a proper long-format fact.
- Converted the resulting text-based month values to real Date types, looked up `product_key` via the standard merge-and-test pattern, and renamed to standards. Connected to `dim_product` — never directly to `fact_sales` — preserving the "facts never touch facts" rule even when two facts share a dimension.

### 7.3 `fact_campaign_spend` + the Factless Fact

- `dim_campaign` was split off as described in [Section 6](#6-the-fact-vs-dimension-decision-framework); the remaining daily transactional columns became `fact_campaign_spend`, looked up against the new campaign dimension by name and converted to `campaign_key`.
- `campaign_skus` stored a comma-separated list of product codes per campaign — a many-to-many bridge, not a dimension in its own right. **Split Column → By Delimiter → Rows** (not Columns) exploded the list into one row per product-campaign pair. A **Trim** transform cleaned up leading whitespace left behind by the "comma-space" delimiter style — a small but easy-to-miss step, since un-trimmed whitespace silently breaks later exact-match merges.
- Both `campaign_name` and `product_code` were converted to their respective surrogate keys, and the result — a table containing **only keys, no measures** — was named and classified as a **factless fact** (`fact_promotion_coverage`): a valid, named pattern for tracking *that an association happened* (which products were promoted in which campaign) without measuring *how much* of anything. It exists purely as a many-to-many bridge between two dimensions that a plain relationship couldn't otherwise express cleanly inside a star schema.

### 7.4 `fact_order_fulfillment` — the Accumulating Snapshot

The final and most conceptually advanced fact table, built from five separate event tables tied to one underlying business process:

```
Order → Ship → Deliver → Invoice → Pay
```

**The naive approach (rejected):** build one fact table per process step (5 fact tables total) mirroring the source system 1:1. Rejected because every step ultimately repeats the *same monetary value*, and duplicating that value five times across five fact tables creates redundant, hard-to-reconcile numbers with little added analytical value — the business rarely needs to compare "order revenue" against "invoice revenue" against "payment revenue" side by side, because in a healthy process they're the same number.

**The real business need, once reframed:** not *how much*, but *how long* — how many days from order to shipment, from shipment to invoice, from invoice to payment. Those questions are about **milestone dates within a single process**, not separate transactions.

**The pattern that fits: an Accumulating Snapshot Fact** — one row per business process instance (per order), with a *column for every milestone date* in that process. As the order moves through its lifecycle, the same row gets progressively "filled in" with more dates.

**Build sequence:**
1. Start from the already-cleaned `orders` table as the **spine** — one row per order, keeping only `order_id`, `customer_id` (via lookup), and `order_date`.
2. Sequentially merge in `shipments` (→ `ship_date`, `delivery_date`), `invoices` (→ `invoice_date`), and `payments` (→ `pay_date`, joined via `invoice_id` since payments reference invoices, not orders directly — requiring the invoice ID to be temporarily retained through an extra merge step specifically to enable that final join).
3. Each merge used **Left Outer Join** and pulled in only the date column needed — deliberately discarding duplicate customer names, amounts, and other data already captured elsewhere in the model.

**The payoff:** with all five raw tables (`invoices`, `invoice_lines`, `payments`, `shipments`, `sheet1`) now redundant — their essential information fully absorbed into either `fact_sales` (money) or `fact_order_fulfillment` (timing) — they were disabled from load entirely, collapsing five separate, overlapping event tables into one purpose-built fact.

### 7.5 Final cleanup pass

The last handful of leftover tables were resolved individually:

| Table | Action | Reasoning |
|---|---|---|
| `sheet1` | **Delete** (not just disable) | Confirmed exact duplicate of `shipments` — a known import artifact, permanently removable |
| `dimension_order` | **Delete** | Confirmed as pure garbage — no usable content |
| `exchange_rate` | **Disable load** (kept, not deleted) | No current connection point to the model, but plausible future relevance — disabled rather than destroyed |
| `sales_targets` | Promoted to a small standalone `fact_sales_targets` | Genuinely transactional (planned revenue by month) — too small for its own dimension, but real fact content |
| `security` | Renamed/cleaned, held for the RLS phase | Deliberately handled last — see [Section 9](#9-row-level-security) |

A guiding principle behind these calls, stated explicitly throughout the build: **don't keep a table "just in case."** Every table that survives into the final model should have an identified use. Genuinely edge-case source tables are better served by a small, purpose-built side model than by bloating the core model everyone else depends on.

---

## 8. Phase 4 — Final Polish

With the star schema structurally complete, the model still needed the layer of trust-building work that separates a "technically correct" model from one a team can actually hand off and rely on.

### 8.1 Standards audit

A full pass through every table and column, checking against the five ground rules from Section 2 — catching a couple of stray naming inconsistencies missed mid-build (naming discipline drifts under momentum; a dedicated audit pass catches what real-time vigilance misses).

### 8.2 Date format standardization

Dates were arriving in inconsistent, verbose formats (full month names, taking up significant visual space in report tables). A single compact format (`YYYY-MM-DD`-style) was chosen once and applied uniformly across **every date column in every table** — a small, tedious, easily-skipped step that materially improves how a report reads to an end user.

### 8.3 Summarization defaults

Power BI auto-flags numeric columns as summable by default — which is convenient until someone accidentally drags a "budget" or "customer ID" column onto a card and gets a nonsensical `SUM`. Every numeric column across the model was reviewed and explicitly set to the correct default aggregation behavior (or **"Don't Summarize"** where summing/averaging makes no analytical sense) — a courtesy to every future report builder who won't have the modeler's context.

### 8.4 The Date Dimension — built with `CALENDARAUTO()`

Rather than hand-maintaining a static calendar table (a well-known source of stale, forgotten date ranges in older BI stacks), the date dimension was generated with a single DAX expression:

```dax
dim_date = CALENDARAUTO()
```

This function scans every date column across the *entire* connected model, determines the true min/max date range in use, and generates one continuous calendar table automatically — meaning the date range self-extends as new years of data are loaded, with zero manual maintenance. Additional calendar attributes were layered on top with simple calculated columns:

```dax
year  = YEAR(dim_date[Date])
month = MONTH(dim_date[Date])
```

`dim_date` was then connected as the **shared spine** across every date-bearing fact table (`fact_sales`, `fact_sales_targets`, `fact_inventory`, `fact_campaign_spend`), enabling true side-by-side, apples-to-apples time analysis across otherwise unrelated fact tables — sales trends, campaign spend, inventory levels, and revenue targets, all filterable from one shared timeline.

For the one fact table with *multiple* date columns (`fact_order_fulfillment`), `dim_date` was deliberately connected multiple times as a **role-playing dimension** (order date active by default; ship/invoice/pay dates connected but inactive, to be invoked with `USERELATIONSHIP()` only when a specific report genuinely needs that angle) — with an explicit caution against wiring up every possible date relationship just because it's technically possible; each additional role-playing link adds real cognitive and DAX-authoring overhead that should be justified by an actual reporting need.

### 8.5 A centralized measures table

Rather than scattering DAX measures across whichever fact table felt most convenient at the time, a dedicated empty table (`_measures`) was created purely as an organizational home:

```dax
_measures = {} 
```

Core, broadly-reusable measures were consolidated here — deliberately limited to metrics with genuine cross-report utility, not an exhaustive catalog of every possible calculation:

```dax
Total Sales = SUM(fact_sales[line_total])

Total Orders = DISTINCTCOUNT(fact_sales[order_id])
-- NOTE: counting order_id directly (not COUNTROWS) is essential here —
-- the fact table's grain is one row per line item, so a naive COUNTROWS
-- would badly overcount orders. This is the grain-awareness rule (Rule #2)
-- expressed directly in DAX.

Total Active Customers = DISTINCTCOUNT(fact_sales[customer_id])
Total Customers        = COUNTROWS(dim_customer)  -- deliberately different: this counts
                                                     -- ALL customers, not just ordering ones

Avg Order to Pay Days  = AVERAGE(fact_order_fulfillment[order_to_pay])
-- built on a calculated column: DATEDIFF(order_date, pay_date, DAY)
```

The underlying motivation goes beyond convenience: in any team setting, leaving core metrics undefined invites *multiple analysts independently reinventing the same calculation*, subtly differently, with no way to reconcile which version is "correct." A single governed measures table — especially the pointed contrast between `Total Active Customers` and `Total Customers` — closes that gap and documents grain-related pitfalls directly inside the model itself, for anyone who opens it later.

---

## 9. Row-Level Security

The final layer applied to the model: restricting each business user to see only the sales data for their assigned region.

**Step 1 — Understand the requirement before building anything.**
The `security` table mapped each employee's email to a single allowed region. This mapping had to be understood and confirmed as a genuine business rule before any relationship was drawn — RLS designed without a clear, confirmed requirement is a common source of over-engineered, hard-to-maintain security models.

**Step 2 — Choose the highest-leverage connection point.**
Two dimensions both carried region information: `dim_geo` (connected to one fact) and `dim_customer` (connected to *two* facts — `fact_sales` and `fact_order_fulfillment`). Since RLS filters propagate outward from wherever it's anchored, connecting security to `dim_customer` secured both downstream facts in a single relationship, rather than requiring two separate security wire-ups.

> A deliberate anti-pattern was called out here too: forcing security onto *every* fact table regardless of actual need — sometimes by artificially merging region columns into facts where they don't belong — was flagged as over-engineering. Not every dimension needs securing (there was, for instance, no business reason to restrict visibility into `dim_product`).

**Step 3 — Build the relationship, then the role.**
A standard one-to-many relationship was drawn from `security` to `dim_customer` on region, filtering in a single direction toward the dimension (which, via its existing one-to-many links, then correctly cascades that filter onward to both connected facts).

A new security role was created with a DAX filter expression built on `USERPRINCIPALNAME()` — matching the logged-in report viewer's identity against the appropriate region from the security table, and restricting `dim_customer` to only rows whose region matches. The conceptual pattern:

```dax
[region] = LOOKUPVALUE(
    security[region],
    security[user_email],
    USERPRINCIPALNAME()
)
```

**Step 4 — Test before trusting.**
Power BI's **"View As"** feature was used to impersonate a specific employee email and confirm, visually, that both the customer list and the total sales card shrank correctly to that employee's assigned region — then reverted to confirm the unrestricted (developer) view returned full totals. Testing both directions (restricted *and* unrestricted) is what actually validates an RLS setup — testing only one gives false confidence.

---

## 10. The Transformation: Before vs. After

| | **Before** | **After** |
|---|---|---|
| Table count in the live model | 23 raw, tangled tables | 4 dimensions + 6 facts + 1 date table + 1 measures table + 1 security table |
| Relationships | Many-to-many, filter directions everywhere, facts wired directly to facts | Exclusively one-to-many, single-direction filters, dimension-mediated |
| Naming | Mixed casing (Pascal, camel, raw source labels) | Uniform `snake_case`, `dim_`/`fact_` prefixed, human-friendly |
| Keys | Mix of business keys, hash keys, and inconsistent IDs | Clean surrogate keys (`_key` suffix) throughout |
| Date handling | No shared date logic; inconsistent formats | Self-maintaining `CALENDARAUTO()` date dimension shared across facts |
| Core metrics | Undefined — everyone would calculate their own | Centralized, tested measures table with grain-aware logic |
| Security | None | Region-based RLS, tested with real user impersonation |
| Trustworthiness | Numbers could silently break with any change | Every merge tested against a protected sentinel total |

---

## 11. Techniques & Tricks — Quick Reference

A condensed cheat-sheet of the reusable tricks demonstrated throughout this build:

- **Reference, don't duplicate.** Always build new tables as a *Reference* off the raw stage query — never edit the raw source directly. Keeps a clean, always-available source of truth.
- **Grain check before every merge.** Group by the intended join key, count rows, and confirm every group returns `1` before merging. This single habit catches fan-out bugs before they happen.
- **Sentinel totals.** Pin one trusted measure on a card visual before touching a fact table, and re-check it after every merge. Any unexpected drift = stop and debug immediately.
- **Left Outer Join, always, for fact-building merges.** Inner joins silently drop rows when a match is missing — far more dangerous in a fact table than a few extra nulls.
- **Model-view cardinality preview.** Temporarily draw a relationship in the Model view before merging in Power Query — Power BI's auto-detected 1:1 / 1:many cardinality is a free, fast safety check.
- **Junk dimensions.** Bundle multiple small, unrelated flag/status columns into a single dimension rather than creating several tiny, disconnected dimension tables.
- **Factless facts.** When a table has plenty of keys but zero measures, it's likely a valid many-to-many bridge table — not a modeling mistake.
- **Accumulating snapshots.** For multi-stage business processes, one row per process instance with a column per milestone date beats one fact table per stage.
- **Role-playing dimensions.** One dimension, multiple relationships to the same fact (one active, others invoked via `USERELATIONSHIP()`) — avoids duplicating an entire dimension just because it's needed from two angles.
- **Unpivot for wide-format sources.** Any table with dates or categories spread across column headers (a spreadsheet habit) needs an Unpivot before it can function as a proper fact.
- **Split Column → Rows** (not Columns) for exploding delimited lists into proper fact/bridge rows — paired with a **Trim** step to catch delimiter-adjacent whitespace before it silently breaks a merge.
- **`CALENDARAUTO()`** for a zero-maintenance, self-extending date dimension.
- **Single point of truth.** If a piece of context (a city, a region, a total) already lives in a dimension, never re-add it to a fact — even when the source system repeats it in five different places.
- **Prefer detail-level facts.** Build from the lowest grain and roll up as needed in DAX — never anchor a fact table on pre-aggregated header totals.

---

## 12. Lessons Carried Forward

Beyond the mechanics, this project reinforced a handful of principles worth remembering on any future modeling work:

1. **Understand before you build.** Every hour spent genuinely understanding the business, the process, and the data pays for itself many times over during the build phase.
2. **Say the grain out loud.** Literally — before merging or connecting anything, state what one row represents. It's the cheapest bug-prevention habit available.
3. **Standards are a contract, not a suggestion.** Define them before you start, and enforce them until the very last table — inconsistency compounds fast in a large model.
4. **Ruthlessly remove what isn't needed.** A leaner model is a faster, more trustworthy, and more understandable one. "Just in case" is rarely a good enough reason to keep a column.
5. **Star schema, always.** It simplifies every DAX calculation that comes after, and it's the one architectural decision that pays dividends across the entire lifetime of a model.
6. **Never connect facts directly.** A shared dimension is the only safe bridge between two fact tables.
7. **Test constantly, not just at the end.** A model that "looks right" and a model that's *verified* right are very different things — and only one of them survives contact with a real report.

---

*This walkthrough documents the full reconstruction process behind the project in this repository — from a 23-table nightmare dataset to a tested, secured, documented star schema. See `README.md` for setup instructions and a summary overview.*
