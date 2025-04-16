# AWS CloudFormation Cheatsheet

This repository contains a concise and practical **AWS CloudFormation Cheatsheet** to help you quickly understand and work with CloudFormation templates. Whether you're a beginner or need a quick reference, this cheatsheet covers the essentials of CloudFormation, including template structure, intrinsic functions, common resources, and useful AWS CLI commands.

---

## Table of Contents

- [Introduction](#introduction)
- [Template Structure](#template-structure)
- [Intrinsic Functions](#intrinsic-functions)
- [Common Resources](#common-resources)
  - [VPC and Network](#vpc-and-network)
  - [EC2](#ec2)
  - [Application Load Balancer (ALB)](#application-load-balancer-alb)
  - [S3](#s3)
- [Parameters and Mappings](#parameters-and-mappings)
- [Conditions](#conditions)
- [Outputs](#outputs)
- [Tips and Notes](#tips-and-notes)
- [Useful AWS CLI Commands](#useful-aws-cli-commands)
- [Additional Resources](#additional-resources)

---

## Introduction

AWS CloudFormation is an Infrastructure as Code (IaC) service that allows you to define and manage AWS resources using JSON or YAML templates. This cheatsheet provides quick references for writing templates, including syntax, common resources, and best practices.

---

## Template Structure

A CloudFormation template (written in JSON or YAML) consists of the following sections:

- **`AWSTemplateFormatVersion`**: Template format version (typically `2010-09-09`).
- **`Description`**: A brief description of the template.
- **`Parameters`**: Input parameters from the user.
- **`Mappings`**: Key-value mappings (e.g., for selecting values based on region or environment).
- **`Conditions`**: Conditional logic for creating resources.
- **`Resources`**: AWS resources to create (required section).
- **`Outputs`**: Values returned after stack creation (e.g., URLs, resource IDs).

### Example (YAML)

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: A simple CloudFormation template
Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: ami-12345678
Outputs:
  InstanceId:
    Value: !Ref MyEC2Instance
```

---

## Intrinsic Functions

| **Function** | **Description**                            | **Example**                                                |
| ------------ | ------------------------------------------ | ---------------------------------------------------------- |
| `!Ref`       | References a resource or parameter.        | `!Ref MyEC2Instance`                                       |
| `!GetAtt`    | Retrieves an attribute of a resource.      | `!GetAtt MyEC2Instance.PublicIp`                           |
| `!Sub`       | Substitutes variables in a string.         | `!Sub "http://${MyEC2Instance.PublicDnsName}"`             |
| `!Join`      | Joins values into a string.                | `!Join [":", ["a", "b"]]` → `a:b`                          |
| `!Select`    | Selects an element from a list.            | `!Select [0, ["us-east-1a", "us-east-1b"]]` → `us-east-1a` |
| `!FindInMap` | Looks up a value in a mapping.             | `!FindInMap [RegionMap, !Ref "AWS::Region", AMI]`          |
| `!GetAZs`    | Gets a list of Availability Zones.         | `!GetAZs ""` → `["us-east-1a", "us-east-1b", ...]`         |
| `!Base64`    | Encodes a string to Base64 (for UserData). | `UserData: !Base64 "yum update -y"`                        |

---

## Common Resources

### VPC and Network

- **VPC**:

  ```yaml
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
  ```

- **Subnet**:

  ```yaml
  Subnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: true
  ```

- **Internet Gateway**:

  ```yaml
  InternetGateway:
    Type: AWS::EC2::InternetGateway
  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
  ```

- **Route Table**:

  ```yaml
  RouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
  Route:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref RouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  ```

- **NAT Gateway**:

  ```yaml
  ElasticIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
  NATGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt ElasticIP.AllocationId
      SubnetId: !Ref PublicSubnet
  ```

### EC2

- **EC2 Instance**:

  ```yaml
  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-12345678
      InstanceType: t2.micro
      SubnetId: !Ref Subnet
      SecurityGroupIds:
        - !Ref SecurityGroup
      KeyName: my-key-pair
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
  ```

- **Security Group**:

  ```yaml
  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP and SSH
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
  ```

### Application Load Balancer (ALB)

- **ALB**:

  ```yaml
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref SecurityGroup
  ```

- **Target Group**:

  ```yaml
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VPC
      Targets:
        - Id: !Ref EC2Instance
          Port: 80
  ```

- **Listener**:

  ```yaml
  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup
  ```

### S3

- **S3 Bucket**:

  ```yaml
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-unique-bucket-name
      AccessControl: Private
  ```

---

## Parameters and Mappings

### Parameters

- Define input parameters:

  ```yaml
  Parameters:
    InstanceType:
      Type: String
      Default: t2.micro
      AllowedValues: [t2.micro, t3.micro]
  ```

- Use a parameter:

  ```yaml
  InstanceType: !Ref InstanceType
  ```

### Mappings

- Define a mapping (e.g., selecting AMI by region):

  ```yaml
  Mappings:
    RegionMap:
      us-east-1:
        AMI: ami-12345678
      us-west-2:
        AMI: ami-87654321
  ```

- Use the mapping:

  ```yaml
  ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", AMI]
  ```

---

## Conditions

- Define a condition:

  ```yaml
  Conditions:
    CreateProdResources: !Equals [!Ref Environment, "prod"]
  ```

- Use the condition in a resource:

  ```yaml
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Condition: CreateProdResources
    Properties:
      InstanceType: t2.micro
      ImageId: ami-12345678
  ```

---

## Outputs

- Export values after stack creation:

  ```yaml
  Outputs:
    WebsiteURL:
      Description: URL of the web server
      Value: !Sub "http://${EC2Instance.PublicDnsName}"
  ```

---

## Tips and Notes

- **Pseudo Parameters**:

  - `AWS::Region`: Current region (e.g., `us-east-1`).
  - `AWS::AccountId`: AWS account ID.
  - `AWS::StackName`: Stack name.
  - Example: `!Sub "arn:aws:s3:::${AWS::AccountId}-bucket"`.

- **Nested Stacks**: Split complex templates into smaller ones:

  ```yaml
  NestedStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/nested-template.yaml
  ```

- **Cost**: CloudFormation is free, but you pay for the resources created.
- **Debugging**: If a stack fails, check the **Events** tab in the CloudFormation Console for error details.

---

## Useful AWS CLI Commands

- Create a stack:

  ```bash
  aws cloudformation create-stack --stack-name MyStack --template-body file://template.yaml --parameters ParameterKey=KeyName,ParameterValue=my-key-pair
  ```

- Update a stack:

  ```bash
  aws cloudformation update-stack --stack-name MyStack --template-body file://template.yaml
  ```

- Delete a stack:

  ```bash
  aws cloudformation delete-stack --stack-name MyStack
  ```

- Check stack status:

  ```bash
  aws cloudformation describe-stacks --stack-name MyStack
  ```

---

## Additional Resources

- [AWS CloudFormation User Guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
- [AWS Resource Types](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)
- [AWS Sample Templates](https://aws.amazon.com/cloudformation/resources/templates/)

---

## Contributing

Feel free to contribute to this cheatsheet by submitting a pull request or opening an issue with suggestions or improvements.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
