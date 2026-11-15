<div align="center">
  <img src="DEPILogo.png" alt="Project Logo" width="180">
  <br><br>
  <a href="README.md">
    <img src="https://img.shields.io/badge/⬅_Back_to_Main_README-README.md-blue?style=for-the-badge" alt="Back to README.md">
  </a>
  &nbsp;
  <a href="#full-raw-sql-script">
    <img src="https://img.shields.io/badge/📄_View_Full_Raw_SQL_Script-T--SQL-27AE60?style=for-the-badge&logo=microsoftsqlserver&logoColor=white" alt="View Full Raw SQL Script">
  </a>
</div>

---

# 3. Database Modeling & Data Loading with SQL Server

In this phase of the project, **Microsoft SQL Server (T-SQL)** was used to design and populate a relational **Star Schema** data warehouse (`Health_Analytics`) using the cleaned datasets exported from Python.

### ⚠️ Execution Workflow (How to Run the Script)

**Do not execute the entire SQL script at once.** Follow this sequential 3-step workflow in **SQL Server Management Studio (SSMS)**:

1. **Run Step 1 Only:** Highlight and execute the `CREATE DATABASE` block to initialize the `Health_Analytics` database.
2. **Manual Data Import (Step 2):** Right-click `Health_Analytics` in SSMS $\rightarrow$ **Tasks** $\rightarrow$ **Import Flat File** to load the 4 cleaned CSV files into staging tables (`ImportObesity`, `ImportDiabetes`, `ImportInactivity`, `ImportEconomic`).
3. **Run Step 3 to the End:** Highlight from Step 3 down to the end of the file and click **Execute** to validate staging tables, create Dimension and Fact tables, and populate the data warehouse.

### Step 1: Creating & Selecting the Database

```sql
-- =============================================================
-- Step 1: Create the database
-- Select Step 1 only and click Execute.

CREATE DATABASE Health_Analytics;
GO

USE Health_Analytics;
GO
```

**Description:**

- **Database Initialization:** Creates a dedicated database named `Health_Analytics` to house the project's analytical tables.
- **Context Switching:** Uses `USE Health_Analytics;` and the `GO` batch separator to ensure all subsequent operations run inside the newly created database.

### Step 2: Importing Cleaned CSVs into Staging Tables (SSMS)

```sql
-- =============================================================
-- Step 2: Import the CSV files using SSMS
-- Right-click Health_Analytics > Tasks > Import Flat File
--
-- obesity_clean.csv                      -> ImportObesity
-- diabetes_clean.csv                     -> ImportDiabetes
-- physical_inactivity_clean.csv          -> ImportInactivity
-- economic_health_indicators_clean.csv   -> ImportEconomic
```

**Description:**

- **Staging Layer Setup:** Documents the manual mapping between the four cleaned CSV files and their corresponding temporary staging tables inside the `dbo` schema (`ImportObesity`, `ImportDiabetes`, `ImportInactivity`, and `ImportEconomic`) using the SSMS Import Flat File Wizard.

### Step 3: Validating Imported Staging Tables

#### 3.1 Verifying Table Existence & Row Counts

```sql
USE Health_Analytics;
GO

-- Check that the imported tables exist.
SELECT TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_NAME IN (
    'ImportObesity',
    'ImportDiabetes',
    'ImportInactivity',
    'ImportEconomic'
);

-- Count the imported rows.
SELECT
    (SELECT COUNT(*) FROM ImportObesity) AS Obesity,
    (SELECT COUNT(*) FROM ImportDiabetes) AS Diabetes,
    (SELECT COUNT(*) FROM ImportInactivity) AS Inactivity,
    (SELECT COUNT(*) FROM ImportEconomic) AS Economic;
```

**Description:**

- **Metadata Check:** Queries `INFORMATION_SCHEMA.TABLES` to confirm all four staging tables were successfully created with exact spelling before running transformations.
- **Row Count Audit:** Counts records across all four staging tables in a single summary row (`138` rows each for Obesity, Diabetes, and Physical Inactivity; `69` rows for Economic Indicators) to ensure zero data loss during import.

