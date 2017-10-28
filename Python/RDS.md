Start and Stop RDS instances through Lambda/Python. Input is as follows:

```json
{
  "instances": ["instance-1", "instance-2"],
  "action": "stop"
}
```

```python
import boto3
RDS = boto3.client('rds')
def lambda_handler(event, context):
    
    # Check that our inputs are valid
    try:
        instances = event.get('instances')
        action = event.get('action')
    except Exception as e:
        return "Exception! Failed with: {0}".format(e)
    
    if (not (action == "stop" or action == "start")) or (not isinstance(instances, list)):
        return "instances must be a list of strings, action must be \"start\" or \"stop\""
    
    # Filter through our databases, only get the instances that are featured in our instances list
    dbs = set([])
    rds_instances = RDS.describe_db_instances()
    for rds_instance in rds_instances['DBInstances']:
        for instance in instances:
            if instance in rds_instance['DBInstanceIdentifier']:
                dbs.add(rds_instance['DBInstanceIdentifier'])
    
    # Apply our action
    for db in dbs:
        try:
            if action == "start":
                response = RDS.start_db_instance(DBInstanceIdentifier=db)
            else:
                response = RDS.stop_db_instance(DBInstanceIdentifier=db)
                
            print("{0} status: {1}".format(db, response['DBInstanceStatus']))
        except Exception as e:
            print("Exception: {0}".format(e))
    
    return "Completed!"
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>