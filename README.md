# Car Company — Sales & Revenue Analytics (Power BI)

An end-to-end **Power BI** analytics solution for a car dealership / importer. It turns four raw
Excel workbooks (sales, inventory, lost leads, and staff) into a five-page interactive report that
answers one question: **where is the revenue, and where is it leaking?**

The project is built in the **PBIP (Power BI Project)** format — the report and the semantic model
are stored as plain-text `TMDL` / `JSON` files, so the whole thing is fully version-controllable in
git.

---

## Table of Contents

- [Report Pages](#report-pages)
- [Data Model](#data-model)
- [KPIs / Measures](#kpis--measures)
- [Conclusions — How to Increase Revenue](#conclusions--how-to-increase-revenue)
- [Installation Guide](#installation-guide)
- [Project Structure](#project-structure)

---

## Report Pages

| # | Page | What it answers |
|---|------|-----------------|
| 1 | **Sales & Profit** | How much did we sell, what did we earn, and which brands/models/months drove it? |
| 2 | **Pipeline & Workforce** | How many leads converted, why we lost the rest, and how the team & payroll look. |
| 3 | **Revenue Opportunities** | Where revenue is leaking (discounting, lost deals) and how much is recoverable. |
| 4 | **Inventory & Demand** | What's in stock vs. sold, sell-through, and which models move. |
| 5 | **Customer Segments** | Who buys (age, sex, reason), average spend per segment, and segment revenue. |

---

## Data Model

A simple star-ish schema with four source tables, a date dimension, and a what-if table.

| Table | Grain | Key fields |
|-------|-------|-----------|
| **Customers** | One row per completed sale (2025) | `Model purchased`, `How much paid`, `Discount applied`, `Reason of purchase`, `Age`, `Sex` |
| **Cars** | One row per model offered | `Brand`, `Model`, `Price for the Importer`, `Price for the Customer`, `In stock`, `Sold (2025)` |
| **Potential Customers** | One row per **lost** lead | `Car they were interested in`, `Price of car`, `Reason why they didn't progress` |
| **Workers** | One row per employee | `Department`, `Salary (monthly)`, `Bonuses (annual)` |
| **DimDate** | One row per calendar date | `Date` (links to sale date) |
| **Win-Back Rate** | What-if parameter | `Win-Back Rate` (assumed % of lost pipeline recoverable, default 25%) |

**Relationships**

- `Customers[Model purchased]` → `Cars[ModelFull]` (each sale ties back to its model's cost & list price)
- `Potential Customers[Car they were interested in]` → `Cars[ModelFull]`
- `Customers[Date of purchase]` → `DimDate[Date]`

Two derived columns are computed in Power Query: an **Age band** on customers (`Under 30`, `30–39`,
`40–49`, `50–59`, `60+`) and per-car **Clean profit** = `Price for the Customer − Price for the Importer`.

---

## KPIs / Measures

All measures live in the **`_Measures`** table. They fall into six themes.

### Sales & Profit
| KPI | Definition | Why it matters |
|-----|-----------|----------------|
| **Total Revenue** | `SUM(Customers[How much paid])` | Actual cash collected from sales. |
| **Units Sold** | `COUNTROWS(Customers)` | Volume of completed deals. |
| **Avg Price Paid** | `AVERAGE(Customers[How much paid])` | Typical ticket size. |
| **Total Importer Cost** | `SUMX(Customers, RELATED(Cars[Price for the Importer]))` | Cost of goods actually sold. |
| **Realized Profit** | `Total Revenue − Total Importer Cost` | Gross profit on what we sold. |
| **Realized Margin %** | `Realized Profit / Total Revenue` | Profitability of the sales mix. |
| **Avg Profit per Sale** | `Realized Profit / Units Sold` | Quality of each deal, not just volume. |

### Price Realization & Leakage
| KPI | Definition | Why it matters |
|-----|-----------|----------------|
| **List Price Total** | `SUMX(Customers, RELATED(Cars[Price for the Customer]))` | What we *should* have collected at list. |
| **Price Realization %** | `Total Revenue / List Price Total` | How much of list price we actually capture. |
| **Revenue Leakage** | `List Price Total − Total Revenue` | Money given away through discounting. |
| **Discounted Sales** | Sales where `Discount applied <> "—"` | How widespread discounting is. |
| **Discounted Sales %** | `Discounted Sales / Units Sold` | Share of deals that needed a discount to close. |

### Inventory & Demand
| KPI | Definition | Why it matters |
|-----|-----------|----------------|
| **Total In Stock** | `SUM(Cars[In stock])` | Capital tied up in unsold inventory. |
| **Total Sold (Cars)** | `SUM(Cars[Sold (2025)])` | Units moved per model. |
| **Sell-Through Rate** | `Sold / (Sold + In stock)` | How fast inventory converts to sales. |
| **Models Offered** | `COUNTROWS(Cars)` | Breadth of the catalogue. |
| **Total List Profit** | `SUM(Cars[Total profit (2025)])` | Catalogue-level profit potential. |

### Pipeline & Conversion
| KPI | Definition | Why it matters |
|-----|-----------|----------------|
| **Potential Customers** | `COUNTROWS('Potential Customers')` | Size of the lost-lead pool. |
| **Conversion Rate** | `Total Customers / (Total Customers + Potential Customers)` | How well leads turn into sales. |
| **Lost Pipeline Value** | `SUM('Potential Customers'[Price of car])` | Revenue value of every deal we lost. |

### Recoverable Revenue (lost-reason breakdown)
Each measure slices **Lost Pipeline Value** by the reason a lead didn't progress, so the business can
attack the biggest, most fixable buckets first.
| KPI | Lost reasons it captures |
|-----|--------------------------|
| **Price-Lost Revenue** | "Price too high" |
| **Supply-Lost Revenue** | "Colour / spec unavailable", "Long delivery time" |
| **Competitor-Lost Revenue** | "Chose a competitor", "Found a better deal elsewhere" |
| **Follow-up-Lost Revenue** | "Decided to wait", "Wanted a different model", "Lease terms unsatisfactory" |
| **Win-Back Rate Value** | What-if % assumed recoverable (default 25%) |
| **Win-Back Revenue** | `Lost Pipeline Value × Win-Back Rate Value` |

### Workforce
| KPI | Definition | Why it matters |
|-----|-----------|----------------|
| **Headcount** | `COUNTROWS(Workers)` | Team size. |
| **Avg Monthly Salary** | `AVERAGE(Workers[Salary (monthly)])` | Pay benchmark. |
| **Total Annual Payroll** | `SUMX(Workers, Salary×12 + Bonuses)` | Fixed cost to cover from margin. |
| **Revenue per Sales Rep** | `Total Revenue / count of Sales-dept workers` | Sales-team productivity. |

---

## Conclusions — How to Increase Revenue

The KPIs above point to four concrete, prioritized levers. Open the relevant page to size each one
against the live data before acting.

1. **Plug the discounting leak (Revenue Opportunities page).**
   `Revenue Leakage` and `Price Realization %` quantify money handed back through discounts. Every
   point of price realization recovered flows **straight to profit** (the car is already sold, so
   cost is fixed). Tighten discount authority on the models and reps with the lowest realization,
   and set a floor price per model. **This is the fastest win — no new customers required.**

2. **Recover the lost pipeline (Pipeline & Revenue pages).**
   `Lost Pipeline Value` is the revenue of deals we failed to close. The lost-reason measures show it
   is *not* one problem but four:
   - **Supply-Lost** ("colour/spec unavailable", "long delivery") → fix stocking & ordering of the
     in-demand specs; this is an inventory decision, not a sales one.
   - **Price-Lost** ("price too high") → targeted, rules-based offers rather than blanket discounts.
   - **Competitor-Lost** → close the gap on the specific models customers comparison-shopped.
   - **Follow-up-Lost** ("decided to wait", "wanted a different model") → a structured follow-up /
     win-back program. `Win-Back Revenue` sizes the realistic upside (default 25% of lost pipeline).

3. **Improve mix, not just volume (Sales & Profit + Customer Segments pages).**
   `Avg Profit per Sale` and `Realized Margin %` reveal that some brands/models sell well but earn
   little. Steer marketing and sales incentives toward **high-margin** models and toward the
   **age/sex segments with the highest `Avg Price Paid`**, instead of chasing unit count alone.

4. **Turn over inventory faster (Inventory & Demand page).**
   A low `Sell-Through Rate` means capital is frozen in slow models while fast-movers risk going out
   of stock (feeding the *Supply-Lost* bucket above). Rebalance ordering toward proven sellers and
   discount-clear the long-tail stock to free up cash.

**Bottom line:** the largest, lowest-effort gains are (1) reducing discount leakage and
(2) recovering supply- and follow-up-lost deals — both raise revenue **without** raising spend.

---

## Installation Guide

### Prerequisites
- **Windows** (Power BI Desktop is Windows-only).
- **[Power BI Desktop](https://www.microsoft.com/en-us/download/details.aspx?id=58494)** — install the
  latest version, or get it from the Microsoft Store.
- The PBIP format is supported out of the box in current Power BI Desktop. (On older builds enable it
  via **File → Options and settings → Options → Preview features → "Power BI Project (.pbip) save
  option"**, then restart.)
- **git** to clone the repository.

### Steps
1. **Clone the repo**
   ```bash
   git clone https://github.com/hickennoace/car-company.git
   cd "car-company"
   ```

2. **Fix the data source paths (important).**
   The three data tables load from **absolute paths** baked into the Power Query (M) scripts, e.g.
   `C:\Users\danie\Desktop\Car Company\Customers.xlsx`. Unless you cloned to that exact folder, update
   them to your location. Either:
   - Open `CarCompany.SemanticModel/definition/tables/*.tmdl` and replace the path in the
     `File.Contents("…")` line of `Customers.tmdl`, `Cars.tmdl`, and `Potential Customers.tmdl`, **or**
   - Open the report in Power BI Desktop → **Transform data → Data source settings → Change Source**
     and point each query at the `.xlsx` files in your clone.

3. **Open the project.** Double-click **`CarCompany.pbip`** (or **File → Open** it from Power BI
   Desktop). The report and semantic model load together.

4. **Refresh.** Click **Refresh** on the Home ribbon to pull the latest data from the Excel files.
   All KPIs recalculate automatically.

> **Note on the Excel files:** `Customers.xlsx`, `Cars.xlsx`, `Potential Customers.xlsx`, and
> `Workers.xlsx` are the source data. Keep their sheet names intact (`Customers 2025`, `Cars`,
> `Potential customers`) — the queries reference them by name.

---

## Project Structure

```
Car Company/
├── CarCompany.pbip                 # Entry point — open this in Power BI Desktop
├── CarCompany.Report/              # Report definition (pages & visuals, as JSON)
│   └── definition/pages/           # 5 pages: sales, pipeline, revenue, inventory, segments
├── CarCompany.SemanticModel/       # Data model (TMDL)
│   └── definition/
│       ├── tables/                 # Customers, Cars, Potential Customers, Workers, DimDate, _Measures
│       ├── relationships.tmdl      # Table relationships
│       └── model.tmdl              # Model-level settings
├── Cars.xlsx                       # Source data — catalogue, costs, stock
├── Customers.xlsx                  # Source data — completed sales (2025)
├── Potential Customers.xlsx        # Source data — lost leads + reasons
└── Workers.xlsx                    # Source data — staff & payroll
```

---

*Built with Power BI Desktop · PBIP format · all KPIs defined in `_Measures.tmdl`.*
