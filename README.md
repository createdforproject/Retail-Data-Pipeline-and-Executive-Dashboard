# End-to-End Retail Sales Analysis Pipeline

## 📌 Project Overview
This project builds a localized data processing and visualization pipeline using a standard flat retail dataset (the Kaggle Superstore Sales dataset). The goal was to practice handling data through its entire lifecycle: cleaning unstructured rows in **Pandas**, structuring a relational schema, executing analytical window queries in an in-memory **DuckDB** instance, and building an interactive operational tracking dashboard in **Power BI**.

The repository contains the data preparation scripts, analytical SQL queries, static exploratory plots, and the functional dashboard file.

## 🛠️ Tech Stack & Components
* **Data Processing:** Python 3.x (Pandas)
* **Local Database Engine:** DuckDB (In-Memory OLAP connection)
* **Static Graphics Engine:** Matplotlib, Seaborn
* **Business Intelligence:** Power BI Desktop & DAX (Data Analysis Expressions)

---

## 🏗️ Technical Workflow Execution

### 1. Data Cleaning & Relational Normalization (Pandas)
The raw source file was ingested as a single flat table containing data type conflicts and missing categorical values. The following pipeline steps were executed:
* **Schema Standardization:** Automated the removal of spaces in column headers, replacing them with underscores to prevent parsing syntax errors during subsequent SQL staging.
* **Missing Value Allocation:** Filled 11 blank rows in the `Postal_Code` column with a placeholder value (`0`) and cast the entire field from a floating-decimal type to an integer format.
* **Temporal Parsing:** Standardized string date inputs into explicit `datetime64[ns]` formats to ensure accurate chronological filtering.
* **Star Schema Normalization:** Deconstructed the single sheet into a relational schema to minimize data redundancy:
  * `dim_customer`: Unique identifier lookup containing 793 rows (`Customer_ID`, `Customer_Name`, `Segment`).
  * `dim_product`: Unique catalog containing 1,861 rows (`Product_ID`, `Product_Name`, `Category`, `Sub_Category`).
  * `fact_sale`: Transaction log retaining all 9,800 historical sales grain rows alongside relational foreign keys.

### 2. Analytical Database Ingestion & Window Queries (DuckDB)
The normalized dataframes were programmatically registered into an in-memory DuckDB instance to practice writing relational joins and analytical aggregations using Common Table Expressions (CTEs) and Window Functions.

* **Query 1: Ranking Product Segments Inside Main Categories**
  Evaluated performance variations by ranking sub-categories by sales within their parent groups. `DENSE_RANK()` was chosen specifically to ensure consecutive integers and avoid rank gaps if a revenue tie occurred.
  ```sql
  WITH category_ranks AS (
      SELECT 
          d.Category AS category,
          d.Sub_Category AS sub_category,
          SUM(f.Sales) AS total_sales,
          DENSE_RANK() OVER(PARTITION BY d.Category ORDER BY SUM(f.Sales) DESC) AS ranks
      FROM fact_sale f
      INNER JOIN dim_product d ON d.Product_ID = f.Product_ID
      GROUP BY d.Category, d.Sub_Category
  )
  SELECT category, sub_category, total_sales, ranks 
  FROM category_ranks
  WHERE ranks <= 3;
  ```

* **Query 2: Cumulative Time-Series Ingestion**
  Calculated monthly sales performance along with a running aggregate total tracking across years to evaluate growth vectors chronologically.
  ```sql
  WITH monthly_totals AS (
      SELECT 
          YEAR(Order_Date) AS order_year,
          MONTH(Order_Date) AS order_month,
          SUM(Sales) AS monthly_sales
      FROM fact_sale
      GROUP BY YEAR(Order_Date), MONTH(Order_Date)
  )
  SELECT 
      order_year,
      order_month,
      ROUND(monthly_sales, 2) AS monthly_sales,
      ROUND(SUM(monthly_sales) OVER(PARTITION BY order_year ORDER BY order_month ASC), 2) AS rolling_sum
  FROM monthly_totals
  ORDER BY order_year, order_month;
  ```

### 3. Programmatic Visualizations (Matplotlib & Seaborn)
Constructed two targeted diagnostic plots using the programmatic Object-Oriented interface (`fig, ax = plt.subplots()`):
* **Dual-Axis Chronological Timeline:** Plotted independent monthly revenue as vertical bars overlaid with a shared-axis line (`twinx()`) tracking cumulative growth without squashing the scale.
* **Log-Scaled Distribution Boxplot:** Merged the dimension profiles to check sales volume densities by category. Applied a logarithmic scale (`ax.set_yscale('log')`) to prevent high-value business order outliers from flattening the view of everyday consumer transactions.

### 4. Interactive Analytical Dashboard (Power BI & DAX)
The three sanitized data tables were imported into Power BI Desktop to build an interactive monitoring dashboard. 
* **Data Modeling:** Reconstructed the relational schema by mapping explicit **1-to-Many relationships** linking the customer and product tables to the sales log.
* **Custom DAX Measures:** Programmed explicit measures instead of generic summaries to exercise contextual calculations:
  * *Sub-Category Share:* Utilizes the `ALL` modifier to wipe out column filtering, calculating a segment's relative weight against the overall product lineup.
    ```dax
    SubCategory Contribution Pct = DIVIDE(SUM(fact_sale[Sales]), CALCULATE(SUM(fact_sale[Sales]), ALL(dim_product)))
    ```
  * *Time Shifts:* Uses `SAMEPERIODLASTYEAR` within a `CALCULATE` expression to contrast dynamic growth margins against past temporal frames.
* **Interface Polish:** Unified the visualization styles inside a unified dark-mode corporate skin with active categorical matrix charts, geographical horizontal bar graphs, and dynamic metric slicers.

---


├── data/
│   ├── dim_customer.csv       # Clean Customer Dimension Table
│   ├── dim_product.csv        # Clean Product Dimension Table
│   └── fact_sale.csv          # Clean Transaction Fact Table
├── notebooks/
│   └── sales_pipeline.ipynb   # Full Python Ingestion, DuckDB Queries, & Static Plots
├── powerbi/
│   └── sales_dashboard.pbix   # Live Power BI Model & Functional Dashboard Canvas
└── README.md                  # System Documentation

## 📁 Repository Structure



## 🎯 Project Learnings
Building this repository provided concrete practical experience in:
1. Identifying and resolving data string/date misalignments in Pandas before database loading.
2. Understanding structural data partitioning constraints in relational setups (knowing when data rows should compress in dimensions vs. remain detailed in facts).
3. Working with advanced SQL syntax (CTEs and Window blocks) to isolate specific dataset ranks.
4. Structuring clean layout alignments and scaling variables in visualization interfaces to prevent data compression or illegible text labels.
