# 🌍 Global CO₂ Emissions Analytics (Power BI)

This repository contains a **production-ready Power BI dashboard** and its **fully documented data-cleaning pipeline**, built using **World Bank CO₂ emissions data (AR5)**.

The project emphasizes **correct data modeling, defensible cleaning logic, and analytics integrity** — not just visuals.

---

## 📌 Project Overview

* **Domain:** Climate & Environmental Analytics
* **BI Tool:** Power BI
* **Data Source:** World Bank Open Data
* **Indicator:** CO₂ emissions (metric tons, AR5)
* **Granularity:** Country–Year
* **Time Range:** 1960–2023

### What the dashboard enables

* Explore long-term global CO₂ trends
* Identify top emitting countries
* Analyze individual country emission profiles
* Compare country emissions against global totals
* Safely aggregate without double counting

---

## 📂 Data Source

* **Indicator code:** `EN.GHG.CO2.MT.CE.AR5`
* **Official link:**
  [https://data.worldbank.org/indicator/EN.GHG.CO2.MT.CE.AR5](https://data.worldbank.org/indicator/EN.GHG.CO2.MT.CE.AR5)
* **Format downloaded:** Excel (`.xls`)
* **Sheets used:**

  * `Data`
  * `Metadata - Countries`
  * `Metadata - Indicators`

> ⚠️ The raw World Bank export is **not BI-ready**. It includes metadata rows, wide-format years, missing values, and aggregate regions mixed with countries.

---

## 🎯 Cleaning Objectives

* Convert data into **tidy (long) format**
* Ensure **correct numeric data types**
* Remove **aggregate regions and income groups**
* Retain **only real sovereign countries**
* Produce a **stable, auditable fact table** for Power BI

---

## 🧹 Data Cleaning Process (Documented & Reproducible)

### 1️⃣ Load Data & Skip Metadata Rows

The `Data` sheet contains non-data rows at the top. These were skipped so headers align correctly.

```python
pd.read_excel(
    path,
    sheet_name="Data",
    skiprows=3
)
```

---

### 2️⃣ Remove Non-Analytical Columns

Dropped:

* `Indicator Name`
* `Indicator Code`

These are constant across the dataset and unnecessary for modeling or analysis.

---

### 3️⃣ Reshape Data (Wide → Long)

Raw data stores each year as a separate column.
This was reshaped to **one row per country per year**.

```python
df.melt(
    id_vars=["Country Name", "Country Code"],
    var_name="Year",
    value_name="CO2_Emissions_MT"
)
```

**Why this matters**

* Enables time-series analysis
* Supports DAX measures
* Matches star-schema best practices

---

### 4️⃣ Fix Data Types & Missing Values

* `Year` → integer
* `CO2_Emissions_MT` → float
* Placeholder values (`..`) → `NaN`

```python
pd.to_numeric(..., errors="coerce")
```

This prevents silent aggregation errors in Power BI.

---

### 5️⃣ Remove Aggregate Regions (Authoritative Method)

World Bank exports include aggregates such as:

* World (`WLD`)
* Regions (`AFE`, `ARB`, `CEB`)
* Income groups (`HIC`, `UMC`, `LMC`)

These **must be removed** to avoid double counting.

#### Source of truth used

The **`Metadata - Countries`** sheet.

A country was kept **only if**:

* `Region` is not null
* `Region` ≠ `"Aggregates"`

```python
real_countries = countries[
    countries["Region"].notna() &
    (countries["Region"] != "Aggregates")
]
```

✔ No heuristics
✔ No string matching
✔ Fully auditable

---

### 6️⃣ Apply Country Filter

```python
df_final = df_long[
    df_long["Country Code"].isin(real_countries["Country Code"])
]
```

Result:

* Only sovereign countries remain
* All aggregates removed safely

---

### 7️⃣ Validation Performed

* Verified known aggregates are absent:

  ```
  WLD, AFE, AFW, ARB, CEB, HIC, UMC, LMC, EAS
  ```
* Confirmed ~210 unique countries
* Ensured one row per country–year
* Verified numeric data types

---

## 📊 Final Dataset Schema (Fact Table)

```
Country Name        (string)
Country Code        (ISO-3)
Year                (integer)
CO2_Emissions_MT    (float, metric tons)
```

* **Grain:** Country–Year
* **Aggregate-free**
* **Safe for totals, trends, and comparisons**

---

## 🧱 Power BI Data Model (Star Schema)

### Fact Table

**fact_co2_emissions**

* Country Code (FK)
* Year (FK)
* CO2_Emissions_MT (measure)

### Dimension Tables

**dim_country**

* Country Code (PK)
* Country Name

**dim_year**

* Year (PK)

### Relationships

* `dim_country[Country Code]` → `fact_co2_emissions[Country Code]`
* `dim_year[Year]` → `fact_co2_emissions[Year]`

✔ One-to-many
✔ Single-direction
✔ No dimension-to-dimension joins

---

## 📊 Dashboard Pages

### 1️⃣ Overview

* Total CO₂ emissions KPI
* Global emissions trend (line)
* Top 10 emitting countries (bar)
* Global emissions map
* Country & year slicers

### 2️⃣ Country Profile

* Country CO₂ total
* Emissions trend over time
* Share of global emissions
* Global rank
* Comparison with top emitters

---

## 🧮 Key DAX Measures

```DAX
Total CO2 (MT) =
SUM ( fact_co2_emissions[CO2_Emissions_MT] )
```

```DAX
World CO2 (MT) =
CALCULATE (
    [Total CO2 (MT)],
    ALL ( dim_country )
)
```

```DAX
Country Share of World (%) =
DIVIDE ( [Total CO2 (MT)], [World CO2 (MT)] )
```

```DAX
Country Rank =
RANKX (
    ALL ( dim_country[Country Name] ),
    [Total CO2 (MT)],
    ,
    DESC
)
```

---

## 🎨 Design & UX Principles

* Minimal, environment-focused color palette
* Clear hierarchy: KPIs → trends → comparisons
* Synced slicers across pages
* No implicit measures
* No auto-date tables

---

## 💾 Repository Outputs

* **`fact_co2_emissions_countries.csv`**
  → Clean, aggregate-free fact table

* **`dim_country.csv`** *(optional)*
  → Country dimension for slicers & relationships

* **`global_co2_emissions.pbix`**
  → Power BI dashboard file

---

## 🚀 How to Use

1. Open the `.pbix` file in Power BI Desktop
2. Use slicers to filter by country and year
3. Navigate between **Overview** and **Country Profile**
4. Explore trends, rankings, and geographic patterns

---
