## ecs-cluster.yaml

This CloudFormation template creates an ECS cluster in a (fixed-size) Autoscaling Group.

```yaml
Description: >
    This template deploys an ECS cluster to the provided VPC and subnets 
    using an Auto Scaling Group
    Created by Luke Youngblood, lukey@amazon.com
Parameters:

    EnvironmentName:
        Description: An environment name that will be prefixed to resource names
        Type: String

    InstanceType: 
        Description: Which instance type should we use to build the ECS cluster?
        Type: String
        Default: t2.medium

    ClusterSize:
        Description: How many ECS hosts do you want to initially deploy?
        Type: Number
        Default: 3

    VPC:
        Description: Choose which VPC this ECS cluster should be deployed to
        Type: AWS::EC2::VPC::Id

    Subnets:
        Description: Choose which subnets this ECS cluster should be deployed to
        Type: List<AWS::EC2::Subnet::Id>

    CIDR:
        Description: CIDR range that you will allow SSH traffic from (default is from Default VPC only)
        Type: String
        Default: "172.31.0.0/16"
        
    KeyPair:
        Description: Select the KeyPair that you would like to use for the ECS cluster hosts
        Type: AWS::EC2::KeyPair::KeyName

Mappings:

    # These are the latest ECS optimized AMIs as of December 2017:
    #
    #   amzn-ami-2017.09.d-amazon-ecs-optimized
    #   ECS agent:    1.16.0
    #   Docker:       17.06.2-ce
    #   ecs-init:     1.16.0-1
    #
    # You can find the latest available on this page of our documentation:
    # http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html
    # (note the AMI identifier is region specific) 

    AWSRegionToAMI:
        us-east-2:
            AMI: ami-58f5db3d
        us-east-1:
            AMI: ami-fad25980
        us-west-2:
            AMI: ami-7114c909
        us-west-1:
            AMI: ami-62e0d802
        eu-west-3:
            AMI: ami-d179ceac
        eu-west-2:
            AMI: ami-dbfee1bf
        eu-west-1:
            AMI: ami-4cbe0935
        eu-central-1:
            AMI: ami-05991b6a
        ap-northeast-2:
            AMI: ami-7267c01c
        ap-northeast-1:
            AMI: ami-56bd0030
        ap-southeast-2:
            AMI: ami-14b55f76
        ap-southeast-1:
            AMI: ami-1bdc8b78
        ca-central-1:
            AMI: ami-918b30f5
        ap-south-1:
            AMI: ami-e4d29c8b
        sa-east-1:
            AMI: ami-d596d2b9

Resources:

    ECSCluster:
        Type: AWS::ECS::Cluster
        Properties:
            ClusterName: !Ref EnvironmentName

    ECSAutoScalingGroup:
        Type: AWS::AutoScaling::AutoScalingGroup
        Properties: 
            VPCZoneIdentifier: !Ref Subnets
            LaunchConfigurationName: !Ref ECSLaunchConfiguration
            MinSize: !Ref ClusterSize
            MaxSize: !Ref ClusterSize
            DesiredCapacity: !Ref ClusterSize
            Tags: 
                - Key: Name
                  Value: !Sub ${EnvironmentName} ECS host
                  PropagateAtLaunch: true
        CreationPolicy:
            ResourceSignal: 
                Timeout: PT15M
        UpdatePolicy:
            AutoScalingRollingUpdate:
                MinInstancesInService: 1
                MaxBatchSize: 1
                PauseTime: PT15M
                SuspendProcesses:
                  - HealthCheck
                  - ReplaceUnhealthy
                  - AZRebalance
                  - AlarmNotification
                  - ScheduledActions
                WaitOnResourceSignals: true
        
    ECSLaunchConfiguration:
        Type: AWS::AutoScaling::LaunchConfiguration
        Properties:
            ImageId:  !FindInMap [AWSRegionToAMI, !Ref "AWS::Region", AMI]
            InstanceType: !Ref InstanceType
            SecurityGroups: 
                - !Ref InstanceSecurityGroup
            IamInstanceProfile: !Ref ECSInstanceProfile
            KeyName: !Ref KeyPair
            UserData: 
                "Fn::Base64": !Sub |
                    #!/bin/bash
                    yum install -y aws-cfn-bootstrap
                    /opt/aws/bin/cfn-init -v --region ${AWS::Region} --stack ${AWS::StackName} --resource ECSLaunchConfiguration
                    /opt/aws/bin/cfn-signal -e $? --region ${AWS::Region} --stack ${AWS::StackName} --resource ECSAutoScalingGroup
        Metadata:
            AWS::CloudFormation::Init:
                config:
                    commands:
                        01_add_instance_to_cluster:
                            command: !Sub echo ECS_CLUSTER=${ECSCluster} >> /etc/ecs/ecs.config
                    files:
                        "/etc/cfn/cfn-hup.conf":
                            mode: 000400
                            owner: root
                            group: root
                            content: !Sub |
                                [main]
                                stack=${AWS::StackId}
                                region=${AWS::Region}
                        
                        "/etc/cfn/hooks.d/cfn-auto-reloader.conf":
                            content: !Sub |
                                [cfn-auto-reloader-hook]
                                triggers=post.update
                                path=Resources.ECSLaunchConfiguration.Metadata.AWS::CloudFormation::Init
                                action=/opt/aws/bin/cfn-init -v --region ${AWS::Region} --stack ${AWS::StackName} --resource ECSLaunchConfiguration
                    services: 
                        sysvinit:
                            cfn-hup: 
                                enabled: true
                                ensureRunning: true
                                files: 
                                    - /etc/cfn/cfn-hup.conf
                                    - /etc/cfn/hooks.d/cfn-auto-reloader.conf

    # This IAM Role is attached to all of the ECS hosts. It is based on the default role
    # published here:
    # http://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html
    #
    # You can add other IAM policy statements here to allow access from your ECS hosts
    # to other AWS services. Please note that this role will be used by ALL containers
    # running on the ECS host.

    ECSRole:
        Type: AWS::IAM::Role
        Properties: 
            Path: /
            RoleName: !Sub ${EnvironmentName}-ECSRole-${AWS::Region}
            AssumeRolePolicyDocument: |
                {
                    "Statement": [{
                        "Action": "sts:AssumeRole",
                        "Effect": "Allow",
                        "Principal": { 
                            "Service": "ec2.amazonaws.com" 
                        }
                    }]
                }
            Policies: 
                - PolicyName: ecs-service
                  PolicyDocument: |
                    {
                        "Statement": [{
                            "Effect": "Allow",
                            "Action": [
                                "ecs:CreateCluster",
                                "ecs:DeregisterContainerInstance",
                                "ecs:DiscoverPollEndpoint",
                                "ecs:Poll",
                                "ecs:RegisterContainerInstance",
                                "ecs:StartTelemetrySession",
                                "ecs:Submit*",
                                "logs:CreateLogStream",
                                "logs:PutLogEvents",
                                "ecr:BatchCheckLayerAvailability",
                                "ecr:BatchGetImage",
                                "ecr:GetDownloadUrlForLayer",
                                "ecr:GetAuthorizationToken"
                            ],
                            "Resource": "*"
                        }]
                    }
    ECSInstanceProfile: 
        Type: AWS::IAM::InstanceProfile
        Properties:
            Path: /
            Roles: 
                - !Ref ECSRole

    InstanceSecurityGroup:
        Type: AWS::EC2::SecurityGroup
        Properties:
            GroupName: !Sub ${EnvironmentName}-SG
            GroupDescription: Allow SSH to EC2 instances from !Ref CIDR
            VpcId: !Ref VPC
            SecurityGroupIngress:
              - IpProtocol: tcp
                FromPort: 22
                ToPort: 22
                CidrIp: !Ref CIDR

Outputs:

    Cluster:
        Description: A reference to the ECS cluster
        Value: !Ref ECSCluster
```

