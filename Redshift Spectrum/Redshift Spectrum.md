Use the RDP information that we have provided to you to connect to Redshift with *SQL Workbench/J*

Create the external table, once done, you will be able to see this database "spectrum_db" in Athena, and Glue. 

Please note your *arn:aws:iam::123456789012:role/mySpectrumRole* will be different, its in the SQL Workbench already.

```sql
create external schema spectrum 
from data catalog 
database 'spectrumdb' 
iam_role 'arn:aws:iam::123456789012:role/mySpectrumRole'
create external database if not exists;
```


To verify, you can check it at Athena console, or run the below query in *SQL Workbench/J*

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
