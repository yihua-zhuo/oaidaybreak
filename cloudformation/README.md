# CloudFormation — management bootstrap (`oaidaybreak`)

One stack: `management-bootstrap.yaml`. It creates the network, security groups,
the management EC2 role and the management EC2 itself.

Everything downstream — S3, Lambda, Step Functions, Bedrock, **and the
application IAM roles** — is provisioned with **Terraform from the instance**,
using **local state**. That is why this template grants the EC2 role those
permissions instead of creating the resources itself.

Region: `us-east-1`

## Naming convention

```text
<prefix>_<type>_<suffix>      →  mpa_sec_ec2_oaidaybreak
```

| Parameter    | Default       |
| ------------ | ------------- |
| `NamePrefix` | `mpa_sec`     |
| `NameSuffix` | `oaidaybreak` |

Underscores everywhere they are legal; hyphens only where AWS rejects or
discourages them:

| Identifier                      | Form                               |
| ------------------------------- | ---------------------------------- |
| Name tags, IAM roles, SGs, EC2  | `mpa_sec_ec2_oaidaybreak`          |
| S3 buckets (underscore illegal) | `mpa-sec-s3-artifacts-oaidaybreak` |
| CFN export names                | `mpa-sec-oaidaybreak-VpcId`        |

## What it creates

| Group | Resources |
| ----- | --------- |
| Network | VPC, 2 public + 2 private subnets, IGW, route tables, S3 gateway endpoint |
| Security groups | `mpa_sec_sg_mgmt_oaidaybreak` (**no inbound**, egress 443), `..._sg_vpce_...`, `..._sg_private_...` |
| IAM | `mpa_sec_role_mgmt_ec2_oaidaybreak` + instance profile |
| Compute | `mpa_sec_ec2_oaidaybreak` + EIP, Terraform & AWS CLI preinstalled |

## Deploy

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --stack-name mpa-sec-oaidaybreak-bootstrap \
  --template-file management-bootstrap.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

`CAPABILITY_NAMED_IAM` is required because the role carries an explicit name.

To override naming or sizing:

```bash
  --parameter-overrides NamePrefix=mpa_sec NameSuffix=oaidaybreak InstanceType=t3.medium
```

## Access

There is no SSH — no key pair, and the management SG has zero inbound rules.

```bash
aws ssm start-session --region us-east-1 \
  --target $(aws cloudformation describe-stacks --region us-east-1 \
    --stack-name mpa-sec-oaidaybreak-bootstrap \
    --query "Stacks[0].Outputs[?OutputKey=='InstanceId'].OutputValue" --output text)
```

The public IP / EIP exists so the box can reach **out** (GitHub, AWS APIs,
package repos, Terraform registry), not so anyone can reach in.

## Terraform on the box

Terraform and the AWS CLI are installed by UserData; work in `/opt/infra`. The
instance role is picked up automatically — no keys to configure.

**The application roles must be created under the IAM path
`/oaidaybreak/`**, or Terraform will get `AccessDenied`:

```hcl
resource "aws_iam_role" "lambda_exec" {
  name = "mpa_sec_role_lambda_oaidaybreak"
  path = "/oaidaybreak/"          # required -- see AppRolePath
  # ...
}
```

Stack outputs are exported, so Terraform can look up the network instead of
hardcoding ids:

```hcl
data "aws_cloudformation_export" "private_subnets" {
  name = "mpa-sec-oaidaybreak-PrivateSubnetIds"
}
```

## Notes / decisions

- **EC2 can create IAM roles**, scoped two ways at once: the role name must
  match `mpa_sec_role_*_oaidaybreak` **and** it must sit under the path
  `/oaidaybreak/`. The instance's own role is at path `/`, so the box cannot
  modify or escalate its own permissions despite the name matching. An explicit
  `Deny` also blocks attaching any managed policy outside
  `aws:policy/service-role/*` or the project's own path, so
  `AdministratorAccess` cannot be attached to a new role.
- **SSM works with 443-only egress.** The instance is in a public subnet routed
  via the IGW, so the agent reaches the SSM endpoints outbound — no interface
  endpoints required. `mpa_sec_sg_vpce_oaidaybreak` is there if you later drop
  the public IP and switch to interface endpoints.
- **No NAT Gateway.** Private subnets therefore have no Internet egress; they
  reach S3 via the gateway endpoint only. A VPC-attached Lambda calling any
  other AWS API needs an interface endpoint for that service, or should run
  outside the VPC.
- **Local Terraform state** lives at `/opt/infra/terraform.tfstate` on a root
  volume created with `DeleteOnTermination: true`. That volume is the only copy:
  terminating the instance destroys the state and orphans everything it tracked.
  Back `/opt/infra` up (e.g. `aws s3 cp` to the artifact bucket — the role
  already has write access) before any instance replacement. No locking either,
  so run Terraform from one session at a time.
- **`DependsOn` on the instance** (public route + subnet association) matters in
  a single stack: without it CloudFormation may launch the EC2 before egress
  exists, and UserData plus the 15-minute `cfn-signal` wait would fail.
- The EC2 role has **no `iam:*` write access outside `/oaidaybreak/`** and no
  permission to create users, groups or access keys.
