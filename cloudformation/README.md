# CloudFormation — bootstrap up to the Management EC2

Implements `setup.md` stacks **01–04** only. Everything from S3 / Lambda /
Step Functions / Bedrock onward is provisioned **with Terraform, from the
management EC2 itself** — that is why stack 03 grants the box those
permissions instead of creating the resources here.

Region: `us-east-1`

## Naming convention

```text
<prefix>_<type>_<suffix>      →  mpa_sec_ec2_oaidaybreak
```

Driven by two parameters present in every template:

| Parameter     | Default       |
| ------------- | ------------- |
| `NamePrefix`  | `mpa_sec`     |
| `NameSuffix`  | `oaidaybreak` |

Underscores are used everywhere they are legal. Hyphens are used only where
AWS rejects or discourages underscores:

| Identifier                     | Form                             |
| ------------------------------ | -------------------------------- |
| Name tags, IAM roles, SGs, EC2 | `mpa_sec_ec2_oaidaybreak`        |
| S3 buckets (underscore illegal)| `mpa-sec-s3-artifacts-oaidaybreak` |
| CFN export names               | `mpa-sec-oaidaybreak-VpcId`      |

## Stacks

| # | File | Creates |
| - | ---- | ------- |
| 01 | `01-network.yaml` | VPC, 2 public + 2 private subnets, IGW, route tables, S3 gateway endpoint |
| 02 | `02-security-groups.yaml` | Management SG (**no inbound**, egress 443), VPC-endpoint SG, private-workload SG |
| 03 | `03-iam.yaml` | `mpa_sec_role_mgmt_ec2_oaidaybreak` + instance profile, `mpa_sec_role_lambda_oaidaybreak`, `mpa_sec_role_sfn_oaidaybreak` |
| 04 | `04-management-ec2.yaml` | `mpa_sec_ec2_oaidaybreak` + EIP, Terraform/AWS CLI preinstalled |

Deploy in order — each consumes the previous stack's outputs.

## Deploy

```bash
REGION=us-east-1
PREFIX=mpa-sec-oaidaybreak

# 01 — network
aws cloudformation deploy \
  --region $REGION \
  --stack-name ${PREFIX}-01-network \
  --template-file 01-network.yaml

VPC_ID=$(aws cloudformation describe-stacks --region $REGION \
  --stack-name ${PREFIX}-01-network \
  --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text)
PUBLIC_SUBNET=$(aws cloudformation describe-stacks --region $REGION \
  --stack-name ${PREFIX}-01-network \
  --query "Stacks[0].Outputs[?OutputKey=='PublicSubnetAId'].OutputValue" --output text)

# 02 — security groups
aws cloudformation deploy \
  --region $REGION \
  --stack-name ${PREFIX}-02-sg \
  --template-file 02-security-groups.yaml \
  --parameter-overrides VpcId=$VPC_ID

MGMT_SG=$(aws cloudformation describe-stacks --region $REGION \
  --stack-name ${PREFIX}-02-sg \
  --query "Stacks[0].Outputs[?OutputKey=='ManagementSecurityGroupId'].OutputValue" --output text)

# 03 — IAM (creates named roles)
aws cloudformation deploy \
  --region $REGION \
  --stack-name ${PREFIX}-03-iam \
  --template-file 03-iam.yaml \
  --capabilities CAPABILITY_NAMED_IAM

INSTANCE_PROFILE=$(aws cloudformation describe-stacks --region $REGION \
  --stack-name ${PREFIX}-03-iam \
  --query "Stacks[0].Outputs[?OutputKey=='ManagementEC2InstanceProfileName'].OutputValue" --output text)

# 04 — management EC2
aws cloudformation deploy \
  --region $REGION \
  --stack-name ${PREFIX}-04-ec2 \
  --template-file 04-management-ec2.yaml \
  --parameter-overrides \
      PublicSubnetId=$PUBLIC_SUBNET \
      ManagementSecurityGroupId=$MGMT_SG \
      ManagementInstanceProfileName=$INSTANCE_PROFILE
```

`--capabilities CAPABILITY_NAMED_IAM` is required on stack 03 only, because the
roles carry explicit names.

## Access

There is no SSH — no key pair, and the SG has zero inbound rules. Use SSM:

```bash
aws ssm start-session --region us-east-1 \
  --target $(aws cloudformation describe-stacks --region us-east-1 \
    --stack-name mpa-sec-oaidaybreak-04-ec2 \
    --query "Stacks[0].Outputs[?OutputKey=='InstanceId'].OutputValue" --output text)
```

The public IP / EIP exists so the box can reach **out** (GitHub, AWS APIs,
package repos, Terraform provider registry), not so anyone can reach in.

## Notes / decisions

- **SSM path.** The instance sits in a public subnet with a 0.0.0.0/0 route via
  the IGW, so the SSM agent reaches the SSM endpoints over egress 443 — no
  interface endpoints needed. `mpa_sec_sg_vpce_oaidaybreak` is provided in case
  you later move to interface endpoints and drop the public IP.
- **Egress is 443-only**, per `setup.md` §3. Amazon Linux 2023 repos are HTTPS,
  so package installs work. `AllowHttpEgress=true` opens 80 if you ever need a
  plain-HTTP mirror.
- **No NAT Gateway.** The private subnets therefore have no Internet egress —
  they reach S3 through the gateway endpoint only. A VPC-attached Lambda that
  needs to call another AWS API will need an interface endpoint for that
  service (attach `mpa_sec_sg_vpce_oaidaybreak`), or to run outside the VPC.
- **`iam:PassRole` is scoped** to exactly the two application role ARNs, with an
  `iam:PassedToService` condition — not `*`.
- The EC2 role deliberately has **no `iam:Create*`/`Put*`**. If your Terraform
  needs to create new roles, add that permission explicitly rather than
  broadening it here.
- **Terraform uses local state** (`/opt/infra/terraform.tfstate` on the box) —
  no state bucket, no DynamoDB lock table, and no IAM policy for either. Two
  consequences: the root volume holds the only copy of your state, and it is
  created with `DeleteOnTermination: true`, so terminating the instance
  destroys the state while leaving the resources it tracks orphaned. Back
  `/opt/infra` up (e.g. `aws s3 cp` to the artifact bucket) before any
  instance replacement. There is also no locking, so run Terraform from one
  session at a time.
