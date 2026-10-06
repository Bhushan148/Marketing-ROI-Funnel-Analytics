# Marketing ROI & Funnel Analytics

**An end-to-end Power BI delivery, written up the way I run a real consulting engagement: requirements → source analysis → Power Query → data model → DAX → report → Power BI Service (RLS, refresh, deployment) → client sharing and handover.**

> **Business question:** *Is marketing spend producing quality leads, customers, and revenue?*

```
Spend → Impressions → Clicks → Leads → Qualified Leads → Customers → Revenue
```

---

## Contents
1. [Project snapshot](#1-project-snapshot)
2. [Engagement overview and requirements](#2-engagement-overview-and-requirements)
3. [Domain knowledge: marketing funnel and metrics](#3-domain-knowledge-marketing-funnel-and-metrics)
4. [How the project flows, step by step](#4-how-the-project-flows-step-by-step)
5. [Solution architecture](#5-solution-architecture)
6. [Repository structure (Azure DevOps)](#6-repository-structure-azure-devops)
7. [Source data and why it is folder-based](#7-source-data-and-why-it-is-folder-based)
8. [Power Query: every step and why](#8-power-query-every-step-and-why)
9. [Data model](#9-data-model)
10. [DAX measures](#10-dax-measures)
11. [Report development](#11-report-development)
12. [Key results and insights from the model](#12-key-results-and-insights-from-the-model)
13. [Power BI Service: deployment, RLS, refresh, sharing](#13-power-bi-service-deployment-rls-refresh-sharing)
14. [Testing and validation](#14-testing-and-validation)
15. [What I did as the Power BI analyst](#15-what-i-did-as-the-power-bi-analyst)
16. [What I gained from this project](#16-what-i-gained-from-this-project)
17. [Decisions, issues and lessons learned](#17-decisions-issues-and-lessons-learned)
18. [Project status](#18-project-status)

> **How to read the status markers in this document**
> ✅ = built and visible in this repository (model, Power Query, DAX, report files).
> 📘 = delivery runbook: the Power BI Service and handover steps I follow on a client engagement, documented here as the plan and configuration for this project. These are not yet configured in a live workspace.

---

## 1. Project snapshot

| | |
|---|---|
| **Client profile** | B2B SaaS company running paid and organic marketing |
| **Channels** | Google Ads, Facebook, Instagram, LinkedIn, Email, SEO |
| **Period covered** | Sep 2023 – Aug 2026 (three fiscal years, FY = Sep–Aug) |
| **Source systems** | Ad-platform exports, CRM exports, campaign and product masters (CSV and Excel, folder-based) |
| **Volume** | ~437K source rows: 360,000 ad rows, 48,000 leads, 6,000 orders, plus master data |
| **Model** | 7 dimensions, 3 facts, 1 helper table, 1 measure table, 16 relationships |
| **Measures** | 47 DAX measures in 7 display folders |
| **Report** | 3 main pages, 2 drill-through pages, 1 tooltip page |
| **Format** | PBIP (TMDL semantic model and PBIR report), version-controlled |

**Headline result (all three fiscal years):** $97.4M ad spend returned $150.9M net revenue, a ROAS of **1.55x**. 47% of leads qualify, and 4,202 customers were acquired.

---

## 2. Engagement overview and requirements

This section is how I open any consulting project: agree the problem, the audience and the definition of done before touching data.

### 2.1 Background
Marketing leadership receives monthly spreadsheets from each ad platform and a separate CRM extract. Nobody can see, in one place, what a dollar of media spend turned into. Channel owners defend their own numbers, and the CMO cannot tell which campaigns to scale or cut.

### 2.2 Objectives
1. Give leadership one trusted view of spend → leads → customers → revenue.
2. Rank channels and campaigns by return, not by clicks.
3. Show where leads are lost between qualification and conversion.
4. Replace manual spreadsheet consolidation with a refreshable, governed dataset.

### 2.3 Stakeholders

| Stakeholder | Role in the project | What they need |
|---|---|---|
| CMO / VP Marketing | Executive sponsor, report consumer | Headline return and trend in ten seconds |
| Marketing Director | Business owner, sign-off | Channel comparison, budget decisions |
| Performance and campaign managers | Power users | Campaign ranking and drill-down |
| Demand generation / Sales | Consumers | Lead quality and conversion leakage |
| Marketing analysts | Consumers, future maintainers | Detail tables and a clean model |
| IT / data team | Gateway, workspace and access owners | Secure, supportable refresh |

### 2.4 Business requirements

| ID | Requirement | Priority | Delivered by |
|---|---|---|---|
| BR-01 | Total spend, net revenue, ROAS, leads, customers and CAC on one page | Must | Executive Overview |
| BR-02 | Compare current fiscal year to the prior year | Must | Time-comparison measures, KPI context text |
| BR-03 | Rank channels by money returned after spend | Must | Channel profit/loss, channel bar |
| BR-04 | Rank and drill into individual campaigns | Must | Campaign leaderboard, Campaign Detail |
| BR-05 | Lead funnel with qualification and conversion rates by channel | Must | Funnel page |
| BR-06 | Revenue by customer segment, country, product and top customers | Should | Funnel & Customers, Customer Detail |
| BR-07 | Managers see only their own region or channel | Must | Row-level security |
| BR-08 | Data refreshes automatically every day | Must | Scheduled refresh |
| BR-09 | Fast refresh as history grows | Should | Incremental refresh |
| BR-10 | Works on laptop and is accessible (contrast, alt text) | Should | Design system |
| BR-11 | Metric definitions documented and consistent | Must | Measure table, descriptions, this README |

### 2.5 KPI definitions agreed with the business

| KPI | Definition |
|---|---|
| ROAS | Net Revenue ÷ Ad Spend |
| CAC | Ad Spend ÷ Distinct customers who ordered |
| CPL | Ad Spend ÷ Leads |
| Lead Qualification Rate | Qualified Leads ÷ Leads |
| Qualified → Converted Rate | Converted Leads ÷ Qualified Leads |
| Marketing Contribution | Net Revenue − Ad Spend |
| Fiscal year | September to August (FY24, FY25, FY26) |

### 2.6 Scope

| In scope (V1) | Out of scope (V1) |
|---|---|
| Ad performance, leads, conversions and revenue | Budget and target tracking |
| Fiscal-year comparison | Multi-product orders and payment history |
| Row-level security by region and channel | Real-time / streaming data |
| Scheduled and incremental refresh | Forecasting and predictive models |
| Drill-through to campaign and customer | Mobile-specific layout |

**Simplification agreed with the client:** one conversion = one order = one customer + one primary product + one revenue amount.

### 2.7 Assumptions and constraints
- Source data is delivered as flat-file exports into a shared folder; no direct database access.
- Spend, leads and revenue sit in three different fact tables, so ratios are only valid by Date, Channel, Campaign and Product (see [section 9](#9-data-model)).
- SEO has no direct media spend and Email has very little, so ROAS is not meaningful for them; the report uses Marketing Contribution instead.
- The data used in this portfolio version is synthetic.

### 2.8 Acceptance criteria
- Model totals reconcile to the validation baseline (see [section 14](#14-testing-and-validation)).
- Every requirement above is traceable to a visual or a configuration.
- Managers confirm that RLS shows only their own region or channel data.
- Refresh succeeds on schedule for two consecutive weeks (hypercare).

---

## 3. Domain knowledge: marketing funnel and metrics

Marketing analytics only works if the numbers are read the way marketers read them. This is the domain background I worked from, and the reasoning behind how the model and report are built.

### 3.1 The funnel and where each stage lives in the data

| Funnel stage | Meaning | Where it comes from | Model table |
|---|---|---|---|
| **Spend** | Money paid to ad platforms | Ad-platform exports | `Fact_AdPerformance` |
| **Impressions** | Times an ad was shown | Ad-platform exports | `Fact_AdPerformance` |
| **Reach** | Unique people who saw it | Ad-platform exports | `Fact_AdPerformance` |
| **Clicks** | Visits driven by the ad | Ad-platform exports | `Fact_AdPerformance` |
| **Lead** | A person who left their details (form, contact request) | CRM lead export | `Fact_Lead` |
| **Qualified lead** | A lead sales judges worth pursuing (often called MQL/SQL) | CRM `QualifiedFlag` | `Fact_Lead` |
| **Converted lead** | A lead that became an order | CRM `ConvertedFlag` | `Fact_Lead` |
| **Customer / order** | A paying account and its purchases | Sales conversion export | `Fact_Conversion` |
| **Revenue** | Money earned (gross, then net of discounts and refunds) | Sales conversion export | `Fact_Conversion` |

**The key domain insight:** the top of the funnel (ads) lives in the ad platforms and the bottom (leads, orders) lives in the CRM. They are different systems with different keys. Joining them correctly is what turns "clicks" into "return".

### 3.2 Channel types

| Type | Channels here | How it behaves |
|---|---|---|
| **Paid search** | Google Ads | Captures existing demand; high intent, high cost per click |
| **Paid social** | Facebook, Instagram, LinkedIn | Creates demand; cheap reach on Meta, expensive but precise on LinkedIn (job title, company) |
| **Owned** | Email | Reaches existing contacts; near-zero media cost; strong for retention, renewal and expansion |
| **Organic** | SEO | No media spend; the cost sits in content and people, which is outside this dataset |

### 3.3 Metric glossary

| Metric | Formula | How to read it |
|---|---|---|
| **CTR** | Clicks ÷ Impressions | Ad relevance. A low CTR means the ad or audience is wrong. |
| **CPC** | Spend ÷ Clicks | Price of a visit |
| **CPM** | Spend ÷ Impressions × 1,000 | Price of attention; compares awareness channels |
| **CPL** | Spend ÷ Leads | Price of a contact. Cheap leads are worthless if they never qualify. |
| **Cost per qualified lead** | Spend ÷ Qualified Leads | A fairer CPL; accounts for lead quality |
| **CAC** | Spend ÷ Customers | Price of a paying customer, the number finance cares about |
| **ROAS** | Net Revenue ÷ Spend | Revenue returned per $1 of media |
| **Marketing contribution** | Net Revenue − Spend | Money left after media cost; shows the *size* of a win or loss, which ROAS hides |
| **Qualification rate** | Qualified ÷ Leads | Lead quality by channel |
| **Qualified → converted rate** | Converted ÷ Qualified | Sales-handoff effectiveness |
| **AOV** | Net Revenue ÷ Orders | Typical deal size |
| **Refund rate** | Refunds ÷ Gross Revenue | Revenue leakage |

Gross revenue is the list value of orders; net revenue is after discounts and refunds. ROAS is calculated on **net** revenue, because a refunded order did not earn anything.

### 3.4 Domain rules that shaped the design

1. **Attribution is a business decision, not a technical one.** This model uses **last-touch via the lead**: each order is credited to the campaign, channel and lead source of the lead that produced it. First-touch or multi-touch would give different channel rankings, so the rule is stated in the documentation.
2. **B2B SaaS has a long cycle and repeat revenue.** A lead can convert months later, and much revenue is **renewal and upsell**, not new business. A campaign can therefore "earn" revenue long after its spend, so results are read over fiscal years, not days.
3. **ROAS is not profit.** It ignores cost of goods, salaries and agency fees. It tells you whether media paid for itself, not whether the company made money. A 1.0x ROAS is break-even on media only.
4. **Free channels break ratios.** SEO has no media spend (ROAS undefined) and Email costs almost nothing (ROAS in the hundreds). Contribution, not ROAS, is the fair ranking measure.
5. **Volume is not quality.** A channel with the most leads can still have the weakest qualification, so every lead number is paired with a qualification and conversion rate.
6. **Platform numbers and CRM numbers rarely agree.** Platforms report their own conversions; finance reports orders. This model uses platform data only for *cost and attention* and the CRM and sales data for *outcomes*.
7. **Fiscal calendars drive reporting.** The business reports Sep–Aug, so all year-on-year comparisons use fiscal years.
8. **Metric definitions must be fixed once.** Agreeing that "customer" means a *distinct customer who ordered* (not a lead, not an order) prevents three different CAC numbers in three meetings.

---

## 4. How the project flows, step by step

| # | Phase | What happens | Output | Status |
|---|---|---|---|---|
| 1 | **Discovery** | Workshops with marketing leadership; agree question, KPIs and scope | Requirements (section 2) | ✅ |
| 2 | **Source analysis** | Profile every export; map keys, grain, quality issues | Source map and channel story | ✅ |
| 3 | **Data modelling design** | Choose star schema, fix grain and relationships, freeze V1 | Model design | ✅ |
| 4 | **Power Query build** | Parameters → source → staging → validation → dimensions and facts | Cleaned, keyed tables | ✅ |
| 5 | **Semantic model and DAX** | Date table, relationships, 47 measures, hiding and formats | Governed model | ✅ |
| 6 | **Report design and build** | Wireframe, design system, three pages, drill-through, tooltip | Report (PBIR) | ✅ |
| 7 | **Testing** | Reconcile totals, test slicers, drill-through, interactions | Test log | ✅ |
| 8 | **Source control** | Commit to Azure DevOps, feature branches, pull-request review | Versioned project | 📘 |
| 9 | **Service setup** | Dev / Test / Prod workspaces, gateway, data source credentials | Workspaces | 📘 |
| 10 | **Security** | RLS roles, test "view as", assign users | Secured dataset | 📘 |
| 11 | **Refresh** | Incremental policy, scheduled refresh, failure alerts | Automated data | 📘 |
| 12 | **Release** | Deployment pipeline Dev → Test → Prod | Production report | 📘 |
| 13 | **Sharing** | Publish a Power BI app to audiences, access via security groups | Live for the client | 📘 |
| 14 | **Handover and hypercare** | Documentation, training, monitoring, support window | Run state | 📘 |

---

## 5. Solution architecture

```
┌─────────────────────────┐
│ SOURCE FILES (folder)   │  Ad exports (Google/LinkedIn/Email/SEO, Meta monthly),
│ CSV + Excel             │  CRM leads/customers, campaign, ad-group, product masters
└────────────┬────────────┘
             │  Folder.Files  (path = p_SourceRoot)
             ▼
┌─────────────────────────┐
│ POWER QUERY             │  00_Parameters → 01_Source → 02_Staging → 03_Validation
│ (shape, clean, key)     │  → 04_Dimensions / 05_Facts
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ SEMANTIC MODEL          │  Star schema, Dim_Date (DAX), 47 measures,
│ (Import mode)           │  RLS roles, incremental refresh policy
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ POWER BI REPORT         │  Executive · Channels & Campaigns · Funnel & Customers
│ (PBIR)                  │  + 2 drill-through pages + tooltip
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ POWER BI SERVICE        │  Dev → Test → Prod workspaces, gateway, scheduled refresh,
│                         │  Power BI app, security groups
└────────────┬────────────┘
             ▼
        CLIENT USERS
```

---

## 6. Repository structure (Azure DevOps)

> **This layout and workflow apply to the Azure DevOps team repository only.** It is the working structure used for delivery and is not mirrored in any public GitHub copy of this project.

### 6.1 Folder layout

```
marketing-roi-funnel-analytics/            (Azure DevOps repo)
│
├── README.md                              Project overview and delivery guide (this file)
├── .gitignore                             Excludes cache.abf, localSettings.json, *.pbix, source data
│
├── powerbi/                               Everything Power BI Desktop opens
│   ├── Marketing ROI & Funnel Analytics.pbip
│   ├── Marketing ROI & Funnel Analytics.Report/            PBIR report (pages, visuals, themes)
│   └── Marketing ROI & Funnel Analytics.SemanticModel/     TMDL model (tables, measures, relationships, Power Query)
│
├── data/
│   └── README.md                          Source folder contract: expected subfolders and file names
│                                          (actual data is never committed)
│
├── docs/
│   ├── requirements.md                    Business requirements and KPI definitions
│   ├── report-spec.md                     Approved report design spec
│   ├── data-model.md                      Schema, relationships, measure catalog
│   ├── project-guide.md                   Design decisions and build notes
│   └── images/                            Screenshots and model diagram
│
├── tools/
│   └── build_report.js                    Report-page generator (Node.js)
│
└── .azuredevops/
    └── pull_request_template.md           PR checklist
```

### 6.2 Why it is organised this way

| Folder | Reason |
|---|---|
| `powerbi/` | Keeps the `.pbip`, `.Report` and `.SemanticModel` together so their relative links never break; this is also the folder the Fabric Git integration points at |
| `data/` | The report depends on a source-folder contract, so the contract is documented even though the files are not stored in Git |
| `docs/` | Requirements, design and decisions live beside the code, so a new analyst can onboard without chasing emails |
| `tools/` | Anything that generates or checks project files stays out of the Power BI folders |

### 6.3 Branching and review

| Branch | Purpose |
|---|---|
| `main` | Production-ready; what Prod is released from |
| `develop` | Integration branch; feeds the Dev and Test workspaces |
| `feature/<ticket>-<short-name>` | One change per branch (e.g. `feature/142-roas-yoy-measure`) |
| `hotfix/<ticket>-<short-name>` | Urgent production fix, merged to `main` and back to `develop` |

- Work items (User Story → Task → Bug) in **Azure Boards**; every commit references the work item (`#142`).
- Commit style: `[142] Add ROAS YoY change measure`.
- **Pull-request policy on `develop` and `main`:** one reviewer minimum, linked work item, and the PR checklist completed (totals reconcile, no hand-edited `visual.json` without the generator, no data files, model opens cleanly).
- Why TMDL and PBIR: they are text, so diffs show exactly which measure or visual changed, and merges are reviewable.

### 6.4 Working rules for the team
1. Close Power BI Desktop fully before hand-editing TMDL or moving files.
2. Edit Power Query through `expressions.tmdl` so query groups are preserved.
3. Keep file encoding **UTF-8 without BOM, CRLF, TAB**.
4. Structural report changes go through the spec and generator, not individual visual files.
5. Never commit `cache.abf`, `localSettings.json`, `.pbix` files or source data.

---

## 7. Source data and why it is folder-based

### 7.1 Decision
The project started with SQL plus files. I moved it to a **file and folder-driven** design for two reasons:

1. **Realism.** In many small and mid-sized marketing teams the real inputs are platform exports, CRM extracts and master Excel files dropped in a shared folder.
2. **Power Query does the visible work.** Folder combine, merges, append, standardisation and attribution are all done in the query layer, where the logic is transparent and reviewable.

### 7.2 Source folder contract

```
<p_SourceRoot>/
├── 01_Marketing_Reference_Data/            Channel_Master, Lead_Source_Lookup, Geography_Lookup, Employee_Master (.xlsx)
├── 02_Campaign_Management_System/          Campaign_Master.xlsx
├── 03_Advertising_Platform_Setup/          Ad_Group_Master.xlsx
├── 04_Google_LinkedIn_Email_SEO_Exports/   Ad_Performance_2023 … 2026 (.csv, yearly)
├── 05_Meta_Ads_Exports/                    Meta_Ads_2023_09 … 2026_08 (.csv, monthly)
├── 06_CRM_Customer_Exports/                Customer_Master, Customer_Profile, Customer_Address (.xlsx)
├── 07_CRM_Lead_Exports/                    Leads_2023 … 2026 (.csv, yearly)
├── 08_Product_Master_Data/                 Product_Master.xlsx
└── 09_Sales_Conversion_Exports/            Conversions_2023 … 2026 (.csv, yearly)
```

Numbered folders mirror the business systems each file comes from: reference data, campaign management, ad platform setup, platform exports, CRM, product catalogue and sales.

### 7.3 Why the numbering, and why different export shapes
- **Numbering** gives a stable order and makes it obvious where a new file belongs.
- **Yearly vs monthly files** is deliberate: Google, LinkedIn, Email and SEO exports are yearly, Meta exports are monthly. This reproduces the real situation where each platform delivers differently, and it is the reason a *folder combine* is needed.
- **Masters as Excel, transactions as CSV** matches how business teams maintain reference data (Excel) versus how systems export bulk data (CSV).

### 7.4 Volumes

| Entity | Rows |
|---|---|
| Channels | 6 |
| Campaigns | 240 |
| Ad groups | 720 |
| Lead sources | 7 |
| Customers | 7,500 |
| Products | 24 |
| Geography | 122 locations across 10 countries |
| Ad performance | 360,000 (216,000 core + 144,000 Meta) |
| Leads | 48,000 |
| Orders | 6,000 |

### 7.5 Channel behaviour in the data

| Channel | Behaviour | Approx. lead qualification |
|---|---|---|
| Google Ads | Strong intent, higher CPC, healthy conversion | 52% |
| Facebook | High reach and lead volume, lower quality | 35% |
| Instagram | Awareness-heavy, weak bottom-funnel conversion | 30% |
| LinkedIn | Low volume, high CPL/CAC, high revenue per customer | 62% |
| Email | Very low media cost, retention and expansion | 58% |
| SEO | Zero direct spend, organic, efficient | 48% |

> The data is **synthetic**, generated for portfolio use. Source files are excluded from Git (`.gitignore`); `data/README.md` documents the folder contract so anyone can recreate the structure.

---

## 8. Power Query: every step and why

### 8.1 Design principles
- **One parameter controls the data location.** `p_SourceRoot` is the only place the path lives, so moving the data from a laptop to SharePoint or a network share is a one-value change.
- **Layered queries.** Each layer has one job, so a problem can be located quickly.
- **Reference, never duplicate.** Staging and model queries reference upstream queries so a fix flows through automatically.
- **Only final tables load.** Source, staging and validation queries have *Enable load* switched off, which keeps the model small.
- **Fix data in Power Query, not in DAX.** Types, keys, merges and cleanup belong upstream.

### 8.2 Query layers

| Group | Prefix | Load | Job |
|---|---|---|---|
| `00_Parameters` | `p_SourceRoot`, `RangeStart`, `RangeEnd` | Off | Configuration |
| `01_Source` | `src_*`, `srcFile_BusinessFilesIndex`, `hlp_*` helpers | Off | Read raw files, nothing else |
| `02_Staging` | `stg_*` | Off | Clean, type, enrich, create keys |
| `03_Validation` | `val_*` | Off | Return rows that break data-quality rules (should be empty) |
| `04_Dimensions` | `Dim_*` | On | Final dimension tables |
| `05_Facts` | `Fact_*` | On | Final fact tables |

Flow: `src_` → `stg_` → `Dim_` / `Fact_`.

### 8.3 Step by step

**Step 1: Parameters**
- Create `p_SourceRoot` (text): the root of the data folder.
- Create `RangeStart` and `RangeEnd` (date/time): required for incremental refresh. Values are 2023-09-01 and 2026-09-01.

**Step 2: One file index**
`srcFile_BusinessFilesIndex` runs `Folder.Files(p_SourceRoot)` once. Every master-file query filters this index by `Folder Path` and `Name` instead of hard-coding a full path. *Why:* one folder scan, one place to fix if a folder moves, and every path is built from the parameter.

**Step 3: Master data (Excel)**
For `Channel_Master`, `Lead_Source_Lookup`, `Geography_Lookup`, `Employee_Master`, `Campaign_Master`, `Ad_Group_Master`, `Customer_*` and `Product_Master`:
1. Filter the index to the folder and file name.
2. Take the file's `Content` and open it with `Excel.Workbook`.
3. Expand the data, **promote headers**.
4. **Set data types** explicitly (text, whole number, date, logical).
5. **Remove helper columns** left by the import.
6. **Trim text** in key and descriptive columns.

*Why:* the explicit types and trimmed keys prevent silent merge failures later (a trailing space is enough to break a join).

**Step 4: Transaction files (CSV), folder combine**
For `src_CoreAdFiles`, `src_MetaAdFiles`, `src_LeadFiles`, `src_ConversionFiles`:
1. Filter the index to the single folder for that source.
2. **Exclude hidden files** (`Attributes[Hidden] <> true`) so temp files never enter the load.
3. Apply a **Transform Sample File** function to each file's content.
4. Expand the combined result and **set data types**.

*Why folder combine:* new yearly or monthly files dropped into the folder are picked up on the next refresh with no query change. This is the main reason the project is folder-based.

**Step 5: Staging (`stg_*`)**
- `stg_Channel`, `stg_Product`, `stg_LeadSource`, `stg_Geography`, `stg_Employee`, `stg_AdGroup`: cleaned masters with surrogate keys.
- `stg_Campaign`: **merge** campaign with employee (owner, department, team, manager) and with geography (region, country, market tier, language), then rename the expanded columns.
- `stg_Customer`: **merge** customer master with profile (segment, industry, size, tier) and address, then with geography (country, state, city).
- `stg_CoreAds` and `stg_MetaAds`: standardised ad exports.
- `stg_AdPerformance`: **append** core and Meta exports, then merge keys.
- `stg_Lead`: merge campaign, channel, product, lead source and geography keys; build `LeadCreatedDateKey`.
- `stg_Conversion`: merge each order back to its originating lead.

*Why merges here:* the dimensions end up wide and self-contained (a campaign carries its owner and region), so report users never need to chase lookups.

**Step 6: Key creation**
- Dimension surrogate keys (`ChannelKey`, `CampaignKey`, `AdGroupKey`, `ProductKey`, `LeadSourceKey`, `CustomerKey`).
- **Date keys** as integers in `yyyyMMdd` form (`ReportDateKey`, `LeadCreatedDateKey`, `ConversionDateKey`) to match `Dim_Date[DateKey]`.
- `LeadKey` and `ConversionKey` parsed from the ID text using `try … otherwise null`, so a malformed ID produces a null instead of failing the whole refresh.

*Why integer keys:* they are smaller and faster than text, and they join cleanly to the date dimension.

**Step 7: Last-touch attribution (conversion)**
Orders do not carry campaign, channel or lead source directly. `stg_Conversion` merges each order to its source lead (`LeadBusinessID`) and brings back the lead's campaign, channel and lead-source keys. *Why:* this is what lets revenue be sliced by channel and campaign, which is the core of the ROI story.

**Step 8: Fact queries**
- `Fact_AdPerformance`: ad rows with channel, campaign, ad group, product and date keys.
- `Fact_Lead`: lead rows with all dimension keys and flags (`QualifiedFlag`, `ConvertedFlag`).
- `Fact_Conversion`: orders with attributed keys and revenue columns.
- Each fact **filters its date column between `RangeStart` and `RangeEnd`** (`ReportDate`, `LeadCreatedDate`, `ConversionDate`), which prepares it for incremental refresh.

**Step 9: Validation queries**

| Query | Rule | Expected |
|---|---|---|
| `val_Channel_Duplicates` | Group by channel key, flag any count above 1 | 0 rows |
| `val_Ads_InvalidMetrics` | Negative metrics, reach > impressions, clicks > impressions, unique clicks > clicks, landing-page views > clicks, SEO with paid spend | 0 rows |
| `val_Conversion_Unmatched` | **Left Anti join** of orders to leads: orders with no matching lead | 0 rows |

*Why Left Anti joins:* they return exactly the rows that failed to match, which is the fastest way to find broken keys.

**Step 10: Dimensions**
`Dim_*` queries are thin passthroughs of their staging queries, so all logic stays in one place. `Dim_Date` is **not** in Power Query; it is built in DAX (see [section 9](#9-data-model)).

### 8.4 Problems solved in Power Query

| Problem | Cause | Fix |
|---|---|---|
| Refresh failed: "key didn't match any rows" | Hard-coded source paths | All paths built from `p_SourceRoot` |
| 144K rows with null `CampaignKey` on Meta data | 18-digit campaign IDs read as Int64 and truncated | `ExternalCampaignID` typed as **text** |
| Revenue did not filter by date | `ConversionDateKey` was text; `Dim_Date[DateKey]` is a number | Key created as `Int64.Type` |
| Validation query was a passthrough | Placeholder logic | Replaced by a real Left Anti join |
| Fragile ID parsing | Bad IDs failed the refresh | Wrapped in `try … otherwise null` |
| Dead code | Orphaned `src_ChannelMaster` | Removed |

---

## 9. Data model

A **star (galaxy) schema**: seven dimensions shared by three fact tables. All relationships are **Dimension (1) → Fact (\*)**, single-direction, with **no fact-to-fact relationships**.

```
                         ┌───────────┐
                         │  Dim_Date │
                         └─────┬─────┘
            ┌──────────────────┼──────────────────┐
   ┌────────▼─────────┐  ┌─────▼──────┐   ┌───────▼─────────┐
   │ Fact_AdPerformance│  │ Fact_Lead  │   │ Fact_Conversion │
   └────────┬─────────┘  └─────┬──────┘   └───────┬─────────┘
            │                  │                   │
   Dim_Channel · Dim_Campaign · Dim_AdGroup · Dim_LeadSource
                    Dim_Customer · Dim_Product
```

| Table | Type | Grain / role |
|---|---|---|
| `Dim_Date` | DAX dimension | 1 row per day, 2023-09-01 to 2026-08-31 (1,096 rows), marked as date table, fiscal calendar and hierarchy |
| `Dim_Channel` | Dimension | 6 channels |
| `Dim_Campaign` | Dimension | 240 campaigns, with owner and geography |
| `Dim_AdGroup` | Dimension | 720 ad groups (format, audience, bidding) |
| `Dim_LeadSource` | Dimension | 7 CRM lead sources |
| `Dim_Customer` | Dimension | 7,500 customers with profile and geography |
| `Dim_Product` | Dimension | 24 products |
| `Fact_AdPerformance` | Fact | Channel · campaign · ad group · day |
| `Fact_Lead` | Fact | One row per lead |
| `Fact_Conversion` | Fact | One row per order |
| `Funnel Stage` | Helper | Disconnected: Leads / Qualified / Converted for the funnel visual |
| `_Measures` | Measure table | All 47 measures |

**Relationships (16):** `Fact_AdPerformance` → Date, Channel, Campaign, AdGroup, Product · `Fact_Lead` → Date, Channel, Campaign, Product, LeadSource · `Fact_Conversion` → Date, Channel, Campaign, Product, LeadSource, Customer.

**Why a dedicated `Dim_Date` in DAX:** the source files carry business dates only. A generated calendar covers exactly the model window, adds the **fiscal year (Sep–Aug)** and sorts months fiscally (Sep first). Auto date/time is switched off so there are no hidden date tables.

**Which slices are safe for ratios**

| Slice by | ROAS, CPL, CAC, Contribution | Note |
|---|---|---|
| Date, Channel, Campaign, Product | Valid | Shared by all three facts |
| Ad Group | Ad metrics only | Filters `Fact_AdPerformance` only |
| Lead Source | Leads and revenue only | Not on the spend fact |
| Customer | Revenue and orders only | Not on the spend or lead fact |

**Model hygiene:** raw measure-backing columns and ID/code columns are hidden; numeric attributes are set to *Don't summarize*; implicit measures are discouraged; display folders organise measures; model culture is en-US while the source query culture stays en-IN so parsing is unchanged.

---

## 10. DAX measures

47 measures in `_Measures`, grouped by display folder.

| Folder | Measures |
|---|---|
| **01 Spend & Volume** | Total Spend, Total Impressions, Total Clicks, Total Reach |
| **02 Cost Efficiency** | CTR %, CPC, CPM, CPL, CAC, Cost per Qualified Lead |
| **03 Funnel & Conversion** | Total Leads, Qualified Leads, Converted Leads, Total Customers, Lead Qualification Rate, Qualified to Converted Rate, Lead to Customer Rate, Click to Lead Rate, Funnel Value, Funnel % of Leads |
| **04 Revenue & ROI** | Total Orders, Gross Revenue, Net Revenue, Total Refunds, ROAS, Avg Order Value, Avg Revenue per Customer, Marketing Contribution |
| **05 Time Comparison** | Total Spend / Net Revenue / Leads / Customers PY, ROAS PY, CAC PY, Spend / Net Revenue / Leads / Customers / CAC YoY %, ROAS YoY Change |
| **06 KPI Context** | KPI Context Spend / Net Revenue / ROAS / Leads / Customers / CAC |
| **07 Formatting** (hidden) | ROAS Status Color |

**Patterns used**
- Safe division with `DIVIDE`, so no divide-by-zero errors.
- Prior year with `SAMEPERIODLASTYEAR` on `Dim_Date[Date]`, which works for fiscal year, quarter and month.
- **YoY only when one fiscal year is in context** (`HASONEVALUE(Dim_Date[FiscalYear])`), so multi-year totals never show a misleading comparison.
- KPI-card reference text built with `SWITCH(TRUE(), …)` and `FORMAT`: "▲ 4.2% vs prior FY", "No prior year" or "All fiscal years".
- A disconnected `Funnel Stage` table plus `Funnel Value` (`SWITCH` on `SELECTEDVALUE`) to drive the funnel from three different measures.
- Every measure has a description, a format string and a display folder.

---

## 11. Report development

I designed and built the whole report myself, from wireframe to final polish.

### 11.1 Design process
1. **Wireframe first.** One page per question: *What did we get? Who earns it? Where do we lose leads?*
2. **Design system before visuals.** Colours, fonts, spacing and card style were defined once and applied everywhere.
3. **Build from the model, not around it.** Visuals only use measures from `_Measures`; implicit measures are discouraged.
4. **Accessibility from the start:** AA contrast, alt text on every visual, values never shown by colour alone.

### 11.2 Design system: "Corporate Cool"
- Cool-grey page `#F1F5F9`, white cards with 1px `#E2E8F0` border and 8px radius, slate text, Segoe UI.
- **Signature rule: cyan = money back, slate = money out.** Revenue and return measures use cyan; spend and cost measures use slate. The code holds on every page.
- Channel colours are fixed and colour-blind safe (Google `#0072B2`, Facebook `#E69F00`, Instagram `#CC79A7`, LinkedIn `#009E73`, Email `#D55E00`, SEO `#56B4E9`).
- 1920 × 1080 canvas, 12 × 12 grid, 32px margin, 24px gutter; header band with title, page navigator and slicers on every main page.

### 11.3 Pages

**1. Executive Overview** – *"Every $1 of ad spend returned $1.55 in net revenue"*
Six KPI cards (Spend, Net Revenue, ROAS, Leads, Customers, CAC) with prior-year context; monthly spend vs net revenue on one shared axis; spend vs net revenue by channel with a hover tooltip. Slicers: Fiscal Year, Channel.

**2. Channels & Campaigns** – *"Paid media: $97.4M spend, ranked by return"*
Profit/loss by channel (teal above zero, red below), campaign spend-vs-return matrix, per-channel monthly trends, CTR by ad format, and a campaign leaderboard with ROAS colour-coding. Slicers: Fiscal Year, Campaign Type.

**3. Funnel & Customers** – *"47% of leads qualify, and 1 in 8 converts"*
Lead funnel, qualification vs conversion by channel, revenue by customer segment, country and product category, top customers. Slicers: Fiscal Year, Channel, Product Category.

**Hidden pages**

| Page | Type | Content |
|---|---|---|
| Campaign Detail | Drill-through on `CampaignName` | Profile, six KPIs, monthly trend, the campaign's funnel, ad-group table |
| Customer Detail | Drill-through on `CustomerName` | Profile rail, four KPIs, monthly revenue, order history |
| Channel Tooltip | Report-page tooltip | Channel spend, revenue, ROAS, CAC and trend on hover |

### 11.4 Interactivity
- **Page navigator** on all main pages; **Back** button on drill-through pages.
- **Right-click drill-through** from any campaign or customer.
- **Synced slicers:** Fiscal Year across all three main pages; Channel across Executive Overview and Funnel & Customers.
- **Edited interactions:** customer-based visuals do not filter the lead funnel, because customer attributes do not reach `Fact_Lead` and the funnel would appear unchanged and misleading.

### 11.5 Design decisions made along the way
- **Channel ranking uses Marketing Contribution, not ROAS**, because SEO has $0 spend and Email almost none.
- **Two funnels rather than one**, because impressions (billions) and customers (thousands) cannot share a scale.
- **Fiscal year instead of calendar year**, because 2023 and 2026 are partial calendar years.
- **Combo chart for per-channel trends**: a plain line chart rendered as a blank placeholder in the Desktop build used.
- **$M / $K display units in tables**, so numbers read the same regardless of the viewer's Windows locale.

### 11.6 Report generation
Pages are generated by a Node.js script from the approved spec, which keeps layout, alignment and styling consistent across about 60 visuals. Structural changes are made in the spec and script, then regenerated, not by hand-editing visual files.

---

## 12. Key results and insights from the model

> **How these figures were produced:** calculated directly from the project's source files using the same logic as the model (spend from ad exports, leads from the CRM export, orders attributed to channel and campaign through their source lead, fiscal year = Sep–Aug). The overall totals match the model's validation baseline exactly ($97,364,735 spend, $150,944,966 net revenue, 4,202 customers). Rates are shown to one decimal place.

### 12.1 Overall performance (Sep 2023 – Aug 2026)

| Metric | Value |
|---|---|
| Total Spend | $97,364,735 |
| Net Revenue | $150,944,966 |
| Gross Revenue | $154,885,248 |
| **ROAS** | **1.55x** |
| **Marketing Contribution** | **$53,580,231** |
| Impressions | 2,726,971,427 |
| Clicks | 65,296,970 |
| CTR | 2.4% |
| CPC | $1.49 |
| CPM | $35.70 |
| Leads | 48,000 |
| Qualified Leads | 22,367 (46.6%) |
| Converted Leads | 6,000 (12.5% of leads, 26.8% of qualified) |
| Customers (distinct) | 4,202 |
| Orders | 6,000 |
| Average Order Value | $25,157 |
| Average Revenue per Customer | $35,922 |
| Refunds | $2,175,803 (1.4% of gross revenue) |

### 12.2 Channel scorecard (all three fiscal years)

| Channel | Spend | Net Revenue | Contribution | ROAS | Leads | Qualified % | Qualified → Converted | CPL | CAC |
|---|---|---|---|---|---|---|---|---|---|
| Email | $0.47M | $45.24M | **+$44.77M** | 97.2x | 4,920 | 59.1% | 37.2% | $95 | $721 |
| LinkedIn | $15.78M | $50.87M | **+$35.08M** | 3.22x | 5,920 | 62.0% | 29.0% | $2,666 | $18,332 |
| SEO | $0 | $6.24M | **+$6.24M** | n/a | 5,760 | 49.2% | 26.4% | n/a | n/a |
| Facebook | $17.16M | $10.08M | **−$7.08M** | 0.59x | 11,310 | 35.5% | 22.5% | $1,517 | $20,904 |
| Instagram | $13.97M | $2.24M | **−$11.73M** | 0.16x | 6,660 | 29.7% | 16.8% | $2,097 | $42,067 |
| Google Ads | $49.99M | $36.29M | **−$13.70M** | 0.73x | 13,430 | 51.8% | 26.9% | $3,722 | $29,986 |

### 12.3 Year-on-year (fiscal years)

| Fiscal year | Spend | Net Revenue | ROAS | Leads | Customers | CAC |
|---|---|---|---|---|---|---|
| FY24 (Sep 23 – Aug 24) | $30.45M | $41.41M | 1.36x | 14,965 | 1,558 | $19,546 |
| FY25 (Sep 24 – Aug 25) | $33.71M | $55.63M | 1.65x | 16,383 | 1,720 | $19,597 |
| FY26 (Sep 25 – Aug 26) | $33.21M | $53.90M | 1.62x | 16,652 | 1,841 | $18,037 |

| Change | Spend | Net Revenue | ROAS | Leads | Customers | CAC |
|---|---|---|---|---|---|---|
| FY25 vs FY24 | +10.7% | +34.3% | +21.4% | +9.5% | +10.4% | +0.3% |
| FY26 vs FY25 | −1.5% | −3.1% | −1.6% | +1.6% | +7.0% | −8.0% |

> Customers are counted as distinct per fiscal year, so the three yearly figures add up to more than the 4,202 distinct customers overall (the same customer can order in several years).

### 12.4 What the data says

1. **The headline hides a split portfolio.** The 1.55x overall return comes from three channels earning **+$86.1M** (Email, LinkedIn, SEO) while three channels lose **−$32.5M** (Google Ads, Instagram, Facebook).
2. **The biggest budget earns the least.** Google Ads takes **51% of all spend** ($50.0M) and returns 0.73x, the largest single loss in the portfolio.
3. **LinkedIn is expensive per lead but still the best paid channel.** Its CPL ($2,666) is not the lowest, but each customer is worth about **$59K** in net revenue versus $6.7K for Instagram, $12.3K for Facebook and $21.8K for Google. That is why it returns 3.22x.
4. **Volume is not quality.** Facebook produces 23.6% of all leads (11,310) at the cheapest paid CPL ($1,517), but only **35.5%** qualify and 22.5% of those convert. Instagram is weaker still (29.7% and 16.8%).
5. **Email is the efficiency leader, with a caution:** its revenue is mostly retention and renewal, so it monetises an existing base and does not prove it can scale new acquisition.
6. **Return improved strongly in FY25 and then held.** ROAS rose from 1.36x to 1.65x (revenue +34% on spend +11%), then eased to 1.62x in FY26. In FY26 CAC fell 8.0% to $18,037 while customers grew 7.0%.
7. **Most campaigns do not pay back.** Of the 199 campaigns that had media spend, **127 (64%) returned less than $1 per $1**, and 72 returned more.
8. **Best and worst campaigns**

   | Best contribution | Channel | Contribution | ROAS |
   |---|---|---|---|
   | Singapore CRM Enterprise Customer Retention | Email | +$22.6M | 197x |
   | India CRM Migration Service Customer Retention | Email | +$7.9M | 70x |
   | United States CRM Enterprise Pipeline Generation | LinkedIn | +$6.9M | 4.9x |
   | Germany Enterprise Analytics Lead Generation | LinkedIn | +$6.6M | 4.8x |

   | Worst contribution | Channel | Contribution | ROAS |
   |---|---|---|---|
   | India Email Marketing Suite Customer Acquisition Q3 2023 | Google Ads | −$3.2M | 0.32x |
   | India Data Connector Pack Lead Generation Q3 2023 | Google Ads | −$2.3M | 0.51x |
   | India CRM Starter Brand Awareness Q3 2023 | Facebook | −$2.2M | 0.20x |
   | France CRM Starter Brand Awareness Q3 2023 | Instagram | −$2.1M | 0.08x |

9. **Ad format matters for attention.** Search text, responsive search and display banner ads all reach about **3.8% CTR**, while carousel and lead-form ads sit at about **1.6%**. Search clicks cost about $2.50 and carousel clicks about $1.18, so the cheaper click is also the less engaged one.

### 12.5 Who the revenue comes from

| By customer segment | Net Revenue | Share |
|---|---|---|
| Enterprise | $68.47M | 45.4% |
| Mid-Market | $48.24M | 32.0% |
| SMB | $27.05M | 17.9% |
| Micro Business | $7.19M | 4.8% |

| By country | Net Revenue | Share |
|---|---|---|
| United States | $28.37M | 18.8% |
| India | $28.23M | 18.7% |
| Germany | $18.42M | 12.2% |
| United Kingdom | $14.89M | 9.9% |
| Australia | $13.71M | 9.1% |
| Canada | $13.52M | 9.0% |

| By product category | Net Revenue | Share |
|---|---|---|
| Software Subscription | $87.65M | 58.1% |
| Professional Service | $48.66M | 32.2% |
| Add-On | $14.64M | 9.7% |

- **Enterprise customers deliver 45% of revenue** from 814 accounts, while 1,444 SMB accounts deliver 18%.
- **Revenue is not concentrated in a few accounts:** the top 10 customers hold only **5.4%** of net revenue. The largest, Nova Technology Works, is $1.17M.
- **New business is only about a quarter of revenue.** New Purchase orders are $40.5M (26.8%); Renewal is $32.0M (21.2%) and Upsell $23.0M (15.2%). Spend therefore feeds a customer base that keeps paying, not just first purchases.

### 12.6 Data-quality finding that needs a business decision

Order status in the sales export is **Completed 5,710, Refunded 123, Pending 105, Cancelled 62**. The 290 orders that are not completed carry **$7.2M** of net revenue (4.8%):

| Status | Orders | Net Revenue |
|---|---|---|
| Completed | 5,710 | $143.74M |
| Refunded | 123 | $3.06M |
| Pending | 105 | $2.71M |
| Cancelled | 62 | $1.44M |

The model currently counts all of them, which is why the baseline is $150.9M. If the business decides that only *Completed* orders count, ROAS falls from 1.55x to about **1.48x**, and the change is a filter in the revenue measures. This is logged as an open decision, not changed silently.

### 12.7 Recommendations I would take to the client
1. **Rebalance Google Ads spend.** Pause or cut the campaigns below 1.0x (the worst are Q3 2023 campaigns, led by India) and move budget toward what already earns back.
2. **Scale LinkedIn enterprise campaigns** in the US, Germany and UK, where return is above 4.5x, while watching CAC.
3. **Protect and extend Email retention programs**, and test whether the same approach can support acquisition.
4. **Fix lead quality at Facebook and Instagram** (targeting, form design, qualification criteria) before adding budget, or treat them strictly as awareness channels and judge them on CPM and reach.
5. **Track completed-order revenue** alongside net revenue once the status rule is agreed.
6. **Add gross margin and sales cost** in a later version, so ROAS can be turned into true profitability.

---

## 13. Power BI Service: deployment, RLS, refresh, sharing

> 📘 **Delivery runbook.** This section is the configuration I apply on a client engagement. It describes the intended setup for this project; it is not yet configured in a live workspace.

### 13.1 Workspace strategy

| Workspace | Purpose | Who has access | Source |
|---|---|---|---|
| `MROI – Dev` | Build and unit test | Developers (Admin / Member) | `develop` branch via Git integration |
| `MROI – Test` | Testing and RLS validation | Developers (Admin / Member), test accounts (Viewer) | Deployment pipeline from Dev |
| `MROI – Prod` | Live reporting | Developers (Admin), end users via app only | Deployment pipeline from Test |

- Workspaces use a licensed capacity as agreed with the client (Pro, Premium Per User or Fabric/Premium).
- Dev is connected to the Azure DevOps repo (`powerbi/` folder); Test and Prod receive content only through the pipeline, never by manual upload.

### 13.2 Release flow (Dev → Test → Prod)
1. Developer works on a `feature/*` branch and opens a pull request into `develop`.
2. After review and merge, the Dev workspace syncs from Git.
3. **Deployment pipeline** promotes Dev → Test.
4. **Deployment rules** switch data-source parameters per stage: `p_SourceRoot` points to the stage's source folder.
5. Testing (totals, RLS, refresh) is performed in Test.
6. Once the checks pass, the pipeline promotes Test → Prod and `develop` is merged to `main`.
7. A release note and version tag are recorded.

### 13.3 Data gateway and credentials
- Because the source is a file share, an **on-premises data gateway** (standard mode) is installed on a server that can read `p_SourceRoot`, run under a service account, with a recovery key stored in the client's password vault. If the files move to SharePoint or ADLS, the gateway is no longer required.
- Data-source credentials are set in dataset settings (Windows authentication for the file share).
- Gateway access is limited to the dataset owners and the IT team; a second gateway member is added for resilience.

### 13.4 Row-level security (RLS)

**Why:** regional and channel managers must only see their own results; executives see everything (BR-07).

**Design:** security filters sit on dimensions that every fact table joins to, so one filter protects spend, leads and revenue together.

| Role | Table filter (DAX) | Who |
|---|---|---|
| `Executive` | none | CMO, Marketing Director, analysts |
| `Regional Manager` | `Dim_Campaign[GlobalRegion] = <user's region>` | Regional marketing managers |
| `Channel Manager` | `Dim_Channel[ChannelName] = <user's channel>` | Channel owners |

Dynamic variant, using a small user-access mapping table (user email → region / channel), joined to the dimension:

```dax
-- Role: Regional Manager (applied on the mapping table, which filters Dim_Campaign)
[UserEmail] = USERPRINCIPALNAME ()
```

**Steps**
1. Create roles in Desktop (Modeling → Manage roles) and add the filters.
2. Test with **View as role**, including *Other user* to impersonate a real manager.
3. Publish, then assign **Microsoft Entra security groups** to each role in the dataset's Security page (groups, not individuals).
4. Confirm that workspace Admin / Member / Contributor roles bypass RLS, so end users are given **app access or Viewer only**.
5. Re-test in Test with real test accounts before release.

**Known behaviour to document for the client**
- Customer attributes have no direct path from `Dim_Campaign`; customer data is restricted through the facts, which carry `CampaignKey`.
- Totals in the report are *the user's* total, not the company total; this is stated on the report footer so no one mistakes it for an error.

### 13.5 Incremental refresh

**Why:** three years of history grows every month. Reloading everything daily wastes time and capacity (BR-09).

**Foundation (✅ built):** `RangeStart` and `RangeEnd` parameters exist, and each fact filters its date column between them.

**Policy (📘 applied in Desktop, per fact table):**

| Setting | `Fact_AdPerformance` | `Fact_Lead` | `Fact_Conversion` |
|---|---|---|---|
| Archive data starting | 36 months before refresh date | 36 months | 36 months |
| Incrementally refresh data starting | 10 days before refresh date | 30 days | 30 days |
| Detect data changes | Yes, on a last-modified column if available | Yes | Yes |
| Only refresh complete periods | Yes | Yes | Yes |

**Notes I record for the client**
- Reasoning for the windows: ad platforms restate recent days, so ad data gets a short rolling window; leads and orders can be updated by CRM for a month.
- With a **folder source**, the date filter does not fold, so every file is still read and filtered in the mashup engine. The benefit is then limited to *what is loaded and processed into the model*. Moving the exports to a **database, SharePoint library or data lake** would give the full benefit, and the parameters are already in place for that.
- The first refresh in the Service builds all partitions and is the slowest; later refreshes only touch the incremental window.
- `Dim_Date` is a fixed range in DAX; it must be extended before the data passes Aug 2026 (tracked as a maintenance task).

### 13.6 Scheduled refresh and monitoring
- **Schedule:** daily at 05:30 local time, after the overnight exports land; a second refresh at 13:30 if the business needs an afternoon update (Pro allows 8 per day).
- **Failure notifications** to the dataset owner and the BI support mailbox.
- **Refresh history** reviewed during hypercare; failures are logged as bugs in Azure Boards.
- **Common failure causes and what I check first:** gateway offline, expired credentials, a source file renamed or open/locked, a new file with a different column layout, or `p_SourceRoot` pointing to the wrong stage.
- **Capacity:** the dataset size and refresh duration are noted after the first full refresh to confirm they fit the capacity.

### 13.7 Sharing with the client
1. Build a **Power BI app** from the Prod workspace: navigation with the three main pages, branded name and description.
2. Create **audiences**: *Executives* (all pages), *Managers* (all pages, RLS applies), *Analysts* (all pages plus the dataset for self-service, **Build** permission).
3. Grant access through **security groups**, not individual users.
4. Apply a **sensitivity label** if the client uses Microsoft Purview.
5. Turn off *Export data* / *Download* where the client requires it, and keep the dataset **certified** or **promoted** so analysts connect to the governed model rather than copies.
6. Send an install link and a one-page "how to read this report" guide.

### 13.8 Handover and hypercare
- **Documentation:** this README, the data model and measure catalog, KPI definitions, runbook for refresh failures, RLS maintenance guide (how to add a manager).
- **Training:** a 60-minute walkthrough for managers (filters, drill-through, tooltips) and a 90-minute session for analysts (model, measures, extending the report).
- **Hypercare:** two weeks of monitoring refreshes and fixing defects, then transfer to the client's BI support team.
- **Ownership matrix:** dataset owner, report owner, gateway owner, RLS administrator, escalation contact.

---

## 14. Testing and validation

### 14.1 Reconciliation baseline (✅)
After a full refresh the model must reproduce:

| Check | Expected |
|---|---|
| Ad rows | 360,000 |
| Total Spend | $97,364,735 |
| Total Impressions | 2,726,971,427 |
| Total Clicks | 65,296,970 |
| Total Leads | 48,000 |
| Qualified Leads | 22,367 |
| Total Orders | 6,000 |
| Distinct Customers | 4,202 |
| Gross Revenue | $154,885,248 |
| Net Revenue | $150,944,966 |
| `Dim_Date` rows | 1,096 |

Plus: Net Revenue by fiscal year splits into three different values (proves revenue filters by date); the three fiscal-year spend totals add up to $97,364,735; every `val_*` query returns zero rows.

### 14.2 Test checklist

| Area | Test |
|---|---|
| Data | Row counts and totals match the baseline |
| Model | No inactive or ambiguous relationships; hidden columns; formats |
| Measures | Spot-check each ratio against a manual calculation |
| Time intelligence | FY25 and FY26 show prior-year context; FY24 shows "No prior year" |
| Report | Slicers sync; drill-through passes the right filter; Back works; tooltips open |
| Interactions | Customer visuals do not alter the lead funnel |
| Accessibility | Alt text on all visuals, keyboard tab order, contrast |
| 📘 RLS | Each role sees only its own region/channel; executives see all; totals differ as expected |
| 📘 Refresh | Manual and scheduled refresh succeed; incremental partitions created |
| 📘 Performance | Pages load in a few seconds; checked with Performance Analyzer |

---

## 15. What I did as the Power BI analyst

- **Requirements:** captured the business question, stakeholders, KPIs and scope; turned them into traceable requirements and acceptance criteria.
- **Source analysis:** profiled nine source folders, mapped keys and grain, and identified data-quality risks before building.
- **Power Query:** designed the parameterised, layered query architecture; built folder combines, merges, append, key generation, last-touch attribution and validation queries; diagnosed and fixed refresh failures.
- **Data modelling:** designed a star schema with shared dimensions across three facts; documented which ratios are valid by which dimensions.
- **DAX:** wrote 47 measures including time intelligence and KPI-context text, with descriptions, formats and display folders; built the fiscal-year date dimension.
- **Report design and build:** defined the design system, built three pages, two drill-throughs and a tooltip, with interaction rules and accessibility.
- **Version control:** structured the project as PBIP (TMDL and PBIR) for reviewable diffs and a pull-request workflow.
- **Service delivery (runbook):** designed workspaces, deployment pipeline, gateway, RLS, incremental and scheduled refresh, app audiences and handover.
- **Documentation:** requirements, model, KPI definitions, design spec, decisions log and this README.

---

## 16. What I gained from this project

### 16.1 Technical skills
- **Power Query at scale:** parameter-driven folder combines, Transform-Sample-File helpers, multi-step merges, append, surrogate and date-key creation, `try … otherwise` hardening, and Left Anti-join validation across ~437K source rows.
- **Dimensional modeling:** choosing grain, designing a galaxy schema with three facts sharing dimensions, and understanding which ratios are valid across fact tables.
- **DAX:** time intelligence with a fiscal calendar, "only when one year is selected" logic, text measures for KPI cards, a disconnected helper table to drive a funnel, and measure organisation.
- **Report engineering:** PBIR/TMDL project format, theme and design-system work, drill-through, tooltip pages, slicer syncing and edited interactions.
- **Source control for BI:** treating a Power BI project as code (text diffs, branches, pull requests, generator scripts) instead of an opaque `.pbix` file.
- **Service-side practice (planned and documented):** RLS design, incremental-refresh policy, gateway, deployment pipelines and app distribution, written as the runbook I would follow for a client.

### 16.2 Analytical and domain skills
- Reading a marketing funnel end to end, and knowing why CPL, CAC, ROAS and contribution answer different questions.
- Understanding **attribution** and its limits, and being explicit about the rule used.
- Spotting when a headline metric hides a split underneath (a 1.55x portfolio made of three winners and three losers).
- Choosing the right metric for the situation: contribution instead of ROAS for free channels, fiscal year instead of calendar year, two funnels instead of one.
- Telling a story with numbers and turning findings into recommendations rather than just charts.

### 16.3 Consulting and delivery skills
- Starting from the business question, stakeholders and scope, then tracing each requirement to a visual.
- Writing KPI definitions the business agrees to before building.
- Documenting decisions, assumptions, caveats and open questions (such as the order-status rule) instead of hiding them.
- Planning a full release path: environments, security, refresh and sharing, not only the report.

### 16.4 Problem-solving experience
Real failures that taught practical lessons: a refresh broken by hard-coded paths, 144K null keys caused by 18-digit IDs read as numbers, revenue that would not filter by date because of a text-versus-number key, a model emptied by editing files while Desktop was open, and a chart type that rendered blank. Each is now a rule in this document.

### 16.5 What I would do differently or next
- Move the source files from a folder to SharePoint, a data lake or a database so incremental refresh can fold and the gateway is no longer needed.
- Add budget and target tables to compare plan against actual.
- Add gross margin, so ROAS becomes profit, and a multi-touch attribution view to compare against last-touch.
- Add automated tests (DAX query checks on totals) to the pull-request pipeline.

---

## 17. Decisions, issues and lessons learned

| Topic | Decision or lesson |
|---|---|
| Folder-based source | Keeps Power Query visible and mirrors small/mid-size marketing reality |
| Date table in DAX | Source has business dates only; the calendar is built for the exact window |
| Fiscal year | Calendar years in the data are partial; fiscal years give three complete years |
| V1 scope | Budget/target facts, multi-product orders and payment history were deliberately left out |
| Cross-fact ratios | Spend, leads and revenue are in different facts, so ratios are limited to shared dimensions |
| Text IDs | Large numeric IDs must be text to avoid truncation |
| Type alignment | Fact date keys and `Dim_Date[DateKey]` must have identical types or time filtering silently fails |
| Editing TMDL | Editing model files while Desktop is open has emptied the model before; close Desktop first |
| Rendering | A plain line chart showed blank in the Desktop build used; the combo chart was used instead |
| Locale | Display follows the viewer's Windows locale; $M/$K units keep tables consistent |
| Open decision | Whether customers and revenue should count completed orders only (`ConversionStatus`) is pending business confirmation |

---

## 18. Project status

| Area | Status |
|---|---|
| Requirements, scope, KPIs | ✅ |
| Source analysis and folder contract | ✅ |
| Power Query (parameters, source, staging, validation, dimensions, facts) | ✅ |
| Semantic model, relationships, `Dim_Date`, 47 measures | ✅ |
| Report: 3 pages, 2 drill-throughs, tooltip, theme | ✅ |
| Incremental-refresh parameters and fact filters | ✅ |
| Azure DevOps repo, branching and PR policy | 📘 |
| Workspaces, deployment pipeline, gateway | 📘 |
| RLS roles and group assignment | 📘 |
| Incremental refresh policy and scheduled refresh | 📘 |
| App publishing, client sharing, hypercare | 📘 |

**Remaining items:** final visual review in Desktop and screenshots in `docs/images/`; confirm the `ConversionStatus` rule with the business; extend `Dim_Date` before the data passes Aug 2026.

---

*Data is synthetic and used for portfolio purposes. Released under the MIT License (see LICENSE).*
