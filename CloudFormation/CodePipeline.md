## Notify an email list/address of CodePipeline events

This CloudFormation template was submitted by Jose Ferraris, joferr@amazon.it.  It created a CloudWatch rule that acts on CodePipeline events and notifies an email list when the pipeline starts, succeeds, or fails.

```yaml
PipelineEventRule:
  Type: "AWS::Events::Rule"
  Properties:
    Description: "Trigger notifications based on pipeline state changes"
    EventPattern:
      source:
        - "aws.codepipeline"
      detail-type:
        - "CodePipeline Pipeline Execution State Change"
      detail:
        state:
          - "FAILED"
          - "STARTED"
          - "SUCCEEDED"
        pipeline:
          - !Ref CodePipeline
    State: "ENABLED"
    Targets:
      -
        Arn: !Ref PipelineSNSTopic
        Id: !Sub "${AWS::StackName}"
        InputTransformer:
          InputTemplate: '"The pipeline <pipeline> from account <account> has <state> at <at>."'
          InputPathsMap:
            pipeline: "$.detail.pipeline"
            state: "$.detail.state"
            at: "$.time"
            account: "$.account"
PipelineSNSTopic:
  Type: AWS::SNS::Topic
  Properties:
    Subscription:
      - Endpoint: yourmailinglist@amazon.com
        Protocol: email
PipelineSNSTopicPolicy:
  Type: AWS::SNS::TopicPolicy
  Properties:
    PolicyDocument:
      Id: MyTopicPolicy
      Version: '2012-10-17'
      Statement:
      - Effect: Allow
        Principal:
          Service:
            - events.amazonaws.com
        Action: sns:Publish
        Resource: !Ref PipelineSNSTopic
    Topics:
    - !Ref PipelineSNSTopic

```