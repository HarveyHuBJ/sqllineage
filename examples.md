# Examples

[TOC]



## Chapter 1 T-SQL Parsing Examples (Basic)
### 1. INSERT / INSERT INTO

~~~sql
INSERT INTO dbo.EmployeeSales  
    SELECT  sp.BusinessEntityID, c.LastName, sp.SalesYTD   
    FROM Sales.SalesPerson AS sp  
    INNER JOIN Person.Person AS c  
        ON sp.BusinessEntityID = c.BusinessEntityID  
    WHERE sp.BusinessEntityID LIKE '2%'  
    ORDER BY sp.BusinessEntityID, c.LastName;  
~~~

![INSERT](/imgs/Insert_into.png)

### 2. SELECT INTO

~~~sql
SELECT c.FirstName, c.LastName, e.JobTitle, a.AddressLine1, a.City,   
    sp.Name AS [State/Province], a.PostalCode  
INTO dbo.EmployeeAddresses  
FROM Person.Person AS c  
    JOIN HumanResources.Employee AS e   
    ON e.BusinessEntityID = c.BusinessEntityID  
    JOIN Person.BusinessEntityAddress AS bea  
    ON e.BusinessEntityID = bea.BusinessEntityID  
    JOIN Person.Address AS a  
    ON bea.AddressID = a.AddressID  
    JOIN Person.StateProvince as sp   
    ON sp.StateProvinceID = a.StateProvinceID;  
GO  
~~~

![INSERT_INTO](/imgs/Select_into.png)

### 3. UPDATE

~~~sql
UPDATE sc1.my_table1
SET    field1 = CASE
                  WHEN b.field1 = 1 THEN b.field1 + dbo.Fun_call(
                                         c.field3, c.field4)
                  WHEN b.field2 = 1 THEN b.field2
                  ELSE b.field3
                END,
       field2 = b.field2,
       field3 = Isnull(b.field1, c.field1_forb)
FROM   sc1.my_table1
       INNER JOIN sc1.my_table2 b
               ON sc1.my_table1.id = b.id
       LEFT JOIN sc1.my_table3 c
              ON b.date = c.datevalue 
~~~

![Update](/imgs/Update.png)

### 4. CTAS (CREATE TABLE AS SELECT)

~~~SQL
CREATE TABLE test  
WITH (HEAP, DISTRIBUTION = ROUND_ROBIN)  
AS  
SELECT  
    CustomerKey AS CustomerKeyNoChange,  
    CustomerKey*1 AS CustomerKeyChangeNullable,  
    CAST(CustomerKey AS DECIMAL(10,2)) AS CustomerKeyChangeDataTypeNullable,  
    ISNULL(CAST(CustomerKey AS DECIMAL(10,2)),0) AS CustomerKeyChangeDataTypeNotNullable,  
    GeographyKey AS GeographyKeyNoChange,  
    ISNULL(GeographyKey,0) AS GeographyKeyChangeNotNullable,  
    CustomerAlternateKey AS CustomerAlternateKeyNoChange,  
    CASE WHEN CustomerAlternateKey = CustomerAlternateKey 
        THEN CustomerAlternateKey END AS CustomerAlternateKeyNullable,  
    CustomerAlternateKey COLLATE Latin1_General_CS_AS_KS_WS AS CustomerAlternateKeyChangeCollation  
FROM [dbo].[DimCustomer2]  
~~~

![CTAS](/imgs/CTAS.PNG)

### 5. CREATE VIEW

~~~sql

CREATE VIEW [HumanResources].[vEmployeeDepartment] 
AS 
SELECT 
    e.[BusinessEntityID] 
    ,p.[Title] 
    ,p.[FirstName] 
    ,p.[MiddleName] 
    ,p.[LastName] 
    ,p.[Suffix] 
    ,e.[JobTitle]
    ,d.[Name] AS [Department] 
    ,d.[GroupName] 
    ,edh.[StartDate] 
FROM [HumanResources].[Employee] e
	INNER JOIN [Person].[Person] p
	ON p.[BusinessEntityID] = e.[BusinessEntityID]
    INNER JOIN [HumanResources].[EmployeeDepartmentHistory] edh 
    ON e.[BusinessEntityID] = edh.[BusinessEntityID] 
    INNER JOIN [HumanResources].[Department] d 
    ON edh.[DepartmentID] = d.[DepartmentID] 
WHERE edh.EndDate IS NULL

~~~

![Create_view](/imgs/Create_view.png)

### 6. MERGE ... USING ... (TBD)



