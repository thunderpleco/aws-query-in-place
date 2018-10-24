# Query-In-Place Workshop

This document proivdes the instruction for AWS builder session.
Understanding of a data lake construct, AWS S3 Select, Glacier Select, Athena and Glue is recommended. 

AWS Accounts:

builder1 https://464361240967.signin.aws.amazon.com/console

builder2 https://725012194027.signin.aws.amazon.com/console

builder3 https://606504329419.signin.aws.amazon.com/console

builder4 https://245730503502.signin.aws.amazon.com/console

builder5 https://485158749081.signin.aws.amazon.com/console


# Section 1 - S3 Select and Glacier Select

Sample Data Description: Two CSV files which contains a list of airport name, code, location, etc. 
1. One small size file, 6M, with ~50k rows of records. 
2. Another large size file, 500M, with 4 millions rows of records. 

Sample File Location: The two files are available in a public s3 bucket: anson-us-east-1.

## S3 Select Builder Instruction:
1. Review the python script provided in this repository, "s3-select-compare-small.py" and "s3-select-compare-large.py". 

2. Launch the pre-created cloud 9 environment on AWS in us-east-1 region. 

3. Run the s3-select-small.py a couple times to observe the difference between query with and without s3 select. 

4. Run the s3-select-large.py a couple times to observe the difference between query with and without s3 select. 

5. Review the results, which shows the significant time imporvement of a query performance with s3 select, the larger the data, the greater the performance. 

## Glacier Select Builder Instruction:
1. Review the python script provided in this repository, "glacier-select-compare-large.py" for running in Cloud 9 IDE and "glacier-get-job-ouput.py" for running in lambda. 

2. Launch the pre-created cloud 9 environment on AWS in us-east-1 region, if not already.

3. Verify that the "glacier-select-compare-large.py" exists in cloud 9 IDE. 

4. Before running the script, create a vault in us-east-1 region. Purchase one provisioned capacity. Record the vault name for input to the python script. Enable the vault notification via SNS. 

5. Upload the sample data file to the vault mentioned in step 4: airport-code-large.csv. Record the archive id for input to the python script. 
E.g. run below CLI to upload a file to Glacier:

`aws glacier upload-archive --vault-name builder0-vault1 --account-id - --body airport-code-large.csv`

The output of the CLI looks like below, which contain the archieveID:

`
{
    "location": "/889111795564/vaults/builder0-vault1/archives/TnsNS4AF_Bo2VkQybTCQHcgoz1PbhKuQoPySP1wu8B_TTlKHkn9pjBizsTNvT0nv4mDGypAZLy36qitLaGWj7G45VyBw_oFR8OUHoIuJZuGfF7lUJh8Spwht4ddN6R7j4lGaOMcoYw",
    "checksum": "da7aac2dc381c0acbe6146d7117f552e09c4e575d116e13ee6dadde5555779b1",
    "archiveId": "TnsNS4AF_Bo2VkQybTCQHcgoz1PbhKuQoPySP1wu8B_TTlKHkn9pjBizsTNvT0nv4mDGypAZLy36qitLaGWj7G45VyBw_oFR8OUHoIuJZuGfF7lUJh8Spwht4ddN6R7j4lGaOMcoYw"
}
`

6. Create a glacier select output bucket in us-east-1 region. Record the bucket name and bucket prefix for input to the python script. 

7. Create a lambda function with the lambda scrip provided. The attached IAM role require lambda execution, S3 full access, and Glacier full access. Configure the lambda trigger to the be SNS of the Glacier Vault notification, which was created previously. Configure the lambda memory to max. Configure the output bucket name and object key. 

8. Run the python script "glacier-select-compare-large", the code will do two things, the first part of the code initiates a "archive-retrieval" job, which will trigger the lambda via SNS once the archive is ready for download, Lambda function will download the file and copy to the output bucket and prefix defined previously. The second part of the code will do Glacier Select and only initiate a job only to retrieve the relevant data of the object and generate the outcome to the bucket location define in the python script. 

9. Review the results, which shows in the s3 bucket, one result would have the whole object retrieved and the other result showing only the relevant data retrieved, with much smaller file size. 

# Section 2 - Glue and Athena

