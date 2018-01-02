### Setting up PSQL to connect to Redshift on a Mac

```sh
export PGPORT=5439
export PGUSER=username
export PGDATABASE=dbname
export PGHOST=rsdss.database.us-east-1.redshift.amazonaws.com
export PATH=$PATH:/Applications/Postgres.app/Contents/Versions/latest/bin
psql
```

### Unload relational data from Redshift to S3 in a delimited format

Without the need for any knowledge of PySpark code, it is possible to export relational tables in Redshift to an S3 bucket. This could potentially be used as part of a lifecycle management policy, retaining the most recent active data within Redshift, while retaining colder more static data on S3, which is obviously more cost-effective.  The resulting output files will be split into the number of slices in your Redshift cluster, and will contain pipe-delimited data.

#### Requirements
While it is possible to provide access keys and secret keys to this command, it is obviously much more appropriate to create an IAM role with the necessary S3 permissions.
 
#### Restrictions
As far as I am aware, it is not possible to export into Parquet format, which would be more cost effective to query with Spectrum as it will retrieve only relevant columns. Exporting to Parquet format would require a PySpark dataframe.
 
#### Example
In this example, we export any rows with a date starting with “2016” to an S3 bucket called “rental-data-archive”, append a prefix of “2016” to the file names, create a manifest file listing all output files, and gzip the output files to save space (which is important, as Redshift Spectrum is priced on data read):

```sql
unload ('select * from s3_rental_data where rental_date like \'2016%\'')  
to 's3://rental-data-archive/2016'
iam_role 'arn:aws:iam::215894926023:role/RedshiftRole'
manifest
gzip;
```

### Load flat file delimited data into Redshift from S3

Just as we exported data out to S3, it is possible to import it back into Redshift using a COPY command. This would be useful where the performance of a relational database is needed for analytical queries.
 
#### Requirements
Again, an IAM role is required for Redshift to access the S3 bucket.
 
#### Example
In this example, we create a new table, called “s3_2016_data” from pipe-delimited files starting with the prefix “2016” in the S3 bucket “rental-data-archive”:

```sql
copy s3_2016_data from  's3://rental-data-archive/2016'
iam_role 'arn:aws:iam::215894926023:role/RedshiftRole'
delimiter '|';
```

### Create a Redshift Spectrum external table referencing objects on S3

Lastly, it is possible to query data on S3 using Redshift Spectrum, even if that data has not actually been imported into Redshift. In order to do this, we need to create an external schema with privileges to access S3, then create an external table within that schema that references the data on S3.
 
#### Requirements
Again, an IAM role is required for Redshift to access the S3 bucket.

#### Restrictions
The documentation suggests that it is not necessary to use a manifest file for Redshift Spectrum external queries. However, when I tested this functionality, the external table would create with no errors but would return no rows when queried. Using a manifest file was the only way that I managed to actually obtain data from the external table.
 
#### Example
In this example, I’m using the same pipe-delimited file unloaded from Redshift earlier. These files are stilled gzipped, and are referred to by the same manifest file.
 
We can create the external schema like this:

```sql
create external schema spectrum
from data catalog
database 'gluedatacatalog'
iam_role 'arn:aws:iam::123456789012:role/RedshiftRole';
```

And then create the external table like this:

```sql
create external table spectrum.ext_rental_archive(
rental_id bigint,
rental_date varchar,
inventory_id bigint,                  
customer_id  bigint,                  
return_date varchar,
staff_id varchar,
last_update varchar)
row format delimited
fields terminated by '|'
stored as textfile
location 's3://rental-data-archive/2016manifest';
```

#### Testing the above example
It is possible to join an internal Redshift table with the above Spectrum external table using a simple select statement. In this example, the sakila\_customer table is internal to Redshift, and the ext\_rental_archive table is external and resides on S3:

```sql
select a.first_name, a.last_name, b.rental_date
from sakila_customer a, ext_rental_archive b
where a.customer_id=b.customer_id;
```

To see an explain plan of this code, we can simply add “explain” at the beginning, e.g.:

```sql
explain select a.first_name, a.last_name, b.rental_date
from sakila_customer a, ext_rental_archive b
where a.customer_id=b.customer_id;
```

When running the above command, it is possible to see that the query against sakila\_customer is using a regular scan of Redshift slices, while the query against ext\_rental\_archive is using an S3 scan via Redshift Spectrum.

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>