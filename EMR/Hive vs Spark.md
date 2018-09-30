SSH to EMR master node

  `ssh -i ~/your-key-name.pem hadoop@ec2-000-000-000-000.compute-1.amazonaws.com`

  `hive`

Run below command to create the external table:

  `create external table orders_hive_vs_spark_test
  (O_ORDERKEY INT, 
  O_CUSTKEY INT, 
  O_ORDERSTATUS STRING, 
  O_TOTALPRICE DOUBLE, 
  O_ORDERDATE STRING, 
  O_ORDERPRIORITY STRING, 
  O_CLERK STRING, 
  O_SHIPPRIORITY INT, 
  O_COMMENT STRING) 
  ROW FORMAT DELIMITED FIELDS TERMINATED BY '|' 
  LOCATION 's3://bigdatalabridge/emrdata/';`



  `create external table lineitem_hive_vs_spark_test (
  L_ORDERKEY INT,
  L_PARTKEY INT,
  L_SUPPKEY INT,
  L_LINENUMBER INT,
  L_QUANTITY INT,
  L_EXTENDEDPRICE DOUBLE,
  L_DISCOUNT DOUBLE,
  L_TAX DOUBLE,
  L_RETURNFLAG STRING,
  L_LINESTATUS STRING,
  L_SHIPDATE STRING,
  L_COMMITDATE STRING,
  L_RECEIPTDATE STRING,
  L_SHIPINSTRUCT STRING,
  L_SHIPMODE STRING, L_COMMENT STRING)
  ROW FORMAT DELIMITED FIELDS TERMINATED BY '|'
  LOCATION 's3://bigdatalabridge/emrdata/';`

Run below query:


`select 
  l_shipmode,
  sum(case
    when o_orderpriority ='1-URGENT'
         or o_orderpriority ='2-HIGH'
    then 1
    else 0
end
  ) as high_line_count,
  sum(case
    when o_orderpriority <> '1-URGENT'
         and o_orderpriority <> '2-HIGH'
    then 1
    else 0
end
  ) as low_line_count
from
  orders_hive_vs_spark_test o join lineitem_hive_vs_spark_test l 
  on 
    o.o_orderkey = l.l_orderkey and l.l_commitdate < l.l_receiptdate
and l.l_shipdate < l.l_commitdate and l.l_receiptdate >= '1996-01-01' 
and l.l_receiptdate < '1997-01-01'
where 
  l.l_shipmode = 'MAIL' or l.l_shipmode = 'SHIP'
group by l_shipmode
order by l_shipmode;`

Time taken: 299.054 seconds, Fetched: 2 row(s)

Exit Hive by:

  `quit;`

Spark-sql is compatible with Hive metadata store, table created in Hive can be used directly by Spark-sql

Start Spark and execute the same query

  `spark-sql`

Then, load the tables into memory

`cache table orders_hive_vs_spark_test;`

Time taken: 536.678 seconds

`cache table lineitem_hive_vs_spark_test;`

Time taken: 1005.38 seconds


Run the same query above:
......

MAIL	62265	875315

SHIP	62260	872950

Time taken: 106.237 seconds, Fetched 2 row(s)



free up the cache by:

`unpersist;`

once cleaned accumulator... quit spark-sql:

`quit;`


Testing result, Spark is much more faster, difference may vary based on the node type and cluster size.

Please note there is time that required for caching, and please unpersist the cache after the test.

Same query can be run directly on Athena, the result: (Run time: 7.95 seconds, Data scanned: 17.74GB)
![](https://laiase.com/png/Athena+Result.png)

