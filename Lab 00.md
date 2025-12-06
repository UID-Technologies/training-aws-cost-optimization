
## 1. How to Deploy This Environment

You’ll use this template as the **starting point for Module 7**.

### Step 1 – Save the Template

1. Copy everything inside the YAML code block below into a file on your PC, e.g.
   `cost-optimization-lab-env.yaml`

### Step 2 – Create the Stack in CloudFormation

1. Log in to AWS Console (region: **us-east-1**).
2. Go to **CloudFormation**.
3. Click **Create stack → With new resources (standard)**.
4. Choose **Upload a template file**.
5. Upload `cost-optimization-lab-env.yaml`.
6. Click **Next**.
7. Enter:

   * **Stack name**: `cost-optimization-lab`
   * **EnvironmentName**: `cost-lab` (or your preferred short name)
8. Leave other parameters as default for now.
9. Click **Next** → **Next**.
10. On the final page:

    * Check the IAM capabilities acknowledgment.
    * Click **Create stack**.

Wait until stack status is **CREATE_COMPLETE**.

### Step 3 – Validate Key Resources

Once created, you should see:

* **VPC** with:

  * 2 public subnets
  * 2 private subnets
  * 1 NAT Gateway
  * VPC Flow Logs enabled (to CloudWatch Logs)
* **EC2 Auto Scaling Group** with 2× `t3.micro` instances behind an **ALB**
* **ECS Cluster** with an EC2-based service (overprovisioned CPU/memory)
* **EKS Cluster** with a small managed node group
* **Lambda function** with **1024 MB** memory
* **Two S3 buckets**:

  * Logs bucket with no lifecycle
  * Data bucket with dummy objects
* **CloudWatch log groups** with “Never expire” retention

That’s your **“bad” starting point** for the case study.

---

## 2. CloudFormation Template (Trainer Version, Low-Cost “Expensive” Env)

> This is a **starter template** designed to be readable and tweakable.
> It focuses on the core things you care about: VPC + NAT + ALB + ASG + EKS + ECS + Lambda + S3 + Logs.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: >
  Cost Optimization Lab - Prebuilt "Expensive" Environment (Low-Cost Trainer Version).
  Includes: VPC, NAT, ALB, ASG, ECS Cluster, minimal EKS Cluster (1 node), Lambda, S3 buckets, CloudWatch logs & VPC Flow Logs.

Parameters:
  EnvironmentName:
    Type: String
    Default: cost-lab
    Description: Short name used as prefix for resource naming.

  VpcCidr:
    Type: String
    Default: 10.10.0.0/16

  PublicSubnet1Cidr:
    Type: String
    Default: 10.10.1.0/24

  PublicSubnet2Cidr:
    Type: String
    Default: 10.10.2.0/24

  PrivateSubnet1Cidr:
    Type: String
    Default: 10.10.11.0/24

  PrivateSubnet2Cidr:
    Type: String
    Default: 10.10.12.0/24

  EC2InstanceType:
    Type: String
    Default: t3.micro
    Description: Instance type for ASG and ECS container instances.

