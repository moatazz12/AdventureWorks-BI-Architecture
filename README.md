# AdventureWorks BI Solution - Decision-Support Architecture

**Timeline:** May 2026  
**Context:** Academic Data Engineering & BI Project @ Institut International de Technologie (IIT)  
**Author:** Moataz Melek  

---

## Overview

Modelled and implemented a comprehensive decision-support architecture using the AdventureWorks dataset to extract actionable business insights.

This end-to-end Business Intelligence solution establishes a modern decision-support framework covering the entire analytical lifecycle: from transactional data extraction, data cleansing, enrichment, and dimensional modeling (Star/Fact-Constellation Schema) to multidimensional OLAP cube design and executive interactive dashboards.

---

## Key Contributions

- Designed and deployed multidimensional OLAP data cubes using SQL Server Analysis Services (SSAS).
- Developed robust ETL pipelines to process, clean, and transform complex enterprise data.
- Created interactive dashboards utilizing DAX formulas and Power BI for advanced data visualization.

---

## Architecture & Data Flow

The project follows an enterprise-grade, four-tier decision-support architecture:

```
[ AdventureWorksDW / Relational Source ]
                   │
                   ▼ (SSIS Pipeline: DIMENSIONS_LOAD & ETL_WORKFLOW)
[ Data Warehouse: DW_Moataz_Melek (Star / Fact-Constellation Schema) ]
                   │
                   ▼ (MOLAP Engine: Aggregations, Hierarchies, Measure Groups)
[ SSAS Multidimensional OLAP Cube: Cube_Ventes_Final ]
                   │
                   ▼ (Analytical Layer: DAX, MDX, Interactive Visualizations)
[ Power BI Executive Dashboard (.pbix) & Excel Pivot Analysis (.xlsx) ]
```

---

## Tech Stack

- **Data Warehouse & Storage:** Microsoft SQL Server, Transact-SQL (T-SQL), Star & Constellation Dimensional Schemas (`DW_Moataz_Melek`)
- **ETL & Data Integration:** SQL Server Integration Services (SSIS), SSIS Data Flow Transformations (Derived Columns, Lookups, Conditional Splits, Data Type Conversions, Audit Logging)
- **OLAP & Semantic Modeling:** SQL Server Analysis Services (SSAS) Multidimensional, MOLAP Storage Architecture, MDX (MultiDimensional eXpressions)
- **Data Visualization & Business Analytics:** Microsoft Power BI Desktop (DAX formulas, Time-Intelligence, KPI Cards, Cross-Filtering), Microsoft Excel (PivotTables, OLAP Querying)
- **Development & Database Administration:** Visual Studio / SQL Server Data Tools (SSDT), SQL Server Management Studio (SSMS), Git / GitHub

---

## Dimensional Data Model

The analytical data warehouse is structured as a **Fact-Constellation (Galaxy) Schema** centered on two business processes: **Internet Sales** and **Sales Quota Targets**.

### Fact Tables
- **`Fact_Ventes_Transformees`**:
  - **Granularity:** Individual sales order line items.
  - **Keys:** `OrderDateKey`, `DueDateKey`, `ShipDateKey`, `CustomerKey`, `ProductKey`, `PromotionKey`, `CurrencyKey`, `SalesTerritoryKey`.
  - **Metrics & Derived Attributes:**
    - `OrderQuantity`, `UnitPrice`, `ExtendedAmount`, `UnitPriceDiscountPct`, `DiscountAmount`.
    - `ProductStandardCost`, `TotalProductCost`, `SalesAmount`, `TaxAmt`, `Freight`.
    - `Profit_Net`: Evaluated as `SalesAmount - TotalProductCost`.
    - `Segment_Vente`: Classified dynamically into `Luxe` (`SalesAmount > 1000`) or `Standard`.
    - `Date_Audit`: System timestamp for data lineage and governance.
- **`Fact_Sales_Quota_Transformees`**:
  - **Granularity:** Sales quota targets per employee per fiscal period.
  - **Keys:** `EmployeeKey`, `DateKey`, `CalendarYear`, `CalendarQuarter`.
  - **Metrics & Derived Attributes:**
    - `SalesAmountQuota`: Performance benchmark metric.
    - `Niveau_Ambition`: Business tiering (`SalesAmountQuota > 500000 ? "Objectif Ambitieux" : "Objectif Standard"`).
    - `Date_Audit`: Audit timestamp.

### Dimension Tables & Hierarchies
- **`Dim_Customer_Final`**:
  - Attributes: `CustomerKey`, `Nom_Complet` (`FirstName + ' ' + LastName`), `Email`, `Localisation` (`City (Country)`), `Tranche_Age` (`DATEDIFF > 40 ? 'Senior' : 'Jeune'`), `Sexe` (`Masculin / Féminin`), `Date_Audit`.
  - **Hierarchy:** `Localisation ➔ Tranche Age ➔ Sexe`
