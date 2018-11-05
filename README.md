# Query-In-Place Builder Session

AWS Accounts:

builder1 https://464361240967.signin.aws.amazon.com/console

builder2 https://725012194027.signin.aws.amazon.com/console

builder3 https://606504329419.signin.aws.amazon.com/console

builder4 https://245730503502.signin.aws.amazon.com/console

builder5 https://485158749081.signin.aws.amazon.com/console


# Topic 1 - S3 Select and Glacier Select

Sample Data Description: Two CSV files which contains a list of airport name, code, location, etc. 
1. One small size file, 6M, with ~50k rows of records. 
2. Another large size file, 500M, with 4 millions rows of records. 

## S3 Select Builder Instruction:

1. Go to S3 console and find the bucket called builder[x]-us-east-1, find the sample data file, and click the tab called "Select From".

2. Tick "File has header row", Run "Preview", and below SQL expression. 

```sql
select name, municipality  from s3object s where municipality = 'Las Vegas' 
```

3. Launch the pre-created cloud 9 environment on AWS in us-west-2 Oregon region. 

4. Review the python script provided in this repository, "s3-select-compare-small.py" and "s3-select-compare-large.py". 

5. Run the s3-select-small.py a couple times to observe the difference between query with and without s3 select. 

6. Run the s3-select-large.py a couple times to observe the difference between query with and without s3 select. 

## Glacier Select Builder Instruction:

1. Watch the demo. 

# Section 2 - Glue and Athena

In this session, you will do the following:
1. Discover the data as is using AWS Glue. 
2. Query the data using the Athena, with the metadata discovered by AWS Glue. 
3. Optionally using AWS Glue to perform ETL to transform the data from CSV format to Parquet format. Compare query performance using Athena.  

Sample Data used consists of all the rides for the green new york city taxis for the month of January 2017.
Sample File Location: Amazon S3 bucket named s3://aws-bigdata-blog/artifacts/glue-data-lake/data/.

## Discover the data as is and query in place

1. Select AWS Glue in AWS console. Choose the us-west-2 AWS Region. Add a new ddatabase, in Database name, type nycitytaxi, and choose Create.

2. Add a table to the database nycitytaxi by using a crawler. A crawler is a program that connects to a data store and progresses through a prioritized list of classifiers to determine the schema for your data. AWS Glue provides classifiers for common file types like CSV, JSON, Avro, and others. You can also write your own classifier using a grok pattern.

3. To add a crawler, enter the data source: an Amazon S3 bucket named s3://aws-bigdata-blog/artifacts/glue-data-lake/data/. 

4. For IAM role, create a role AWSGlueServiceRole-Default. Make sure it has S3 full access. 

5. For Frequency, choose Run on demand. The crawler can be run on demand or set to run on a schedule.

6. For Database, choose nycitytaxi.

7. Review the steps, and choose Finish. The crawler is ready to run. Choose Run it now. When the crawler has finished, one table has been added.

8. Choose Tables in the left navigation pane, and then choose data. This screen describes the table, including schema, properties, and other valuable information.

9. You can query the data using standard SQL.

    Choose the nytaxigreenparquet
    Type `sql Select * From "nycitytaxi"."data" limit 10;`
    Choose Run Query.

## Athena New Feature: Creating a Table from Query Results (CTAS)

Run below CTAS queries:

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

Use the RDP information that we have provided to you to connect to Redshift with SQL Workbench/J

Create the external table, once done, you will be able to see this database "spectrum_db" in Athena, and Glue. 

Please note your *arn:aws:iam::123456789012:role/mySpectrumRole* will be different, its in the SQL Workbench already.

```sql
create external schema spectrum 
from data catalog 
database 'spectrumdb' 
iam_role 'arn:aws:iam::123456789012:role/mySpectrumRole'
create external database if not exists;
```


To verify, you can check it at Athena console, or run the below query in SQL Workbench/J

```sql
select * from svv_external_schemas where schemaname='spectrum';
```


Then, we create the sales table, once its done, you can directly query it from Athena (Keep your larger fact tables in Amazon S3 and your smaller dimension tables in Amazon Redshift, as a best practice:

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


To verify:

```sql
select count(*) from spectrum.sales;
```

Now, let's create the smaller table:

```sql
create table event(
eventid integer not null distkey,
venueid smallint not null,
catid smallint not null,
dateid smallint not null sortkey,
eventname varchar(200),
starttime timestamp);
```

Load the data to Event table (small table, less than 10K records):

```sql
copy event from 's3://awssampledbuswest2/tickit/allevents_pipe.txt' 
iam_role 'arn:aws:iam::123456789012:role/mySpectrumRole'
delimiter '|' timeformat 'YYYY-MM-DD HH:MI:SS' region 'us-west-2';
```

>Load into table 'event' completed, 8798 record(s) loaded successfully.


The following example joins the external table SPECTRUM.SALES with the local table EVENT to find the total sales for the top ten events.


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


By default, Amazon Redshift creates external tables with the pseudocolumns $path and $size. Select these columns to view the path to the data files on Amazon S3 and the size of the data files for each row returned by a query. 

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

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-04') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-04/';
```

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-05') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-05/';
```

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-06') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-06/';
```

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-07') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-07/';
```

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-08') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-08/';
```

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-09') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-09/';
```

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-10') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-10/';
```

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-11') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-11/';
```

```sql 
alter table spectrum.sales_part
add partition(saledate='2008-12') 
location 's3://awssampledbuswest2/tickit/spectrum/sales_partition/saledate=2008-12/';
```


To view external table partitions, query the SVV_EXTERNAL_PARTITIONS system view.

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


You can monitor Amazon Redshift Spectrum queries using the following system views:

`SVL_S3QUERY`

Use the SVL_S3QUERY view to get details about Redshift Spectrum queries (S3 queries) at the segment and node slice level.

`SVL_S3QUERY_SUMMARY`

Use the SVL_S3QUERY_SUMMARY view to get a summary of all Amazon Redshift Spectrum queries (S3 queries) that have been run on the system.
