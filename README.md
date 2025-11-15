# PortfolioProjects
Data Cleaning Portfolio Project Queries,sql
-- Data Cleaning

SELECT *
FROM layoffs;

-- 1. Remove Duplicates
-- 2. Standardize the Data
-- 3. Null Values or blank values
-- 4. Remove Any Columns or Rows


-- Creating a Staging Table identical to the original
-- This creates empty tables layoffs_staging with the same structure as the original layoffs table
-- A staging table allows me to safely clean data without touching the raw source 
CREATE TABLE layoffs_staging
LIKE layoffs;


SELECT *
FROM layoffs_staging;

-- This inserts all raw data from layoffs into the new staging table
INSERT layoffs_staging
SELECT *
FROM layoffs;

-- Checking duplicates using ROW_NUMBER()
-- This assigns row numbers to potentially duplicated rows
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`) AS row_num
FROM layoffs_staging;

-- Better duplicate detection using more columns
-- Rows with row_num > 1 are duplicates
WITH duplicate_cte AS
(
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, location, 
industry, total_laid_off, percentage_laid_off, 'date', stage, country, funds_raised_millions) AS row_num
FROM layoffs_staging
)
SELECT *
FROM duplicate_cte
WHERE row_num > 1;

SELECT *
FROM layoffs_staging
WHERE company = 'Casper';

-- Attempt to delete duplicates inside a CTE
-- MySQL does not allow DELETE from CTEs, which is why the workaround was needed later
WITH duplicate_cte AS
(
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, location, 
industry, total_laid_off, percentage_laid_off, 'date', stage, country, funds_raised_millions) AS row_num
FROM layoffs_staging
)
DELETE
FROM duplicate_cte
WHERE row_num > 1;



-- Creating a Second Staging Table (with row_num stored physically)
-- Because MySQL cannot delete from a CTE directly, a second staging table is created
-- Create layoffs_staging2, This table includes a row_num column.
CREATE TABLE `layoffs_staging2` (
`company` text,
`location` text,
`industry` text,
`total_laid_off` int DEFAULT NULL,
`percentage_laid_off` text,
`date` text,
`stage` text,
`country` text,
`fund_raised_millions` int DEFAULT NULL,
`row_num` INT
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

SELECT *
FROM layoffs_staging2
WHERE row_num > 1;

-- Insert data with ROW_NUMBER()
-- Now each row has a row number permanently stored
INSERT INTO layoffs_staging2
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, location,
industry, total_laid_off, percentage_laid_off, 'date', stage,
country, funds_raised_millions) AS row_num
FROM layoffs_staging;


-- Delete actual duplicates
-- This removes all duplicated rows, keeping only the first occurrence
DELETE 
FROM layoffs_staging2
WHERE row_num > 1;

SELECT *
FROM layoffs_staging2;



-- Standardizing / Cleaning data

-- Removing the extra whitespace
SELECT company,TRIM(company)
FROM layoffs_staging2;

UPDATE layoffs_staging2
SET company = TRIM(company);

-- Fixing inconsistent industry names, checking variation
SELECT DISTINCT industry
FROM layoffs_staging2
ORDER BY 1;

SELECT *
FROM layoffs_staging2
WHERE industry LIKE 'Crypto%';

SELECT DISTINCT location
FROM layoffs_staging2
ORDER BY 1;

SELECT *
FROM layoffs_staging2
ORDER BY 1;

SELECT DISTINCT country
FROM layoffs_staging2
ORDER BY 1;

SELECT *
FROM layoffs_staging2
WHERE country LIKE 'United States%'
ORDER BY 1;

SELECT DISTINCT country, TRIM(country)
FROM layoffs_staging2
ORDER BY 1;

SELECT DISTINCT country, TRIM(TRAILING ' , ' FROM country)
FROM layoffs_staging2
ORDER BY 1;

-- Cleaning country values (e.g., removing trailing commas)
UPDATE layoffs_staging2
SET country = TRIM(TRAILING ' , ' FROM country)
WHERE country LIKE 'United States%';

SELECT `date`
FROM layoffs_staging2;

-- Converting date column from string to DATE format
SELECT `date`,
STR_TO_DATE(`date`, '%m/%d/%Y')
FROM layoffs_staging2;

-- Converting string-MySQL date
UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y'); 

-- Convert the column's type
ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;

SELECT *
FROM layoffs_staging2;

-- Handling NULL or Missing Industry Values

SELECT *
FROM layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;

-- Replace blank industry values with NULL
UPDATE layoffs_staging2
SET industry = NULL
WHERE industry = '';


SELECT DISTINCT industry
FROM layoffs_staging2;

-- Identifying missing industries
SELECT *
FROM layoffs_staging2
WHERE industry IS NULL
OR industry = '';


SELECT *
FROM layoffs_staging2
WHERE company = 'Airbnb';

SELECT *
FROM layoffs_staging2
WHERE company LIKE 'Bally%';


-- Fill missing industries using rows with same company and location
-- This is a self-join technique to fill missing values using known data
SELECT *
FROM layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company
    AND t1.location = t2.location
WHERE (t1.industry IS NULL OR t1.industry = '')
AND t2.industry IS NOT NULL;

SELECT *
FROM layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company
    AND t1.location = t2.location
WHERE t1.industry IS NULL
AND t2.industry IS NOT NULL;

SELECT t1.industry, t2.industry
FROM layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company
WHERE (t1.industry IS NULL OR t1.industry = '')
AND t2.industry IS NOT NULL;

UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL
AND t2.industry IS NOT NULL;


SELECT *
FROM layoffs_staging2;


SELECT *
FROM layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;

-- Removing Meaningless Rows
-- if both total_laid_off and percentage_laid_off are NULL = Delete
-- These rows contain no useful layoff data
DELETE
FROM layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;


SELECT *
FROM layoffs_staging2;

-- Remove the helper column
-- This column is no longer needed now that duplicates are removed
ALTER TABLE layoffs_staging2
DROP COLUMN row_num;


-- Summary of what the Project Does
-- Creates clean working tables (staging)
-- Identifies and removes duplicate rows using window functions
-- Standardizes inconsistent text values (company, industry, country)
-- Converts date strings into MySQL DATE format
-- Handle NULL and blank values properly
-- Backfills missing industry data using a self-join
-- Removes rows with no useful layoff information
-- Produces a fully cleaned dataset ready for analysis
















