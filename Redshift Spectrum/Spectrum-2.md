###
Further test you can do at home, to create an external table partitioned by date and eventid, run the following command.


```sql
create external table spectrum.sales_event(
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
partitioned by (salesmonth char(10), event integer)
row format delimited
fields terminated by '|'
stored as textfile
location 's3://awssampledbuswest2/tickit/spectrum/salesevent/'
table properties ('numRows'='172000');
```



```sql 
alter table spectrum.sales_event
add partition(salesmonth='2008-01', event='101') 
location 's3://awssampledbuswest2/tickit/spectrum/salesevent/salesmonth=2008-01/event=101/';
```

```sql 
alter table spectrum.sales_event
add partition(salesmonth='2008-01', event='102') 
location 's3://awssampledbuswest2/tickit/spectrum/salesevent/salesmonth=2008-01/event=102/';
```

```sql 
alter table spectrum.sales_event
add partition(salesmonth='2008-01', event='103') 
location 's3://awssampledbuswest2/tickit/spectrum/salesevent/salesmonth=2008-01/event=103/';
```

```sql 
alter table spectrum.sales_event
add partition(salesmonth='2008-02', event='101') 
location 's3://awssampledbuswest2/tickit/spectrum/salesevent/salesmonth=2008-02/event=101/';
```

```sql 
alter table spectrum.sales_event
add partition(salesmonth='2008-02', event='102') 
location 's3://awssampledbuswest2/tickit/spectrum/salesevent/salesmonth=2008-02/event=102/';
```

```sql 
alter table spectrum.sales_event
add partition(salesmonth='2008-02', event='103') 
location 's3://awssampledbuswest2/tickit/spectrum/salesevent/salesmonth=2008-02/event=103/';
```

```sql 
alter table spectrum.sales_event
add partition(salesmonth='2008-03', event='101') 
location 's3://awssampledbuswest2/tickit/spectrum/salesevent/salesmonth=2008-03/event=101/';
```

```sql 
alter table spectrum.sales_event
add partition(salesmonth='2008-03', event='102') 
location 's3://awssampledbuswest2/tickit/spectrum/salesevent/salesmonth=2008-03/event=102/';
```

```sql 
alter table spectrum.sales_event
add partition(salesmonth='2008-03', event='103') 
location 's3://awssampledbuswest2/tickit/spectrum/salesevent/salesmonth=2008-03/event=103/';

```


Run the following query to select data from the partitioned table.


```sql
select spectrum.sales_event.salesmonth, event.eventname, sum(spectrum.sales_event.pricepaid) 
from spectrum.sales_event, event
where spectrum.sales_event.eventid = event.eventid
  and salesmonth = '2008-02'
	and (event = '101'
	or event = '102'
	or event = '103')
group by event.eventname, spectrum.sales_event.salesmonth
order by 3 desc;
```


You can monitor Amazon Redshift Spectrum queries using the following system views:

`SVL_S3QUERY`

Use the SVL_S3QUERY view to get details about Redshift Spectrum queries (S3 queries) at the segment and node slice level.

`SVL_S3QUERY_SUMMARY`

Use the SVL_S3QUERY_SUMMARY view to get a summary of all Amazon Redshift Spectrum queries (S3 queries) that have been run on the system.
