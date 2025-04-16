# Bảng Tóm Tắt AWS CloudFormation

Kho lưu trữ này chứa một **Bảng Tóm Tắt AWS CloudFormation** ngắn gọn và thực tế, giúp bạn nhanh chóng hiểu và làm việc với các mẫu CloudFormation. Dù bạn là người mới bắt đầu hay cần tham khảo nhanh, bảng tóm tắt này bao gồm các yếu tố cơ bản của CloudFormation, như cấu trúc mẫu, các hàm nội tại, tài nguyên phổ biến và các lệnh AWS CLI hữu ích.

---

## Mục Lục

- [Giới Thiệu](#giới-thiệu)
- [Cấu Trúc Mẫu](#cấu-trúc-mẫu)
- [Các Hàm Nội Tại](#các-hàm-nội-tại)
- [Các Tài Nguyên Phổ Biến](#các-tài-nguyên-phổ-biến)
  - [VPC và Mạng](#vpc-và-mạng)
  - [EC2](#ec2)
  - [Application Load Balancer (ALB)](#application-load-balancer-alb)
  - [S3](#s3)
- [Tham Số và Ánh Xạ](#tham-số-và-ánh-xạ)
- [Điều Kiện](#điều-kiện)
- [Đầu Ra](#đầu-ra)
- [Mẹo và Lưu Ý](#mẹo-và-lưu-ý)
- [Các Lệnh AWS CLI Hữu Ích](#các-lệnh-aws-cli-hữu-ích)
- [Tài Nguyên Tham Khảo](#tài-nguyên-tham-khảo)

---

## Giới Thiệu

AWS CloudFormation là một dịch vụ Cơ Sở Hạ Tầng dưới dạng Mã (Infrastructure as Code - IaC) cho phép bạn định nghĩa và quản lý các tài nguyên AWS bằng các mẫu JSON hoặc YAML. Bảng tóm tắt này cung cấp các tham chiếu nhanh để viết mẫu, bao gồm cú pháp, các tài nguyên phổ biến và các phương pháp hay nhất.

---

## Cấu Trúc Mẫu

Một mẫu CloudFormation (viết bằng JSON hoặc YAML) bao gồm các phần sau:

- **`AWSTemplateFormatVersion`**: Phiên bản định dạng mẫu (thường là `2010-09-09`).
- **`Description`**: Mô tả ngắn gọn về mẫu.
- **`Parameters`**: Tham số đầu vào từ người dùng.
- **`Mappings`**: Ánh xạ giá trị (ví dụ: chọn giá trị dựa trên vùng hoặc môi trường).
- **`Conditions`**: Logic điều kiện để tạo tài nguyên.
- **`Resources`**: Các tài nguyên AWS cần tạo (phần bắt buộc).
- **`Outputs`**: Giá trị trả về sau khi tạo stack (ví dụ: URL, ID tài nguyên).

### Ví dụ (YAML)

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Một mẫu CloudFormation đơn giản
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

## Các Hàm Nội Tại

| **Hàm**      | **Mô Tả**                                 | **Ví Dụ**                                                  |
| ------------ | ----------------------------------------- | ---------------------------------------------------------- |
| `!Ref`       | Tham chiếu đến tài nguyên hoặc tham số.   | `!Ref MyEC2Instance`                                       |
| `!GetAtt`    | Lấy thuộc tính của tài nguyên.            | `!GetAtt MyEC2Instance.PublicIp`                           |
| `!Sub`       | Thay thế biến trong chuỗi.                | `!Sub "http://${MyEC2Instance.PublicDnsName}"`             |
| `!Join`      | Nối các giá trị thành chuỗi.              | `!Join [":", ["a", "b"]]` → `a:b`                          |
| `!Select`    | Chọn một phần tử từ danh sách.            | `!Select [0, ["us-east-1a", "us-east-1b"]]` → `us-east-1a` |
| `!FindInMap` | Tìm giá trị trong ánh xạ.                 | `!FindInMap [RegionMap, !Ref "AWS::Region", AMI]`          |
| `!GetAZs`    | Lấy danh sách Availability Zones.         | `!GetAZs ""` → `["us-east-1a", "us-east-1b", ...]`         |
| `!Base64`    | Mã hóa chuỗi thành Base64 (cho UserData). | `UserData: !Base64 "yum update -y"`                        |

---

## Các Tài Nguyên Phổ Biến

### VPC và Mạng

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
      GroupDescription: Cho phép HTTP và SSH
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

## Tham Số và Ánh Xạ

### Tham Số

- Định nghĩa tham số đầu vào:

  ```yaml
  Parameters:
    InstanceType:
      Type: String
      Default: t2.micro
      AllowedValues: [t2.micro, t3.micro]
  ```

- Sử dụng tham số:

  ```yaml
  InstanceType: !Ref InstanceType
  ```

### Ánh Xạ

- Định nghĩa ánh xạ (ví dụ: chọn AMI theo vùng):

  ```yaml
  Mappings:
    RegionMap:
      us-east-1:
        AMI: ami-12345678
      us-west-2:
        AMI: ami-87654321
  ```

- Sử dụng ánh xạ:

  ```yaml
  ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", AMI]
  ```

---

## Điều Kiện

- Định nghĩa điều kiện:

  ```yaml
  Conditions:
    CreateProdResources: !Equals [!Ref Environment, "prod"]
  ```

- Sử dụng điều kiện trong tài nguyên:

  ```yaml
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Condition: CreateProdResources
    Properties:
      InstanceType: t2.micro
      ImageId: ami-12345678
  ```

---

## Đầu Ra

- Xuất giá trị sau khi tạo stack:

  ```yaml
  Outputs:
    WebsiteURL:
      Description: URL của máy chủ web
      Value: !Sub "http://${EC2Instance.PublicDnsName}"
  ```

---

## Mẹo và Lưu Ý

- **Tham Số Giả (Pseudo Parameters)**:

  - `AWS::Region`: Vùng hiện tại (ví dụ: `us-east-1`).
  - `AWS::AccountId`: ID tài khoản AWS.
  - `AWS::StackName`: Tên stack.
  - Ví dụ: `!Sub "arn:aws:s3:::${AWS::AccountId}-bucket"`.

- **Nested Stacks**: Chia nhỏ các mẫu phức tạp:

  ```yaml
  NestedStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/nested-template.yaml
  ```

- **Chi Phí**: CloudFormation miễn phí, nhưng bạn phải trả phí cho các tài nguyên được tạo.
- **Gỡ Lỗi**: Nếu stack thất bại, kiểm tra tab **Events** trong CloudFormation Console để xem chi tiết lỗi.

---

## Các Lệnh AWS CLI Hữu Ích

- Tạo stack:

  ```bash
  aws cloudformation create-stack --stack-name MyStack --template-body file://template.yaml --parameters ParameterKey=KeyName,ParameterValue=my-key-pair
  ```

- Cập nhật stack:

  ```bash
  aws cloudformation update-stack --stack-name MyStack --template-body file://template.yaml
  ```

- Xóa stack:

  ```bash
  aws cloudformation delete-stack --stack-name MyStack
  ```

- Kiểm tra trạng thái stack:

  ```bash
  aws cloudformation describe-stacks --stack-name MyStack
  ```

---

## Tài Nguyên Tham Khảo

- [Hướng Dẫn Sử Dụng AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
- [Danh Sách Loại Tài Nguyên AWS](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)
- [Mẫu CloudFormation Mẫu của AWS](https://aws.amazon.com/cloudformation/resources/templates/)

---

## Đóng Góp

Bạn có thể đóng góp cho bảng tóm tắt này bằng cách gửi pull request hoặc mở issue với các đề xuất hoặc cải tiến.

## Giấy Phép

Dự án này được cấp phép theo Giấy Phép MIT. Xem file [LICENSE](LICENSE) để biết thêm chi tiết.
