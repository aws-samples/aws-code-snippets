Reload all partitions in a table:

```python
import boto3
import json
import os

client 		= boto3.client('athena')
DATABASE 	= os.environ['DATABASE_NAME']
TABLE		  = os.environ['TABLE_NAME']
# The S3 bucket\folder\ location where you would like query results saved.
OUTPUT 		= os.environ['OUTPUT_LOCATION']
# One of the following: 'SSE_S3'|'SSE_KMS'|'CSE_KMS'
ENCRYPT 	= os.environ['ENCRYPT_METHOD']

try:
	response = client.start_query_execution(
	    QueryString='MSCK REPAIR TABLE ' + TABLE + ';',
	    QueryExecutionContext={
	        'Database': DATABASE
	    },
	    ResultConfiguration={
	        'OutputLocation': OUTPUT,
	        'EncryptionConfiguration': {
	            'EncryptionOption': ENCRYPT
	        }
	    }
	)
except ClientError as err:
	print(err.response['Error']['Message'])
else: 
	print("Query succeeded:")
	print(json.dumps(response, indent=4))
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>