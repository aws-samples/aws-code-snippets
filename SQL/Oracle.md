### ora_discovery.sql

A script to capture sizing and performance metrics from an Oracle database.  This can be useful when planning database migrations to AWS to "right-size" the instance and storage.  Contributed by Simon Rice, ricesimo@amazon.com.

Sample output:
```
METRIC_NAME                                     MIN                    MAX                     AVG
---------------------------------------------------------------- ------------------------------ ------------------------------
Database Size (Allocated)                                                     1.46 GB
Database Size (Used)                                                          .97 GB
Physical Read Total Bytes Per Sec                     0 KB                   1826.3 KB                  93.1 KB
Physical Read Total IO Requests Per Sec               0                      104.4                   4.2
Physical Write Total Bytes Per Sec                    0 KB                   903.9 KB                  21.2 KB
Physical Write Total IO Requests Per Sec              0                      23.8                    1.2
Redo Writes Per Sec                                   0                      22.4                    .1
```

```sql
define start_date = to_date('01_DEC-2017');
define end_date = to_date('02_DEC-2017');
 
set lines 180
set pages 1000
col min for a30
col max for a30
col avg for a30
 
SELECT metric_name,
  (CASE WHEN metric_name LIKE '%Bytes%' THEN TO_CHAR(ROUND(MIN(minval / 1024),1)) || ' KB' ELSE TO_CHAR(ROUND(MIN(minval),1)) END) min,
  (CASE WHEN metric_name LIKE '%Bytes%' THEN TO_CHAR(ROUND(MAX(maxval / 1024),1)) || ' KB' ELSE TO_CHAR(ROUND(MAX(maxval),1)) END) max,
  (CASE WHEN metric_name LIKE '%Bytes%' THEN TO_CHAR(ROUND(AVG(average / 1024),1)) || ' KB' ELSE TO_CHAR(ROUND(AVG(average),1)) END) avg
FROM dba_hist_sysmetric_summary
WHERE metric_name
  IN ('Physical Read Total IO Requests Per Sec',
  'Physical Write Total IO Requests Per Sec',
  'Physical Read Total Bytes Per Sec',
  'Physical Write Total Bytes Per Sec',
  'Redo Writes Per Sec')
  and begin_time>(&start_date)
  and begin_time<(&end_date)
group by METRIC_NAME
union
select 'Database Size (Allocated)' METRIC_NAME, ' ' MIN, to_char(round(sum(bytes/1024/1024/1024),2)) || ' GB' MAX, ' ' AVG
from dba_data_files
union
select 'Database Size (Used)' METRIC_NAME, ' ' MIN, to_char(round(sum(bytes/1024/1024/1024),2)) || ' GB' MAX, ' ' AVG
from dba_segments
ORDER BY 1
;
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>