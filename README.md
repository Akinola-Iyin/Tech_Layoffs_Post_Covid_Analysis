# Data Cleaning and Exploratory Data Analysis (EDA) using SQL

## Project Overview
This project involved cleaning and analyzing a dataset containing information on company layoffs. The data was sourced from Kaggle and imported into MySQL for structured processing. The primary objectives were:

- Cleaning and standardizing the dataset
- Removing duplicates and handling missing values
- Conducting exploratory data analysis (EDA) to uncover insights

## Dataset
- **Source:** [Kaggle - Layoffs Dataset (2022)](https://www.kaggle.com/datasets/swaptr/layoffs-2022)
- **Rows:** 2,361
- **Columns:** Company, Location, Industry, Total Laid Off, Percentage Laid Off, Date, Stage, Country, Funds Raised (in millions)

## SQL Workflow

### 1. Data Preparation
To ensure data integrity, the original dataset was duplicated before performing cleaning operations.

```sql
CREATE TABLE layoffs_staging LIKE layoffs;
INSERT INTO layoffs_staging SELECT * FROM layoffs;
```

### 2. Removing Duplicates
Since the dataset lacked a unique identifier, duplicates were identified using the `ROW_NUMBER()` function.

```sql
WITH duplicate_cte AS (
    SELECT *, 
        ROW_NUMBER() OVER(
            PARTITION BY company, location, industry, total_laid_off, percentage_laid_off,
            `date`, stage, country, funds_raised_millions
        ) row_num
    FROM layoffs_staging
)
SELECT * FROM duplicate_cte WHERE row_num > 1;
```

To remove duplicates, a new table was created, and rows with `row_num > 1` were deleted.

```sql
DELETE FROM layoffs_staging2 WHERE row_num > 1;
```

### 3. Data Standardization
- **Trimming extra spaces** in company names:

```sql
UPDATE layoffs_staging2 SET company = TRIM(company);
```

- **Fixing inconsistent spellings**:

```sql
UPDATE layoffs_staging2 SET location = 'Dusseldorf' WHERE location LIKE '%sseldorf%';
UPDATE layoffs_staging2 SET industry = 'Crypto' WHERE industry LIKE 'Crypto%';
```

- **Standardizing country names**:

```sql
UPDATE layoffs_staging2 SET country = TRIM(TRAILING '.' FROM country) WHERE country LIKE 'United State%';
```

### 4. Handling Missing Values
- Replacing blanks with NULL:

```sql
UPDATE layoffs_staging2 SET industry = NULL WHERE industry = '';
```

- Updating NULL values where possible:

```sql
UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2 ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL AND t2.industry IS NOT NULL;
```

- Deleting rows with both `total_laid_off` and `percentage_laid_off` missing:

```sql
DELETE FROM layoffs_staging2 WHERE total_laid_off IS NULL AND percentage_laid_off IS NULL;
```

- Converting `date` column from TEXT to DATE:

```sql
UPDATE layoffs_staging2 SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');
ALTER TABLE layoffs_staging2 MODIFY COLUMN `date` DATE;
```

### 5. Exploratory Data Analysis (EDA)

#### Key Insights:
1. **Largest single layoff:**

```sql
SELECT company, total_laid_off FROM layoffs_staging2 ORDER BY total_laid_off DESC LIMIT 1;
```

2. **Top 5 companies with the highest layoffs:**

```sql
SELECT company, SUM(total_laid_off) AS total_layoffs FROM layoffs_staging2 GROUP BY company ORDER BY total_layoffs DESC LIMIT 5;
```

3. **Companies that shut down completely (100% layoffs):**

```sql
SELECT COUNT(company) FROM layoffs_staging2 WHERE percentage_laid_off = 1;
```

4. **Total layoffs per year:**

```sql
SELECT YEAR(date) AS year, SUM(total_laid_off) FROM layoffs_staging2 GROUP BY year ORDER BY year ASC;
```

5. **Top industries affected:**

```sql
SELECT industry, SUM(total_laid_off) AS layoffs FROM layoffs_staging2 GROUP BY industry ORDER BY layoffs DESC LIMIT 5;
```

## Conclusion
- The dataset required significant cleaning, including duplicate removal, standardization, and handling of missing values.
- The majority of layoffs occurred in **2022 and 2023**, with **Amazon, Google, Meta, and Microsoft** being the most affected companies.
- The **tech and finance** industries had the highest layoffs.

## Repository Structure
```
📂 SQL-Layoffs-Analysis
│── 📂 data                   # Raw dataset
│── 📂 queries                # SQL scripts
│── 📜 README.md              # Project documentation
│── 📜 analysis.sql           # SQL queries used in EDA
│── 📜 cleaning.sql           # Data cleaning SQL scripts
```

## Next Steps
- **Create a visualization dashboard** using Tableau or Power BI
- **Perform deeper analysis** on industry trends and economic impacts

---
This project demonstrates SQL skills in data cleaning, transformation, and analysis. Feel free to explore the dataset and modify queries to gain additional insights!