- **`Dim_Product_Final`**:
  - Attributes: `ProductKey`, `Nom_Produit`, `Categorie`, `Prix_Standard`, `Couleur`, `Status_Prix` (`StandardCost > 1000 ? 'Haut de gamme' : 'Standard'`), `Date_Audit`.
  - **Hierarchy:** `Categorie ➔ Couleur ➔ Nom Produit`
- **`Dim_Date_Final`**:
  - Attributes: `DateKey`, `Date_Brute`, `Nom_Jour`, `Nom_Mois`, `Num_Mois`, `Trimestre`, `Annee`, `Saison` (engineered meteorological seasonality: `Hiver`, `Printemps`, `Été`, `Automne`), `Date_Audit`.
  - **Hierarchy:** `Annee ➔ Trimestre ➔ Nom Mois`
- **`Dim_Employee_Final`**:
  - Attributes: `EmployeeKey`, `Nom_Complet`, `Titre`, `Genre`, `Email`, `Date_Chargement`.

---

## ETL Pipelines (SSIS Implementation)

The ETL solution is encapsulated in `BI_Project_GLID_Moataz_Melek/Package.dtsx` using modular control and data flow containers:

1. **`DIMENSIONS_LOAD_TRIKI_REKIK` (Sequence Container):**
   - **`Dim_Customer_Load`**: Extracts customer data, performs in-memory cache lookup against territory/geography tables to resolve country and city, executes string concatenation and age grouping, enforces unicode data casting, and loads to the destination.
   - **`Dim_Product_Load`**: Cleans product catalogs, resolves categories, derives cost tiering flags (`Haut de gamme` vs. `Standard`), and applies audit metadata.
   - **`Dim_Date_Load`**: Transforms raw date records into an analytical calendar, mapping quarters, months, and conditional seasonality logic.
   - **`Dim_Employee_Load`**: Standardizes staff records, harmonizes gender values, and applies load timestamps.

2. **`ETL_WORKFLOW_TRIKI_REKIK` (Sequence Container):**
   - **`Flux_Objectifs_Ventes`**: Extracts historical sales quotas, filters invalid/null records using **Conditional Split** (`ISNULL(SalesAmountQuota) == FALSE`), categorizes target ambition levels, and populates `Fact_Sales_Quota_Transformees`.
   - **`Flux_Ventes_Internet`**: Ingests high-volume sales transactions, computes row-level net profitability (`Profit_Net = SalesAmount - TotalProductCost`), flags luxury segments, and bulk-inserts into `Fact_Ventes_Transformees`.

---

## Multidimensional OLAP Cube (SSAS Implementation)

The project leverages a multidimensional SSAS cube (`Cube_Ventes_Final`) built with **MOLAP (Multidimensional OLAP)** storage mode for sub-second analytical querying:
- **Measure Groups:**
  - *Fact Ventes Transformees*: Pre-aggregates `Sales Amount`, `Total Product Cost`, `Profit Net`, `Order Quantity`, `Tax Amt`, and `Freight`.
  - *Fact Sales Quota Transformees*: Pre-aggregates `Sales Amount Quota` across calendar quarters and sales personnel.
- **Multidimensional Roll-ups & Drill-downs:** Natural multi-level hierarchies allow decision-makers to analyze revenues from global regions down to individual customer demographics, product categories down to specific colors, and annual performance down to monthly trends.

---

## Executive Dashboards & Analytics

1. **Power BI Dashboard (`Dashboard_Ventes_Final_Triki_Rekik.pbix.pbix`):**
   - Designed for senior leadership to monitor corporate KPIs, revenue vs. quota variance, customer segmentation, and product performance.
   - Implements advanced DAX calculations for profit margin ratios, period-over-period sales tracking, and territory contribution.
2. **Excel OLAP Pivot Model (`Dashboard_Ventes_Moataz_Melek.xlsx`):**
   - Direct connection to the multidimensional cube for operational ad-hoc slice-and-dice, cross-tabular analysis, and financial reporting.

---

## Project Structure

