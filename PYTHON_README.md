<div align="center">
  <img src="DEPILogo.png" alt="Project Logo" width="180">
  <br><br>
  <a href="README.md">
    <img src="https://img.shields.io/badge/⬅_Back_to_Main_README-README.md-blue?style=for-the-badge" alt="Back to README.md">
  </a>
</div>

---

# 2. Cleaning the Data with Python

In this phase of the project, **Python (Pandas)** was used to clean, standardize, validate, and structure raw datasets sourced from the **NCD Risk Factor Collaboration (NCD-RisC)**, the **World Health Organization (WHO)**, and economic health repositories for **Egypt**, **Japan**, and the **United States** from **2000 to 2022**.

Below is a detailed, step-by-step breakdown of the Python script used to prepare the datasets for analysis and SQL database integration.

---

### Step 1: Loading Raw Datasets & Initial Inspection

```python
import pandas as pd

# 1. load raw 4 datasets
df_diabetes = pd.read_csv('Diabetes Dataset – NCD-RisC.csv')
df_obesity = pd.read_csv('Obesity Dataset – NCD-RisC.csv')
df_inactivity = pd.read_csv('Physical Inactivity.csv')
df_economic = pd.read_csv('economic_health_indicators_egypt_japan_usa_2000_2022.csv')

# check row and column counts
print(f"Diabetes shape: {df_diabetes.shape}")
print(f"Obesity shape: {df_obesity.shape}")
print(f"Physical Inactivity shape: {df_inactivity.shape}")
print(f"Economic Indicators shape: {df_economic.shape}")
```

**Description:**
* **Library Import:** Imports `pandas`, the primary Python library used for tabular data manipulation and cleaning.
* **Data Ingestion:** Reads the four raw CSV files into separate DataFrames (`df_diabetes`, `df_obesity`, `df_inactivity`, `df_economic`).
* **Baseline Dimensions:** Prints `.shape` (rows, columns) for each dataset to record the starting size before applying any filters or transformations.

---

### Step 2: Duplicate & Missing Values Assessment

#### 2.1 Checking for Duplicate Rows
```python
# count exact duplicate rows (True = 1, False = 0, so .sum() gives the total count)
print("Duplicate rows:")
print("Diabetes:", df_diabetes.duplicated().sum())
print("Obesity:", df_obesity.duplicated().sum())
print("Inactivity:", df_inactivity.duplicated().sum())
print("Economic:", df_economic.duplicated().sum())
```

**Description:**
* Uses `.duplicated().sum()` across all four DataFrames to verify data uniqueness.
* All datasets returned `0`, confirming there are no exact duplicate records that need removal.

#### 2.2 Checking for Missing Values (Nulls)
```python
# first .sum() counts nulls per column, second .sum() adds them up for the whole dataset
print("\nNull values per dataset:")
print("Diabetes total nulls:", df_diabetes.isnull().sum().sum())
print("Obesity total nulls:", df_obesity.isnull().sum().sum())
print("Economic total nulls:", df_economic.isnull().sum().sum())

# for inactivity, we only use one .sum() to see nulls per column
# because the raw WHO file has empty metadata columns (like Dim3) that we drop later anyway
inctv_nulls = df_inactivity.isnull().sum()
print("\nInactivity columns with nulls:")
print(inctv_nulls[inctv_nulls > 0])
```

**Description:**
* **NCD-RisC & Economic Files:** Confirms `0` missing values across all cells using `.isnull().sum().sum()`.
* **WHO Physical Inactivity File:** Inspects nulls at the column level and isolates columns where null counts are greater than zero. This reveals that the missing values exist exclusively in unused WHO metadata columns (e.g., `Dim3`, `FactComments`) rather than core analytical fields.

---

### Step 3: Scope Filtering & Schema Standardization

#### 3.1 Defining Global Project Scope & Mapping Rules
```python
# project scope: Egypt, Japan, USA from 2000 to 2022
countries = ['EGY', 'JPN', 'USA']
country_names = {'EGY': 'Egypt', 'JPN': 'Japan', 'USA': 'United States'}
```

