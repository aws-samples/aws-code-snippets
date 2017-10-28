Check to see if there are messages in an SQS queue, and do something if there are:

```python
import boto3
from botocore.exceptions import ClientError

REGION				= os.environ['REGION_NAME']
SQS_QUEUE			= os.environ['CONFIG_SQS_QUEUE']

sqs			= boto3.resource('sqs', region_name=REGION)
queue		= sqs.get_queue_by_name(QueueName=SQS_QUEUE)

try:
		message = queue.receive_messages(
			MessageAttributeNames=['All'],
			VisibilityTimeout=0
			)
	except ClientError as e:
		print(e.get_response['Error']['Message'])
	else: # read queue succeeded; did we get a message?
		if message:
			DoSomething();
		else:
			print("No messages in SQS queue; exiting...")
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>