### Nashville Housing - Data Cleaning and Exploring in SQL

Context
This is home value data for the hot Nashville market.

Content
There are 56,000+ rows altogether. However, I'm missing home detail data for about half.

```sql
CREATE DATABASE NashvilleHousing
USE NashvilleHousing
```

-- Cleaning data in SQL Queries

-- Checking the table content
```sql
EXEC sp_columns HousingData
```

SELECT * 
FROM HousingData

--1. Standardize date format
```sql
SELECT SaleDate, CONVERT(DATE, SaleDate) 
FROM HousingData
```
```sql
UPDATE HousingData
SET SaleDate = CONVERT(DATE, SaleDate)

ALTER TABLE HousingData
ADD SaleDateConverted DATE

UPDATE HousingData
SET SaleDateConverted = CONVERT(DATE, SaleDate)
```
-- Populate missing property address data 

```sql
SELECT *
FROM HousingData
--WHERE PropertyAddress IS NULL
WHERE ParcelID = '025 07 0 031.00'
ORDER BY PropertyAddress
```

-- Using the self join to check the missing values
```sql
SELECT 
	A.ParcelID,
	A.PropertyAddress,
	B.ParcelID,
	B.PropertyAddress,
	ISNULL(A.PropertyAddress, B.PropertyAddress)
FROM 
	HousingData A
JOIN HousingData B
	ON A.ParcelID = B.ParcelID
	AND A.[UniqueID ] <> B.[UniqueID ]
WHERE A.PropertyAddress IS NULL
```

--Populate de missing values
```sql
UPDATE A
SET PropertyAddress = ISNULL(A.PropertyAddress, B.PropertyAddress)
FROM 
	HousingData A
JOIN HousingData B
	ON A.ParcelID = B.ParcelID
	AND A.[UniqueID ] <> B.[UniqueID ]
WHERE A.PropertyAddress IS NULL
```

--Breaking out address into individual columns (Address, City, State)
```sql
SELECT
	SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress)-1) AS Address,
	SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) +1, LEN(PropertyAddress)) AS City
FROM
	HousingData
 ``` 
```sql
ALTER TABLE HousingData
ADD PropertySplitAddress VARCHAR(255)

UPDATE HousingData
SET PropertySplitAddress = SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress)-1)

ALTER TABLE HousingData
ADD PropertySplitCity VARCHAR(255)

UPDATE HousingData
SET PropertySplitCity = SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) +1, LEN(PropertyAddress))
```
```sql
SELECT
PARSENAME(REPLACE(OwnerAddress, ',', '.'), 3) AS Address,
PARSENAME(REPLACE(OwnerAddress, ',', '.'), 2) AS City,
PARSENAME(REPLACE(OwnerAddress, ',', '.'), 1) AS State
FROM
	HousingData
```  
```sql
ALTER TABLE HousingData
ADD OwnerSplitAddress VARCHAR(255)

UPDATE HousingData
SET OwnerSplitAddress = PARSENAME(REPLACE(OwnerAddress, ',', '.'), 3)

ALTER TABLE HousingData
ADD OwnerSplitCity VARCHAR(255)

UPDATE HousingData
SET OwnerSplitCity = PARSENAME(REPLACE(OwnerAddress, ',', '.'), 2)

ALTER TABLE HousingData
ADD OwnerSplitState VARCHAR(255)

UPDATE HousingData
SET OwnerSplitState = PARSENAME(REPLACE(OwnerAddress, ',', '.'), 1)
```

--Change Y and N to Yes and No in "Sold as Vacant" field
```sql
SELECT
	DISTINCT(SoldAsVacant),
	COUNT(SoldAsVacant) AS Qtde
FROM
	HousingData
GROUP BY SoldAsVacant
ORDER BY 2
```
```sql
SELECT
	SoldAsVacant,
CASE WHEN SoldAsVacant = 'Y' THEN 'Yes'
	 WHEN SoldAsVacant = 'N' THEN 'No'
	 ELSE SoldAsVacant
END
FROM
	HousingData
```  
```sql
UPDATE HousingData
SET SoldAsVacant = CASE WHEN SoldAsVacant = 'Y' THEN 'Yes'
						WHEN SoldAsVacant = 'N' THEN 'No'
				        ELSE SoldAsVacant
                   END
```
--Remove duplicates
```sql
WITH RowNumCTE AS (
SELECT
	*,
	ROW_NUMBER() OVER(
	PARTITION BY ParcelID,
				 PropertyAddress,
				 SalePrice,
				 SaleDate,
				 LegalReference
				 ORDER BY
					UniqueID
					) row_num
FROM
	HousingData
--ORDER BY ParcelID
)
SELECT *
FROM RowNumCTE
WHERE row_num > 1
ORDER BY PropertyAddress
```
--Delete the duplicates
```sql
WITH RowNumCTE AS (
SELECT
	*,
	ROW_NUMBER() OVER(
	PARTITION BY ParcelID,
				 PropertyAddress,
				 SalePrice,
				 SaleDate,
				 LegalReference
				 ORDER BY
					UniqueID
					) row_num
FROM
	HousingData
--ORDER BY ParcelID
)
DELETE
FROM RowNumCTE
WHERE row_num > 1
--ORDER BY PropertyAddress
```
--Delete unused columns
```sql
ALTER TABLE HousingData
DROP COLUMN PropertyAddress, SaleDate, OwnerAddress
```
### Exploring the dataset

```sql
SELECT * FROM HousingData WHERE OwnerSplitAddress IS NULL
--OWNERNAME = 31158
--ACREAGE, TAXDISTRICT, LANDVALUE, BUILDINGVALUE, TOTAL VALUE, OWNERSPLITADDRESS, CITY, STATE = 30404
--YEARBUILT = 32255
--BEDROONS = 32261
--FULLBATH = 32143
--HALFBATH = 32274
```

--How many land use we have
```sql
SELECT
	DISTINCT(LandUse)
FROM
	HousingData
 ```

--Total and percentage of properties by land use
```sql
SELECT
	DISTINCT(LandUse),
	COUNT(LandUse) AS Total,
	CAST((COUNT(LandUse)/Tmp.Total)*100 AS numeric(10,2)) AS Percentage
FROM
	HousingData,
	(SELECT CAST(COUNT(LandUse) AS numeric(10,2)) AS TOTAL FROM HousingData) Tmp
GROUP BY LandUse, Tmp.Total
ORDER BY Percentage DESC
```
--Highest Sale price by land use
```sql
SELECT
	LandUse,
	MAX(SalePrice) AS MaxPrice
FROM
	HousingData
GROUP BY LandUse
ORDER BY MaxPrice DESC
```
--Porperties by cities
```sql
SELECT
	DISTINCT(PropertySplitCity),
	COUNT(PropertySplitCity) AS Total
FROM
	HousingData
GROUP BY PropertySplitCity
ORDER BY Total DESC
```
--How many were sold as vacant