##  Chapter 2 T-SQL Parsing Examples (Advanced)
### 1. Complex Stored Procedure

~~~sql
CREATE PROCEDURE [dbo].[uspGetEmployeeManagers]
    @BusinessEntityID [int]
AS
BEGIN
    SET NOCOUNT ON;

    -- Use recursive query to list out all Employees required for a particular Manager
    WITH [EMP_cte]([BusinessEntityID], [OrganizationNode], [FirstName], [LastName], [JobTitle], [RecursionLevel]) -- CTE name and columns
    AS (
        SELECT e.[BusinessEntityID], e.[OrganizationNode], p.[FirstName], p.[LastName], e.[JobTitle], 0  [RecursionLevel]  -- Get the initial Employee
        FROM [HumanResources].[Employee] e 
			INNER JOIN [Person].[Person] as p
			ON p.[BusinessEntityID] = e.[BusinessEntityID]
        WHERE e.[BusinessEntityID] = @BusinessEntityID
        UNION ALL
        SELECT e.[BusinessEntityID], e.[OrganizationNode], p.[FirstName], p.[LastName], e.[JobTitle], [RecursionLevel] + 1 -- Join recursive member to anchor
        FROM [HumanResources].[Employee] e 
            INNER JOIN [EMP_cte]
            ON e.[OrganizationNode] = [EMP_cte].[OrganizationNode].GetAncestor(1)
            INNER JOIN [Person].[Person] p 
            ON p.[BusinessEntityID] = e.[BusinessEntityID]
    )
    -- Join back to Employee to return the manager name 
    SELECT [EMP_cte].[RecursionLevel], [EMP_cte].[BusinessEntityID], [EMP_cte].[FirstName], [EMP_cte].[LastName], 
        [EMP_cte].[OrganizationNode].ToString() AS [OrganizationNode], p.[FirstName] AS 'ManagerFirstName', p.[LastName] AS 'ManagerLastName'  -- Outer select from the CTE
    INTO tgt.EmployeeManagers
    FROM [EMP_cte] 
        INNER JOIN [HumanResources].[Employee] e 
        ON [EMP_cte].[OrganizationNode].GetAncestor(1) = e.[OrganizationNode]
        INNER JOIN [Person].[Person] p 
        ON p.[BusinessEntityID] = e.[BusinessEntityID]
    ORDER BY [RecursionLevel], [EMP_cte].[OrganizationNode].ToString()
    OPTION (MAXRECURSION 25) 
END;
~~~

![Complex_stored_procedure](/imgs/Complex_stored_procedure.png)

### 2. Table Joins

~~~sql
INSERT INTO tgt.StoreWithAddresses
SELECT 
    s.[BusinessEntityID] 
    ,s.[Name] 
    ,at.[Name] AS [AddressType]
    ,a.[AddressLine1] 
    ,a.[AddressLine2] 
    ,a.[City] 
    ,sp.[Name] AS [StateProvinceName] 
    ,a.[PostalCode] 
    ,cr.[Name] AS [CountryRegionName] 
FROM [Sales].[Store] s
    INNER JOIN [Person].[BusinessEntityAddress] bea 
    ON bea.[BusinessEntityID] = s.[BusinessEntityID] 
    INNER JOIN [Person].[Address] a 
    ON a.[AddressID] = bea.[AddressID]
    INNER JOIN [Person].[StateProvince] sp 
    ON sp.[StateProvinceID] = a.[StateProvinceID]
    INNER JOIN [Person].[CountryRegion] cr 
    ON cr.[CountryRegionCode] = sp.[CountryRegionCode]
    INNER JOIN [Person].[AddressType] at 
    ON at.[AddressTypeID] = bea.[AddressTypeID];
~~~

![Table_joins](/imgs/Table_joins.png)

### 3. CTE or SubQuery

~~~sql
WITH Parts  AS
(
    SELECT b.ProductAssemblyID as AssemblyID , b.ComponentID, b.PerAssemblyQty PerAssemblyQty,
        b.EndDate, 0 AS ComponentLevel
    FROM Production.BillOfMaterials AS b
    WHERE b.ProductAssemblyID = 800
          AND b.EndDate IS NULL
)
SELECT AssemblyID, ComponentID, pr.Name, PerAssemblyQty, EndDate, ComponentLevel
INTO tgt.BillOfMaterials
FROM Parts AS p
    INNER JOIN Production.Product AS pr
    ON p.ComponentID = pr.ProductID
