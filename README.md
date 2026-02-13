# Merchant-Interchange-Finance– Data Warehouse Project

## Project Overview

This project builds a dimensional data warehouse model using MS SQL Server to analyze interchange revenue performance across merchants, cards, and acquirers.

The solution includes:

- Data validation and cleaning
- Duplicate detection and handling
- Star schema design
- Dimension and fact table creation
- Business KPI development
- Merchant ranking and trend analysis

Database: MS SQL Server  
Source Table: [Sample Interchange Dataset]

---

# Data Validation & Cleaning

## View Raw Data

```sql
SELECT * FROM [Sample Interchange Dataset];
```

---

## Null Check (Overall)

```sql
SELECT 
    CASE 
        WHEN EXISTS (
            SELECT 1
            FROM [Sample Interchange Dataset]
            WHERE 
                InterChangeID IS NULL OR
                ReportingDate IS NULL OR
                CardNumber IS NULL OR
                AcquirerNetworkGroup IS NULL OR
                Merchant IS NULL OR
                UsageType IS NULL OR
                ProductType IS NULL OR
                SettlementAmount IS NULL OR
                InterchangeRevenue IS NULL OR
                TransactionCount IS NULL
        )
        THEN 'NULLS Found'
        ELSE 'NO NULLS Found'
    END AS NullCheck;
```

---

## Column-wise Null Check

```sql
SELECT 
    SUM(CASE WHEN InterChangeID IS NULL THEN 1 ELSE 0 END) AS Null_InterChangeID,
    SUM(CASE WHEN ReportingDate IS NULL THEN 1 ELSE 0 END) AS Null_ReportingDate,
    SUM(CASE WHEN CardNumber IS NULL THEN 1 ELSE 0 END) AS Null_CardNumber,
    SUM(CASE WHEN AcquirerNetworkGroup IS NULL THEN 1 ELSE 0 END) AS Null_AcquirerNetworkGroup,
    SUM(CASE WHEN Merchant IS NULL THEN 1 ELSE 0 END) AS Null_Merchant,
    SUM(CASE WHEN UsageType IS NULL THEN 1 ELSE 0 END) AS Null_UsageType,
    SUM(CASE WHEN ProductType IS NULL THEN 1 ELSE 0 END) AS Null_ProductType,
    SUM(CASE WHEN SettlementAmount IS NULL THEN 1 ELSE 0 END) AS Null_SettlementAmount,
    SUM(CASE WHEN InterchangeRevenue IS NULL THEN 1 ELSE 0 END) AS Null_InterchangeRevenue,
    SUM(CASE WHEN TransactionCount IS NULL THEN 1 ELSE 0 END) AS Null_TransactionCount
FROM [Sample Interchange Dataset];
```

---

## Data Quality Checks

### Duplicate InterChangeID

```sql
SELECT InterChangeID, COUNT(*) AS Cnt
FROM [Sample Interchange Dataset]
GROUP BY InterChangeID
HAVING COUNT(*) > 1;
```

### Negative Amounts

```sql
SELECT *
FROM [Sample Interchange Dataset]
WHERE SettlementAmount < 0 
   OR InterchangeRevenue < 0;
```

### Missing Merchant Names

```sql
SELECT *
FROM [Sample Interchange Dataset]
WHERE Merchant IS NULL 
   OR LTRIM(RTRIM(Merchant)) = '';
```

### Invalid Transaction Count

```sql
SELECT *
FROM [Sample Interchange Dataset]
WHERE TransactionCount <= 0;
```

---

## Duplicate Detection Using ROW_NUMBER

```sql
WITH Dups AS
(
    SELECT *,
           ROW_NUMBER() OVER (
                PARTITION BY 
                    InterChangeID,
                    ReportingDate,
                    CardNumber,
                    AcquirerNetworkGroup,
                    Merchant,
                    UsageType,
                    ProductType,
                    SettlementAmount,
                    InterchangeRevenue,
                    TransactionCount
                ORDER BY InterChangeID
           ) AS rn
    FROM [Sample Interchange Dataset]
)
SELECT *
FROM Dups
WHERE rn > 1;
```

---

## Trim White Spaces

```sql
UPDATE [Sample Interchange Dataset]
SET 
    InterChangeID = LTRIM(RTRIM(InterChangeID)),
    CardNumber = LTRIM(RTRIM(CardNumber)),
    AcquirerNetworkGroup = LTRIM(RTRIM(AcquirerNetworkGroup)),
    Merchant = LTRIM(RTRIM(Merchant)),
    UsageType = LTRIM(RTRIM(UsageType)),
    ProductType = LTRIM(RTRIM(ProductType));
```

---

# Dimensional Modelling – Star Schema

## Dimension Tables

### Dim_Date

```sql
CREATE TABLE Dim_Date (
    DateKey INT PRIMARY KEY,
    FullDate DATE,
    Year INT,
    Month INT,
    MonthName VARCHAR(20),
    Quarter INT
);
```

