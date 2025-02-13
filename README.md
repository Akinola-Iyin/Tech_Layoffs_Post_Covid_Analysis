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

To remove duplicates, a new table was created similar to the layoffs_staging table but with the row number as a new column, the table was populated with the values from the cte above and rows with `row_num > 1` were deleted.

```sql
CREATE TABLE `layoffs_staging2` (
  `company` text,
  `location` text,
  `industry` text,
  `total_laid_off` int DEFAULT NULL,
  `percentage_laid_off` text,
  `date` text,
  `stage` text,
  `country` text,
  `funds_raised_millions` int DEFAULT NULL,
  `row_num` int
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

```sql
INSERT INTO layoffs_staging2
SELECT *, 
row_number() OVER(
PARTITION BY company, location, industry, total_laid_off, percentage_laid_off,
 `date`, stage, country, funds_raised_millions) row_num
FROM layoffs_staging;
```

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
UPDATE layoffs_staging2 SET location = 'Florianopolis' WHERE location LIKE 'Florian%';
UPDATE layoffs_staging2 SET location = 'Malmo' WHERE location LIKE 'Malm%';
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
1. **Maximum and Minimum Layoffs:**

```sql
SELECT max(total_laid_off) max_layoff, min(total_laid_off) min_layoff FROM layoffs_staging2;
```
![image](https://github.com/user-attachments/assets/b865ee44-cc7e-4d75-911e-97721a969adc)

2. **Largest single layoff:**

```sql
SELECT company, total_laid_off FROM layoffs_staging2 ORDER BY total_laid_off DESC LIMIT 1;
```
![image](https://github.com/user-attachments/assets/a0152457-0132-4aea-9908-09b8bb1a9e1a)

3. **Top 5 companies with the highest layoffs:**

```sql
SELECT company, SUM(total_laid_off) AS total_layoffs FROM layoffs_staging2 GROUP BY company ORDER BY total_layoffs DESC LIMIT 5;
```
![image](https://github.com/user-attachments/assets/786adf0c-61e9-447f-af9b-c85f0c7fe9ac)


4. **Top 5 countries with the highest layoffs:**

```sql
SELECT country, SUM(total_laid_off) total_laid_off FROM layoffs_staging2 GROUP BY country ORDER BY total_laid_off DESC LIMIT 5;
```
![image](https://github.com/user-attachments/assets/a6e05fa8-3867-4bfa-93d8-5566b1a846d2)


5. **Top 5 locations with the highest layoffs:**

```sql
SELECT location, SUM(total_laid_off) total_laid_off FROM layoffs_staging2 GROUP BY location ORDER BY total_laid_off DESC LIMIT 5;
```
![image](https://github.com/user-attachments/assets/08df4dcb-8ac5-44e1-ada6-1714c93527bb)


6. **Companies that shut down completely (100% layoffs):**

```sql
SELECT COUNT(company) total_no_of_shutdown_companies FROM layoffs_staging2 WHERE percentage_laid_off = 1;
```
![image](https://github.com/user-attachments/assets/7175d76b-5d15-499e-8514-fd1891070881)


7. **Total layoffs per year:**

```sql
SELECT YEAR(date) AS year, SUM(total_laid_off) total_laid_off FROM layoffs_staging2 GROUP BY year ORDER BY year ASC;
```
![image](https://github.com/user-attachments/assets/1e4cb48e-df48-4a2b-8e06-d31d8e3fa799)


8. **Top 5 industries affected:**

```sql
SELECT industry, SUM(total_laid_off) AS layoffs FROM layoffs_staging2 GROUP BY industry ORDER BY layoffs DESC LIMIT 5;
```
![image](https://github.com/user-attachments/assets/f54b3847-2bc2-468d-86fa-7979e8ed7dc9)


9. **Top 3 companies with the most layoffs in each year:**

```sql
WITH Company_Year AS
(
SELECT company, YEAR(date) `YEARS`, SUM(total_laid_off) total_laid_off
FROM layoffs_staging2
GROUP BY company, `YEARS`
ORDER BY `YEARS`
),
Company_Ranking AS
(
SELECT company, `YEARS`, total_laid_off, dense_rank() OVER(PARTITION BY `YEARS` ORDER BY total_laid_off DESC) AS Ranking
FROM Company_Year
)
SELECT company, `YEARS`, total_laid_off, Ranking
FROM Company_Ranking
WHERE Ranking <=3
AND `YEARS` IS NOT NULL;
```
![image](https://github.com/user-attachments/assets/23fca0f1-0e1a-4027-a022-4e4a8ce46581)


10. **Rolling total of layoffs per month:**

```sql
WITH monthly_layoffs AS
(
SELECT SUBSTRING(date,1,7) months, SUM(total_laid_off) total_laid_off
FROM layoffs_staging2
GROUP BY months
ORDER BY months
)
SELECT months, SUM(total_laid_off) OVER(ORDER BY months ASC) rolling_total
FROM monthly_layoffs
ORDER BY months;
```

## Conclusion
- The dataset required significant cleaning, including duplicate removal, standardization, and handling of missing values.
- The majority of layoffs occurred in **2022 and 2023**, with **Amazon, Google, Meta, and Microsoft** being the most affected companies.
- The **consumer and retail** industries had the highest layoffs.

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

## Contact Information
Feel free to reach out with feedback or questions:
- **Email**: [akinolaiyin@gmail.com]

---