**Description:**
* Establishes centralized filter lists and mapping dictionaries so that country codes (`EGY`, `JPN`, `USA`) and standardized country names (`Egypt`, `Japan`, `United States`) remain 100% consistent across all tables.

#### 3.2 Cleaning & Standardizing the Diabetes Dataset
```python
# --- 1. clean diabetes data ---
# filter scope and use .copy() to avoid pandas memory warnings when editing columns
df_diabetes_clean = df_diabetes[
    (df_diabetes['ISO'].isin(countries)) &
    (df_diabetes['Year'].between(2000, 2022))
].copy()

# rename columns so all tables share the exact same key names for SQL joins later
df_diabetes_clean = df_diabetes_clean.rename(columns={
    'Country/Region/World': 'Country',
    'ISO': 'Country_Code',
    'Prevalence of diabetes (18+ years)': 'Diabetes_Prevalence',
    'Prevalence of diabetes (18+ years) lower 95% uncertainty interval': 'Diabetes_Lower_95_UI',
    'Prevalence of diabetes (18+ years) upper 95% uncertainty interval': 'Diabetes_Upper_95_UI',
    'Proportion of people with diabetes who were treated (30+ years)': 'Diabetes_Treatment_Coverage'
})

# standardize country names using our dictionary so spelling matches across all files
df_diabetes_clean['Country'] = df_diabetes_clean['Country_Code'].map(country_names)

# keep only the columns we need, sort, and reset row numbers to start cleanly from 0
df_diabetes_clean = df_diabetes_clean[[
    'Country', 'Country_Code', 'Year', 'Sex',
    'Diabetes_Prevalence', 'Diabetes_Lower_95_UI', 'Diabetes_Upper_95_UI', 'Diabetes_Treatment_Coverage'
]].sort_values(['Country_Code', 'Year', 'Sex']).reset_index(drop=True)
```

**Description:**
* **Filtering:** Extracts rows matching the three target ISO codes and the `2000–2022` timeframe, using `.copy()` to prevent `SettingWithCopyWarning`.
* **Column Renaming:** Converts verbose NCD-RisC headers into concise, SQL-ready column names (`Diabetes_Prevalence`, `Diabetes_Treatment_Coverage`, and 95% uncertainty intervals).
* **Structuring:** Applies standardized country names, selects only the 8 required columns, sorts chronologically by country and sex, and resets the index.

#### 3.3 Cleaning & Standardizing the Obesity Dataset
```python
# --- 2. clean obesity data (same logic as diabetes) ---
df_obesity_clean = df_obesity[
    (df_obesity['ISO'].isin(countries)) &
    (df_obesity['Year'].between(2000, 2022))
].copy()

df_obesity_clean = df_obesity_clean.rename(columns={
    'Country/Region/World': 'Country',
    'ISO': 'Country_Code',
    'Prevalence of BMI<18.5 kg/m² (underweight)': 'Underweight_Prevalence',
    'Prevalence of BMI 25 kg/m² to <30 kg/m²': 'Overweight_Prevalence',
    'Prevalence of BMI>=30 kg/m² (obesity)': 'Obesity_Prevalence',
    'Prevalence of BMI>=30 kg/m² (obesity) lower 95% uncertainty interval': 'Obesity_Lower_95_UI',
    'Prevalence of BMI>=30 kg/m² (obesity) upper 95% uncertainty interval': 'Obesity_Upper_95_UI',
    'Prevalence of BMI 30 kg/m² to <35 kg/m²': 'Obesity_Class_1',
    'Prevalence of BMI 35 kg/m² to <40 kg/m²': 'Obesity_Class_2',
    'Prevalence of BMI >=40 kg/m² (morbid obesity)': 'Morbid_Obesity_Class_3'
})
df_obesity_clean['Country'] = df_obesity_clean['Country_Code'].map(country_names)
df_obesity_clean = df_obesity_clean[[
    'Country', 'Country_Code', 'Year', 'Sex',
    'Underweight_Prevalence', 'Overweight_Prevalence',
    'Obesity_Prevalence', 'Obesity_Lower_95_UI', 'Obesity_Upper_95_UI',
    'Obesity_Class_1', 'Obesity_Class_2', 'Morbid_Obesity_Class_3'
]].sort_values(['Country_Code', 'Year', 'Sex']).reset_index(drop=True)
```

