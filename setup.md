# AWS Infrastructure Plan

## 1. Overall Architecture

Region：

```text
us-east-1
```

VPC：

```text
VPC
│
├── Public Subnet
│   └── Management EC2
│       └── Public IP / Elastic IP
│
└── Private Subnet
    ├── Lambda
    ├── Step Functions
    └── other private resources
│
├── S3
└── Bedrock
```

核心网络：

```text
                         Internet
                            |
                    Public IP / EIP
                            |
                    +---------------+
                    | Management EC2 |
                    | Public Subnet  |
                    +---------------+
                            |
                     Instance Profile
                            |
          +-----------------+------------------+
          |                 |                  |
         S3            CloudFormation      Lambda
                            |
                       Step Functions
                            |
                         Bedrock
```

---

# 2. Management EC2

Management EC2 放在：

```text
Public Subnet
```

并且：

* Public IP / Elastic IP：**Yes**
* SSH Port 22：**No**
* Public inbound access：**No**
* SSM：**Yes**
* IAM Instance Profile：**Yes**

也就是说：

```text
EC2
├── Public Subnet
├── Elastic IP
├── No inbound :22
├── SSM Agent
└── IAM Instance Profile
```

Public IP 的作用主要是让 EC2 可以直接访问 Internet，例如：

```text
GitHub
AWS APIs
Package repositories
Docker registry
```

而不是为了让别人 SSH 进来。

---

# 3. Security Group

Management EC2 Security Group：

### Inbound

```text
NONE
```

也就是：

```text
22   ❌
80   ❌
443  ❌
```

没有任何 Internet inbound rule。

### Outbound

允许：

```text
HTTPS / TCP 443
0.0.0.0/0
```

例如：

```yaml
ManagementSecurityGroup:
  Type: AWS::EC2::SecurityGroup

  Properties:
    GroupDescription: Management EC2 Security Group
    VpcId: !Ref VpcId

    SecurityGroupEgress:
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0
```

这样：

```text
Internet ---> EC2 :22
              X

Internet ---> EC2 :443
              X

EC2 ---> Internet :443
         OK
```

---

# 4. EC2 Management Access

虽然没有 SSH，但是仍然可以通过：

```text
AWS Systems Manager
```

管理 EC2。

例如：

```bash
aws ssm start-session --target i-xxxxxxxx
```

访问路径：

```text
Developer
    |
    | AWS API
    v
AWS Systems Manager
    |
    v
Management EC2
```

EC2 上配置：

```yaml
IamInstanceProfile:
  !Ref ManagementInstanceProfile
```

Role：

```yaml
ManagedPolicyArns:
  - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

所以：

> **Public IP ≠ 必须开放 SSH。**

---

# 5. IAM Instance Profile

EC2 使用独立的 Management Role：

```text
ManagementEC2Role
        |
        v
Instance Profile
        |
        v
Management EC2
```

权限分成几个部分。

### SSM

```text
AmazonSSMManagedInstanceCore
```

### S3

例如：

```text
s3:GetObject
s3:PutObject
s3:ListBucket
```

只允许指定 bucket。

### Lambda

例如：

```text
lambda:GetFunction
lambda:UpdateFunctionCode
lambda:UpdateFunctionConfiguration
lambda:PublishVersion
```

### Step Functions

例如：

```text
states:DescribeStateMachine
states:StartExecution
states:StopExecution
```

### CloudFormation

例如：

```text
cloudformation:CreateStack
cloudformation:UpdateStack
cloudformation:DeleteStack
cloudformation:DescribeStacks
```

### IAM PassRole

如果 EC2 通过 CloudFormation 创建 Lambda / Step Functions：

```text
iam:PassRole
```

但是只允许指定的 application roles。

---

# 6. CloudFormation Plan

建议拆成以下几个 Stack：

```text
01-network.yaml
        |
        v
02-security-groups.yaml
        |
        v
03-iam.yaml
        |
        v
04-management-ec2.yaml
        |
        v
05-s3.yaml
        |
        v
06-lambda.yaml
        |
        v
07-step-functions.yaml
```

---

## Stack 01 — Network

创建：

```text
VPC
├── Public Subnet AZ-a
├── Public Subnet AZ-b
├── Private Subnet AZ-a
├── Private Subnet AZ-b
├── Internet Gateway
├── Public Route Table
└── Private Route Table / NAT
```

Public route：

```text
0.0.0.0/0
    |
Internet Gateway
```

所以 Management EC2 可以：

```text
EC2
 |
Internet Gateway
 |
Internet
```

---

## Stack 02 — Security Groups

主要创建：

```text
ManagementEC2SecurityGroup
```

Inbound：

```text
NONE
```

Outbound：

```text
TCP 443 -> 0.0.0.0/0
```

---

## Stack 03 — IAM

创建：

```text
ManagementEC2Role
ManagementEC2InstanceProfile

LambdaExecutionRole
StepFunctionExecutionRole
```

关系：

```text
ManagementEC2Role
       |
       v
ManagementEC2
```

而：

```text
LambdaExecutionRole
       |
       v
Lambda
```

不要让 Lambda 和 EC2 共用一个 Role。

---

# 7. Stack 04 — Management EC2

CloudFormation 创建：

```text
Management EC2
├── Public Subnet
├── Public IP / EIP
├── Security Group
├── Instance Profile
├── Encrypted EBS
└── IMDSv2
```

例如：

```yaml
ManagementEC2:
  Type: AWS::EC2::Instance

  Properties:
    ImageId: !Ref AmiId

    InstanceType: !Ref InstanceType

    SubnetId: !Ref PublicSubnetId

    SecurityGroupIds:
      - !Ref ManagementSecurityGroup

    IamInstanceProfile:
      !Ref ManagementInstanceProfile

    BlockDeviceMappings:
      - DeviceName: /dev/xvda
        Ebs:
          VolumeSize: 30
          VolumeType: gp3
          Encrypted: true
          DeleteOnTermination: true

    MetadataOptions:
      HttpTokens: required
      HttpEndpoint: enabled

    Tags:
      - Key: Name
        Value: management-ec2
```

如果需要固定 IP：

```yaml
ManagementEIP:
  Type: AWS::EC2::EIP
  Properties:
    Domain: vpc

ManagementEIPAssociation:
  Type: AWS::EC2::EIPAssociation
  Properties:
    InstanceId: !Ref ManagementEC2
    EIP: !Ref ManagementEIP
```

---

# 8. 最终网络模型

最终就是：

```text
                         Internet
                            |
                            |
                       Internet GW
                            |
                     +------+------+
                     |             |
               Public Subnet   Private Subnet
                     |             |
              +-------------+   +----------+
              | Management  |   | Lambda   |
              |    EC2      |   +----------+
              |             |
              | Public IP   |
              | EIP         |
              +-------------+
                     |
                     | IAM Role
                     |
        +------------+-------------+
        |            |             |
       S3       CloudFormation   SSM
                     |
              +------+------+
              |             |
           Lambda     Step Functions
                                  |
                               Bedrock
```