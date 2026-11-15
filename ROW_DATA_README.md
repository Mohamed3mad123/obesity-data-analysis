<p align="center">
  <img src="DEPILogo.png" alt="DEPI Logo" width="180">
</p>

<p align="center">
  <a href="README.md">
    <img src="https://img.shields.io/badge/⬅️_Back_to_Main_Project-2E86C1?style=for-the-badge" alt="Back to Main">
  </a>
</p>

---

# <span style="color: #2E86C1;">Raw Data Documentation & Initial Audit</span>

## <span style="color: #2E86C1;">Overview</span>
This folder contains the original, unprocessed datasets (`Raw Data`) collected for the final project. These files serve as the foundational input for our data cleaning, preprocessing, and analysis pipeline using Python and SQL.

Before building the ETL pipeline and Star Schema data warehouse, an initial audit was conducted across these raw CSV datasets to understand their structure, coverage, and data quality issues.

---

## <span style="color: #2E86C1;">Dataset Files & Audit</span>

### 1. Diabetes Dataset
📄 **File:** [`Diabetes Dataset – NCD-RisC.csv`](./Row%20Data/Diabetes%20Dataset%20%E2%80%93%20NCD-RisC.csv)[cite: 2]

* **Description:** Global and country-specific data tracking diabetes prevalence, uncertainty intervals, and treatment proportions across various age groups (starting from 1990), sourced from the NCD Risk Factor Collaboration (NCD-RisC)[cite: 1].
* **Key Columns:** `Country/Region/World`, `ISO`, `Sex`, `Year`, `Age`, `Prevalence of diabetes...`, and `Proportion of people with diabetes who were treated...` (along with their lower and upper 95% uncertainty intervals)[cite: 1].
* **Data Quality Issues Identified:**
  * **Verbose & Unstandardized Headers:** Column names are full descriptive sentences rather than clean snake_case identifiers, making programmatic querying difficult[cite: 1].
  * **Redundant Uncertainty Columns:** Includes lower and upper confidence bound columns for every metric that need to be filtered out to keep only point estimates[cite: 1].
  * **Granularity Mismatch:** Contains both age-standardized (`Age-standardised`) and specific age brackets, requiring filtering to match the target analytical grain[cite: 1].
  * **Decimals Instead of Percentages:** Values are stored as raw proportions (e.g., `0.1021` instead of `10.21%`)[cite: 1].

---

### 2. Obesity Dataset
📄 **File:** [`Obesity Dataset – NCD-RisC.csv`](./Row%20Data/Obesity%20Dataset%20%E2%80%93%20NCD-RisC.csv)[cite: 2]

* **Description:** Comprehensive records on adult body mass index (BMI) distributions, including obesity prevalence, underweight rates, and uncertainty intervals for Egypt, Japan, and the United States from 2000 to 2022[cite: 1].
* **Key Columns:** `Year`, `Sex`, `Country/Region/World`, `ISO`, and multiple prevalence columns (`Prevalence of BMI>=30 kg/m² (obesity)`, `Combined prevalence...`, etc.)[cite: 1].
* **Data Quality Issues Identified:**
  * **Inconsistent Column Ordering:** Unlike the Diabetes file, `Year` and `Sex` appear before `Country/Region/World` and `ISO`, requiring schema alignment before joining[cite: 1].
  * **Wide Format with Excess Metrics:** Contains numerous BMI sub-categories and confidence intervals in a wide structure that must be trimmed down to core obesity indicators[cite: 1].
  * **Lengthy Column Names:** Requires renaming for clean SQL/Python ingestion and dimensional modeling[cite: 1].

---

### 3. Physical Inactivity Dataset
📄 **File:** [`Physical Inactivity.csv`](./Row%20Data/Physical%20Inactivity.csv)[cite: 2]

* **Description:** Statistical indicators measuring insufficient physical activity levels across populations from WHO Global Health Observatory (GHO) indicators to analyze lifestyle correlations with health outcomes[cite: 1].
* **Key Columns:** `IndicatorCode`, `Indicator`, `ParentLocationCode`, `ParentLocation`, `Location type`, `SpatialDimValueCode` (ISO), `Location`, `Period` (Year), `Dim1` (Sex), and numeric value columns[cite: 1].
* **Data Quality Issues Identified:**
  * **Schema & Naming Mismatch:** Uses WHO-specific metadata naming (`SpatialDimValueCode` instead of `ISO`, `Period` instead of `Year`, `Dim1` instead of `Sex`), which must be mapped to match the NCD-RisC datasets[cite: 1].
  * **Mixed Geographic Levels:** The `Location type` column mixes country-level records with regional aggregates, risking double-counting if not filtered strictly to `Country`[cite: 1].
  * **Category Naming Differences:** Uses `"Male"` / `"Female"` / `"Both sexes"` under `Dim1`, whereas the NCD-RisC files use `"Men"` / `"Women"`, requiring value standardization[cite: 1].
  * **Metadata Bloat:** Contains multiple static/redundant indicator and dimension description columns (`IndicatorCode`, `ValueType`, `Dim1 type`, etc.) that add noise to the pipeline[cite: 1].

---

### 4. Economic & Health Indicators
📄 **File:** [`economic_health_indicators_egypt_japan_usa_2000_2022.csv`](./Row%20Data/economic_health_indicators_egypt_japan_usa_2000_2022.csv)[cite: 2]

* **Description:** Macroeconomic and health system indicators—such as GDP per capita (PPP) and health expenditures—structured for comparative analysis across the three target countries from 2000 to 2022[cite: 1].
* **Key Columns:** `Country`, `Country_Code` (ISO), `Year`, `GDP_per_capita_PPP`, and `Health_expenditure_per_capita_current_USD`[cite: 1].
* **Data Quality Issues Identified:**
  * **Time-Range Misalignment:** Data begins in **2000**, whereas the NCD-RisC epidemiological datasets start in **1990**, requiring the overall analysis window to be standardized to **2000–2022**[cite: 1].
  * **Column Naming Inconsistency:** Uses `Country_Code` instead of `ISO`, requiring key alignment for relational joins[cite: 1].
  * **Floating-Point Precision:** Monetary indicators carry unrounded multi-decimal values that require formatting standardization for reporting[cite: 1].

---

## <span style="color: #2E86C1;">Data Source Integrity & Note</span>
* All raw files are maintained in their original formats to ensure traceability and reproducibility. 
* Subsequent cleaning, transformation, and merging steps are handled via Python scripts before loading into the relational database.