### Dim_Merchant

```sql
CREATE TABLE Dim_Merchant (
    MerchantKey INT IDENTITY(1,1) PRIMARY KEY,
    MerchantName VARCHAR(255)
);
```

### Dim_Card

```sql
CREATE TABLE Dim_Card (
    CardKey INT IDENTITY(1,1) PRIMARY KEY,
    CardNumber VARCHAR(50)
);
```

### Dim_Acquirer

```sql
CREATE TABLE Dim_Acquirer (
    AcquirerKey INT IDENTITY(1,1) PRIMARY KEY,
    AcquirerNetworkGroup VARCHAR(100)
);
```

### Dim_Category

```sql
CREATE TABLE Dim_Category (
    CategoryKey INT IDENTITY(1,1) PRIMARY KEY,
    UsageType VARCHAR(100),
    ProductType VARCHAR(100),
    CombinedCategory VARCHAR(200)
);
```

---

## Fact Table

```sql
CREATE TABLE Fact_Interchange (
    InterChangeID INT PRIMARY KEY,
    DateKey INT,
    MerchantKey INT,
    CardKey INT,
    AcquirerKey INT,
    CategoryKey INT,
    SettlementAmount DECIMAL(18,2),
    InterchangeRevenue DECIMAL(18,2),
    TransactionCount INT,

    FOREIGN KEY (DateKey) REFERENCES Dim_Date(DateKey),
    FOREIGN KEY (MerchantKey) REFERENCES Dim_Merchant(MerchantKey),
    FOREIGN KEY (CardKey) REFERENCES Dim_Card(CardKey),
    FOREIGN KEY (AcquirerKey) REFERENCES Dim_Acquirer(AcquirerKey),
    FOREIGN KEY (CategoryKey) REFERENCES Dim_Category(CategoryKey)
);
```

---

# Business Analysis Queries

## Top 10 Merchants by Interchange Volume

```sql
SELECT  
    M.MerchantName,
    SUM(F.SettlementAmount) AS InterchangeVolume
FROM Fact_Interchange F
JOIN Dim_Merchant M 
    ON F.MerchantKey = M.MerchantKey
GROUP BY M.MerchantName
ORDER BY InterchangeVolume DESC
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;
```

---

## Interchange Rate per Merchant

```sql
SELECT  
    M.MerchantName,
    SUM(F.InterchangeRevenue) AS TotalInterchangeRevenue,
    SUM(F.SettlementAmount) AS TotalSettlementAmount,
    FORMAT(
        (SUM(F.InterchangeRevenue) * 1.0 / SUM(F.SettlementAmount)),
        'P2'
    ) AS InterchangeRate_Percentage
FROM Fact_Interchange F
JOIN Dim_Merchant M 
    ON F.MerchantKey = M.MerchantKey
GROUP BY M.MerchantName
ORDER BY 
    (SUM(F.InterchangeRevenue) * 1.0 / SUM(F.SettlementAmount)) DESC;
```

---

## Merchant Ranking by Volume

```sql
SELECT  
    dm.MerchantName,
    SUM(fi.SettlementAmount) AS TotalInterchangeVolume,
    RANK() OVER (ORDER BY SUM(fi.SettlementAmount) DESC) AS MerchantRank
FROM Fact_Interchange fi
JOIN Dim_Merchant dm 
      ON fi.MerchantKey = dm.MerchantKey
GROUP BY dm.MerchantName
ORDER BY TotalInterchangeVolume DESC;
```

---

## Monthly Ranking Comparison (Current vs Previous Month)

```sql
WITH MonthlyData AS 
(
    SELECT  
        dm.MerchantKey,
        dm.MerchantName,
        d.Year,
        d.Month,
        SUM(fi.InterchangeRevenue) AS TotalRevenue
    FROM Fact_Interchange fi
    JOIN Dim_Merchant dm ON fi.MerchantKey = dm.MerchantKey
    JOIN Dim_Date d ON fi.DateKey = d.DateKey
    GROUP BY dm.MerchantKey, dm.MerchantName, d.Year, d.Month
),
RankedData AS
(
    SELECT 
        *,
        RANK() OVER (
            PARTITION BY Year, Month 
            ORDER BY TotalRevenue DESC
        ) AS RevenueRank
    FROM MonthlyData
)
SELECT *
FROM RankedData;
```

---

# SQL Concepts Used

- Data Validation
- Window Functions (ROW_NUMBER, RANK)
- CTE (Common Table Expressions)
- Foreign Keys & Referential Integrity
- Star Schema Design
- Revenue Ratio Calculations
- Ranking & Trend Comparison

---

# Conclusion

This project demonstrates:

- End-to-end data warehouse implementation
- Strong SQL proficiency
- Dimensional modeling expertise
- Business KPI development
- Merchant performance analytics
- Advanced ranking and month-over-month analysis
