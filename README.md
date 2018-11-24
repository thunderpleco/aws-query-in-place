# Query-In-Place Workshop

This document proivdes the instruction for AWS builder session.
Understanding of a data lake construct, AWS S3 Select, Glacier Select, Athena and Glue is recommended. 

AWS Accounts:

Workstation1 

**https://464361240967.signin.aws.amazon.com/console**



Workstation2 

**https://725012194027.signin.aws.amazon.com/console**



Workstation3 

**https://606504329419.signin.aws.amazon.com/console**



Workstation4 

**https://245730503502.signin.aws.amazon.com/console**



Workstation5 

**https://485158749081.signin.aws.amazon.com/console**



# Topic 1 - S3 Select and Glacier Select

**Sample Data**: Two CSV files which contains a list of airport name, code, location, etc. 
1. One small size file, 6M, with ~50k rows of records. 
2. Another large size file, 500M, with 4 millions rows of records. 

## S3 Select Builder Instruction:

1. Go to S3 console and find the bucket called **builder[x]-us-west-2**, find the sample data file, and click the tab called **"Select From"**.

2. Tick **"File has header row"**, Run **"Preview"**, as well as the following SQL query:

```sql
select name, municipality  from s3object s where municipality = 'Las Vegas' 
```

3. Launch the pre-created cloud 9 environment on AWS in **us-west-2** Oregon region. 

4. Review the pre-loaded python script, **"s3-select-compare-small.py"** and **"s3-select-compare-large.py"**. 

5. Run the **s3-select-small.py** a couple times to observe the difference between query with and without s3 select. 

6. Run the **s3-select-large.py** a couple times to observe the difference between query with and without s3 select. 

## Glacier Select Builder Instruction:

1. Watch the demo, which shows the difference between normal Glacier retrival and Glacier Select. 

# Topic 2 - Glue and Athena
 
**Sample Data**: Infomation of the rides for the green new york city taxis for the month of January 2017.

Sample File Location: Amazon S3 bucket named **s3://aws-bigdata-blog/artifacts/glue-data-lake/data/**

## Discover the data as is and query in place

1. Select AWS Glue in AWS console. Choose the **us-west-2** AWS Region. Add a new ddatabase, in Database name, type **nycitytaxi**, and choose Create.

2. Add a table to the database **nycitytaxi** by using a crawler. Choose crawler, add crawler, enter the data source: an Amazon S3 bucket named **s3://aws-bigdata-blog/artifacts/glue-data-lake/data/** 

3. For IAM role, create a role e.g. **AWSGlueServiceRole-builderxxxx**. 

4. For Frequency, choose Run on demand. The crawler can be run on demand or set to run on a schedule.

5. For Database, choose **nycitytaxi**.

6. Review the steps, and choose Finish. The crawler is ready to run. Choose **Run it now**. When the crawler has finished, one table has been added.

7. Now, let's go to Athena, choose Tables in the left navigation pane, and then choose table **"data"**. This screen describes the table, including schema, properties, and other valuable information. You can preview the table. 

9. You can query the data using standard SQL, such as:

```sql 
Select * From "nycitytaxi"."data" limit 10;
```



## Athena New Feature: Creating a Table from Query Results (CTAS)
A CREATE TABLE AS SELECT (CTAS) query creates a new table in Athena from the results of a SELECT statement from another query. Athena stores data files created by the CTAS statement in a specified location in **Amazon S3**. Try to run below sample queries. 


```sql
CREATE TABLE nyctaxi_new_table AS 
SELECT * 
FROM "data";
```

```sql
CREATE TABLE nyctaxi_new_table_pq
WITH (
      format = 'Parquet',
      parquet_compression = 'SNAPPY')
AS SELECT *
FROM "data";
```

```sql
CREATE TABLE nyctaxi_new_table_pq_snappy
WITH (
      external_location='s3://builder[x]-us-west-2/nyctaxi_pq_snappy',
      format = 'Parquet',
      parquet_compression = 'SNAPPY')
AS SELECT *
FROM "data";
```

# Topic 3 - Redshift Spectrum

Use the RDP information that we have provided to you to connect to **Amazon Redshift** with **SQL Workbench/J**

As **SQL Workbench/J** is already opened, please go to menu "File", "Connect Window" to re-establish the connection to **Amazon Redshift**

Create the external table, once done, you will be able to see this database "spectrum_db" in **Amazon Athena**, and **AWS Glue*. 

Please note your **arn:aws:iam::123456789012:role/mySpectrumRole** will be different, its in the SQL Workbench already.

```sql
create external schema spectrum 
from data catalog 
database 'spectrumdb' 
iam_role 'arn:aws:iam::123456789012:role/mySpectrumRole'
create external database if not exists;
```