## ecs-service.yaml

This CloudFormation template creates an NLB, and an ECS service and task definition for a TCP-based application.  In addition, the task definition has an IAM role assigned to it that enables it to Get and List objects in S3 buckets.  One other thing to note is that this service demonstrates the ability for ECS to assign dynamic ports to tasks (in bridge networking mode), then register those ports in a load balancing target group.  This allows you to run multiple copies of a task on a single EC2 instance, without having port conflicts, or needing to assign secondary IP addresses to the tasks.
* Note: in order to allow the traffic to the dynamic ports, you'll need to open ports 32768-61000 in the security group assigned to your ECS cluster instances.

```yaml
Description: >
    This template deploys a Network Load Balancer that acts as a front-end to the Parity service.
    It also deploys a long-running service that launches Parity nodes listening for JSON RPC requests.
    Created by Luke Youngblood, lukey@amazon.com
Parameters: 

    EnvironmentName:
        Description: An environment name that will be prefixed to resource names
        Type: String

    DockerImage:
        Description: The Docker image to pull from your container registry
        Type: String
        Default: "AWS account ID.dkr.ecr.us-east-1.amazonaws.com/parity:latest"

    VPC:
        Type: AWS::EC2::VPC::Id
        Description: Choose which VPC the Network Load Balancer should be deployed to

    Subnets:
        Description: Choose which subnets the Network Load Balancer should be deployed to
        Type: List<AWS::EC2::Subnet::Id>

    Cluster:
        Description: Please provide the ECS Cluster ID that this service should run on
        Type: String

    DesiredCount: 
        Description: How many instances of this task should we run across our cluster?
        Type: Number
        Default: 12

Resources:

    Service: 
        Type: AWS::ECS::Service
        DependsOn: Listener
        Properties: 
            Cluster: !Ref Cluster
            Role: !Ref ServiceRole
            DesiredCount: !Ref DesiredCount
            TaskDefinition: !Ref TaskDefinition
            
            LoadBalancers: 
                - ContainerName: "parity-service"
                  ContainerPort: 8545
                  TargetGroupArn: !Ref TargetGroup

    TaskDefinition:
        Type: AWS::ECS::TaskDefinition
        Properties:
            Family: parity-service
            TaskRoleArn: !Ref TaskRole
            NetworkMode: bridge
            ContainerDefinitions:
                - Name: parity-service
                  Essential: true
                  Image: !Ref DockerImage
                  Command: 
                    - "--tx-queue-mem-limit 0 --tx-queue-size 40000 --rpccorsdomain * --rpcaddr 0.0.0.0 --jsonrpc-hosts all --warp"
                  Environment:
                    - Name: "region"
                      Value: !Ref AWS::Region
                  Cpu: 256
                  MemoryReservation: 512
                  PortMappings:
                    - ContainerPort: 8545
                  LogConfiguration:
                    LogDriver: awslogs
                    Options:
                        awslogs-group: !Ref AWS::StackName
                        awslogs-region: !Ref AWS::Region
                        awslogs-stream-prefix: !Ref EnvironmentName

    CloudWatchLogsGroup:
        Type: AWS::Logs::LogGroup
        Properties: 
            LogGroupName: !Ref AWS::StackName
            RetentionInDays: 14

    LoadBalancer:
        Type: AWS::ElasticLoadBalancingV2::LoadBalancer
        Properties:
            Name: !Ref EnvironmentName
            Type: network
            Scheme: internal
            Subnets: !Ref Subnets
            Tags: 
                - Key: Name
                  Value: !Ref EnvironmentName

    TargetGroup:
        Type: AWS::ElasticLoadBalancingV2::TargetGroup
        DependsOn: LoadBalancer
        Properties:
            VpcId: !Ref VPC
            Port: 8545
            Protocol: TCP

    Listener:
        Type: AWS::ElasticLoadBalancingV2::Listener
        DependsOn: LoadBalancer
        Properties: 
            DefaultActions:
            - Type: forward
              TargetGroupArn: !Ref TargetGroup
            LoadBalancerArn: !Ref LoadBalancer
            Port: 8545
            Protocol: TCP

    # This IAM Role grants the task access to download files stored in S3
    TaskRole:
        Type: AWS::IAM::Role
        Properties:
            RoleName: !Sub ecs-task-${AWS::StackName}
            Path: /
            AssumeRolePolicyDocument: |
                {
                    "Statement": [{
                        "Sid": "",
                        "Effect": "Allow",
                        "Principal": { "Service": [ "ecs-tasks.amazonaws.com" ]},
                        "Action": "sts:AssumeRole"
                    }]
                }
            Policies: 
                - PolicyName: !Sub ecs-task-${AWS::StackName}
                  PolicyDocument: 
                    {
                        "Version": "2012-10-17",
                        "Statement": [{
                                "Effect": "Allow",
                                "Action": [
                                    "s3:Get*",
                                    "s3:List*"
                                ],
                                "Resource": "*"
                        }]
                    }

    # This IAM Role grants the service access to register/unregister with the 
    # Network Load Balancer (NLB). It is based on the default documented here:
    # http://docs.aws.amazon.com/AmazonECS/latest/developerguide/service_IAM_role.html
    ServiceRole: 
        Type: AWS::IAM::Role
        Properties: 
            RoleName: !Sub ecs-service-${AWS::StackName}
            Path: /
            AssumeRolePolicyDocument: |
                {
                    "Statement": [{
                        "Effect": "Allow",
                        "Principal": { "Service": [ "ecs.amazonaws.com" ]},
                        "Action": [ "sts:AssumeRole" ]
                    }]
                }
            Policies: 
                - PolicyName: !Sub ecs-service-${AWS::StackName}
                  PolicyDocument: 
                    {
                        "Version": "2012-10-17",
                        "Statement": [{
                                "Effect": "Allow",
                                "Action": [
                                    "ec2:AuthorizeSecurityGroupIngress",
                                    "ec2:Describe*",
                                    "elasticloadbalancing:DeregisterInstancesFromLoadBalancer",
                                    "elasticloadbalancing:Describe*",
                                    "elasticloadbalancing:RegisterInstancesWithLoadBalancer",
                                    "elasticloadbalancing:DeregisterTargets",
                                    "elasticloadbalancing:DescribeTargetGroups",
                                    "elasticloadbalancing:DescribeTargetHealth",
                                    "elasticloadbalancing:RegisterTargets"
                                ],
                                "Resource": "*"
                        }]
                    }
```