```
├── BI_Project_GLID_Moataz_Melek/             # SSIS ETL Solution
│   ├── BI_Project_GLID_Moataz_Melek.dtproj   # Integration Services Project
│   ├── Package.dtsx                          # Main ETL Workflow & Data Flow Pipelines
│   └── Project.params                        # ETL Runtime Parameters
├── Cube_Ventes_Final/                        # SSAS OLAP Solution
│   ├── Cube_Ventes_Final.dwproj              # Analysis Services Multidimensional Project
│   ├── DS_Projet.ds                          # Relational Data Source Definition
│   ├── DSV_Ventes.dsv                        # Data Source View & Relational Schema
│   ├── DSV_Ventes.cube                       # Multidimensional Cube & Measures Configuration
│   ├── Dim Customer Final.dim                # Customer Dimension & Hierarchies
│   ├── Dim Date Final.dim                    # Date/Calendar Dimension & Hierarchies
│   ├── Dim Employee Final.dim                # Employee Dimension
│   ├── Dim Product Final.dim                 # Product Dimension & Hierarchies
│   └── DSV_Ventes.partitions                 # Partition & Storage Configurations
├── BI_Project_GLID_Moataz_Melek.sln          # Visual Studio Solution (SSIS)
├── Cube_Ventes_Final.sln                     # Visual Studio Solution (SSAS)
├── Dashboard_Ventes_Final_Triki_Rekik.pbix.pbix # Power BI Executive Dashboard
├── Dashboard_Ventes_Moataz_Melek.xlsx        # Excel Multidimensional Pivot Model
├── .gitignore                                # Git Ignore Configuration
└── README.md                                 # Project Documentation & Architecture Guide
```

---

## Getting Started & Local Exploration

### Prerequisites
- **Database Engine:** Microsoft SQL Server 2019 / 2022
- **OLAP Engine:** Microsoft SQL Server Analysis Services (SSAS Multidimensional Instance)
- **ETL Engine:** SQL Server Integration Services (SSIS)
- **IDE:** Visual Studio (with SQL Server Data Tools / SSDT, SSIS & SSAS extensions installed)
- **Client Tools:** SQL Server Management Studio (SSMS), Microsoft Power BI Desktop, Microsoft Excel

---

### Step 1: Database Setup & Data Warehouse Creation
1. Restore or ensure access to the source **AdventureWorksDW** sample database in your SQL Server instance.
2. Create the target Data Warehouse database named `DW_Moataz_Melek` in SQL Server Management Studio:
   ```sql
   CREATE DATABASE DW_Moataz_Melek;
   GO
   ```
3. Create the target dimension and fact tables matching the schema defined in the SSIS package (`Dim_Customer_Final`, `Dim_Date_Final`, `Dim_Employee_Final`, `Dim_Product_Final`, `Fact_Ventes_Transformees`, `Fact_Sales_Quota_Transformees`).

---

### Step 2: Configure & Execute SSIS ETL Pipelines
1. Open Visual Studio and load `BI_Project_GLID_Moataz_Melek.sln`.
2. In the Solution Explorer, double-click `Package.dtsx`.
3. In the **Connection Managers** pane, update the OLE DB connection strings for:
   - Source database: Pointer to your local `AdventureWorksDW` instance.
   - Destination database: Pointer to your local `DW_Moataz_Melek` instance.
4. Execute the package (**Start** or `F5`). Verify that both sequence containers (`DIMENSIONS_LOAD_TRIKI_REKIK` and `ETL_WORKFLOW_TRIKI_REKIK`) complete successfully with green checkmarks.

---

### Step 3: Deploy & Process SSAS OLAP Cubes
1. Open Visual Studio and load `Cube_Ventes_Final.sln`.
2. In Solution Explorer, open `DS_Projet.ds` and update the connection string to target your local `DW_Moataz_Melek` database.
3. Right-click the `Cube_Ventes_Final` project in Solution Explorer and select **Properties**:
   - Under **Deployment**, ensure the **Server** field targets your SSAS Multidimensional instance (e.g., `localhost` or `localhost\SSAS`).
4. Right-click the project and click **Deploy**.
5. Once deployment succeeds, right-click `DSV_Ventes.cube` and select **Process** ➔ Click **Run** to process the MOLAP partitions and generate cube aggregations.
6. Switch to the **Browser** tab in the cube designer to perform interactive MDX queries and test dimension hierarchies.

---

### Step 4: Explore Dashboards & Analytics
- **Power BI Dashboard:** Launch `Dashboard_Ventes_Final_Triki_Rekik.pbix.pbix` in Microsoft Power BI Desktop. If prompted, adjust the data source settings under *Home > Transform Data > Data Source Settings* to connect to your local SQL Server / SSAS database, and click **Refresh**.
- **Excel Analytical Model:** Open `Dashboard_Ventes_Moataz_Melek.xlsx` in Microsoft Excel. Under the *Data* ribbon, click **Refresh All** to query the processed SSAS cube live via PivotTables.

---

## Academic & Research Alignment

This project demonstrates proficiency in:
- **Decision-Support Engineering:** Designing resilient information systems capable of translating massive transactional enterprise data into actionable analytical indicators.
- **Data Governance & Integrity:** Applying deterministic ETL cleansing rules, handling missing values, standardizing foreign keys, and maintaining continuous audit logging (`Date_Audit`).
- **Scalable Semantic Modeling:** Architecting star/constellation schemas and MOLAP pre-computation pipelines that optimize analytical query throughput.
