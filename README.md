# Case Management Analytics Platform

A production SQL Server-to-Power BI analytics platform supporting organisation-wide case-management reporting for a UK organisation. The solution combines SQL extraction, Power Query transformation, dimensional modelling, DAX and governed data-quality controls across 3M+ operational records.

This project documents the technical design and implementation, from transactional SQL data through transformation and modelling to Power BI reporting.

---

## The Problem

Operational reporting relied heavily on manual Excel workflows and CRM exports, creating long preparation times and inconsistent results across teams. KPI definitions were not consistently applied, while the existing reporting process provided limited ability to explore performance by service, location, demographic group or reporting period.

**My role:** As Data Analyst within the organisation's data team, I led the technical design and development of the SQL-to-Power BI solution, working with data colleagues, operational teams and senior stakeholders to translate reporting requirements into reliable models, measures and dashboards.

---

## Scale

> **Anonymisation note:** Operational scale and data-quality volume figures in this public portfolio have been deliberately approximated and modified. They demonstrate the scale and complexity of the engineering problem without reproducing the organisation's internal reporting figures.

| Entity | Approximate public portfolio scale |
|---|---|
| Clients | 120,000+ |
| Enquiries / Cases | 450,000+ |
| Actions / Contacts | 3M+ |
| Raw postcode variants | 30,000+ |
| Raw nationality variants | 700+ |
| Cleaned columns | 13+ |
| DAX measures | 60+ |

---

### Solution Architecture
```
Microsoft SQL Server (Live OLTP CRM Source)
        ↓
Inbound Staging Layer (Native T-SQL Interface Queries / "Bronze Layer")
        ↓
Power Query ETL Engine (Structured Cleaning & Schema Validation / "Silver Layer")
        ↓
Dimensional Data Model (High-Performance Star Schema / "Gold Layer")
        ↓
Enterprise Semantic Layer (60+ DAX Measures & Direct-Lake Ready Modeling)
```
---

## Technical Stack

| Layer | Technology |
|---|---|
| Source database | Microsoft SQL Server |
| Query language | T-SQL (CTEs, window functions, EAV pivots) |
| BI platform | Microsoft Power BI (Import mode) |
| Transformation | Power Query M |
| Semantic layer | DAX |
| Postcode enrichment | postcodes.io API + SharePoint cache |

---

## SQL Architecture

The core of the pipeline is a single CTE-based SQL query connecting 9 tables via a chain of natural key joins, pulling several million rows of transactional data into a clean, analysis-ready flat structure.

**Key technical challenges solved:**

- **EAV schema unpivoting** - the source database stored client and case attributes in an Entity-Attribute-Value pattern for two tables. Built MAX(CASE WHEN) pivots to flatten 36 client fields and 5 case fields into a wide analytical structure.

- **Household reference logic** - clients exist in complex household relationships with many-to-many links across cases and family members. Implemented ROW_NUMBER() OVER (PARTITION BY) with descending date ordering to resolve the most recent household relationship per client without fan-out.

- **Previous owners** - case ownership history was stored in a separate audit table with no deleted flag. Used STRING_AGG to concatenate previous case owners into a single column per case.

- **Window functions** - calculated Last Action Date, Latest Deadline Date, Case Time and Last Action Notes at action grain using FIRST_VALUE and MAX OVER without pre-aggregating data.

- **Dual-view architecture** - designed two logical views of the data: a full 79-column version including PII for authorised users, and a 67-column analytics version with PII columns excluded for reporting consumers.

- **Soft delete handling** - the source database used inconsistent soft delete patterns across tables (datetime IS NULL vs BIT flag). Mapped and applied the correct deleted filter per table across the full join chain.

→ See [`sql/core_query.sql`](sql/core_query.sql)

---

## Data Cleaning - Power Query

The staging layer applies structured cleaning across 13+ demographic and operational columns, handling years of historical free-text entries before dropdown controls were introduced.

**Cleaning pattern per column:**

```
Raw value → Lookup merge (controlled list) → Custom column (unmatched variants) → Clean value
```

**DQ signal convention:**

| Signal | Meaning |
|---|---|
| Clean value | Mapped to controlled list |
| `Missing entry` | Null or blank at source |
| `Invalid entry` | Value entered but unclassifiable |
| `null` | New variant not yet mapped — surfaces for review |

**Cleaning coverage:**

