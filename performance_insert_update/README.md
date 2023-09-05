# Performance Testing SSIS vs. SQL Server    

The Purpose of this Repo is to test different methods of inserting or updating new data into a table



<!-- vscode-markdown-toc -->
1. [Datasets](#Datasets)
2. [Testing Procedure](#TestingProcedure)
3. [Use Case 1 - Data will only be added](#UseCase1-Datawillonlybeadded)\
	3.1. [Lookup in SSIS](#LookupinSSIS)\
	3.2. [Insert in SQL Server](#InsertinSQLServer)\
	3.3. [Truncate that B!&%?](#TruncatethatB)
4. [UseCase 2 - Data will be updated and added](#UseCase2-Datawillbeupdatedandadded)\
	4.1. [Update/Insert in SQL Server](#Update/InsertinSQLServer)\
	4.2. [Update/Insert in SSIS](#Update/InsertinSSIS)\
	4.3. [Updated only Changed in SQL Server](#updateSome)\
	4.4. [Truncate that B!&%?](#TruncatethatB-1)
5. [Results](#Results)
6. [Conclusion](#Conclusion)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->




##  1. <a name='Datasets'></a>Datasets

All datasets consist of the following columns:

- customer_id
- customer_name
- country
- email
- website


customers_actual --> 500k rows. This represents the data we already have in our customers table\
customers_added --> 1mio rows. The first 500k are the same as in customers_actual, the last 500k is new data\
customers_added_and_changed --> 1mio rows. First 500k have same customer_id as in customers_actual, but other values for the rest. Last 500k  are new entries


##  2. <a name='TestingProcedure'></a>Testing Procedure

All Procedures/Flows will be run under the same conditions.\
All tables are truncated/preloaded to mimic a real life environment.


##  3. <a name='UseCase1-Datawillonlybeadded'></a>Use Case 1 - Data will only be added

In this case we receive a flatfile where new data gets added to the file without the old data being deleted or changed.


###  3.1. <a name='LookupinSSIS'></a>Lookup in SSIS

customers_added Dataset gets compared with the existing costumer table, customer_id's that are not matching are inserted to the existing data

![Lookup in SSIS](screenshots\lookup_in_ssis.png)



###  3.2. <a name='InsertinSQLServer'></a>Insert in SQL Server

customers_added gets loaded into temp table.\
Call stored procedure that checks if the customer_id already exists, if not, insert the row.

![Insert in SQL Server](screenshots\insert_in_sql_server.png)


**Stored Procedure**
```sql
INSERT INTO [testing].[dbo].[customers] (customer_id, customer_name, country,email,website)
	SELECT tmp.customer_id, tmp.customer_name, tmp.country, tmp.email, tmp.website
	FROM [testing].[dbo].[customers] tmp
	WHERE NOT EXISTS (
		SELECT 1
		FROM [testing].[dbo].[customers] c
		WHERE c.customer_id = tmp.customer_id
		);
```


###  3.3. <a name='TruncatethatB'></a>Truncate that B!&%?

Customer table gets truncated and the full csv gets uploaded again

![TTB](screenshots\ttb.png)

##  4. <a name='UseCase2-Datawillbeupdatedandadded'></a>UseCase 2 - Data will be updated and added

In this case we receive a flatfile with new and updated data and we want to add the new data and update the old entrys in our database.

###  4.1. <a name='Update/InsertinSQLServer'></a>Update/Insert in SQL Server

New Data gets uploaded into temp table, a stored procedure is called and the tmp gets deleted.

![Merge SQL Server](screenshots\merge_in_sql_server.png)

**Stored Procedure**
```sql
MERGE INTO [testing].[dbo].[customers] AS Ziel
USING [testing].[dbo].[tmp_customers] AS Quelle
ON Ziel.customer_id = Quelle.customer_id
WHEN MATCHED THEN
  UPDATE SET
    Ziel.customer_name = Quelle.customer_name,
    Ziel.country = Quelle.country,
    Ziel.email = Quelle.email,
    Ziel.website = Quelle.website
WHEN NOT MATCHED THEN
  INSERT (customer_id, customer_name, country, email, website)
  VALUES (Quelle.customer_id, Quelle.customer_name, Quelle.country, Quelle.email, Quelle.website);
```

###  4.2. <a name='Update/InsertinSSIS'></a>Update/Insert in SSIS

Performs Full Join on half_data_updated.csv and the existing customer table.\
Splits the data on the result of the join and directs it to the insert or stored update procedure.

![Merge SSIS](screenshots\upsert_in_ssis.png)

**Stored Procedure**
```sql
@customer_id int,
@customer_name nvarchar(50),
@country nvarchar(50),
@email nvarchar(50),
@website nvarchar(50)
	

   
UPDATE [testing].[dbo].[customers]
SET
	[customer_name] = @customer_name,
	[country] = @country,
	[email] = @email,
	[website] = @website
WHERE [customer_id] = @customer_id
```

### 4.3 <a name='updateSome'></a> Update only changed entrys in SQL Server

Within a stored procedure the tmp_customers.customer_id  compared with customers.customer_id. When matched, we check if the name changed and if yes, we update the entry. When there is no match, the row gets inserted into customers.

The dataset hast been altered so that 20 entrys in the first 500k rows in customers_full have been altered. Everything else stays the same. 

![UPDATE Changed](screenshots\update_changed.png)

**Stored Procedure**
```sql
MERGE INTO dbo.customers as target
	USING dbo.tmp_customers AS source
	ON target.customer_id = source.customer_id
	WHEN MATCHED AND target.customer_name <> source.customer_name THEN
		UPDATE SET
			target.customer_name = source.customer_name,
			target.country = source.country,
			target.email = source.email,
			target.website = source.website
	WHEN NOT MATCHED THEN
		INSERT (customer_id, customer_name, country, email, website)
		VALUES (source.customer_id, source.customer_name, source.country, source.email, source.website);
```

###  4.3. <a name='TruncatethatB-1'></a>Truncate that B!&%?

See above

##  5. <a name='Results'></a>Results
**Lookup in SSIS**\

1. datetime.timedelta(seconds=10, microseconds=70)
2. datetime.timedelta(seconds=9, microseconds=999813)
3. datetime.timedelta(seconds=10, microseconds=227)

Average: 10.000036666666666 seconds

**Insert in SQL Server**\

1. datetime.timedelta(seconds=16, microseconds=210)
2. datetime.timedelta(seconds=15, microseconds=999622)
3. datetime.timedelta(seconds=16, microseconds=195)

Average: 16.000009 seconds

**Truncate that B!&%?**\

1. datetime.timedelta(seconds=15, microseconds=672)
2. datetime.timedelta(seconds=14, microseconds=999356)
3. datetime.timedelta(seconds=14, microseconds=999623)

Average: 14.999883666666666 seconds

**Update/Insert in SQL Server**\

1. datetime.timedelta(seconds=29, microseconds=86)
2. datetime.timedelta(seconds=28, microseconds=120)
3. datetime.timedelta(seconds=36, microseconds=999742)

Average: 31.333316 seconds

**Update/Insert ins SSIS**\

Stopped the Process after a few Minutes of waiting with 1 mio rows, same with 500k. So working with 20k
1. datetime.timedelta(seconds=209, microseconds=907)
2. datetime.timedelta(seconds=146, microseconds=999783)
3. datetime.timedelta(seconds=145, microseconds=128)

Average: 167.00027266666666 seconds

**Update only changed entrys in SQL Server**

1. datetime.timedelta(seconds=21, microseconds=381)
2. datetime.timedelta(seconds=20, microseconds=999707)
3. datetime.timedelta(seconds=20, microseconds=999980)

Average: 21.000022666666666 seconds

##  6. <a name='Conclusion'></a>Conclusion

The unsatisfying outcome is: Truncate and reloading is the fastest.

The thing to consider here is, that you loose the info about what row has been changed.

If you cant Truncate and reload, use SSIS to Move Data from one place to another and use stored Procedures on SQL Server for transformations.
