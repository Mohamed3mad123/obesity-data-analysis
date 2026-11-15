
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

Below is a step-by-step breakdown of the Python script used to prepare the datasets for analysis and SQL database integration.

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

This block imports the `pandas` library and loads the four raw CSV files into separate DataFrames. It then prints the dimensions (number of rows and columns) of each dataset to establish a baseline understanding of the data volume before cleaning.

### Step 2: Checking for Duplicates and Missing Values

```python
# 2. null & duplicate checks

# count exact duplicate rows
print("Duplicate rows:")
print("Diabetes:", df_diabetes.duplicated().sum())
print("Obesity:", df_obesity.duplicated().sum())
print("Inactivity:", df_inactivity.duplicated().sum())
print("Economic:", df_economic.duplicated().sum())

# count total missing values per dataset
print("\nNull values per dataset:")
print("Diabetes total nulls:", df_diabetes.isnull().sum().sum())
print("Obesity total nulls:", df_obesity.isnull().sum().sum())
print("Economic total nulls:", df_economic.isnull().sum().sum())

# inspect nulls per column for the WHO physical inactivity dataset
inctv_nulls = df_inactivity.isnull().sum()
print("\nInactivity columns with nulls:")
print(inctv_nulls[inctv_nulls > 0])
```

**Description:**

This step performs data quality checks across all four datasets:

* **Duplicate Check:** Verifies that there are no exact duplicate records in any of the files (all returned `0` duplicates).
* **Null Value Check:** Confirms that the Diabetes, Obesity, and Economic datasets have zero missing values. For the WHO Physical Inactivity dataset, it identifies completely empty metadata columns (such as `Dim3` and `FactComments`) that are irrelevant to the analysis and will be excluded during filtering.

### Step 3: Scope Filtering, Column Standardization & Harmonization

```python
# 3. standardization & scope filtering

# project scope: Egypt, Japan, USA from 2000 to 2022
countries = ['EGY', 'JPN', 'USA']
country_names = {'EGY': 'Egypt', 'JPN': 'Japan', 'USA': 'United States'}

# --- 1. clean diabetes data ---
df_diabetes_clean = df_diabetes[
    (df_diabetes['ISO'].isin(countries)) &
    (df_diabetes['Year'].between(2000, 2022))
].copy()

df_diabetes_clean = df_diabetes_clean.rename(columns={
    'Country/Region/World': 'Country',
    'ISO': 'Country_Code',
    'Prevalence of diabetes (18+ years)': 'Diabetes_Prevalence',
    'Prevalence of diabetes (18+ years) lower 95% uncertainty interval': 'Diabetes_Lower_95_UI',
    'Prevalence of diabetes (18+ years) upper 95% uncertainty interval': 'Diabetes_Upper_95_UI',
    'Proportion of people with diabetes who were treated (30+ years)': 'Diabetes_Treatment_Coverage'
})

df_diabetes_clean['Country'] = df_diabetes_clean['Country_Code'].map(country_names)

df_diabetes_clean = df_diabetes_clean[[
    'Country', 'Country_Code', 'Year', 'Sex',
    'Diabetes_Prevalence', 'Diabetes_Lower_95_UI', 'Diabetes_Upper_95_UI', 'Diabetes_Treatment_Coverage'
]].sort_values(['Country_Code', 'Year', 'Sex']).reset_index(drop=True)


# --- 2. clean obesity data ---
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


# --- 3. clean physical inactivity data ---
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
df_inactivity_clean['Sex'] = df_inactivity_clean['Sex'].replace({'Male': 'Men', 'Female': 'Women'})

df_inactivity_clean = df_inactivity_clean[[
    'Country', 'Country_Code', 'Year', 'Sex',
    'Physical_Inactivity_Prevalence', 'Physical_Inactivity_Lower_95_UI', 'Physical_Inactivity_Upper_95_UI'
]].sort_values(['Country_Code', 'Year', 'Sex']).reset_index(drop=True)


# --- 4. clean economic data ---
df_economic_clean = df_economic[
    (df_economic['Country_Code'].isin(countries)) &
    (df_economic['Year'].between(2000, 2022))
].copy()

df_economic_clean['Country'] = df_economic_clean['Country_Code'].map(country_names)
df_economic_clean = df_economic_clean.sort_values(['Country_Code', 'Year']).reset_index(drop=True)

# check final shapes
print("Cleaned Shapes & Null Counts:")
print("Diabetes:", df_diabetes_clean.shape, "| Nulls:", df_diabetes_clean.isnull().sum().sum())
print("Obesity:", df_obesity_clean.shape, "| Nulls:", df_obesity_clean.isnull().sum().sum())
print("Inactivity:", df_inactivity_clean.shape, "| Nulls:", df_inactivity_clean.isnull().sum().sum())
print("Economic:", df_economic_clean.shape, "| Nulls:", df_economic_clean.isnull().sum().sum())
```

