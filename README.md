## Data Cleaning using SQL

### Dataset
This is home value data for the hot Nashville market available on Kaggle <br>
Link: https://www.kaggle.com/datasets/tmthyjames/nashville-housing-data

### 0. Creating the Database
```sql
CREATE DATABASE NashvilleHousing
USE NashvilleHousing
```

### 1. Standardize date format with Convert

The SaleDate column includes the hour, since it's empty, we don't need it. So I converted to date format.

```sql
SELECT SaleDate, CONVERT(DATE, SaleDate) 
FROM HousingData

UPDATE HousingData
SET SaleDate = CONVERT(DATE, SaleDate)

ALTER TABLE HousingData
ADD SaleDateConverted DATE

UPDATE HousingData
SET SaleDateConverted = CONVERT(DATE, SaleDate)
```
![image](https://user-images.githubusercontent.com/100388639/195446756-f7a343a7-b853-48c4-9581-bdbd855d7ba8.png)

### 2. Populate missing property address data

**2.1. Checking the missing values**

```sql
SELECT *
FROM HousingData
WHERE PropertyAddress IS NULL
```
![image](https://user-images.githubusercontent.com/100388639/195454241-adf3d8ff-b068-4d66-bf85-0518a05e682f.png)

**2.2. Identified the duplicated pattern**

Checking one specific ParcelID (025 07 0 031.00), we can see that we have duplicates, where only one of them has a PropertyAddress. So, we can use this PropertyAddress to populate the missing values.

```sql
SELECT *
FROM HousingData
--WHERE PropertyAddress IS NULL
WHERE ParcelID = '025 07 0 031.00'
ORDER BY PropertyAddress
```
![image](https://user-images.githubusercontent.com/100388639/195454298-b4ac6137-2fe0-45b5-9abd-9fb3a47f6bda.png)

**2.3. Using the self join to check the missing values**

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

**2.4. Populate the missing values**

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

### 3. Breaking out address into individual columns (Address, City, State)

**3.1. Separating the City from the address** 

```sql
SELECT
	SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress)-1) AS Address,
	SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) +1, LEN(PropertyAddress)) AS City
FROM
	HousingData
 ``` 
 ![image](https://user-images.githubusercontent.com/100388639/195455461-1fe99be7-b861-4425-8cda-4a39e31b8b72.png)
 
 **3.2. Creating and updating new columns for PropertyAddress and City**
 
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
**3.3. Separating Address, City and State from Ower address**

```sql
SELECT
PARSENAME(REPLACE(OwnerAddress, ',', '.'), 3) AS Address,
PARSENAME(REPLACE(OwnerAddress, ',', '.'), 2) AS City,
PARSENAME(REPLACE(OwnerAddress, ',', '.'), 1) AS State
FROM
	HousingData
```  
![image](https://user-images.githubusercontent.com/100388639/195455702-a5500938-8349-492c-bf81-6024661cbd4e.png)

**3.4. Creating and updating new columns for OwerAddress, City and State**

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

**4. Change Y and N to Yes and No in "Sold as Vacant" field**

```sql
SELECT
	DISTINCT(SoldAsVacant),
	COUNT(SoldAsVacant) AS Qtde
FROM
	HousingData
GROUP BY SoldAsVacant
ORDER BY 2
```
![image](https://user-images.githubusercontent.com/100388639/195455876-b225b71b-634d-4f6f-8952-811ebd208f6a.png)

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

**4.1. Updating the SoldAsVacant column**

```sql
UPDATE HousingData
SET SoldAsVacant = CASE WHEN SoldAsVacant = 'Y' THEN 'Yes'
						WHEN SoldAsVacant = 'N' THEN 'No'
				        ELSE SoldAsVacant
                   END
```
### 5. Remove duplicates

**5.1. Identify duplicates using CTE**

Creating the CTE to return the row_number that shows the number of times a row with the same data appears in the dataset. When we have a duplicate row we see the number 2 on this column.

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
![image](https://user-images.githubusercontent.com/100388639/195462865-b1a93637-39fe-4487-9267-4c3f382029cd.png)

**5.2. Delete the duplicates**

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

**5.3. Delete unused columns**

Usually we do not delete columns from the database, however for this cleaning practice I decided to delete the columns PropertyAddress, SaleDate, OwnerAddress because we alreary have those information in separate columns.

```sql
ALTER TABLE HousingData
DROP COLUMN PropertyAddress, SaleDate, OwnerAddress
```
### Credits

 Alex Freberg - Data Analyst Portfolio Project | Data Cleaning in SQL | Project 3/4 <br>
 Link: https://www.youtube.com/watch?v=8rO7ztF4NtU&list=PLUaB-1hjhk8H48Pj32z4GZgGWyylqv85f&index=3