ORDER BY ComponentLevel, AssemblyID, ComponentID;
~~~

![CTE](/imgs/cte.png)

* Another one:

~~~sql
SELECT AssemblyID, ComponentID, pr.Name, PerAssemblyQty, EndDate, ComponentLevel
INTO tgt.BillOfMaterials
FROM
(
    SELECT b.ProductAssemblyID as AssemblyID , b.ComponentID, b.PerAssemblyQty PerAssemblyQty,
        b.EndDate, 0 AS ComponentLevel
    FROM Production.BillOfMaterials AS b
    WHERE b.ProductAssemblyID = 800
          AND b.EndDate IS NULL
) AS p
    INNER JOIN Production.Product AS pr
    ON p.ComponentID = pr.ProductID
ORDER BY ComponentLevel, AssemblyID, ComponentID;
~~~

![subquery](/imgs/subquery.png)



### 4. Union / Union All

~~~sql
   WITH [BOM_cte]([ProductAssemblyID], [ComponentID], [ComponentDesc], [PerAssemblyQty], [StandardCost], [ListPrice], [BOMLevel], [RecursionLevel]) -- CTE name and columns
    AS (
        SELECT b.[ProductAssemblyID], b.[ComponentID], p.[Name], b.[PerAssemblyQty], p.[StandardCost], p.[ListPrice], b.[BOMLevel], 0  [RecursionLevel]-- Get the initial list of components for the bike assembly
        FROM [Production].[BillOfMaterials] b
            INNER JOIN [Production].[Product] p 
            ON b.[ComponentID] = p.[ProductID] 
        WHERE b.[ProductAssemblyID] = @StartProductID 
            AND @CheckDate >= b.[StartDate] 
            AND @CheckDate <= ISNULL(b.[EndDate], @CheckDate)
        UNION ALL
        SELECT b.[ProductAssemblyID], b.[ComponentID], p.[Name], b.[PerAssemblyQty], p.[StandardCost], p.[ListPrice], b.[BOMLevel], [RecursionLevel] + 1 -- Join recursive member to anchor
        FROM [BOM_cte] cte
            INNER JOIN [Production].[BillOfMaterials] b 
            ON b.[ProductAssemblyID] = cte.[ComponentID]
            INNER JOIN [Production].[Product] p 
            ON b.[ComponentID] = p.[ProductID] 
        WHERE @CheckDate >= b.[StartDate] 
            AND @CheckDate <= ISNULL(b.[EndDate], @CheckDate)
        )
    -- Outer select from the CTE
    SELECT b.[ProductAssemblyID], b.[ComponentID], b.[ComponentDesc], SUM(b.[PerAssemblyQty]) AS [TotalQuantity] , b.[StandardCost], b.[ListPrice], b.[BOMLevel], b.[RecursionLevel]
    INTO tgt.BOM
    FROM [BOM_cte] b
    GROUP BY b.[ComponentID], b.[ComponentDesc], b.[ProductAssemblyID], b.[BOMLevel], b.[RecursionLevel], b.[StandardCost], b.[ListPrice]
    ORDER BY b.[BOMLevel], b.[ProductAssemblyID], b.[ComponentID]
~~~



![Union_all](/imgs/Union_all.png)

### 5. Scalar Subquery

~~~sql
SELECT Ord.SalesOrderID, Ord.OrderDate,
    (SELECT MAX(OrdDet.UnitPrice)
     FROM Sales.SalesOrderDetail AS OrdDet
     WHERE Ord.SalesOrderID = OrdDet.SalesOrderID) AS MaxUnitPrice
INTO tgt.SalesOrderHeader
FROM Sales.SalesOrderHeader AS Ord;
~~~

![scalar_subquery](/imgs/scalar_subquery.png)

* Another one:

~~~sql
SELECT [Name], ListPrice,
(SELECT AVG(ListPrice) FROM Production.Product) AS Average,
    ListPrice - (SELECT AVG(ListPrice) FROM Production.Product)
    AS Difference
INTO tgt.Product
FROM Production.Product
WHERE ProductSubcategoryID = 1;
~~~

![scalar_subquery_exp](/imgs/scalar_subquery_exp.png)



### 6. PIVOT

~~~sql
SELECT 
    pvt.[SalesPersonID]
    ,pvt.[FullName]
    ,pvt.[JobTitle]
    ,pvt.[SalesTerritory]
    ,pvt.[2002]
    ,pvt.[2003]
    ,pvt.[2004] 