### Step 4: Building & Populating Dimension Tables

```sql
-- Create dimension tables.
CREATE TABLE dbo.DimCountry (Country_Code VARCHAR(3) PRIMARY KEY, Country_Name NVARCHAR(50) NOT NULL);
CREATE TABLE dbo.DimYear ([Year] INT PRIMARY KEY);
CREATE TABLE dbo.DimSex (Sex_ID INT PRIMARY KEY, Sex_Name VARCHAR(10) UNIQUE NOT NULL);

INSERT dbo.DimSex VALUES (1,'Men'),(2,'Women');

INSERT dbo.DimCountry (Country_Code, Country_Name)
SELECT DISTINCT Country_Code, Country FROM dbo.ImportEconomic;

INSERT INTO dbo.DimYear ([Year])
SELECT DISTINCT [Year]
FROM dbo.ImportEconomic;
```

**Description:**

- **Star Schema Dimensions:** Creates three normalized lookup/dimension tables (`DimCountry`, `DimYear`, and `DimSex`) with enforced `PRIMARY KEY` and `UNIQUE` constraints to eliminate text redundancy and optimize query performance.
- **Surrogate Key Encoding (`DimSex`):** Assigns numeric IDs (`1 = Men`, `2 = Women`) so fact tables store compact integers instead of repeated strings.
- **Dynamic Population:** Extracts distinct country codes/names (`EGY`, `JPN`, `USA`) and the 23 distinct years (`2000–2022`) directly from `ImportEconomic`.

### Step 5: Creating & Populating Fact Tables

#### 5.1 Obesity Fact Table (`dbo.FactObesity`)

```sql
-- Obesity fact table.
CREATE TABLE dbo.FactObesity (
    Country_Code VARCHAR(3) NOT NULL,
    [Year] INT NOT NULL,
    Sex_ID INT NOT NULL,
    [Underweight_Prevalence] DECIMAL(20,12) NOT NULL,
    [Overweight_Prevalence] DECIMAL(20,12) NOT NULL,
    [Obesity_Prevalence] DECIMAL(20,12) NOT NULL,
    [Obesity_Lower_95_UI] DECIMAL(20,12) NOT NULL,
    [Obesity_Upper_95_UI] DECIMAL(20,12) NOT NULL,
    [Obesity_Class_1] DECIMAL(20,12) NOT NULL,
    [Obesity_Class_2] DECIMAL(20,12) NOT NULL,
    [Morbid_Obesity_Class_3] DECIMAL(20,12) NOT NULL,
    CONSTRAINT PK_FactObesity PRIMARY KEY ([Country_Code], [Year], [Sex_ID]),
    CONSTRAINT FK_FactObesity_Country FOREIGN KEY (Country_Code) REFERENCES dbo.DimCountry(Country_Code),
    CONSTRAINT FK_FactObesity_Year FOREIGN KEY ([Year]) REFERENCES dbo.DimYear([Year]),
    CONSTRAINT FK_FactObesity_Sex FOREIGN KEY (Sex_ID) REFERENCES dbo.DimSex(Sex_ID)
);
GO

INSERT INTO FactObesity (
    Country_Code, [Year], Sex_ID,
    Underweight_Prevalence, Overweight_Prevalence, Obesity_Prevalence,
    Obesity_Lower_95_UI, Obesity_Upper_95_UI,
    Obesity_Class_1, Obesity_Class_2, Morbid_Obesity_Class_3
)
SELECT
    i.Country_Code, i.[Year], s.Sex_ID,
    i.Underweight_Prevalence, i.Overweight_Prevalence, i.Obesity_Prevalence,
    i.Obesity_Lower_95_UI, i.Obesity_Upper_95_UI,
    i.Obesity_Class_1, i.Obesity_Class_2, i.Morbid_Obesity_Class_3
FROM ImportObesity i
JOIN DimSex s
    ON i.Sex = s.Sex_Name;
GO
```

**Description:**