| Column | Approximate public portfolio scale | Notes |
|---|---|---|
| Gender | 300+ variants | Direction-aware trans handling; dozens of spelling variants |
| Nationality | 700+ variants | Country names, demonyms, typos, ISO codes, dual entries |
| Ethnic Origin | 300+ variants | Mapped to controlled values |
| Spoken Language | 1,000+ variants | Dialect and direction-aware handling |
| Employment Status | 75+ variants | Numerous unemployment spelling variants |
| Housing Status | 600+ variants | Mapped to controlled values |
| Housing Provider | 150+ variants | Named organisations |
| Action Type | 450+ variants | Channel vs topic distinction |
| Immigration Status 1 | 300+ variants | Mapped to controlled values |
| Immigration Status 2 | 25+ variants | Mapped to controlled values |
| Referral Agency | 650+ variants | Named organisations |
| Signpost Agency | 600+ variants | Named organisations |
| Postcode | 30,000+ variants | Three-layer validation, see note below |

**Notable cleaning decisions:**

*Postcode - three-layer validation:*
1. Format normalisation - strips brackets, slashes, dashes, applies O→0/I→1/L→1 substitution, validates inward code format
2. SharePoint cache lookup - previously validated postcodes resolve instantly without an API call
3. postcodes.io API - only runs on cache misses, returns Local Authority, Ward and Region; falls back to prefix-based inference for Scottish and Welsh postcodes

→ See [`powerquery/cleaning/`](powerquery/cleaning/)

---

## Dimensional Model

Star schema with natural key relationships throughout — no surrogate keys.

```
fact_Actions (grain: one row per action/contact)
    ├── dim_Client    [Client Reference]   - active
    ├── dim_Enquiry   [Enquiry Reference]  - active
    ├── dim_Staff     [Name]               - active  (Action By)
    └── dim_Date      [Date]               - active  (Action Date)
```

**Full relationship map:**

| # | From | Cardinality | To | Status |
|---|---|---|---|---|
| 1 | fact_Actions (Action By) | Many to one | dim_Staff (Name) | **Active** |
| 2 | fact_Actions (Action Date) | Many to one | dim_Date (Date) | **Active** |
| 3 | fact_Actions (Client Reference) | Many to one | dim_Client (Client Reference) | **Active** |
| 4 | fact_Actions (Enquiry Reference) | Many to one | dim_Enquiry (Enquiry Reference) | **Active** |
| 5 | fact_Actions (Enquiry Owner) | Many to one | dim_Staff (Name) | Inactive |
| 6 | fact_Actions (Enquiry Closed By) | Many to one | dim_Staff (Name) | Inactive |
| 7 | dim_Enquiry (Client Reference) | Many to one | dim_Client (Client Reference) | Inactive |
| 8 | dim_Enquiry (Enquiry Date) | Many to one | dim_Date (Date) | Inactive |
| 9 | dim_Enquiry (Enquiry Closed Date) | Many to one | dim_Date (Date) | Inactive |
| 10 | dim_Enquiry (Enquiry Owner) | Many to one | dim_Staff (Name) | Inactive |
| 11 | dim_Enquiry (Enquiry Created By) | Many to one | dim_Staff (Name) | Inactive |
| 12 | dim_Enquiry (Enquiry Closed By) | Many to one | dim_Staff (Name) | Inactive |
| 13 | dim_Client (Client Added By) | Many to one | dim_Staff (Name) | Inactive |

Inactive relationships are activated via USERELATIONSHIP() in DAX measures where needed, for example filtering enquiries by their open date, filtering by closed date, or attributing actions to specific staff roles.

**dim_Client** - 120,000+ public portfolio-scale clients, 13 cleaned demographic columns, age banding, postcode-derived geography (Ward, Local Authority, Region).

**dim_Enquiry** - 450,000+ public portfolio-scale cases, IAA classification logic, immigration status, case type, outcome tracking, closure time banding.

**dim_Staff** - built directly from the Users table via SQL query (not derived from fact data), Text.Proper normalisation applied to resolve case inconsistencies across staff name entry points.

**dim_Date** - financial year aware, FY label, FY month number, quarter, week.

<img width="968" height="747" alt="data_model_diagram" src="https://github.com/user-attachments/assets/e732ed78-bd69-4926-9fe9-ea57727ea639" />

---

## DAX Measures

60+ measures organised across six categories:

**Client measures** - Total/New/Returning Clients Served, YoY growth, gender, disability, global majority backgrounds, NRPF, geography, % by Local Authority

**Enquiry measures** - Total/New/Returning Enquiries, Open/Closed, IAA, Avg Days to Closure, Q Answer analysis (INTERSECT-based), % by type, YoY growth

**Action measures** — Total Actions, IAA Actions, Referrals, Signposts, action time, rates, YoY growth

**Staff measures** — workload distribution, avg actions per staff, avg enquiries owned and closed, closed with no attributed staff

**Geography measures** — in service area vs out of service area across clients, enquiries and actions

**Data quality measures** — client record completeness (missing and invalid rates), postcode DQ, staff accountability

→ See [`dax/`](dax/)

---

## Data Quality Framework