**Description:**
* **BMI Classification Extraction:** Filters the obesity dataset to the same scope and renames complex BMI range descriptions into structured clinical categories (`Underweight_Prevalence`, `Overweight_Prevalence`, `Obesity_Prevalence`, `Obesity_Class_1`, `Obesity_Class_2`, and `Morbid_Obesity_Class_3`).
* **Key Alignment:** Ensures primary keys (`Country`, `Country_Code`, `Year`, `Sex`) match the Diabetes table identically for seamless relational joins.

#### 3.4 Cleaning & Harmonizing the Physical Inactivity Dataset (WHO)
```python
# --- 3. clean physical inactivity data ---
# in WHO data: Dim2 is age group (we only want 18+ adults, na=False prevents crashing on empty cells)
# Dim1 is sex (we keep only Male/Female and drop 'Both sexes' to match the 138 rows in diabetes & obesity)
df_inactivity_clean = df_inactivity[
    (df_inactivity['SpatialDimValueCode'].isin(countries)) &
    (df_inactivity['Period'].between(2000, 2022)) &
    (df_inactivity['Dim2'].str.startswith('18+', na=False)) &
    (df_inactivity['Dim1'].isin(['Male', 'Female']))
].copy()

df_inactivity_clean = df_inactivity_clean.rename(columns={
    'Location': 'Country',
    'SpatialDimValueCode': 'Country_Code',
    'Period': 'Year',
    'Dim1': 'Sex',
    'FactValueNumeric': 'Physical_Inactivity_Prevalence',
    'FactValueNumericLow': 'Physical_Inactivity_Lower_95_UI',
    'FactValueNumericHigh': 'Physical_Inactivity_Upper_95_UI'
})
df_inactivity_clean['Country'] = df_inactivity_clean['Country_Code'].map(country_names)

# change Male/Female to Men/Women so the Sex column matches the NCD-RisC tables
df_inactivity_clean['Sex'] = df_inactivity_clean['Sex'].replace({'Male': 'Men', 'Female': 'Women'})

df_inactivity_clean = df_inactivity_clean[[
    'Country', 'Country_Code', 'Year', 'Sex',
    'Physical_Inactivity_Prevalence', 'Physical_Inactivity_Lower_95_UI', 'Physical_Inactivity_Upper_95_UI'
]].sort_values(['Country_Code', 'Year', 'Sex']).reset_index(drop=True)
```

**Description:**
* **Granularity Filtering:** Filters WHO dimension columns to retain only adults aged `18+` (`Dim2`) and excludes the aggregate `'Both sexes'` category (`Dim1`).
* **Value Harmonization:** Maps WHO column names (`SpatialDimValueCode`, `Period`, `FactValueNumeric`) to the project schema and converts `'Male'`/`'Female'` labels into `'Men'`/`'Women'` to match the NCD-RisC tables.
* **Metadata Removal:** Drops all 27 irrelevant WHO metadata columns, keeping only the 7 essential analytical columns.

#### 3.5 Cleaning the Economic Indicators Dataset & Verifying Final Shapes
```python
# --- 4. clean economic data ---
df_economic_clean = df_economic[
    (df_economic['Country_Code'].isin(countries)) &
    (df_economic['Year'].between(2000, 2022))
].copy()

df_economic_clean['Country'] = df_economic_clean['Country_Code'].map(country_names)
df_economic_clean = df_economic_clean.sort_values(['Country_Code', 'Year']).reset_index(drop=True)

# check final shapes (all 3 health tables should be 138 rows, economic 69 rows, with 0 nulls)
print("Cleaned Shapes & Null Counts:")
print("Diabetes:", df_diabetes_clean.shape, "| Nulls:", df_diabetes_clean.isnull().sum().sum())
print("Obesity:", df_obesity_clean.shape, "| Nulls:", df_obesity_clean.isnull().sum().sum())
print("Inactivity:", df_inactivity_clean.shape, "| Nulls:", df_inactivity_clean.isnull().sum().sum())
print("Economic:", df_economic_clean.shape, "| Nulls:", df_economic_clean.isnull().sum().sum())
```

