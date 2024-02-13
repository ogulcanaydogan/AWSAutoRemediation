AWSTemplateFormatVersion: 2010-09-09
Description: |
  This YAML is resolving AWS Auto Remediation Project

Parameters:
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Default: ziya
  IAMRole:
    Type: String
    Default: arn:aws:iam::621627302500:role/LambdaAutoScaling

Resources:
# ------------------------------------------
# ----------- VPC CONFIGURATIONS -----------
# ------------------------------------------

  # Create a VPC
  ProjVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags: 
        - Key: 'Name'
          Value: 'Proj-VPC'

  # Create an Internet Gateway
  ProjIGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags: 
        - Key: 'Name'
          Value: 'Proj-InternetGW'

  # Attach Internet Gateway to VPC
  AttachGateway:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref ProjVPC
      InternetGatewayId: !Ref ProjIGW

  # Create a Public Subnet under us-east-1a
  Public1a:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1a
      CidrBlock: 10.0.1.0/24
      VpcId: !Ref ProjVPC
      MapPublicIpOnLaunch: true
      Tags: 
        - Key: 'Name'
          Value: 'Public1a'

  # Create a Public Subnet under us-east-1b
  Public1b:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1b
      CidrBlock: 10.0.2.0/24
      VpcId: !Ref ProjVPC
      MapPublicIpOnLaunch: true
      Tags: 
        - Key: 'Name'
          Value: 'Public1b'

  # Create a Public Route table
  PublicRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref ProjVPC
      Tags: 
        - Key: 'Name'
          Value: 'PublicRT'

  # New Route for Public subnets to access Internet Gateway
  IGWRoute:
    Type: 'AWS::EC2::Route'
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRT
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId: !Ref ProjIGW

  # Associate Public Subnet under Public Route Table
  PublicRTASS1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRT
      SubnetId: !Ref Public1a

# ------------------------------------------
# ------ SECURITY GROUP CONFIGURATIONS -----
# ------------------------------------------

  # Create Security Group for Auto Scaling Group
  EC2SecGrp:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: 22 - 80 - 443 Ports
      GroupName: EC2SecGrp
      SecurityGroupIngress: 
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
      VpcId: !Ref ProjVPC
      Tags: 
        - Key: 'Name'
          Value: 'EC2SecGrp'

# ------------------------------------------
# ----- LAUNCH TEMPLATE CONFIGURATIONS -----
# ------------------------------------------

  # Create LaunchTemplate for PROD. AutoScaling
  LTPROD:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: ami-0cff7528ff583bf9a
        InstanceType: t2.micro
        KeyName: !Ref KeyName
        SecurityGroupIds:
          - !Ref EC2SecGrp
        TagSpecifications:
          - ResourceType: instance
            Tags: 
              - Key: Name
                Value: 'PROD'
      LaunchTemplateName: LTPROD
      VersionDescription: 'v1'

  # Create LaunchTemplate for DEV. AutoScaling
  LTDEV:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: ami-0cff7528ff583bf9a
        InstanceType: t2.micro
        KeyName: !Ref KeyName
        SecurityGroupIds:
          - !Ref EC2SecGrp
        TagSpecifications:
          - ResourceType: instance
            Tags: 
              - Key: Name
                Value: 'DEV'
      LaunchTemplateName: LTDEV
      VersionDescription: 'v1'

# ------------------------------------------
# ------- AUTO SCALING CONFIGURATIONS ------
# ------------------------------------------

  # Create AutoScaling Group
  AutoScalingPROD:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      DesiredCapacity: "2"
      HealthCheckGracePeriod: 90
      HealthCheckType: EC2
      LaunchTemplate:
        LaunchTemplateId: !Ref LTPROD
        Version: !GetAtt LTPROD.LatestVersionNumber
      MaxSize: "3"
      MinSize: "2"
      VPCZoneIdentifier: [!Ref Public1a]
      MetricsCollection:
        - Granularity: "1Minute"
      Tags: 
        - Key: 'Name'
          Value: 'AutoScalingPROD'
          PropagateAtLaunch: true

  # Create AutoScaling Group
  AutoScalingDEV:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      DesiredCapacity: "2"
      HealthCheckGracePeriod: 90
      HealthCheckType: EC2
      LaunchTemplate:
        LaunchTemplateId: !Ref LTDEV
        Version: !GetAtt LTDEV.LatestVersionNumber
      MaxSize: "3"
      MinSize: "2"
      VPCZoneIdentifier: [!Ref Public1a]
      MetricsCollection:
        - Granularity: "1Minute"
      Tags: 
        - Key: 'Name'
          Value: 'AutoScalingDEV'
          PropagateAtLaunch: true

