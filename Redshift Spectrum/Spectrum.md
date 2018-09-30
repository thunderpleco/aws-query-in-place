Connect to Redshift with SQL Workbench/J

Create the external table, once done, you will be able to see this database "spectrum_db" in Athena, and Glue.

`create external schema spectrum_schema from data catalog 
database 'spectrum_db' 
iam_role 'arn:aws:iam::AWS-ACCOUNT:role/Your-RedshiftSpectrumRole' 
create external database if not exists;`

To verify, you can check it at Athena console, or run the below query in SQL Workbench/J

`select * from svv_external_schemas where schemaname='spectrum_schema';`