- **High-Precision Schema & Integrity:** Stores all BMI prevalence metrics using `DECIMAL(20,12)` to preserve exact floating-point precision without rounding errors. Enforces a composite primary key `(Country_Code, Year, Sex_ID)` and foreign keys linking to all three dimension tables.
- **Relational Transformation:** Joins `ImportObesity` with `DimSex` on `Sex = Sex_Name` during insertion to translate text gender labels into integer `Sex_ID` foreign keys.

#### 5.2 Diabetes Fact Table (`dbo.FactDiabetes`)

```sql
-- Diabetes fact table.
CREATE TABLE dbo.FactDiabetes (
    Country_Code VARCHAR(3) NOT NULL,
    [Year] INT NOT NULL,
    Sex_ID INT NOT NULL,
    [Diabetes_Prevalence] DECIMAL(20,12) NOT NULL,
    [Diabetes_Lower_95_UI] DECIMAL(20,12) NOT NULL,
    [Diabetes_Upper_95_UI] DECIMAL(20,12) NOT NULL,
    [Diabetes_Treatment_Coverage] DECIMAL(20,12) NOT NULL,
    CONSTRAINT PK_FactDiabetes PRIMARY KEY ([Country_Code], [Year], [Sex_ID]),
    CONSTRAINT FK_FactDiabetes_Country FOREIGN KEY (Country_Code) REFERENCES dbo.DimCountry(Country_Code),
    CONSTRAINT FK_FactDiabetes_Year FOREIGN KEY ([Year]) REFERENCES dbo.DimYear([Year]),
    CONSTRAINT FK_FactDiabetes_Sex FOREIGN KEY (Sex_ID) REFERENCES dbo.DimSex(Sex_ID)
);
GO

INSERT INTO FactDiabetes (
    Country_Code, [Year], Sex_ID,
    Diabetes_Prevalence, Diabetes_Lower_95_UI, Diabetes_Upper_95_UI, Diabetes_Treatment_Coverage
)
SELECT
    i.Country_Code, i.[Year], s.Sex_ID,
    i.Diabetes_Prevalence, i.Diabetes_Lower_95_UI, i.Diabetes_Upper_95_UI, i.Diabetes_Treatment_Coverage
FROM ImportDiabetes i
JOIN DimSex s
    ON i.Sex = s.Sex_Name;
```

**Description:**

- **Schema & Constraints:** Mirrors the grain of `FactObesity` (`Country_Code, Year, Sex_ID`) to store diabetes prevalence, 95% uncertainty intervals, and treatment coverage rates with full referential integrity.
- **Data Loading:** Maps `Men`/`Women` to `Sex_ID` via an `INNER JOIN` with `DimSex` while inserting records from `ImportDiabetes`.

#### 5.3 Physical Inactivity Fact Table (`dbo.FactPhysicalInactivity`)

```sql
-- Physical inactivity fact table.
CREATE TABLE dbo.FactPhysicalInactivity (
    Country_Code VARCHAR(3) NOT NULL,
    [Year] INT NOT NULL,
    Sex_ID INT NOT NULL,
    [Physical_Inactivity_Prevalence] DECIMAL(20,12) NOT NULL,
    [Physical_Inactivity_Lower_95_UI] DECIMAL(20,12) NOT NULL,
    [Physical_Inactivity_Upper_95_UI] DECIMAL(20,12) NOT NULL,
    CONSTRAINT PK_FactPhysicalInactivity PRIMARY KEY ([Country_Code], [Year], [Sex_ID]),
    CONSTRAINT FK_FactPhysicalInactivity_Country FOREIGN KEY (Country_Code) REFERENCES dbo.DimCountry(Country_Code),
    CONSTRAINT FK_FactPhysicalInactivity_Year FOREIGN KEY ([Year]) REFERENCES dbo.DimYear([Year]),
    CONSTRAINT FK_FactPhysicalInactivity_Sex FOREIGN KEY (Sex_ID) REFERENCES dbo.DimSex(Sex_ID)
);
GO

INSERT INTO FactPhysicalInactivity (
    Country_Code, [Year], Sex_ID,
    Physical_Inactivity_Prevalence, Physical_Inactivity_Lower_95_UI, Physical_Inactivity_Upper_95_UI
)
SELECT
    i.Country_Code, i.[Year], s.Sex_ID,
    i.Physical_Inactivity_Prevalence, i.Physical_Inactivity_Lower_95_UI, i.Physical_Inactivity_Upper_95_UI
FROM ImportInactivity i
JOIN DimSex s
    ON i.Sex = s.Sex_Name;
GO
```