INTO tgt.SalesPersonSalesByFiscalYears
FROM (SELECT 
        soh.[SalesPersonID]
        ,p.[FirstName] + ' ' + COALESCE(p.[MiddleName], '') + ' ' + p.[LastName] AS [FullName]
        ,e.[JobTitle]
        ,st.[Name] AS [SalesTerritory]
        ,soh.[SubTotal]
        ,YEAR(DATEADD(m, 6, soh.[OrderDate])) AS [FiscalYear] 
    FROM [Sales].[SalesPerson] sp 
        INNER JOIN [Sales].[SalesOrderHeader] soh 
        ON sp.[BusinessEntityID] = soh.[SalesPersonID]
        INNER JOIN [Sales].[SalesTerritory] st 
        ON sp.[TerritoryID] = st.[TerritoryID] 
        INNER JOIN [HumanResources].[Employee] e 
        ON soh.[SalesPersonID] = e.[BusinessEntityID] 
		INNER JOIN [Person].[Person] p
		ON p.[BusinessEntityID] = sp.[BusinessEntityID]
	 ) AS soh 
PIVOT 
(
    SUM([SubTotal]) 
    FOR [FiscalYear] 
    IN ([2002], [2003], [2004])
) AS pvt;
~~~

![Pivot](/imgs/Pivot.png)

### 7. UNPIVOT

~~~sql
CREATE TABLE dbo.pvt (VendorID INT, Emp1 INT, Emp2 INT,  
    Emp3 INT, Emp4 INT, Emp5 INT);  
GO  
  
-- Unpivot the table.  
CREATE VIEW dbo.vUnpiovt as
SELECT VendorID, Employee, Orders  
FROM   
   (SELECT VendorID, Emp1, Emp2, Emp3, Emp4, Emp5  
   FROM dbo.pvt) p  
UNPIVOT  
   (Orders FOR Employee IN   
      (Emp1, Emp2, Emp3, Emp4, Emp5)  
)AS unpvt;  

~~~

![Unpivot](/imgs/Unpivot.png)



### 8. String_Split

~~~sql
SELECT a.ProductId, a.Name, value  as Tag
INTO tgt.ProductWithTag
FROM dbo.Product a  
    CROSS APPLY STRING_SPLIT(Tags, ',') ;  
~~~



![String_split](/imgs/String_split.png)

### 9. Cross Apply

~~~sql
SELECT TOP 111    A.SalesOrderID, A.OrderDate, T.ProductID
INTO  r1   
FROM   A
  CROSS APPLY (
     SELECT B.SalesOrderDetailID, B.ProductID
     FROM   B
     WHERE A.SalesOrderID = B.SalesOrderID  
  ) T
~~~

![Cross_apply](/imgs/Cross_apply.png)



##  Chapter 3 Implicit Inferring  Columns

Assuming we've had metadata of a database, which contains all table/column metadata information we will use in our scripts. That is the prerequisites for column inferring when we come across scripts like 'select *' , 'insert into {table}' without specified columns or select multiply columns from two or more tables without table or table alias decorated.    

假设我们已经具备了某个数据库的元数据， 包括了所有的将会在脚本中用到的表、列元数据信息。这是我们做隐式列名推断的前提条件。 这样当我们遇到像 "select * ", "insert into {table}"但不指定列，以及从多个表SELECT多个列但不加表名限定的时候， 能够推测出准确的血缘关系。

### 1. SELECT *

~~~sql
insert into tgt1
select * from Sales.CurrencyRate

go
insert into tgt2
select * from Sales.Currency
~~~

![INFER_SELECT_STAR](/imgs/INFER_SELECT_STAR)

### 2. INSERT INTO {TABLE} 

~~~SQL
INSERT INTO Sales.Currency
SELECT cur_code, cur_name, date 
from dbo.src_currency
~~~

![INFER_INSERT_UNSPECIFICED](/imgs/INFER_INSERT_UNSPECIFICED.PNG)

### 3. SELECT Undecorated Columns FROM Multiple Tables

 ~~~sql
 SELECT Name, ToCurrencyCode, AverageRate, EndOfDayRate
 into tgt
 from sales.CurrencyRate a
   INNER JOIN sales.Currency b on a.ToCurrencyCode = b.CurrencyCode
   WHERE A.FromCurrencyCode='USD' AND a.ModifiedDate = '2011-05-31 00:00:00.000'
 ~~~

![INFER_MULTIPLE_TABLES](/imgs/INFER_MULTIPLE_TABLES.png)

##  Chapter 4 PowerQuery Parsing Example(TODO)