# ------------------------------------------
# -------- SNS&ALARM CONFIGURATIONS --------
# ------------------------------------------

  # Create SNS Topic for Email Notification
  SNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      DisplayName: AutoScalingSNS
      FifoTopic: false
      Subscription: 
        - Protocol: email
          Endpoint: yusuf.asici@gmail.com
      Tags: 
        - Key: Name
          Value: AutoScalingSNS

  # Create Alarm for PROD AutoScaling DesiredCapacity
  DesiredCapacityforPROD:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: "DesiredCapacity Change Alarm for PROD"
      Namespace: AWS/AutoScaling
      MetricName: GroupDesiredCapacity
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref AutoScalingPROD
      Period: 60
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: LessThanOrEqualToThreshold
      Statistic: Maximum
      AlarmActions:
        - !Ref SNSTopic
      AlarmName: "DesiredCapacityforPROD" 

  # Create Alarm for DEV AutoScaling DesiredCapacity
  DesiredCapacityforDEV:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: "DesiredCapacity Change Alarm for DEV"
      Namespace: AWS/AutoScaling
      MetricName: GroupDesiredCapacity
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref AutoScalingDEV
      Period: 60
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: LessThanOrEqualToThreshold
      Statistic: Maximum
      AlarmActions:
        - !Ref SNSTopic
      AlarmName: "DesiredCapacityforDEV" 

# ------------------------------------------
# --------- LAMBDA CONFIGURATIONS ----------
# ------------------------------------------

  # Create Lambda Function
  LabmdaAutoScaling:
    Type: AWS::Lambda::Function
    Properties:
      Handler: index.lambda_handler
      Role: !Ref IAMRole
      Code:
        ZipFile: |
          import boto3
          def update_auto_scaling_group_desired_capacity(auto_scaling_group_name, desired_capacity):
              autoscaling_client = boto3.client('autoscaling')
              
              response = autoscaling_client.update_auto_scaling_group(
                  AutoScalingGroupName=auto_scaling_group_name,
                  DesiredCapacity=desired_capacity
              )
              
              print("Desired capacity updated successfully for Auto Scaling group:", auto_scaling_group_name)

          def lambda_handler(event, context):
              autoscaling_client = boto3.client('autoscaling')
              
              try:
                  # Retrieve information about all Auto Scaling groups
                  response = autoscaling_client.describe_auto_scaling_groups()
                  auto_scaling_group_names = [group['AutoScalingGroupName'] for group in response['AutoScalingGroups']]
                  
                  # Update desired capacity for each Auto Scaling group
                  for auto_scaling_group_name in auto_scaling_group_names:
                      update_auto_scaling_group_desired_capacity(auto_scaling_group_name, 2)
                  
                  return {
                      'statusCode': 200,
                      'body': 'Desired capacity updated successfully for all Auto Scaling groups'
                  }
              except Exception as e:
                  return {
                      'statusCode': 500,
                      'body': f'Error: {str(e)}'
                  }
      Runtime: python3.12

  # Add SNS as a TRIGGER to LAmbda Function
  SNSTopicPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !GetAtt LabmdaAutoScaling.Arn
      Principal: sns.amazonaws.com
      SourceArn: !Ref SNSTopic

  # Lambda Function Subscription to the SNS
  LambdaSNSSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: 'lambda'
      Endpoint: !GetAtt LabmdaAutoScaling.Arn
      TopicArn: !Ref SNSTopic