**Description:**

- **Behavioral Risk Storage:** Structures the WHO adult physical inactivity prevalence rates and confidence bounds at the same `(Country_Code, Year, Sex_ID)` granularity so all three health tables can be queried together effortlessly.

#### 5.4 Economic Indicators Fact Table (`dbo.FactEconomic`)

```sql
-- Economic fact table (no Sex_ID here).
CREATE TABLE dbo.FactEconomic (
    Country_Code VARCHAR(3) NOT NULL,
    [Year] INT NOT NULL,
    [GDP_per_capita_PPP_constant_2021_intl_dollars] DECIMAL(20,12) NOT NULL,
    [Health_expenditure_per_capita_current_USD] DECIMAL(20,12) NOT NULL,
    CONSTRAINT PK_FactEconomic PRIMARY KEY ([Country_Code], [Year]),
    CONSTRAINT FK_FactEconomic_Country FOREIGN KEY (Country_Code) REFERENCES dbo.DimCountry(Country_Code),
    CONSTRAINT FK_FactEconomic_Year FOREIGN KEY ([Year]) REFERENCES dbo.DimYear([Year])
);
GO

INSERT INTO FactEconomic (
    Country_Code, [Year],
    GDP_per_capita_PPP_constant_2021_intl_dollars, Health_expenditure_per_capita_current_USD
)
SELECT
    Country_Code, [Year],
    GDP_per_capita_PPP_constant_2021_intl_dollars, Health_expenditure_per_capita_current_USD
FROM ImportEconomic;
GO
```

**Description:**

- **Country-Level Granularity:** Because macroeconomic indicators (`GDP per capita` and `Health expenditure per capita`) apply at the national level rather than by gender, this fact table uses a two-column composite primary key `(Country_Code, Year)` and links only to `DimCountry` and `DimYear` (omitting `Sex_ID`).
- **Direct Insertion:** Loads all 69 country-year records directly from `ImportEconomic` without requiring a join to `DimSex`.

---

## Full Raw SQL Script

Below is the complete, unsplit T-SQL script for quick copying or reference:

```sql
-- =============================================================
-- --- IMPORTANT: Do not run the whole script at once. ---
-- -- Run Step 1, import files, then run Step 3 to the end.
-- =============================================================
-- Step 1: Create the database
-- Select Step 1 only and click Execute.

CREATE DATABASE Health_Analytics;
GO

USE Health_Analytics;
GO

-- =============================================================
-- Step 2: Import the CSV files using SSMS
-- Right-click Health_Analytics > Tasks > Import Flat File
--
-- obesity_clean.csv                      -> ImportObesity
-- diabetes_clean.csv                     -> ImportDiabetes
-- physical_inactivity_clean.csv          -> ImportInactivity
-- economic_health_indicators_clean.csv   -> ImportEconomic

-- =============================================================
-- Step 3: Build the tables and run the analysis.
-- Select Step 3 to the end and click Execute AFTER importing the CSV files.
-- Required imported table names: dbo.ImportObesity, dbo.ImportDiabetes,
-- dbo.ImportInactivity, dbo.ImportEconomic.

USE Health_Analytics;
GO

-- Check that the imported tables exist.
SELECT TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_NAME IN (
    'ImportObesity',
    'ImportDiabetes',
    'ImportInactivity',
    'ImportEconomic'
);

-- Count the imported rows.
SELECT
    (SELECT COUNT(*) FROM ImportObesity) AS Obesity,
    (SELECT COUNT(*) FROM ImportDiabetes) AS Diabetes,
    (SELECT COUNT(*) FROM ImportInactivity) AS Inactivity,
    (SELECT COUNT(*) FROM ImportEconomic) AS Economic;

-- Create dimension tables.
CREATE TABLE dbo.DimCountry (Country_Code VARCHAR(3) PRIMARY KEY, Country_Name NVARCHAR(50) NOT NULL);
CREATE TABLE dbo.DimYear ([Year] INT PRIMARY KEY);
CREATE TABLE dbo.DimSex (Sex_ID INT PRIMARY KEY, Sex_Name VARCHAR(10) UNIQUE NOT NULL);
INSERT dbo.DimSex VALUES (1,'Men'),(2,'Women');
INSERT dbo.DimCountry (Country_Code, Country_Name)
SELECT DISTINCT Country_Code, Country FROM dbo.ImportEconomic;
INSERT INTO dbo.DimYear ([Year])
SELECT DISTINCT [Year]
FROM dbo.ImportEconomic;

-- Obesity fact table.
CREATE TABLE dbo.FactObesity (
    Country_Code VARCHAR(3) NOT NULL,
    [Year] INT NOT NULL,
    Sex_ID INT NOT NULL,
    [Underweight_Prevalence] DECIMAL(20,12) NOT NULL,
    [Overweight_Prevalence] DECIMAL(20,12) NOT NULL,
    [Obesity_Prevalence] DECIMAL(20,12) NOT NULL,
    [Obesity_Lower_95_UI] DECIMAL(20,12) NOT NULL,
    [Obesity_Upper_95_UI] DECIMAL(20,12) NOT NULL,
    [Obesity_Class_1] DECIMAL(20,12) NOT NULL,
    [Obesity_Class_2] DECIMAL(20,12) NOT NULL,
    [Morbid_Obesity_Class_3] DECIMAL(20,12) NOT NULL,
    CONSTRAINT PK_FactObesity PRIMARY KEY ([Country_Code], [Year], [Sex_ID]),
    CONSTRAINT FK_FactObesity_Country FOREIGN KEY (Country_Code) REFERENCES dbo.DimCountry(Country_Code),
    CONSTRAINT FK_FactObesity_Year FOREIGN KEY ([Year]) REFERENCES dbo.DimYear([Year]),
    CONSTRAINT FK_FactObesity_Sex FOREIGN KEY (Sex_ID) REFERENCES dbo.DimSex(Sex_ID)
);
GO

INSERT INTO FactObesity (
    Country_Code,
    [Year],
    Sex_ID,
    Underweight_Prevalence,
    Overweight_Prevalence,
    Obesity_Prevalence,
    Obesity_Lower_95_UI,
    Obesity_Upper_95_UI,
    Obesity_Class_1,
    Obesity_Class_2,
    Morbid_Obesity_Class_3
)
SELECT
    i.Country_Code,
    i.[Year],
    s.Sex_ID,
    i.Underweight_Prevalence,
    i.Overweight_Prevalence,
    i.Obesity_Prevalence,
    i.Obesity_Lower_95_UI,
    i.Obesity_Upper_95_UI,
    i.Obesity_Class_1,
    i.Obesity_Class_2,
    i.Morbid_Obesity_Class_3
FROM ImportObesity i
JOIN DimSex s
    ON i.Sex = s.Sex_Name;
GO

-- Diabetes fact table.
CREATE TABLE dbo.FactDiabetes (
    Country_Code VARCHAR(3) NOT NULL,
    [Year] INT NOT NULL,
    Sex_ID INT NOT NULL,
    [Diabetes_Prevalence] DECIMAL(20,12) NOT NULL,
    [Diabetes_Lower_95_UI] DECIMAL(20,12) NOT NULL,
    [Diabetes_Upper_95_UI] DECIMAL(20,12) NOT NULL,
    [Diabetes_Treatment_Coverage] DECIMAL(20,12) NOT NULL,
    CONSTRAINT PK_FactDiabetes PRIMARY KEY ([Country_Code], [Year], [Sex_ID]),
    CONSTRAINT FK_FactDiabetes_Country FOREIGN KEY (Country_Code) REFERENCES dbo.DimCountry(Country_Code),
    CONSTRAINT FK_FactDiabetes_Year FOREIGN KEY ([Year]) REFERENCES dbo.DimYear([Year]),
    CONSTRAINT FK_FactDiabetes_Sex FOREIGN KEY (Sex_ID) REFERENCES dbo.DimSex(Sex_ID)
);
GO

INSERT INTO FactDiabetes (
    Country_Code,
    [Year],
    Sex_ID,
    Diabetes_Prevalence,
    Diabetes_Lower_95_UI,
    Diabetes_Upper_95_UI,
    Diabetes_Treatment_Coverage
)
SELECT
    i.Country_Code,
    i.[Year],
    s.Sex_ID,
    i.Diabetes_Prevalence,
    i.Diabetes_Lower_95_UI,
    i.Diabetes_Upper_95_UI,
    i.Diabetes_Treatment_Coverage
FROM ImportDiabetes i
JOIN DimSex s
    ON i.Sex = s.Sex_Name;

-- Physical inactivity fact table.
CREATE TABLE dbo.FactPhysicalInactivity (
    Country_Code VARCHAR(3) NOT NULL,
    [Year] INT NOT NULL,
    Sex_ID INT NOT NULL,
    [Physical_Inactivity_Prevalence] DECIMAL(20,12) NOT NULL,
    [Physical_Inactivity_Lower_95_UI] DECIMAL(20,12) NOT NULL,
    [Physical_Inactivity_Upper_95_UI] DECIMAL(20,12) NOT NULL,
    CONSTRAINT PK_FactPhysicalInactivity PRIMARY KEY ([Country_Code], [Year], [Sex_ID]),
    CONSTRAINT FK_FactPhysicalInactivity_Country FOREIGN KEY (Country_Code) REFERENCES dbo.DimCountry(Country_Code),
    CONSTRAINT FK_FactPhysicalInactivity_Year FOREIGN KEY ([Year]) REFERENCES dbo.DimYear([Year]),
    CONSTRAINT FK_FactPhysicalInactivity_Sex FOREIGN KEY (Sex_ID) REFERENCES dbo.DimSex(Sex_ID)
);
GO

INSERT INTO FactPhysicalInactivity (
    Country_Code,
    [Year],
    Sex_ID,
    Physical_Inactivity_Prevalence,
    Physical_Inactivity_Lower_95_UI,
    Physical_Inactivity_Upper_95_UI
)
SELECT
    i.Country_Code,
    i.[Year],
    s.Sex_ID,
    i.Physical_Inactivity_Prevalence,
    i.Physical_Inactivity_Lower_95_UI,
    i.Physical_Inactivity_Upper_95_UI
FROM ImportInactivity i
JOIN DimSex s
    ON i.Sex = s.Sex_Name;
GO

-- Economic fact table (no Sex_ID here).
CREATE TABLE dbo.FactEconomic (
    Country_Code VARCHAR(3) NOT NULL,
    [Year] INT NOT NULL,
    [GDP_per_capita_PPP_constant_2021_intl_dollars] DECIMAL(20,12) NOT NULL,
    [Health_expenditure_per_capita_current_USD] DECIMAL(20,12) NOT NULL,
    CONSTRAINT PK_FactEconomic PRIMARY KEY ([Country_Code], [Year]),
    CONSTRAINT FK_FactEconomic_Country FOREIGN KEY (Country_Code) REFERENCES dbo.DimCountry(Country_Code),
    CONSTRAINT FK_FactEconomic_Year FOREIGN KEY ([Year]) REFERENCES dbo.DimYear([Year])
);
GO

INSERT INTO FactEconomic (
    Country_Code,
    [Year],
    GDP_per_capita_PPP_constant_2021_intl_dollars,
    Health_expenditure_per_capita_current_USD
)
SELECT
    Country_Code,
    [Year],
    GDP_per_capita_PPP_constant_2021_intl_dollars,
    Health_expenditure_per_capita_current_USD
FROM ImportEconomic;
GO
```
