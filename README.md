# CloudFormation – Café App Infrastructure

CloudFormation templates that recreate the highly available and auto scaling café application
from the *Elasticity and Availability* assignment. They are the class lab templates
`lab-network.yaml` and `lab-application.yaml`, updated for this assignment.

**Every change compared with the lab templates is commented in the files** with
`# ADDED:`, `# CHANGED:` or `# REMOVED:`, and each file starts with a change log.
The first commit of this repository contains the original lab templates, so all changes can also be
seen in the commit diff.

## Architecture

| Stack | Template | Creates |
|---|---|---|
| `lab-network` | `lab-network.yaml` | VPC (10.0.0.0/16) in 2 AZs: 2 public subnets, 2 private app subnets, 2 private DB subnets, Internet gateway, 1 NAT gateway per AZ, route tables. Exports the VPC and all subnet IDs. |
| `lab-application` | `lab-application.yaml` | Imports the network exports and creates: bastion host (SSH only from your workstation IP), Application Load Balancer, launch template (cafe-app-ami), Auto Scaling group with CPU target tracking, RDS MariaDB Multi-AZ, Secrets Manager secrets for the café app, IAM role, 4 security groups. |

```
Internet ──► ALB (public subnets AZ1 + AZ2)
               │
               ▼
        Auto Scaling group (private app subnets AZ1 + AZ2) ──► NAT gateway of its AZ ──► Internet
               │
               ▼
        RDS MariaDB Multi-AZ (private DB subnets AZ1 + AZ2)

Workstation IP ──SSH──► Bastion (public subnet AZ1) ──SSH──► app servers
```

## Parameters

Required by the assignment:

| Parameter | Template | Default | Purpose |
|---|---|---|---|
| `WorkstationIP` | application | – (required) | Your public IP in CIDR form (e.g. `203.0.113.25/32`). The bastion only accepts SSH from it. |
| `MinSize` / `MaxSize` | application | 2 / 4 | Minimum and maximum size of the Auto Scaling group |

Other parameters that make the templates reusable:

| Parameter | Template | Default |
|---|---|---|
| `VpcCidr`, `PublicSubnet1Cidr`, `PublicSubnet2Cidr`, `AppSubnet1Cidr`, `AppSubnet2Cidr`, `DBSubnet1Cidr`, `DBSubnet2Cidr` | network | 10.0.0.0/16, 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/23, 10.0.4.0/23, 10.0.128.0/24, 10.0.129.0/24 |
| `NatGatewayPerAZ` (`false` = one shared NAT gateway, cheaper) | network | true |
| `NetworkStackName` | application | lab-network |
| `KeyName` | application | – (required) |
| `AppAMIID`, `AppInstanceType` | application | ami-0c9a853f2177ce3a9, t3.micro |
| `BastionAMIID`, `BastionInstanceType` | application | latest Amazon Linux 2023, t3.micro |
| `DesiredCapacity`, `CPUTargetValue`, `HealthCheckPath` | application | 2, 50, /cafe/ |
| `DBInstanceClass`, `DBEngineVersion`, `DBAllocatedStorage`, `DBMultiAZ` | application | db.t3.micro, 10.6, 20, true |
| `DBName`, `DBUsername`, `DBPassword` (NoEcho) | application | cafe_db, admin, – |
| `DBSnapshotIdentifier` (restore existing data; empty = new database) | application | empty |
| `SecretsPrefix` (use another prefix to run a 2nd copy in the same account) | application | /cafe |
| `Currency`, `TimeZone`, `ShowServerInfo` | application | $, America/New_York, false |

## Deploy

```bash
# 1. Network stack (~3 min)
aws cloudformation deploy --stack-name lab-network --template-file lab-network.yaml

# 2. Application stack (~20 min, mostly the Multi-AZ database)
aws cloudformation deploy --stack-name lab-application --template-file lab-application.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    NetworkStackName=lab-network \
    WorkstationIP=$(curl -s https://checkip.amazonaws.com)/32 \
    KeyName=<your-key-pair> \
    MinSize=2 MaxSize=4 \
    DBPassword=<password> \
    DBSnapshotIdentifier=<optional-snapshot> \
    SecretsPrefix=/cafe

# 3. Open the café website
aws cloudformation describe-stacks --stack-name lab-application \
  --query "Stacks[0].Outputs[?OutputKey=='URL'].OutputValue" --output text
```

The same can be done in the CloudFormation console (**Create stack → Upload a template file**);
the application stack needs the *"I acknowledge that AWS CloudFormation might create IAM resources"* checkbox.

Requirements: one free Elastic IP per NAT gateway (2 by default) and the `cafe-app-ami` AMI in the Region.

## Validate

```bash
aws cloudformation validate-template --template-body file://lab-network.yaml
aws cloudformation validate-template --template-body file://lab-application.yaml
cfn-lint lab-network.yaml lab-application.yaml
```

## Clean up

```bash
aws cloudformation delete-stack --stack-name lab-application
aws cloudformation wait stack-delete-complete --stack-name lab-application
aws cloudformation delete-stack --stack-name lab-network
```

The application stack must be deleted first because it imports the network exports.
The database keeps a final snapshot (`DeletionPolicy: Snapshot`); delete it in the RDS console when it is no longer needed.
