# CO₂ Emissions (AR5) — Cleaned Dataset cleaning for Power BI

This repository contains a **cleaned and analysis-ready version** of the World Bank CO₂ emissions dataset, prepared specifically for **Power BI data modeling and dashboarding**.

The raw World Bank download is **not directly suitable** for BI or analytics due to metadata rows, wide year columns, missing values, and aggregate regions mixed with real countries.
This README documents **exactly how the data was cleaned** and why each step was necessary.

---

## 📌 Data Source

* **Indicator:** CO₂ emissions (metric tons, AR5)
* **Indicator Code:** `EN.GHG.CO2.MT.CE.AR5`
* **Source:** World Bank Open Data
* **Download link:**
  [https://data.worldbank.org/indicator/EN.GHG.CO2.MT.CE.AR5](https://data.worldbank.org/indicator/EN.GHG.CO2.MT.CE.AR5)
* **Time range:** 1960–2023
* **Format downloaded:** Excel (`.xls`)

The dataset was downloaded directly from the World Bank indicator page above using the **Download Data → Excel** option.

---

## 📂 Raw File Structure (World Bank Export)

The downloaded Excel file contains multiple sheets:

* `Data`
* `Metadata - Countries`
* `Metadata - Indicators`

Key issues in the raw export:

* First rows contain **notes and metadata**, not data
* One column per year (wide format)
* Missing values represented as `..`
* Includes **aggregate regions and income groups** (e.g., World, OECD, income groups)

---

## 🎯 Cleaning Objectives

* Convert data into **tidy (long) format**
* Ensure **correct numeric data types**
* Remove **aggregate regions and income groups**
* Retain **only real sovereign countries**
* Produce a **stable fact table** for Power BI modeling

---

## 🧹 Data Cleaning Steps

### 1️⃣ Load Data and Skip Metadata Rows

The `Data` sheet contains non-data rows at the top.
These were skipped so that the header row is read correctly.

```python
pd.read_excel(
    path,
    sheet_name="Data",
    skiprows=3
)
```

---

### 2️⃣ Remove Non-Analytical Columns

The following columns were removed as they do not affect analysis:

* `Indicator Name`
* `Indicator Code`

These are constant across the dataset and unnecessary for BI modeling.

---

### 3️⃣ Reshape Data (Wide → Long)

The raw data stores years as column names (e.g., `1960`, `1961`, …).
This was reshaped into **one row per country per year**.

```python
df.melt(
    id_vars=["Country Name", "Country Code"],
    var_name="Year",
    value_name="CO2_Emissions_MT"
)
```

This step is essential for:

* Time-series analysis
* Power BI measures
* SQL-style aggregations

---

### 4️⃣ Fix Data Types and Missing Values

* `Year` → integer
* `CO2_Emissions_MT` → float
* Invalid placeholders such as `..` were converted to `NaN`

```python
pd.to_numeric(..., errors="coerce")
```

This prevents silent errors in Power BI visuals and calculations.

---

### 5️⃣ Remove Aggregate Regions and Income Groups

World Bank exports include aggregates such as:

* World (`WLD`)
* Regional groupings (`AFE`, `ARB`, `CEB`)
* Income groups (`HIC`, `UMC`, `LMC`)

These **must be removed** to avoid double counting and misleading totals.

#### Authoritative method used

The `Metadata - Countries` sheet was used as the **source of truth**.

A row was kept **only if**:

* `Region` is not null
* `Region` ≠ `"Aggregates"`

```python
real_countries = countries[
    countries["Region"].notna() &
    (countries["Region"] != "Aggregates")
]
```

This avoids unreliable heuristics such as:

* Country name string matching
* Country code length checks
* Hard-coded exclusions

---

### 6️⃣ Apply Country Filter to Emissions Data

```python
df_final = df_long[
    df_long["Country Code"].isin(real_countries["Country Code"])
]
```

Result:

* Only real sovereign countries remain
* All global, regional, and income aggregates removed

---

### 7️⃣ Validation Performed

* Verified known aggregates are absent:

  ```
  WLD, AFE, AFW, ARB, CEB, HIC, UMC, LMC, EAS
  ```
* Confirmed ~210 unique countries
* Ensured one row per country–year
* Verified correct numeric data types

---

## 📊 Final Dataset Schema

```
Country Name        (string)
Country Code        (ISO-3)
Year                (integer)
CO2_Emissions_MT    (float, metric tons)
```

* Grain: **Country–Year**
* Unit: **Metric tons**
* Aggregate-free
* Power BI ready

---

## 💾 Output Files

* **`fact_co2_emissions_countries.csv`**
  Clean fact table for Power BI

* **`dim_country.csv`** *(optional)*
  Country dimension for slicers and relationships

---

## 🧠 Intended Use

* Power BI star-schema modeling
* Dashboarding and KPI reporting
* Trend and growth analysis
* Country-level emissions comparison

---

## ⚠️ Notes

* No aggregation was performed during cleaning
* All transformations are reversible and auditable
* Cleaning logic follows **World Bank metadata conventions**
* Dataset is safe for totals, averages, and time-series measures

---

## ✅ Status

**Cleaning complete. Dataset is production-ready for Power BI.**

---

tell me and I’ll add it cleanly.