Every cleaned column outputs one of three DQ signals. This means the cleaning layer and the reporting layer are unified; no separate DQ pipeline required.

An unpivoted DQ table approach was evaluated but rejected due to volume: unpivoting 13 columns across a six-figure client dimension would produce well over 10M rows. Instead, DQ is measured directly from cleaned dimension tables using multi-condition OR filters, producing efficient headline metrics. Per-field breakdown is available via visual-level filters on the Data Quality report page.

---
### Software Engineering & Governance Principles
* **Version Control & CI/CD Ready:** SQL and Power Query assets are structured modularly to support Git integration, team code reviews, and automated deployment pipelines via Microsoft Fabric or Azure DevOps.
* **Idempotency & Reusability:** Core SQL transformation scripts are designed to be entirely idempotent, ensuring they can be re-run safely at any point without causing duplicate records or data validation anomalies.
* **Performance Optimization:** Implemented a single-evaluation native staging strategy to enforce data caching. This design choice reduced overall data gateway processing overhead and cut local database lock times by over 40% during concurrent data refreshes.
---
## Impact

- Reduced annual reporting turnaround by over 50%, moving core preparation from a multi-week manual process to a matter of days and enabling refreshed outputs within hours
- Created a consistent reporting layer connecting operational SQL data, agreed KPI definitions and Power BI reports/dashboards
- Improved access to operational and performance information across multiple offices and service areas
- Enabled senior leaders and data colleagues to explore performance through governed, refreshable reporting
- Embedded data-quality monitoring within the reporting model, making missing, invalid and newly occurring values visible for review
- Helped demonstrate the organisational value of Microsoft Fabric & Power BI and inform wider adoption for reporting

---

## Project Structure

```
├── sql/
│   └── core_query.sql              # Full CTE pipeline query (anonymised)
├── powerquery/
│   ├── staging.m                   # Staging layer — trim, clean, type, staff normalisation
│   └── cleaning/
│       ├── gender_clean.m          # high-volume variants, direction-aware trans handling
│       ├── nationality_clean.m     # high-volume variants, controlled list merge
│       ├── postcode_clean.m        # Three-layer: format, cache, postcodes.io API
│       ├── employment_status_clean.m
│       └── action_type_clean.m     # Channel vs topic distinction
├── dax/
│   ├── client_measures.dax
│   ├── enquiry_measures.dax
│   ├── action_measures.dax
│   ├── staff_measures.dax
│   └── dq_measures.dax
├── docs/
│   └── data_model_diagram.png     
└── README.md
```

---

## Key Design Decisions

**Import mode over DirectQuery** - weekly off-hours refresh acceptable for this use case; Import delivers faster report performance for end users and avoids query folding limitations on complex CTEs.

**Native SQL statement over database views** - read-only analytics account had no DDL permissions; native SQL statement in Power BI connection string achieved the same result without requiring CREATE VIEW access.

**Natural keys throughout** - no surrogate keys in the dimensional model; relationships built on Client Reference, Enquiry Reference and Staff Name for transparency and maintainability.

**Cleaning in Power Query, not SQL** - DQ classification (Missing entry / Invalid entry) requires business logic that belongs in the transformation layer, not the source database. Keeping it in Power Query also makes it visible, auditable and editable without database access.

**Single staging query with load enabled** - prevents the long-running SQL query from executing multiple times across downstream dimension queries. All dims read from the cached staging result; disabling load caused each dim to trigger a fresh SQL execution.

**dim_Staff from Users table** - initial approach derived staff names from an unpivot of all staff columns across fact and dimension tables. Replaced with a direct query to the Users table, which is cleaner, more stable, and avoids fan-out from the unpivot causing many-to-many relationship issues.

**SharePoint postcode cache** - API calls across tens of thousands of distinct raw values would be prohibitively slow on every refresh. A SharePoint-hosted Excel cache stores previously validated postcodes; the API only runs on cache misses, reducing external calls to a small fraction of the total on each refresh.

## Related Projects

- [retail-analytics-dbt](https://github.com/tessogamba/retail-analytics-dbt) - Retail analytics project using dbt and Snowflake to create tested staging models and dimensional marts
- [retail-analytics-tableau](https://github.com/tessogamba/retail-analytics-tableau) - Tableau dashboard for customer, sales and revenue analysis using the transformed retail datasets
- [financial-analytics-bigquery](https://github.com/tessogamba/financial-analytics-bigquery) - Financial analytics project using BigQuery and reusable SQL models across 12 public companies and 23 metrics
- [financial-analytics-looker](https://github.com/tessogamba/financial-analytics-looker) - Looker Studio dashboard for exploring financial growth, profitability and performance

---
*Built by Tess Ogamba · [github.com/tessogamba](https://github.com/tessogamba) · [LinkedIn](https://linkedin.com/in/tessogamba)*
