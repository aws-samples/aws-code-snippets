DynamoDB Setup/Headers

```python
from __future__ import print_function # Python 2/3 compatibility
import json
import boto3
import os
import time
import decimal
from boto3.dynamodb.conditions import Key, Attr
from botocore.exceptions import ClientError

# Helper class to convert a DynamoDB item to JSON.
class DecimalEncoder(json.JSONEncoder):
    def default(self, o):
        if isinstance(o, decimal.Decimal):
            if o % 1 > 0:
                return float(o)
            else:
                return int(o)
        return super(DecimalEncoder, self).default(o)
        
REGION		= os.environ['REGION_NAME']
DDB_TABLE	= os.environ['DDB_TABLE_NAME']
dynamodb	= boto3.resource('dynamodb', region_name=REGION)
table 		= dynamodb.Table(DDB_TABLE)
```

Create an Item:

```python
def CreateItem():
	try:
		put_response = table.put_item(
			Item={
				'key': value,
				'region': REGION
			}
		)
	except ClientError as err:
		print(err.put_response['Error']['Message'])
	else: 
		print("PutItem succeeded:")
		print(json.dumps(put_response, indent=4))
```

Try to get an item, create it if it doesn’t exist, but if it does, compare the data and take some action:

```python
try:
  item = get_response['Item']
except KeyError: # if get_response is empty, create an item in table
	CreateItem();
else:
	print("GetItem succeeded:")
	print(json.dumps(get_response, indent=4, cls=DecimalEncoder))
	print("Current time: " + str(curr_time))
	print("Last build time: " + str(get_response['Item']['lastbuild']))
	print("Last build time + cooldown: " + str(int(get_response['Item']['lastbuild']) + COOLDOWN))
  if curr_time > (get_response['Item']['lastbuild'] + COOLDOWN):
	  UpdateItem();
		StartBuild();
	else:
		print("The build last ran " + str(curr_time - get_response['Item']['lastbuild']) + " seconds ago; exiting...")
```

Update an item:

```python
def UpdateItem():
	try:
		response = table.update_item(
		    Key={
				'cb_project': CODEBUILD_PROJECT,
				'region': REGION
			},
		    UpdateExpression="set lastbuild = :t",
		    ExpressionAttributeValues={
				':t': curr_time
		    },
		    ReturnValues="UPDATED_NEW"
		)
	except ClientError as err:
		print(err.response['Error']['Message'])
	else:
		print("UpdateItem succeeded:")
		print(json.dumps(response, indent=4, cls=DecimalEncoder))
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>