**Description:**
* **Economic Table Alignment:** Filters and sorts the macroeconomic indicators by country and year.
* **Post-Cleaning Verification:** Confirms that all three gender-disaggregated health tables have exactly **138 rows** ($3 \text{ countries} \times 23 \text{ years} \times 2 \text{ sexes}$), the country-level economic table has **69 rows** ($3 \text{ countries} \times 23 \text{ years}$), and all four tables contain **0 nulls**.

---

### Step 4: Time-Series Continuity & Completeness Validation

```python
# 4. check year coverage and make sure no years are missing

# check min and max years in raw files (shows why 2000-2022 is the shared timeline across all 4 sources)
print("Raw Year Coverage (EGY, JPN, USA):")
print("Diabetes:", df_diabetes[df_diabetes['ISO'].isin(countries)]['Year'].min(), "to", df_diabetes['Year'].max())
print("Obesity:", df_obesity[df_obesity['ISO'].isin(countries)]['Year'].min(), "to", df_obesity['Year'].max())
print("Inactivity:", df_inactivity[df_inactivity['SpatialDimValueCode'].isin(countries)]['Period'].min(), "to", df_inactivity['Period'].max())
print("Economic:", df_economic['Year'].min(), "to", df_economic['Year'].max())

# create a set of all expected years from 2000 to 2022
expected_years = set(range(2000, 2023))

print("\nChecking for missing years (2000-2022) in cleaned data:")

# check per country AND per sex so no missing year hides inside a group
for name, df in [('Diabetes', df_diabetes_clean), ('Obesity', df_obesity_clean), ('Inactivity', df_inactivity_clean)]:
    missing_found = False
    for code in countries:
        for sex in df['Sex'].unique():
            sub = df[(df['Country_Code'] == code) & (df['Sex'] == sex)]
            actual_years = set(sub['Year'])
            # subtract actual years from expected years to spot any gaps
            missing = sorted(list(expected_years - actual_years))
            if missing:
                missing_found = True
                print(f"{name} - {code} ({sex}) missing years:", missing)
    if not missing_found:
        print(f"{name}: Complete (0 missing years for all countries and sexes)")

# economic data is at the country level only (no Sex column), so we check it separately
econ_missing = False
for code in countries:
    sub = df_economic_clean[df_economic_clean['Country_Code'] == code]
    missing = sorted(list(expected_years - set(sub['Year'])))\
    if missing:
        econ_missing = True
        print(f"Economic - {code} missing years:", missing)
if not econ_missing:
    print("Economic: Complete (0 missing years for all countries)")
```

**Description:**
* **Shared Timeline Proof:** Checks minimum and maximum years in the raw datasets, proving that `2000–2022` is the exact overlapping period across all sources (NCD-RisC starts in 1990, whereas WHO and Economic datasets start in 2000).
* **Set-Difference Gap Audit:** Uses Python `set` subtraction (`expected_years - actual_years`) across every individual `(Country, Sex)` subgroup to guarantee there are no hidden missing years inside any cohort.

---

### Step 5: Exporting Cleaned Datasets

```python
# 5: export cleaned datasets

from google.colab import files

# save cleaned dataframes to Excel files
df_obesity_clean.to_excel('obesity.xlsx', index=False)
df_diabetes_clean.to_excel('diabetes.xlsx', index=False)
df_inactivity_clean.to_excel('physical_inactivity.xlsx', index=False)
df_economic_clean.to_excel('economic_health_indicators.xlsx', index=False)

# download files to local machine
files.download('obesity.xlsx')
files.download('diabetes.xlsx')
files.download('physical_inactivity.xlsx')
files.download('economic_health_indicators.xlsx')
```

**Description:**
* **Clean File Generation:** Exports each standardized DataFrame into an Excel workbook (`.xlsx`) with `index=False` so Pandas row numbers are not saved as an extra column.
* **Local Download & SQL Readiness:** Triggers automated downloads in Google Colab for immediate use in Excel, Power BI, or direct ingestion into SQL Server via `SQLAlchemy`.
