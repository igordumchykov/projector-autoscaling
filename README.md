# projector-autoscaling

Consists of Cloudformation template that is used for provisioning:
- VPC
- 2 Subnets
- Internet Gateway
- Route table
- Autoscaling group with 2 EC2 instances
- Load Balancer in front of EC2 instances
- Scaling policies for scaling up EC2 instances based on AVG CPU usage and number of requests during particular period of time

# Template
```yaml
AWSTemplateFormatVersion: 2010-09-09

Resources:
  #Creating the VPC
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: yes
      EnableDnsHostnames: yes

  #Creaating Public Subnet 1
  PublicSubnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: eu-central-1a
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: yes

  #Creating Public Subnet 2
  PublicSubnet2:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: eu-central-1b
      CidrBlock: 10.0.2.0/24
      MapPublicIpOnLaunch: yes

  #Creating Internet Gateway
  InternetGateway:
    Type: 'AWS::EC2::InternetGateway'

  #Attaching the ITG to the VPC
  InternetGatewayAttachment:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  #Creating a Public Route Table
  PublicRouteTable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId: !Ref VPC

  #Configuring Public Route
  PublicRoute:
    Type: 'AWS::EC2::Route'
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  #Associating Subnet 1 and Route Table
  PublicSubnet1RouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1

  #Associating Subnet 2 and Route Table
  PublicSubnet2RouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2

  #Creating Security Group
  InstanceSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupName: MyInstanceSecurityGroup
      GroupDescription: Enable SSH access and HTTP
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  #Creating launch template that will install Apache upon any EC2 Instance startup
  MyLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: MyLaunchTemplate
      LaunchTemplateData:
        ImageId: ami-088e71edb8795252f
        InstanceType: t2.micro
        KeyName: web-server-key-pair
        SecurityGroupIds:
          - !Ref InstanceSecurityGroup
        InstanceMarketOptions:
          MarketType: spot
          SpotOptions:
            SpotInstanceType: one-time
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            yum update -y 
            yum install -y httpd 
            systemctl start httpd 
            systemctl enable httpd

  #Creating autoscaling group with desired minimum and maximum size
  MyASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      LaunchTemplate:
        LaunchTemplateId: !Ref MyLaunchTemplate
        Version: !GetAtt MyLaunchTemplate.LatestVersionNumber
      MinSize: 2
      MaxSize: 5
      DesiredCapacity: 2
      TargetGroupARNs:
        - !Ref ALBTargetGroup
      VPCZoneIdentifier:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2

  #Creating an Application Load Balancer
  ApplicationLoadBalancer :
    Type : AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties :
      SecurityGroups:
        - !Ref InstanceSecurityGroup
      Subnets :
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2

  #Configuring Application Load Balancer Target Group
  ALBTargetGroup :
    Type : AWS::ElasticLoadBalancingV2::TargetGroup
    Properties :
      HealthCheckIntervalSeconds : '30'
      HealthCheckTimeoutSeconds : '5'
      Port : '80'
      Protocol : HTTP
      VpcId: !Ref VPC

  #Scaling Policy For AVG CPU Usage
  ScalingPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref MyASG
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        TargetValue: 60.0
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization

  #Scaling Policy For Scaling Up EC2 Instances
  WebServerScaleUpPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AdjustmentType: ChangeInCapacity
      AutoScalingGroupName: !Ref MyASG
      ScalingAdjustment: 1

  #Alarm That Sends Notification to WebServerScaleUpPolicy to Scale Up EC2 Instances
  #When Number of Requests is More Then 100 during 5 min
  RequestsThresholdAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: requests-exceed-100-in-5-min-alarm
      ComparisonOperator: GreaterThanOrEqualToThreshold
      EvaluationPeriods: 5
      MetricName: RequestCountPerTarget
      Namespace: AWS/ApplicationELB
      Period: 60
      Statistic: Sum
      Threshold: 100
      AlarmActions:
        - !Ref WebServerScaleUpPolicy
```
# Run
```shell
aws cloudformation create-stack --stack-name AutoscalingStack --template-body file:///<path-to-file>/template.yml
```

# Clean Up Resources
```shell
 aws cloudformation delete-stack --stack-name AutoscalingStack
```
