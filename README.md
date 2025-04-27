```sql
-- 1️⃣ BACKUP ORIGINAL DATA
CREATE TABLE layoff_dup LIKE layoffs;

INSERT INTO layoff_dup
SELECT * FROM layoffs;

SELECT * FROM layoff_dup;

-- 2️⃣ FIND DUPLICATES
WITH duplicates AS (
  SELECT *,
         ROW_NUMBER() OVER(PARTITION BY company, location, total_laid_off, 
                           percentage_laid_off, `date`, stage, country, 
                           funds_raised_millions) AS row_num
  FROM layoff_dup
)
SELECT * FROM duplicates
WHERE row_num > 1;

-- 3️⃣ CREATE TABLE WITH ROW_NUM COLUMN
CREATE TABLE layoff_dup1 (
  company TEXT,
  location TEXT,
  industry TEXT,
  total_laid_off INT DEFAULT NULL,
  percentage_laid_off TEXT,
  `date` TEXT,
  stage TEXT,
  country TEXT,
  funds_raised_millions INT DEFAULT NULL,
  row_num INT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 4️⃣ INSERT DATA INTO NEW TABLE WITH ROW_NUM
INSERT INTO layoff_dup1
SELECT *, ROW_NUMBER() OVER(PARTITION BY company, location, total_laid_off, 
                            percentage_laid_off, `date`, stage, country, 
                            funds_raised_millions) AS row_num
FROM layoff_dup;

-- 5️⃣ REMOVE DUPLICATE RECORDS
SET SQL_SAFE_UPDATES = 0;

DELETE FROM layoff_dup1
WHERE row_num > 1;

-- 6️⃣ STANDARDIZE DATA

-- 6.1 Trim extra spaces in 'company' names
UPDATE layoff_dup1
SET company = TRIM(company);

-- 6.2 Standardize 'industry' names
UPDATE layoff_dup1
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';

-- 6.3 Standardize 'country' names
UPDATE layoff_dup1
SET country = 'USA'
WHERE country LIKE 'United States%';

-- 6.4 Convert 'date' column into proper DATE format
UPDATE layoff_dup1
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');

ALTER TABLE layoff_dup1
MODIFY COLUMN `date` DATE;

-- 7️⃣ HANDLE NULL VALUES

-- 7.1 Find rows with NULL industry
SELECT *
FROM layoff_dup1
WHERE industry IS NULL OR industry = '';

-- 7.2 Update blanks ('') to NULL
UPDATE layoff_dup1
SET industry = NULL
WHERE industry = '';

-- 7.3 Fill NULL industry values using self-join
UPDATE layoff_dup1 l1
JOIN layoff_dup1 l2 ON l1.company = l2.company
SET l1.industry = l2.industry
WHERE l1.industry IS NULL AND l2.industry IS NOT NULL;

-- 7.4 Delete records where both 'total_laid_off' and 'percentage_laid_off' are NULL
DELETE FROM layoff_dup1
WHERE total_laid_off IS NULL AND percentage_laid_off IS NULL;

-- 8️⃣ FINAL CLEANUP

-- Drop 'row_num' column
ALTER TABLE layoff_dup1
DROP COLUMN row_num;

-- View final cleaned data
SELECT * FROM layoff_dup1;

```