Resources:
  ########################
  # Networking - VPC
  ########################
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-vpc"

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-igw"

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnet1Cidr
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select [0, !GetAZs ""]
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-public-az1"

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnet2Cidr
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select [1, !GetAZs ""]
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-public-az2"

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PrivateSubnet1Cidr
      MapPublicIpOnLaunch: false
      AvailabilityZone: !Select [0, !GetAZs ""]
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-private-az1"

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PrivateSubnet2Cidr
      MapPublicIpOnLaunch: false
      AvailabilityZone: !Select [1, !GetAZs ""]
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-private-az2"

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-public-rt"

  PublicRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable

  NatEip:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc

  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEip.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-nat"

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-private-rt"

  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRouteTable

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateRouteTable

  ########################
  # VPC Flow Logs
  ########################
  FlowLogsLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub "/vpc/flowlogs/${EnvironmentName}"
      RetentionInDays: 365  # intentionally high; students will optimize later

  FlowLogsRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: vpc-flow-logs.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs

  VPCFlowLog:
    Type: AWS::EC2::FlowLog
    Properties:
      ResourceId: !Ref VPC
      ResourceType: VPC
      TrafficType: ALL
      LogDestinationType: cloud-watch-logs
      LogGroupName: !Ref FlowLogsLogGroup
      DeliverLogsPermissionArn: !GetAtt FlowLogsRole.Arn

  ########################
  # Security Groups
  ########################
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB SG
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-alb-sg"

  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: EC2/App SG
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-ec2-sg"

  ########################
  # S3 Buckets (Logs & Data)
  ########################
  LogsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${EnvironmentName}-logs-${AWS::AccountId}-${AWS::Region}"
      VersioningConfiguration:
        Status: Enabled
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName

  DataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${EnvironmentName}-data-${AWS::AccountId}-${AWS::Region}"
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName

  ########################
  # IAM Roles
  ########################
  InstanceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
        - arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-ec2-role"

  InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref InstanceRole

  ########################
  # Launch Template + ASG
  ########################
  AppLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Sub "${EnvironmentName}-app-lt"
      LaunchTemplateData:
        InstanceType: !Ref EC2InstanceType
        IamInstanceProfile:
          Arn: !GetAtt InstanceProfile.Arn
        ImageId: !Ref AWS::NoValue  # uses default AMI via AutoScaling (or you can hardcode)
        NetworkInterfaces:
          - AssociatePublicIpAddress: false
            DeviceIndex: 0
            Groups:
              - !Ref EC2SecurityGroup
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            yum update -y
            yum install -y httpd stress-ng
            systemctl enable httpd
            systemctl start httpd
            echo "Cost Lab Demo App - High Cost Pattern" > /var/www/html/index.html
            # generate fake CPU load every minute (crude)
            (while true; do stress-ng --cpu 1 --timeout 60; sleep 60; done) &

  AppAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
      LaunchTemplate:
        LaunchTemplateId: !Ref AppLaunchTemplate
        Version: !GetAtt AppLaunchTemplate.LatestVersionNumber
      MinSize: "2"
      MaxSize: "4"
      DesiredCapacity: "2"
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-app-asg"
          PropagateAtLaunch: true

  ########################
  # Application Load Balancer
  ########################
  AppTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub "${EnvironmentName}-tg"
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VPC
      TargetType: instance
      HealthCheckPath: /
      Matcher:
        HttpCode: 200

  AppALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub "${EnvironmentName}-alb"
      Scheme: internet-facing
      Type: application
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref ALBSecurityGroup

  AppALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref AppALB
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref AppTargetGroup

  ASGToTGAttachment:
    Type: AWS::AutoScaling::LifecycleHook
    Condition: "AlwaysFalse"
    Properties:
      AutoScalingGroupName: !Ref AppAutoScalingGroup
      LifecycleTransition: autoscaling:EC2_INSTANCE_LAUNCHING
      LifecycleHookName: Dummy
  # NOTE: Instances will register themselves if you add appropriate user-data or you can attach via console.
  # To keep template simple, you can associate targets manually during the lab.

  ########################
  # Lambda - Overprovisioned
  ########################
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  SampleLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub "${EnvironmentName}-expensive-lambda"
      Handler: index.handler
      Runtime: python3.11
      Role: !GetAtt LambdaExecutionRole.Arn
      Timeout: 30
      MemorySize: 1024  # intentionally high
      Code:
        ZipFile: |
          import time
          def handler(event, context):
              # simulate some CPU work
              total = 0
              for i in range(5_000_000):
                  total += i
              time.sleep(1)
              return {"status": "ok", "total": total}

  ########################
  # EKS - Minimal Cluster + Nodegroup
  ########################
  EKSClusterRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: eks.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

  EKSCluster:
    Type: AWS::EKS::Cluster
    Properties:
      Name: !Sub "${EnvironmentName}-eks"
      RoleArn: !GetAtt EKSClusterRole.Arn
      Version: "1.29"
      ResourcesVpcConfig:
        SubnetIds:
          - !Ref PrivateSubnet1
          - !Ref PrivateSubnet2
        SecurityGroupIds:
          - !Ref EC2SecurityGroup

  EKSNodeRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-eks-node-role"

  EKSNodeInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref EKSNodeRole

  EKSNodegroup:
    Type: AWS::EKS::Nodegroup
    Properties:
      ClusterName: !Ref EKSCluster
      NodeRole: !GetAtt EKSNodeRole.Arn
      Subnets:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
      ScalingConfig:
        MinSize: 1
        MaxSize: 3
        DesiredSize: 1
      InstanceTypes:
        - t3.small

Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC

  ALBURL:
    Description: Public URL of the Application Load Balancer
    Value: !Sub "http://${AppALB.DNSName}"

  LogsBucketName:
    Description: Logs bucket - intentionally no lifecycle rule
    Value: !Ref LogsBucket

  DataBucketName:
    Description: Data bucket
    Value: !Ref DataBucket

  LambdaName:
    Description: Overprovisioned Lambda function
    Value: !Ref SampleLambda

  EKSClusterName:
    Description: EKS Cluster Name
    Value: !Ref EKSCluster
```

---

## 3. How This Template Aligns With Module 7 Labs

You now have a **single deployable stack** that:

* Looks like an expensive environment
* Actually runs on low-cost resources
* Seeds all the **anti-patterns** your labs will fix:

Where to point students:

* **Lab 1 – Cost Explorer & VPC Flow Logs**

  * VPCFlowLog + FlowLogsLogGroup
  * NAT Gateway, ALB, ASG, EC2 metrics

* **Lab 2 – S3 + Logs optimization**

  * `LogsBucket` and `DataBucket` (no lifecycle rules)
  * CloudWatch log group retention

* **Lab 3 – Compute optimization**

  * `AppAutoScalingGroup` (overprovisioned)
  * EC2 instances behind `AppALB`
  * `SampleLambda` with 1024MB memory
  * `EKSCluster` + `EKSNodegroup` for EKS labs

* **Lab 5 – Networking optimization**

  * NAT Gateway + VPC endpoints you’ll add later