In this session, you will do the following:
1. Discover the data as is using AWS Glue. 
2. Query the data using the Athena, with the metadata discovered by AWS Glue. 
3. Optionally using AWS Glue to perform ETL to transform the data from CSV format to Parquet format. Compare query performance using Athena.  

Sample Data used consists of all the rides for the green new york city taxis for the month of January 2017.
Sample File Location: Amazon S3 bucket named s3://aws-bigdata-blog/artifacts/glue-data-lake/data/.

## Discover the data as is and query in place

1. Select AWS Glue in AWS console. Choose the us-east-1 AWS Region. Add database, in Database name, type nycitytaxi, and choose Create.

2. Choose Tables in the navigation pane. A table consists of the names of columns, data type definitions, and other metadata about a dataset. There should be no table at the moment. 

3. Add a table to the database nycitytaxi by using a crawler. A crawler is a program that connects to a data store and progresses through a prioritized list of classifiers to determine the schema for your data. AWS Glue provides classifiers for common file types like CSV, JSON, Avro, and others. You can also write your own classifier using a grok pattern.

4. To add a crawler, enter the data source: an Amazon S3 bucket named s3://aws-bigdata-blog/artifacts/glue-data-lake/data/. 

6. For IAM role, create a role AWSGlueServiceRole-Default. Make sure it has S3 full access. 

7. For Frequency, choose Run on demand. The crawler can be run on demand or set to run on a schedule.

8. For Database, choose nycitytaxi.

9. Review the steps, and choose Finish. The crawler is ready to run. Choose Run it now. When the crawler has finished, one table has been added.

10. Choose Tables in the left navigation pane, and then choose data. This screen describes the table, including schema, properties, and other valuable information.

11. You can query the data using standard SQL.

    Choose the nytaxigreenparquet
    Type `sql Select * From "nycitytaxi"."data" limit 10;`
    Choose Run Query.


## Optionally, transform the data from CSV to Parquet format, and query in place
Now you can configure and run a job to transform the data from CSV to Parquet. Parquet is a columnar format that is well suited for AWS analytics services like Amazon Athena and Amazon Redshift Spectrum.

1. Under ETL in the left navigation pane, choose Jobs, and then choose Add job.
2. For the Name, type nytaxi-csv-parquet.
3. For the IAM role, choose AWSGlueServiceRoleDefault.
4. For This job runs, choose A proposed script generated by AWS Glue.
5. Provide a unique Amazon S3 path to store the scripts.
6. Provide a unique Amazon S3 directory for a temporary directory.
7. Choose Next.
8. Choose data as the data source.
9. Choose Create tables in your data target.
10. Choose Parquet as the format.
11. Choose a new location (a new prefix location without any existing objects) to store the results.
12. Verify the schema mapping, and choose Finish.
13. View the job.This screen provides a complete view of the job and allows you to edit, save, and run the job.AWS Glue created this script. However, if required, you can create your own.
14. Choose Save, and then choose Run job.

Add the Parquet table and crawler
When the job has finished, add a new table for the Parquet data using a crawler.

1. For Crawler name, type nytaxiparquet.
2. Choose S3 as the Data store.
3. Include the Amazon S3 path chosen in the ETL
4. For the IAM role, choose AWSGlueServiceRoleDefault.
5. For Database, choose nycitytaxi.
6. For Frequency, choose Run on demand.

After the crawler has finished, there are two tables in the nycitytaxi database: a table for the raw CSV data and a table for the transformed Parquet data.

You can query the data using standard SQL.

Choose the nytaxigreenparquet
Type `sql Select * From "nycitytaxi"."data" limit 10;`
Choose Run Query.

## Athena New Feature: Creating a Table from Query Results (CTAS)

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
CREATE TABLE nyctaxi_new_table_pq
WITH (
      external_location='s3://builder0-us-east-1/nyctaxi_pq'
      format = 'Parquet',
      parquet_compression = 'SNAPPY')
AS SELECT *
FROM "data";
```

Conclusion
This post demonstrates how easy it is to build the foundation of a data lake using AWS Glue and Amazon S3. By using AWS Glue to crawl your data on Amazon S3 and build an Apache Hive-compatible metadata store, you can use the metadata across the AWS analytic services and popular Hadoop ecosystem tools. This combination of AWS services is powerful and easy to use, allowing you to get to business insights faster.



