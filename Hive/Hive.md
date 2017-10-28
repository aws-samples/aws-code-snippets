DDL for Apache Extended Log Format:

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS mytable (
  `epochtime` bigint,
  `time-taken` int,
  `c-ip` string,
  `filesize` int,
  `s-ip` string,
  `s-port` int,
  `sc-status` string,
  `sc-bytes` int,
  `cs-method` string,
  `cs-uri-stem` string,
  `dash` string,
  `rs-duration` int,
  `rs-bytes` int,
  `c-referrer` string,
  `c-user-agent` string,
  `customer-id` int,
  `x-ec_custom-1` string 
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
  'serialization.format' = '1',
  'input.regex' = '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ \"]*|\".*\") ([^ ]*) ([^ \"]*|\".*\")'
) LOCATION 's3://mybucket/path/'
TBLPROPERTIES ('has_encrypted_data'='false');
```

Setting up Hive to compress output and split it into multiple files (mapred.reduce.tasks):

- note 7/12/17 - this may no longer be necessary on recent versions of EMR - need to confirm.

```sql
set mapred.output.compress=true;
set hive.exec.compress.output=true;
set mapred.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec;
set io.compression.codecs=org.apache.hadoop.io.compress.GzipCodec;
set mapred.reduce.tasks=16;
```

Useful input regex to capture different text log file fields:

```sql
-- Capture a standard text field, without quotes around it:
([^ ]*) 
-- Capture a double-quoted text field like "field":
([^ \"]*|\".*\")
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>