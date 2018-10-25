Start EMR cluster with release 5.17.0 or above.

Assume we have a file located in S3 folder "ridge-temp-emr-presto-s3-select"

`Number	Name`

`1	Ridge`

`2	Anson`

`3	James`

`4	Tom`


SSH into the master node

Turn off Spark-SQL verbose mode

`sudo sed -i -e 's/rootCategory=INFO/rootCategory=WARN/' /etc/spark/conf/log4j.properties`

run `spark-sql`

Create a view 
```sql
CREATE TEMPORARY VIEW MyView (number INT, name STRING) USING s3selectCSV OPTIONS (path "s3://ridge-temp-emr-presto-s3-select/spark-select.csv", header "true", delimiter "\t");
```

You can then query the file by using statement like:

```sql
Select Name from MyView Where Number=2;
```
