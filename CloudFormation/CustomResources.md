### Idempotent S3 Bucket Creation with Custom Resources

Dave Liggat - dliggat@amazon.com - contributed this example of how to properly structure Custom Resources.  It has a self-contained custom resource for an S3 "idempotent bucket" that exports its ServiceToken ARN, and a "client" stack that makes use of it below.

```yaml
AWSTemplateFormatVersion: "2010-09-09"

Description: A 'idempotent_bucket_creator' custom resource.

Parameters:
  ResourcePrefix:
    Type: String
    Description: A description to identify resources  (e.g. "my-perf-test")
    MinLength: 2

Resources:
  CustomResourceLambdaFunction:
    Type: AWS::Lambda::Function
    Metadata:
      Comment:
        "Fn::Sub":
          "Function for ${ResourcePrefix}"
    DependsOn: [ CustomResourceLambdaFunctionExecutionRole ]
    Properties:
      Code:
        ZipFile:
          "Fn::Sub": |
            import boto3
            import json
            import logging
            import traceback
            import cfnresponse

            logging.basicConfig()
            logger = logging.getLogger(__name__)
            logger.setLevel(logging.INFO)

            def update_resource(event, context):
                return {'CreatedByCustomResource': 'unknown'}

            def delete_resource(event, context):
                return {'CreatedByCustomResource': 'unknown'}

            def create_resource(event, context):
                client = boto3.client('s3')
                my_buckets = [ b['Name'] for b in client.list_buckets()['Buckets'] ]
                bucket_name = event['ResourceProperties']['Name']
                if bucket_name in my_buckets:
                    created = 'false'
                else:
                    result = client.create_bucket(Bucket=bucket_name)
                    logger.info('Result: ' + json.dumps(result))
                    created = 'true'
                return {'CreatedByCustomResource': created}, bucket_name

            def handler(event, context):
                logger.info('Event: ' + json.dumps(event))
                logger.info('Context: ' + str(dir(context)))
                operation = event['RequestType']
                physical_id = None
                data = { }
                try:
                    if operation == 'Create':
                        data, physical_id = create_resource(event, context)
                    elif operation == 'Delete':
                        data = delete_resource(event, context)
                    else:
                        data = update_resource(event, context)
                except Exception as e:
                    logger.error('CloudFormation custom resource {0} failed. Exception: {1}'.format(operation, traceback.format_exc()))
                    status = cfnresponse.FAILED
                else:
                    status = cfnresponse.SUCCESS
                    logger.info('CloudFormation custom resource {0} succeeded. Result data {1}'.format(operation, json.dumps(data)))
                cfnresponse.send(event, context, status, data, physical_id)


      Role: { "Fn::GetAtt": [ CustomResourceLambdaFunctionExecutionRole, Arn ] }
      Timeout: "30"  # Seconds.
      Handler: index.handler
      Runtime: python2.7
      MemorySize: "128"  # MB.

  Policy:
    Type: AWS::IAM::Policy
    Properties:
      Roles:
        - { Ref: CustomResourceLambdaFunctionExecutionRole }
      PolicyName: CommonPolicyForLambdaAndDevelopment
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action:
              - "logs:CreateLogGroup"
              - "logs:CreateLogStream"
              - "logs:PutLogEvents"
            Resource: "arn:aws:logs:*:*:*"
          - Effect: Allow
            Action:
              - "s3:CreateBucket"
              - "s3:ListAllMyBuckets"
            Resource: "*"


  CustomResourceLambdaFunctionExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: [ lambda.amazonaws.com ]
            Action:
              - sts:AssumeRole
          - Effect: Allow
            Principal:
              AWS:
                - "Fn::Join":
                  - ""
                  - - "arn:aws:iam::"
                    - { Ref: "AWS::AccountId" }
                    - ":"
                    - "root"
            Action:
              - sts:AssumeRole
      Path: /

Outputs:
  CustomResourceLambdaFunction:
    Value: { Ref : CustomResourceLambdaFunction }
  CustomResourceLambdaFunctionARN:
    Value: { "Fn::GetAtt": [ CustomResourceLambdaFunction, Arn ] }
    Export:
      Name: CustomResourceArn-IdempotentBucketCreator
  CustomResourceLambdaFunctionExecutionRole:
    Value: { Ref : CustomResourceLambdaFunctionExecutionRole }
  CustomResourceLambdaFunctionExecutionRoleARN:
    Value: { "Fn::GetAtt": [ CustomResourceLambdaFunctionExecutionRole, Arn ] }
  SigninUrl:
    Value:
      "Fn::Sub": |
        https://signin.aws.amazon.com/switchrole?account=${AWS::AccountId}&roleName=${CustomResourceLambdaFunctionExecutionRole}&displayName=assumed-role
  TestCommand:
    Value:
      "Fn::Sub": |
        aws lambda invoke --function-name ${CustomResourceLambdaFunction} /tmp/${CustomResourceLambdaFunction}-output.txt; cat /tmp/${CustomResourceLambdaFunction}-output.txt
```

#### Custom Resource "client" stack that leverages the above custom resource.

```yaml
AWSTemplateFormatVersion: "2010-09-09"

Description: An idempotent bucket creator client.

Parameters:
  BucketName:
    Type: String

Resources:
  IdempotentBucket1:
    Type: Custom::IdempotentBucket
    Properties:
      ServiceToken: { "Fn::ImportValue": CustomResourceArn-IdempotentBucketCreator }
      Region: { Ref: "AWS::Region" }
      Name: { Ref: BucketName }

Outputs:
  IdempotentBucket1:
    Value: { Ref: IdempotentBucket1 }
  IdempotentBucketCreatedByCustomResource1:
    Value:
      "Fn::GetAtt": [IdempotentBucket1, CreatedByCustomResource]

```