## ecs-scheduled-task.yaml

This CloudFormation template demonstrates how to create an ECS task definition with a IAM task role assigned to it, that gets executed on a schedule (in this case hourly) through CloudWatch events.  This allows you to execute maintenance tasks on a cron-like schedule.

```yaml
Description: >
    This template creates a single task definition in ECS that will launch the Parity updater container.
    The Parity updater container is designed to download the stored Ethereum blockchain from S3,
    synchronize it to the latest block on the chain, then cleanly exit the Parity process
    (after a 30 minute timer expires), and sync the delta blocks back up to the S3 saved copy.
    It is designed to be scheduled to run on a periodic basis through a CloudWatch event.
    Created by Luke Youngblood, lukey@amazon.com
Parameters: 

    EnvironmentName:
        Description: An environment name that will be prefixed to resource names
        Type: String

    DockerImage:
        Description: The Docker image to pull from your container registry
        Type: String
        Default: "AWS account ID.dkr.ecr.us-east-1.amazonaws.com/parity-updater:latest"

    Cluster:
        Description: Please provide the ECS Cluster ID that this service should run on
        Type: String

    Schedule:
        Description: The schedule expression that this task will run on
        Type: String
        Default: "rate(1 hour)"

Resources:

    CWEventRule:
        Type: AWS::Events::Rule
        Properties: 
            Description: CloudWatch Events Rule to launch the scheduled task definition
            Name: !Sub event-rule-${AWS::StackName}
            ScheduleExpression: !Ref Schedule
            State: ENABLED
            Targets:
                - Arn: !Sub arn:aws:ecs:${AWS::Region}:${AWS::AccountId}:cluster/${Cluster}
                  RoleArn: !GetAtt 
                    - EventRole
                    - Arn
                  Id: !Sub rule-target-${AWS::StackName}
                  EcsParameters:
                    TaskCount: 1
                    TaskDefinitionArn: !Ref TaskDefinition

    TaskDefinition:
        Type: AWS::ECS::TaskDefinition
        Properties:
            Family: parity-updater
            TaskRoleArn: !Ref TaskRole
            NetworkMode: bridge
            ContainerDefinitions:
                - Name: parity-updater
                  Essential: true
                  Image: !Ref DockerImage
                  Command: 
                    - "--tx-queue-mem-limit 0 --tx-queue-size 40000 --rpccorsdomain * --rpcaddr 0.0.0.0 --jsonrpc-hosts all --warp"
                  Environment:
                    - Name: "region"
                      Value: !Ref AWS::Region
                  Cpu: 512
                  MemoryReservation: 512
                  PortMappings:
                    - ContainerPort: 8545
                  LogConfiguration:
                    LogDriver: awslogs
                    Options:
                        awslogs-group: !Ref AWS::StackName
                        awslogs-region: !Ref AWS::Region
                        awslogs-stream-prefix: !Ref EnvironmentName

    CloudWatchLogsGroup:
        Type: AWS::Logs::LogGroup
        Properties: 
            LogGroupName: !Ref AWS::StackName
            RetentionInDays: 14

    # This IAM Role grants CloudWatch events the ability to run the task
    EventRole:
        Type: AWS::IAM::Role
        Properties:
            RoleName: !Sub run-task-${AWS::StackName}
            Path: /
            AssumeRolePolicyDocument: |
                {
                    "Statement": [{
                        "Sid": "",
                        "Effect": "Allow",
                        "Principal": { "Service": [ "events.amazonaws.com" ]},
                        "Action": "sts:AssumeRole"
                    }]
                }
            Policies: 
                - PolicyName: !Sub run-task-${AWS::StackName}
                  PolicyDocument: 
                    {
                        "Version": "2012-10-17",
                        "Statement": [{
                                "Effect": "Allow",
                                "Action": [
                                    "ecs:RunTask"
                                ],
                                "Resource": "*"
                        }]
                    }

    # This IAM Role grants the task access to update the blockchain saved in S3
    TaskRole:
        Type: AWS::IAM::Role
        Properties:
            RoleName: !Sub ecs-task-${AWS::StackName}
            Path: /
            AssumeRolePolicyDocument: |
                {
                    "Statement": [{
                        "Sid": "",
                        "Effect": "Allow",
                        "Principal": { "Service": [ "ecs-tasks.amazonaws.com" ]},
                        "Action": "sts:AssumeRole"
                    }]
                }
            Policies: 
                - PolicyName: !Sub ecs-task-${AWS::StackName}
                  PolicyDocument: 
                    {
                        "Version": "2012-10-17",
                        "Statement": [{
                                "Effect": "Allow",
                                "Action": [
                                    "s3:Get*",
                                    "s3:List*",
                                    "s3:PutObject*",
                                    "s3:DeleteObject*"
                                ],
                                "Resource": "*"
                        }]
                    }
```