To verify, you can check it at **Amazon Athena** console, or run the below query in **SQL Workbench/J**

```sql
select * from svv_external_schemas where schemaname='spectrum';
```


The following example creates a table named SALES in the Amazon Redshift external schema named spectrum. The data is in tab-delimited text files. Once done, you can directly query it from **Amazon Athena** (Keep your larger fact tables in **Amazon S3** and your smaller dimension tables in **Amazon Redshift**, as one of the best practices:

```sql
create external table spectrum.sales(
salesid integer,
listid integer,
sellerid integer,
buyerid integer,
eventid integer,
dateid smallint,
qtysold smallint,
pricepaid decimal(8,2),
commission decimal(8,2),
saletime timestamp)
row format delimited
fields terminated by '\t'
stored as textfile
location 's3://awssampledbuswest2/tickit/spectrum/sales/'
table properties ('numRows'='172000');
```


To check how many records count:

```sql
select count(*) from spectrum.sales;
```

Now, let's create the smaller table in Amazon Redshift:

```sql
create table event(
eventid integer not null distkey,
venueid smallint not null,
catid smallint not null,
dateid smallint not null sortkey,
eventname varchar(200),
starttime timestamp);
```

Load the data to Event table in the Amazon Redshift (small table, less than 10K records):

```sql
copy event from 's3://awssampledbuswest2/tickit/allevents_pipe.txt' 
iam_role 'arn:aws:iam::123456789012:role/mySpectrumRole'
delimiter '|' timeformat 'YYYY-MM-DD HH:MI:SS' region 'us-west-2';
```

>Load into table 'event' completed, 8798 record(s) loaded successfully.


The following example joins the external table **SPECTRUM.SALES** with the local table EVENT to find the total sales for the top ten events.


```sql
select top 10 spectrum.sales.eventid, sum(spectrum.sales.pricepaid) from spectrum.sales, event
where spectrum.sales.eventid = event.eventid
and spectrum.sales.pricepaid > 30
group by spectrum.sales.eventid
order by 2 desc;
```


View the query plan for the previous query. Note the S3 Seq Scan, S3 HashAggregate, and S3 Query Scan steps that were executed against the data on Amazon S3.


```sql
explain
select top 10 spectrum.sales.eventid, sum(spectrum.sales.pricepaid) 
from spectrum.sales, event
where spectrum.sales.eventid = event.eventid
and spectrum.sales.pricepaid > 30
group by spectrum.sales.eventid
order by 2 desc;
```


By default, Amazon Redshift creates external tables with the pseudocolumns **$path** and **$size**. Select these columns to view the path to the data files on Amazon S3 and the size of the data files for each row returned by a query. 

```sql
select "$path", "$size" from spectrum.sales where dateid = '1983';
```


Question? Does above query incur charges?

###
Create an external table that is partitioned by month

```sql
create external table spectrum.sales_part(
salesid integer,
listid integer,
sellerid integer,
buyerid integer,
eventid integer,
dateid smallint,
qtysold smallint,
pricepaid decimal(8,2),
commission decimal(8,2),
saletime timestamp)
partitioned by (saledate char(10))
row format delimited
fields terminated by '|'
stored as textfile
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/'
table properties ('numRows'='172000');
```


```sql 
alter table spectrum.sales_part
add partition(saledate='2008-01') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-01/';
```

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-02') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-02/';
```

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-03') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-03/';
```

**We only need to partition several months, as we will use 2008-01 in our next query, let's just partition 3 of them to understand how it works, and you can feel free to partition more months and use them in your own query later.**


To view external table partitions, query the **SVV_EXTERNAL_PARTITIONS** system view.

```sql
select schemaname, tablename, values, location from svv_external_partitions
where tablename = 'sales_part';
```


Run the following query to select data from the partitioned table.


```sql
select top 10 spectrum.sales_part.eventid, sum(spectrum.sales_part.pricepaid) 
from spectrum.sales_part, event
where spectrum.sales_part.eventid = event.eventid
  and spectrum.sales_part.pricepaid > 30
  and saledate = '2008-01'
group by spectrum.sales_part.eventid
order by 2 desc;
```


You can monitor **Amazon Redshift Spectrum** queries using the following system views:

`SVL_S3QUERY`

Use the **SVL_S3QUERY** view to get details about **Amazon Redshift Spectrum** queries (S3 queries) at the segment and node slice level.

`SVL_S3QUERY_SUMMARY`

Use the **SVL_S3QUERY_SUMMARY** view to get a summary of all **Amazon Redshift Spectrum** queries (Amazon S3 queries) that have been run on the system.


# Additional Material:
https://github.com/thunderpleco/aws-query-in-place