**Description:**

This is the core transformation step where all datasets are aligned to a unified schema:

* **Scope Filtering:** Filters records to include only **Egypt (`EGY`)**, **Japan (`JPN`)**, and the **United States (`USA`)** between **2000 and 2022**.
* **Demographic Alignment (WHO Data):** Filters the Physical Inactivity dataset to focus exclusively on adults aged `18+` and excludes the aggregate `'Both sexes'` category so that its granularity matches the NCD-RisC tables (`Men` and `Women`).
* **Schema & Value Standardization:** Renames columns to clean, SQL-friendly identifiers (`Country`, `Country_Code`, `Year`, `Sex`), maps consistent country names (`Egypt`, `Japan`, `United States`), and standardizes gender labels (`Male`/`Female` $\rightarrow$ `Men`/`Women`).
* **Sorting & Indexing:** Selects only relevant analytical columns, sorts records chronologically by country and gender, and resets row indices. The three health datasets result in **138 rows** each ($3 \text{ countries} \times 23 \text{ years} \times 2 \text{ sexes}$), while the country-level economic dataset results in **69 rows** ($3 \text{ countries} \times 23 \text{ years}$).

### Step 4: Time-Series Continuity & Completeness Validation

```python
# 4. check year coverage and make sure no years are missing

# check min and max years in raw files
print("Raw Year Coverage (EGY, JPN, USA):")
print("Diabetes:", df_diabetes[df_diabetes['ISO'].isin(countries)]['Year'].min(), "to", df_diabetes['Year'].max())
print("Obesity:", df_obesity[df_obesity['ISO'].isin(countries)]['Year'].min(), "to", df_obesity['Year'].max())
print("Inactivity:", df_inactivity[df_inactivity['SpatialDimValueCode'].isin(countries)]['Period'].min(), "to", df_inactivity['Period'].max())
print("Economic:", df_economic['Year'].min(), "to", df_economic['Year'].max())

# create a set of all expected years from 2000 to 2022
expected_years = set(range(2000, 2023))

print("\nChecking for missing years (2000-2022) in cleaned data:")

# check per country AND per sex
for name, df in [('Diabetes', df_diabetes_clean), ('Obesity', df_obesity_clean), ('Inactivity', df_inactivity_clean)]:
    missing_found = False
    for code in countries:
        for sex in df['Sex'].unique():
            sub = df[(df['Country_Code'] == code) & (df['Sex'] == sex)]
            actual_years = set(sub['Year'])
            missing = sorted(list(expected_years - actual_years))
            if missing:
                missing_found = True
                print(f"{name} - {code} ({sex}) missing years:", missing)
    if not missing_found:
        print(f"{name}: Complete (0 missing years for all countries and sexes)")

# check economic data separately
econ_missing = False
for code in countries:
    sub = df_economic_clean[df_economic_clean['Country_Code'] == code]
    missing = sorted(list(expected_years - set(sub['Year'])))
    if missing:
        econ_missing = True
        print(f"Economic - {code} missing years:", missing)
if not econ_missing:
    print("Economic: Complete (0 missing years for all countries)")
```

**Description:**

This validation step ensures there are no hidden gaps in the time series:

* **Timeline Verification:** Demonstrates why `2000–2022` was chosen as the shared timeframe (since Physical Inactivity and Economic data start in 2000, whereas NCD-RisC starts in 1990).
* **Granular Gap Detection:** Compares the actual years present in each `(Country, Sex)` subgroup against the expected set of 23 years (`2000–2022`). It confirms **100% completeness** with zero missing years across all four cleaned datasets.

### Step 5: Exporting Cleaned Data (Excel & SQL Ready)

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

Finally, the four cleaned and validated DataFrames are exported into clean `.xlsx` files without index columns (`index=False`) and downloaded locally from Google Colab. Additionally, the notebook includes a commented-out `SQLAlchemy` template for pushing the standardized tables directly into a Microsoft SQL Server database for relational querying.
