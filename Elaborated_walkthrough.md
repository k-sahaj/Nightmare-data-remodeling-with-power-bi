# From Chaos to Star Schema: An Elaborated Data Modeling Walkthrough

> **A behind-the-scenes look at how a 23-table "nightmare dataset" was rebuilt, table by table, into a clean, secure, production-style star schema in Power BI.**

This document is the companion to my main `README.md`. Where the README tells you *what* the project is, this file tells you *how* it was built — the reasoning, the trade-offs, the debugging moments, and the small tricks that separate a "working" data model from a *trustworthy* one.

---

## Table of Contents

1. [The Setup and The Plan](#1-the-setup-and-the-plan)
2. [Modeling Principles](#2-modeling-principles)
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

## 1. The Setup and The Plan

### 1.1 The Setup: A Deliberately Broken Data Model

Every mature BI project eventually reaches the same point: a dashboard that appears correct until someone asks, *"Why don't these two numbers match?"* In most cases, the visuals aren't the problem—the underlying data model is.

This project begins with a dataset intentionally designed to expose those failures: 23 loosely connected source tables, uncontrolled many-to-many relationships, duplicate imports, inconsistent naming conventions, undocumented identifiers, and several structural decisions that make reliable reporting almost impossible.

Rather than building reports on top of this foundation, the objective was to redesign it into a governed semantic model following dimensional modeling best practices. Every transformation was validated, every relationship justified, and every modeling decision documented.

The goal wasn't to build a dashboard. It was to take that spider-web of tables and turn it into a **healthy, testable, secure star schema** — using the exact same discipline you'd want to see in a professional BI engagement.

### 1.2 The Plan: Phases, rules and standards
![Plan](docs/plan.png)
---

## 2. Modeling Principles

Before a single table was touched, five standards were locked in. Before a single transformation was made, a small set of engineering principles was established. Every modeling decision throughout the project was evaluated against these principles to ensure consistency, maintainability, and analytical correctness.

| # | Rule | Why it matters |
|---|------|-----------------|
| 1 | **Star schema only.** Fact tables sit in the center, surrounded by dimensions. Facts are *never* connected directly to other facts. | Prevents ambiguous filter paths and circular logic — the #1 cause of "wrong numbers" bugs. |
| 2 | **Know the grain before touching a table.** Say out loud what one row represents. | Merging tables of different grains without knowing it is the single most common modeling mistake (see the fan-out incident below). |
| 3 | **Every column must earn its place.** If it's not needed for analysis, drop it. | Unused columns bloat file size, slow refreshes, and confuse report builders. |
| 4 | **Protect the numbers.** Know your key totals by heart; re-check them after every structural change. | A model can *look* correct and still silently duplicate or drop rows. Trust nothing — verify. |
| 5 | **Define standards up front, and follow them until the end.** | English only. `snake_case` for every table/column. Prefixes: `dim_` for dimensions, `fact_` for facts. Surrogate keys always end in `_key` (to distinguish them from natural/source IDs like `_id`). All table, column, and value names must be human-friendly — no raw technical labels from the source system. |

Rules:
![Rules](docs/rules%20%26%20standards/rules.png)

Standard:
![Standard](docs/rules%20%26%20standards/standard.png)

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

### 5.1 `dim_customer` — Building the Customer Dimension

The customer dimension was the largest and most involved dimension in the model, consolidating six operational tables into a single conformed dimension:

- `customer_master`
- `customer_contacts`
- `user_details`
- `addresses`
- `cities`
- `regions`

This phase established several modeling patterns—including grain validation, merge safety checks, and naming standards—that were reused throughout the remainder of the project.

---

#### Selecting the Anchor Table

`customer_master` served as the foundation of the dimension.

Rather than duplicating the query, a **Reference** was created to preserve an untouched staging layer while maintaining a clean dependency graph inside Power Query.

A dummy validation record (`customer_id = 999`) was removed immediately, leaving **60 valid customers** as the expected baseline. This row count became the reference point against which every subsequent merge was validated.

---

#### Resolving a Grain Mismatch

The first merge immediately exposed a classic dimensional modeling problem.

While `customer_master` stores one row per customer, `customer_contacts` stores one row per **contact**. Several customers had multiple contacts, meaning a direct merge multiplied customer records and violated the intended grain of the dimension.

Three approaches were evaluated:

- Aggregate contacts into a single customer-level record.
- Create a separate contact dimension with its own relationship.
- Filter to the **primary contact** only.

The third option was selected because the reporting requirements required only one contact per customer while preserving a simple star schema.

Filtering to `is_primary = true` restored the expected **60-to-60** relationship, making the merge safe.

---

#### Establishing a Merge Validation Strategy

Before trusting any merge key, its uniqueness was verified by grouping the prospective key column and counting occurrences.

A count of **1** confirmed a unique key suitable for merging.

Any value greater than **1** indicated duplicate business keys and the potential for fan-out during joins.

This simple cardinality check became the standard validation technique used throughout the project before every major merge.

---

#### Standardizing Business Terminology

After incorporating `user_details`, the semantic model contained two competing names for the same business entity: **customer** and **user**.

Rather than preserving inconsistent terminology from the operational systems, the model standardized on **customer**, reflecting the terminology used across the majority of source tables.

This decision established consistent business language throughout the semantic layer while avoiding unnecessary ambiguity for report developers.

---

#### Favoring a Star Schema over a Snowflake

The supporting `addresses`, `cities`, and `regions` tables could have been modeled as separate linked dimensions, producing a snowflake schema.

Instead, these attributes were folded directly into `dim_customer` to maintain a simpler and more efficient star schema.

Before each merge, Power BI's **Model View** was used as a secondary validation tool by creating temporary relationships and reviewing the automatically detected cardinality.

Relationships identified as **1:1** or **1:Many** confirmed that the merge could proceed safely, while any indication of **Many:Many** triggered further investigation before continuing.

---

#### Eliminating Redundant Information

During integration, the `regions` table was found to contribute no new business information.

The region attribute had already been introduced through the `cities` table, making an additional merge unnecessary.

Rather than retaining duplicate information "just in case," the redundant table was excluded entirely, reinforcing the principle that every attribute should provide unique analytical value.

---

#### Model Cleanup

With the dimension complete, temporary and operational columns were removed, including:

- `address_id`
- source identifiers
- hash keys
- the `is_primary` flag
- account manager information (outside project scope)

Remaining columns were renamed using a consistent `snake_case` convention, and verbose source-system names such as `cities.region_name` were simplified to concise business-friendly names like `region`.

---

#### Retiring the Staging Tables

Once `dim_customer` had been validated, the six source queries had **Enable Load** disabled.

This preserved the staging layer for future maintenance while preventing unnecessary tables from appearing in the semantic model or consuming refresh resources.

The result was a cleaner model that remained fully traceable back to its operational sources.

### 5.2 `dim_product` — Building the Product Dimension

The product dimension was comparatively straightforward, but it still surfaced several common data-modeling challenges and design decisions.

| Step | Decision | Reasoning |
|------|----------|-----------|
| **Anchor Table** | Started with `products` after removing a dummy **"DO NOT USE"** record, leaving **60 valid products**. | Establish a clean baseline before any transformations. |
| **Duplicate Validation** | Grouped by `product_code` to verify uniqueness. | Confirmed the business key could safely identify one product per row. |
| **Category Normalization** | Split the pipe-delimited `category\|subcategory` field into separate `category` and `subcategory` columns using **Split Column → By Delimiter**. | Normalized the hierarchy for cleaner filtering and drill-down. |
| **Case Standardization** | Standardized subcategory names using **Transform → Capitalize Each Word** before merging. | Prevented a silent join failure caused by case-sensitive text matching while creating consistent report-friendly labels. |
| **Surrogate Key** | Generated an **Index Column** (`product_key`) because no numeric identifier existed in the source. | Introduced a stable surrogate key independent of the business key (`product_code`). |
| **Model Simplification** | Removed long free-text product descriptions. | Semantic models should prioritize analytical attributes over operational metadata. |
| **Column Organization** | Ordered columns as `product_key` → Product Attributes → `subcategory` → `category`. | Improves navigation and mirrors the natural drill-down hierarchy for report builders. |

> **Key takeaway:** Even a relatively simple dimension benefits from disciplined validation, standardized business semantics, and thoughtful organization. Small design decisions at the dimension level significantly improve the maintainability of the overall semantic model.

---

## 6. The Fact vs. Dimension Decision Framework

Choosing whether a table should become a fact or a dimension was driven by business semantics rather than the presence of dates or numeric columns.

Tables that primarily described business entities became dimensions, while tables representing recurring business events became facts.

The campaign data illustrates this distinction well. Campaign metadata—such as name, start date, budget, and owner—forms the descriptive context and therefore belongs in `dim_campaign`. Daily campaign performance, however, represents recurring measurable activity and naturally belongs in `fact_campaign_spend`.

Separating descriptive attributes from transactional activity keeps the model aligned with dimensional modeling principles while avoiding duplicated context across facts.

### The Header/Detail pattern (the deepest concept in this project)

Almost every transactional business system stores events as a **header** (one row per event — order, invoice, shipment) plus **details** (multiple rows per event — line items, products, quantities). This is standard OLTP design, and it appears constantly in Orders, Invoices, and similar tables here.

The classic beginner mistake: treat the header and the detail as *two separate fact tables*, then connect them to each other using the shared order ID. This technically "works" but is a landmine — connecting fact-to-fact breaks the star schema, produces ambiguous filter paths, tanks performance, and eventually produces wrong numbers in visuals that pull from both.

**The rule applied instead:** measures almost always live at the *lowest* grain — the details. So the **fact table is built from the detail rows**, and the header is used purely as a **source of dimensional context** (dates, customer, channel, status) that gets merged *into* the fact and/or split off into new dimensions. You can always aggregate detail-level facts back up to header-level totals — but you can never safely go the other direction.

This single principle — "facts are built from the finest grain, headers just supply context" — was reused for Orders, Invoices, and the later Order Fulfillment fact.

---

## 7. Phase 3 — Building the Facts

### 7.1 `fact_sales` — Building the Core Transaction Fact

The construction of `fact_sales` was the most involved phase of the project. It combined data from `orders_2025`, `orders_2026`, and `order_line_items` into a single transaction-level fact table while preserving analytical correctness at every stage.

---

#### Consolidating Historical Orders

The first task was to combine the two yearly order tables using **Append Queries → As New Query**. Although both tables represented the same business process at the same grain, the append immediately exposed schema drift between years.

The 2025 extract contained a legacy reference field and a gift-message column that no longer existed in the 2026 data. Both fields consisted entirely of null or non-reporting values and were removed before continuing, ensuring a consistent structure across both years.

---

#### Extracting a Junk Dimension

The order header contained three independent, low-cardinality attributes:

- `order_channel`
- `status`
- `priority`

Each attribute was useful for filtering, yet none justified its own standalone dimension. Rather than creating three tiny lookup tables, these fields were consolidated into a single **junk dimension** (`dim_order_flags`).

The dimension was created by referencing the order header, retaining only the three categorical fields, removing duplicate combinations (resulting in just **14 unique records**), and assigning a surrogate `flag_key`.

This reduced repetition inside the fact table while preserving all filtering capabilities.

---

#### Manual Reference Data (Documented Technical Debt)

One challenge emerged during the construction of the junk dimension.

The `order_channel` field stored only numeric codes (`10`, `20`, `30`, `40`) without any accompanying lookup table describing their business meaning.

Since no governed reference table existed, a small mapping table was created manually using **Home → Enter Data**, translating each code into a business-friendly label (Online Store, Retail Partner, Wholesale, and Field Sale).

After merging the lookup into `dim_order_flags`, the raw numeric codes were removed.

> ⚠️ **Technical Debt**
>
> This manual lookup is intentionally documented as a temporary solution rather than a best practice. If the operational system later introduces additional channel codes, the mapping would silently become incomplete, producing null values during reporting.
>
> In a production environment, this mapping should exist within the source system—or another governed reference table—rather than inside the semantic model.

---

#### Building the Transaction Fact

Unlike the order header, which represents one row per order, `order_line_items` represents the true transactional grain.

For that reason, `order_line_items` became the foundation of `fact_sales`, while the consolidated order header was merged only to provide additional descriptive context.

Every merge into the fact table used a **Left Outer Join**.

Using an Inner Join may appear cleaner, but it risks silently discarding valid transaction rows whenever corresponding header records are missing. Preserving transactional completeness is significantly more important than eliminating a handful of null lookup values.

---

#### Protecting Analytical Integrity

Before any structural transformation touched the fact table, a trusted sales measure (`SUM(line_total)`) was placed on a report card and treated as a **sentinel measure**.

After every merge, enrichment, or key replacement, this value was immediately rechecked.

Any unexpected change was treated as a failed validation, requiring investigation before further modeling work continued.

Rather than relying solely on Power Query previews or row counts, this approach validated what ultimately matters: the business numbers themselves.

---

#### Investigating a Fan-Out Bug

This validation strategy quickly proved its value.

Immediately after merging the product dimension, the protected sales total increased unexpectedly.

The root cause was not the merge itself but duplicate records within `dim_product`. Because the join relied on **product name**—the only shared business key—two products matched multiple dimension rows, causing every affected sales record to duplicate during the merge.

Grouping the dimension by product name exposed the duplicates immediately.

Further inspection showed that each duplicated product consisted of one complete record and one incomplete record missing descriptive attributes such as brand and price.

Instead of patching the issue inside the fact build, the duplicate was removed directly within `dim_product` immediately after the raw import step using its `source_id`.

Correcting the problem upstream ensured that every future consumer of the dimension benefited from the fix.

Once the duplicate was removed, the sentinel measure returned exactly to its original value, confirming the integrity of the fact table before modeling continued.

> This incident demonstrates why continuous validation is an engineering discipline rather than a procedural checkbox. Without a protected business metric, the duplicate records would have propagated silently into every downstream report.

---

#### Replacing Business Attributes with Surrogate Keys

With the fact table validated, descriptive attributes were systematically replaced by surrogate keys.

- `customer_name` → `customer_id`
- `product_name` → `product_key`
- `order_channel`, `status`, and `priority` → `flag_key`

Each replacement was validated independently using the sentinel measure to ensure no unintended changes had been introduced.

---

#### Separating Master Data from Transactional Context

Not every descriptive attribute belongs in a dimension.

Customer city and region were removed from the fact because they already existed within `dim_customer`, preserving a single authoritative source of truth.

Shipping and billing locations, however, represent characteristics of individual orders rather than permanent customer attributes.

A customer may have one registered address while shipping different orders to different destinations.

Recognizing this distinction prevented legitimate transactional information from being incorrectly treated as master data.

---

#### Eliminating Redundant Aggregates

Once line-level transactions formed the foundation of the fact table, the pre-aggregated `order_total` supplied by the source system was removed.

Whenever detailed transactions exist, aggregate values become redundant.

Keeping both introduces unnecessary duplication and creates opportunities for inconsistencies if one value changes while the other does not.

The model therefore derives order totals directly from transactional detail whenever required.

---

#### Introducing a Role-Playing Geography Dimension

During fact construction, it became clear that shipping and billing locations deserved their own reusable geography dimension.

A new `dim_geo` was therefore created from the previously deferred cities dataset, illustrating that dimension discovery is often iterative rather than something completed entirely upfront.

The resulting dimension connects twice to `fact_sales`:

- Ship-to City
- Bill-to City

Because Power BI permits only one active relationship between two tables, the shipping relationship remained active while the billing relationship was intentionally left inactive and activated only within DAX measures using `USERELATIONSHIP()`.

This implementation demonstrates a classic **role-playing dimension**.

---

#### Final Structure

The completed fact table follows a consistent column organization adopted throughout the entire model:

1. Surrogate keys
2. Dates
3. Transactional attributes
4. Measures

Maintaining a predictable structure across every fact table improves navigation, simplifies maintenance, and creates a more intuitive modeling experience for report developers.

### 7.2 `fact_inventory` — Reshaping Inventory into an Analytical Fact

Compared to `fact_sales`, the inventory fact required relatively little business logic. The primary challenge was transforming a spreadsheet-oriented layout into a structure suitable for dimensional modeling.

| Phase | Implementation | Why it Matters |
|-------|----------------|----------------|
| **Source Structure** | The raw inventory table stored one row per product with a separate column for each month. | Although readable in spreadsheets, this wide format is unsuitable for analytical models because months are stored as column headers rather than data values. |
| **Normalization** | Applied **Transform → Unpivot Columns** to convert the monthly columns into two fields: `month` and `units`. | Transformed the dataset into a proper long-format fact table where time becomes a filterable dimension instead of part of the schema. |
| **Data Standardization** | Converted the month values from text into proper `Date` data types and renamed columns according to the project's naming standards. | Ensures consistent filtering, sorting, and integration with the shared date dimension. |
| **Dimension Integration** | Replaced the business identifier with `product_key` using the standard merge-and-validation process. | Maintains surrogate key discipline and consistent relationships across the model. |
| **Relationship Design** | Connected the fact only to `dim_product` (and the shared date dimension), never to `fact_sales`. | Reinforces the core modeling principle that fact tables communicate through shared dimensions rather than directly with one another. |

> **Key takeaway:** Although technically simple, this transformation illustrates one of the most common tasks in Power Query—converting spreadsheet-style data into a normalized fact table suitable for dimensional modeling.

### 7.3 `fact_campaign_spend` & `fact_promotion_coverage`

The campaign data naturally split into two distinct analytical structures: one transactional fact table containing campaign performance metrics, and one **factless fact** representing the relationship between campaigns and products.

| Component | Implementation | Why it Matters |
|-----------|----------------|----------------|
| **Campaign Spend Fact** | After extracting `dim_campaign` (see [Section 6](#6-the-fact-vs-dimension-decision-framework)), the remaining daily transactional data became `fact_campaign_spend`. Campaign names were replaced with `campaign_key` using the standard merge-and-validation workflow. | Separates descriptive campaign attributes from recurring transactional activity while maintaining surrogate key discipline. |
| **Exploding Product Lists** | The `campaign_skus` table stored multiple product codes as a comma-separated list within a single field. Applying **Split Column → By Delimiter → Rows** expanded each campaign into one row per associated product. | Converts a denormalized spreadsheet-style structure into a normalized table where each row represents a single business relationship. |
| **Data Cleanup** | Applied **Trim** after splitting the values into rows. | Removes leading whitespace introduced by comma-separated values, preventing silent merge failures caused by text mismatches. |
| **Surrogate Key Conversion** | Replaced `campaign_name` with `campaign_key` and `product_code` with `product_key`. | Ensures consistent relationships across the semantic model while eliminating dependence on business keys. |
| **Factless Fact Pattern** | The resulting table contains only foreign keys and no numeric measures. It was modeled as `fact_promotion_coverage`. | Demonstrates the **factless fact** pattern, recording that a business relationship exists (which products participated in which campaigns) without measuring a numeric event. This enables many-to-many analysis while preserving a clean star schema. |

> **Key takeaway:** Not every fact table stores measures. Factless facts are a recognized dimensional modeling pattern used to represent business events or relationships whose analytical value lies in their existence rather than in a numeric quantity.

### 7.4 `fact_order_fulfillment` — Modeling an Accumulating Snapshot

The final—and most conceptually advanced—fact table models the lifecycle of an order across five operational event tables:

```text
Order → Ship → Deliver → Invoice → Pay
```

Unlike the previous fact tables, the challenge here was not data transformation but selecting the correct dimensional modeling pattern.

---

#### The Naïve Design

The most obvious implementation would have been to create **one fact table per process stage**, directly mirroring the operational system:

- Orders
- Shipments
- Deliveries
- Invoices
- Payments

Although technically valid, this design would duplicate the **same business transaction** across multiple fact tables.

The revenue associated with an order does not meaningfully change as it progresses through fulfillment, so storing the same monetary value in five different places introduces redundancy, complicates reconciliation, and increases maintenance effort without improving analytical capability.

---

#### Reframing the Business Question

The important realization was that the business was not asking:

> **"How much revenue exists at each stage?"**

Instead, it wanted to understand:

- How many days from **Order → Shipment**?
- How many days from **Shipment → Delivery**?
- How many days from **Invoice → Payment**?

These are questions about **process duration**, not financial transactions.

---

#### Choosing the Right Modeling Pattern

This requirement naturally fits an **Accumulating Snapshot Fact**.

Instead of recording each process stage as a separate transaction, the model stores **one row per order**, with additional milestone dates populated as the order progresses through its lifecycle.

Each row gradually accumulates information over time:

```text
Order
│
├── order_date
├── ship_date
├── delivery_date
├── invoice_date
└── payment_date
```

This structure makes process-cycle analysis straightforward while avoiding duplicated transactional data.

---

#### Build Sequence

The fact table was assembled incrementally.

| Phase | Implementation | Purpose |
|-------|----------------|---------|
| **Foundation** | Started with the cleaned `orders` table, retaining only `order_id`, `customer_id`, and `order_date`. | Establish one row per business process (one order). |
| **Shipping Events** | Merged `shipments` to add `ship_date` and `delivery_date`. | Capture fulfillment milestones. |
| **Invoice Events** | Merged `invoices` to add `invoice_date`. | Extend the lifecycle timeline. |
| **Payment Events** | Merged `payments` using `invoice_id`, since payments reference invoices rather than orders directly. | Complete the process chain while preserving the correct relationship path. |
| **Data Minimization** | Imported only milestone dates during each merge, intentionally excluding duplicated customer information, monetary values, and other descriptive fields already modeled elsewhere. | Keep the fact table focused solely on process timing. |

Every merge followed the standard **Left Outer Join** strategy to preserve the complete order history while enriching it with milestone information.

---

#### Final Outcome

Once the accumulating snapshot had been validated, several operational event tables became redundant.

Their analytical responsibilities had been cleanly separated:

- **`fact_sales`** → Financial transactions
- **`fact_order_fulfillment`** → Process timing and operational performance

As a result, the following staging tables had **Enable Load** disabled:

- `shipments`
- `invoices`
- `invoice_lines`
- `payments`
- `sheet1`

Rather than exposing multiple overlapping operational tables, the semantic model now presents two purpose-built facts, each optimized for a distinct analytical objective.

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

With the star schema structurally complete, the remaining work focused on improving usability, governance, and long-term maintainability. These refinements don't change the model's analytical capability, but they significantly improve its reliability and day-to-day experience for report developers.

---

### 8.1 Standards Audit

Before considering the model complete, a final audit was performed against the engineering principles established in Section 2.

This review focused on identifying inconsistencies that naturally emerge during long modeling sessions—particularly naming deviations and structural drift that are easy to overlook while actively building the model.

Rather than assuming standards had been followed perfectly throughout the project, a dedicated review ensured the finished semantic model remained internally consistent.

---

### 8.2 Date Standardization

Source tables contained dates in multiple display formats, including verbose month names that consumed unnecessary report space.

A single ISO-style format (`YYYY-MM-DD`) was adopted and applied consistently across every date column in the model.

Although purely cosmetic, consistent date formatting improves readability, simplifies comparison across visuals, and creates a more professional reporting experience.

---

### 8.3 Summarization Audit

Power BI automatically assigns aggregation behavior to numeric columns, which can easily produce misleading visuals if left unchecked.

Every numeric field was reviewed individually and assigned an explicit default summarization.

| Column Type | Default Behavior | Reason |
|-------------|------------------|--------|
| Measures (Sales, Units, Budget) | **Sum** | Represents additive business metrics. |
| Identifiers (`customer_id`, `product_key`, etc.) | **Don't Summarize** | Prevents meaningless totals such as summed IDs. |
| Non-additive values | Appropriate aggregation (or **Don't Summarize**) | Ensures report authors cannot accidentally create misleading visuals. |

Although this step produces no visible dashboard improvements, it reduces report-authoring errors and makes the semantic model significantly more intuitive for future developers.

---

### 8.4 Building the Date Dimension

Rather than maintaining a static calendar table, the shared date dimension was generated dynamically using:

```dax
dim_date = CALENDARAUTO()
```

`CALENDARAUTO()` scans every connected date column in the model, determines the overall date range, and generates a continuous calendar automatically. As additional years of data are loaded, the calendar extends without requiring manual updates.

Common calendar attributes were then added as calculated columns:

```dax
year  = YEAR(dim_date[Date])
month = MONTH(dim_date[Date])
```

The completed `dim_date` became the shared timeline for every date-driven fact table:

| Connected Fact | Primary Date |
|----------------|--------------|
| `fact_sales` | Order Date |
| `fact_sales_targets` | Target Date |
| `fact_inventory` | Inventory Month |
| `fact_campaign_spend` | Campaign Date |

Using a single conformed date dimension enables consistent time intelligence and side-by-side analysis across otherwise unrelated business processes.

#### Role-Playing Dates

`fact_order_fulfillment` required multiple business dates for the same order:

- Order Date
- Ship Date
- Delivery Date
- Invoice Date
- Payment Date

Rather than creating multiple calendar tables, `dim_date` was reused as a **role-playing dimension**.

Only the Order Date relationship remains active by default, while the remaining relationships are intentionally inactive and activated within DAX using `USERELATIONSHIP()` only when required.

This avoids unnecessary model complexity while preserving analytical flexibility.

---

### 8.5 Centralizing Business Measures

Rather than scattering DAX calculations across individual fact tables, a dedicated measures table was created:

```dax
_measures = {}
```

This table contains reusable business metrics shared across multiple reports.

```dax
Total Sales = SUM(fact_sales[line_total])

Total Orders = DISTINCTCOUNT(fact_sales[order_id])

Total Active Customers = DISTINCTCOUNT(fact_sales[customer_id])

Total Customers = COUNTROWS(dim_customer)

Avg Order to Pay Days =
AVERAGE(fact_order_fulfillment[order_to_pay])
```

The distinction between measures is intentional.

For example:

- `Total Orders` uses `DISTINCTCOUNT(order_id)` because the fact table is stored at **line-item grain**.
- `COUNTROWS(fact_sales)` would incorrectly count order lines rather than orders.
- `Total Customers` counts every customer in the dimension, while `Total Active Customers` counts only customers who actually placed an order.

These examples reinforce how grain awareness extends beyond data modeling and directly influences DAX design.

Beyond organization, a centralized measures table also establishes a single governed definition for core business metrics, preventing multiple report authors from independently creating slightly different versions of the same calculation.

---

## Summarizing phase plan:
<table>
  <tr>
    <td><img src="docs/phases/Screenshot%202026-07-09%20111430.png" width="100%"></td>
    <td><img src="docs/phases/Screenshot%202026-07-09%20111438.png" width="100%"></td>
  </tr>
  <tr>
    <td><img src="docs/phases/Screenshot%202026-07-09%20111450.png" width="100%"></td>
    <td><img src="docs/phases/Screenshot%202026-07-09%20111503.png" width="100%"></td>
  </tr>
</table>

---

## 9. Row-Level Security

The final layer added to the semantic model was **Row-Level Security (RLS)**, ensuring that each business user could access only the sales data associated with their assigned region.

---

### Understanding the Security Requirement

The `security` table mapped each employee's email address to a single authorized region.

Before implementing any relationships, this mapping was verified as a genuine business rule rather than simply assuming the source structure represented the desired security model.

Establishing the business requirement first avoids one of the most common RLS mistakes: building an unnecessarily complex security model before understanding what actually needs to be protected.

---

### Choosing the Security Anchor

Two dimensions contained regional information:

| Candidate Dimension | Connected Facts | Decision |
|---------------------|-----------------|----------|
| `dim_geo` | One fact table | Rejected |
| `dim_customer` | `fact_sales`, `fact_order_fulfillment` | **Selected** |

Because security filters propagate through existing model relationships, anchoring RLS to `dim_customer` automatically secured both downstream fact tables through a single relationship.

This eliminated the need to duplicate security logic across multiple parts of the model.

> **Design Principle**
>
> Security should be applied at the highest logical point in the model rather than being duplicated across every fact table. Likewise, dimensions should not be secured unless a genuine business requirement exists. For example, there was no need to restrict access to `dim_product`, so no unnecessary security rules were introduced.

---

### Implementing the Security Model

A standard one-to-many relationship was created between `security` and `dim_customer` using the regional attribute, with filters flowing toward the dimension and then propagating naturally through the existing star schema.

A security role was then created using `USERPRINCIPALNAME()` to identify the logged-in user and determine the appropriate region.

Conceptually, the filter follows this pattern:

```dax
[region] =
LOOKUPVALUE(
    security[region],
    security[user_email],
    USERPRINCIPALNAME()
)
```

This dynamically filters `dim_customer` to only the rows associated with the current user's assigned region, with the restriction automatically cascading to every connected fact table.

---

### Validating the Security Rules

The completed RLS configuration was tested using Power BI's **View As** feature.

Validation included two complementary checks:

| Test | Expected Result |
|------|-----------------|
| **Impersonate a business user** | Customer records and sales totals are restricted to the assigned region. |
| **Return to the developer view** | Full, unrestricted totals are restored. |

Testing both restricted and unrestricted scenarios confirmed that the security rules behaved correctly without unintentionally filtering the underlying model.

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

| Technique | Why it matters |
|------------|---------------|
| Reference instead of Duplicate | Preserves an untouched staging layer while keeping dependencies clear. |
| Grain Validation | Prevents fan-out before merges occur. |
| Sentinel Measure | Detects unintended changes immediately after structural transformations. |
| Left Outer Joins | Prevents accidental loss of transactional records. |
| Model View Cardinality Check | Quickly validates relationship assumptions before merging. |
| Junk Dimensions | Consolidates low-cardinality attributes without creating unnecessary dimensions. |
| Factless Facts | Models business relationships without introducing artificial measures. |
| Accumulating Snapshots | Captures business process milestones without duplicating transactions. |
| Role-playing Dimensions | Reuses a single dimension across multiple business contexts. |
| Unpivot | Converts spreadsheet-style layouts into analytical tables. |
| CALENDARAUTO() | Generates a self-maintaining calendar dimension. |

---

## 12. Lessons Carried Forward

Beyond the mechanics, this project reinforced a handful of principles worth remembering on any future modeling work:

1. **Understand the business before modeling the data.** Technical correctness is meaningless if the model fails to represent real business processes.

2. **Grain determines everything.** Relationships, DAX, and aggregation all become simpler once the grain is explicitly defined.

3. **Consistency scales.** Naming standards, surrogate keys, and modeling conventions prevent complexity from compounding as models grow.

4. **Validation is continuous.** Every structural change should be verified immediately rather than assumed correct until final testing.

5. **Dimensions describe. Facts record.** Maintaining this separation produces models that remain understandable, extensible, and performant.

6. **A semantic model is a long-term asset.** Dashboards evolve rapidly, but a well-designed model continues serving new analytical requirements with minimal structural change.

---

## Closing Thoughts

This project was intentionally designed to simulate the type of data modeling work performed during real Business Intelligence engagements.

While the final star schema is the visible outcome, the primary objective was to practice the engineering discipline behind trustworthy analytical models: understanding business processes, defining grain, validating transformations, standardizing semantics, and building a model that remains maintainable long after the first dashboard is published.

The techniques documented here are not specific to this dataset—they represent reusable patterns applicable across Power BI, SQL-based data warehouses, and modern dimensional modeling projects.

---

*This walkthrough documents the full reconstruction process behind the project in this repository — from a 23-table nightmare dataset to a tested, secured, documented star schema. See `README.md` for setup instructions and a summary overview.*
