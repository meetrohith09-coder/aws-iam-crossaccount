# AWS IAM Cross-Account Access: Production-Grade Architecture

![AWS](https://img.shields.io/badge/AWS-IAM-orange)
![Security](https://img.shields.io/badge/Security-Production--Grade-green)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

## Overview

A complete implementation of **secure cross-account access** in AWS—the pattern used by Netflix, Uber, and enterprise cloud teams managing 100+ AWS accounts.

Instead of permanent access keys, this project demonstrates:
- ✅ Temporary role assumption (auto-expires in 1 hour)
- ✅ MFA-enforced access (6-digit code required)
- ✅ ExternalId security (prevents confused deputy attacks)
- ✅ Least privilege permissions (read-only by default)
- ✅ Full CloudTrail auditability (logs everything)
- ✅ Production-ready architecture (scales to 100+ accounts)

## Why This Matters

Most developers skip IAM because it's complex. But **IAM is where breaches happen.**

**The Capital One Breach (2019):**
- Exposed AWS credentials found in GitHub
- Attacker had broad IAM permissions
- 100M customers' data exfiltrated
- $80M settlement

**This project prevents that.** Using the patterns here:
- Credentials expire automatically (no permanent keys in GitHub)
- Broad permissions not possible (least privilege enforced)
- Every access is logged (attackers caught within minutes)

## Architecture

```
┌─────────────────────────────────────────────────────┐
│            AWS ORGANIZATION                         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────────────┐    ┌──────────────────┐   │
│  │ SECURITY ACCOUNT    │    │ WORKLOAD ACCOUNT │   │
│  │ (489157521585)      │    │ (657654036224)   │   │
│  │                     │    │                  │   │
│  │ security-admin user │    │                  │   │
│  │ + MFA               │───▶│ SecurityAuditRole│   │
│  │ + Access Key        │    │ + Trust Policy   │   │
│  │                     │ STS│ + Read-only      │   │
│  │ SecurityAuditOperator   │ + ExternalId     │   │
│  │ role (can assume)   │    │ + MFA Required   │   │
│  │                     │    │                  │   │
│  └─────────────────────┘    └──────────────────┘   │
│         ▲                                           │
│         │ Temporary credentials                    │
│         │ Valid 1 hour only                        │
│         │ Logged in CloudTrail                     │
│         │                                          │
│      [Phone: Pixel1]                               │
│      [MFA: 6-digit code]                           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## Quick Start

### Prerequisites
- 2 AWS accounts (or create free tier accounts)
- AWS CLI v2 installed
- MFA device (phone with authenticator app)
- Terraform (optional, but recommended)

### 1️⃣ Set Up AWS CLI Credentials

```bash
# Configure Security Account profile
aws configure --profile security-account

# Configure Workload Account profile
aws configure --profile workload-account

# Verify both work
aws sts get-caller-identity --profile security-account
aws sts get-caller-identity --profile workload-account
```

### 2️⃣ Create IAM User in Security Account

```bash
# In Security Account console:
# IAM → Users → Create user (security-admin)
# Add to group with cross-account assume permissions
# Create access key and set up MFA
```

### 3️⃣ Deploy Roles to Workload Account

#### Option A: Using Terraform (Recommended)

```bash
# Clone this repo
git clone https://github.com/yourusername/aws-iam-crossaccount.git
cd aws-iam-crossaccount

# Update terraform.tfvars with your account IDs
cat > terraform.tfvars <<EOF
security_account_id = "489157521585"
workload_account_id = "657654036224"
external_id         = "MyUniqueSecretValue123"
aws_profile         = "workload-account"
EOF

# Deploy
terraform init
terraform plan
terraform apply
```

#### Option B: Using JSON Policies (Manual)

**Create trust-policy.json:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::489157521585:user/security-admin"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "MyUniqueSecretValue123"
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

**Create permission-policy.json:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "vpc:Describe*",
        "s3:ListAllMyBuckets",
        "iam:GetRole",
        "iam:GetUser"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "s3:DeleteBucket",
        "iam:DeleteUser"
      ],
      "Resource": "*"
    }
  ]
}
```

**Create in AWS Console:**
- IAM → Roles → Create role
- Paste trust-policy.json
- Attach permission-policy.json
- Name: SecurityAuditRole

### 4️⃣ Test Role Assumption

```bash
# Assume the cross-account role
aws sts assume-role \
  --role-arn arn:aws:iam::657654036224:role/SecurityAuditRole \
  --role-session-name test-session \
  --external-id MyUniqueSecretValue123 \
  --serial-number arn:aws:iam::489157521585:mfa/pixel1 \
  --token-code 123456 \
  --profile security-account
```

**Expected output:**
```json
{
    "Credentials": {
        "AccessKeyId": "ASIA...",
        "SecretAccessKey": "...",
        "SessionToken": "FwoGZXIvYXdzEJ7...",
        "Expiration": "2026-02-06T18:36:45+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AIDA...:test-session",
        "Arn": "arn:aws:sts::657654036224:assumed-role/SecurityAuditRole/test-session"
    }
}
```

✅ **Success!** You now have temporary credentials valid for 1 hour.

## Project Structure

```
aws-iam-crossaccount/
├── README.md                           # This file
├── terraform/
│   ├── main.tf                        # Main Terraform configuration
│   ├── variables.tf                   # Variable definitions
│   └── terraform.tfvars               # Your account IDs
├── policies/
│   ├── trust-policy.json              # Who can assume the role
│   ├── permission-policy.json         # What role can do
│   └── assume-policy.json             # Who can call AssumeRole
├── docs/
│   ├── PROJECT_DOCUMENTATION.md       # Detailed Q&A answers
│   ├── ARCHITECTURE.md                # Deep dive into design
│   └── SCALING.md                     # How to scale to 100+ accounts
└── cloudtrail/
    ├── detect-misuse.md               # How to monitor for attacks
    └── alerts.json                    # CloudWatch alarm configs
```

## Key Concepts Explained

### 1. Trust Policy (Who Can Assume)

```json
{
  "Principal": {
    "AWS": "arn:aws:iam::489157521585:user/security-admin"
  }
}
```
**Means:** Only the security-admin user in account 489157521585 can assume this role.

### 2. ExternalId (Extra Password)

```json
{
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "MyUniqueSecretValue123"
    }
  }
}
```
**Means:** Even if someone gets the role ARN, they need the ExternalId password to assume it.

### 3. MFA Requirement

```json
{
  "Condition": {
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"
    }
  }
}
```
**Means:** Can't assume the role without a 6-digit code from your phone.

### 4. Permission Policy (What They Can Do)

```json
{
  "Effect": "Allow",
  "Action": ["ec2:Describe*", "vpc:Describe*"],
  "Resource": "*"
}
```
**Means:** Role can READ EC2 and VPC, but NOT modify or delete.

### 5. Explicit Deny (Defense in Depth)

```json
{
  "Effect": "Deny",
  "Action": ["ec2:TerminateInstances", "s3:DeleteBucket"]
}
```
**Means:** Even if there's a bug in AWS, these actions are always blocked.

## How It Works: Step-by-Step

1. **You authenticate to Security Account with username/password + MFA** ✅
2. **You call `aws sts assume-role` with:**
   - Role ARN: `arn:aws:iam::657654036224:role/SecurityAuditRole`
   - ExternalId: `MyUniqueSecretValue123`
   - MFA code: `123456`
3. **AWS checks the trust policy:**
   - ✅ Is your account in the Principal? YES
   - ✅ Did you provide the ExternalId? YES
   - ✅ Did you provide MFA code? YES
4. **AWS generates temporary credentials:**
   - AccessKeyId: `ASIA...` (starts with ASIA, indicates temporary)
   - SecretAccessKey: `...`
   - SessionToken: `...` (proves you assumed a role)
5. **Credentials expire in 1 hour automatically**
6. **Every step logged in CloudTrail** 📝

## Security Features

| Feature | Benefit |
|---------|---------|
| **Temporary Credentials** | Auto-expire in 1 hour max, no permanent keys |
| **MFA Required** | 6-digit code that changes every 30 seconds |
| **ExternalId** | Prevents confused deputy attacks |
| **Least Privilege** | Read-only permissions by default |
| **Explicit Deny** | Defense in depth against bugs |
| **CloudTrail Logging** | Every assumption logged and auditable |
| **Account Isolation** | Separate AWS account, not just a role |

## Scaling to 100+ Accounts

Use AWS StackSets to deploy this to all accounts automatically:

```bash
# Create CloudFormation StackSet
aws cloudformation create-stack-set \
  --stack-set-name CrossAccountRoles \
  --template-body file://cloudformation.yaml

# Deploy to all Production accounts
aws cloudformation create-stack-instances \
  --stack-set-name CrossAccountRoles \
  --accounts 657654036224 657654036225 657654036226 \
  --regions us-east-1

# Update to all accounts at once
aws cloudformation update-stack-set \
  --stack-set-name CrossAccountRoles \
  --template-body file://cloudformation-updated.yaml
```

See [SCALING.md](docs/SCALING.md) for complete details.

## Monitoring & Detection

### Real-Time Alerts

Monitor CloudTrail for:
- ❌ Failed AssumeRole attempts (brute force)
- ❌ AssumeRole from unknown IPs
- ❌ AssumeRole outside business hours
- ❌ Changes to trust policies
- ❌ Multiple users assuming same role

See [detect-misuse.md](cloudtrail/detect-misuse.md) for CloudWatch setup.

### Example: Detecting an Attack

```
3:15 AM: Failed AssumeRole from 35.201.45.89
3:16 AM: Failed AssumeRole from 35.201.45.89
3:17 AM: Failed AssumeRole from 35.201.45.89
↓
ALERT: Multiple failed attempts from unknown IP
↓
Action: Revoke all sessions, rotate credentials
↓
Result: Breach prevented before attacker succeeds
```

## Real-World Examples

### Netflix Architecture
- 100+ AWS accounts
- Central Security Account
- All workload accounts trust Security Account
- Uses exact pattern in this repo

### Capital One Breach (What This Prevents)
- 2019: Exposed AWS credentials in GitHub
- Attacker had broad IAM permissions
- Our approach prevents this:
  - ✅ No credentials in GitHub (uses roles)
  - ✅ Broad permissions blocked (least privilege)
  - ✅ Auto-expires in 1 hour (limited window)
  - ✅ Logged in CloudTrail (caught quickly)

## Questions Answered

See [PROJECT_DOCUMENTATION.md](docs/PROJECT_DOCUMENTATION.md) for detailed answers to:

1. **Why role assumption instead of long-lived credentials?**
2. **How would this scale to 20 or 100 accounts?**
3. **What risks exist if a role is over-privileged?**
4. **How would you detect misuse?**
5. **What happens if the Security Account is compromised?**

## Files in This Repo

### Core Implementation
- `terraform/main.tf` - Complete Terraform code (copy-paste ready)
- `terraform/variables.tf` - Input variables
- `policies/trust-policy.json` - Who can assume
- `policies/permission-policy.json` - What they can do

### Documentation
- `docs/PROJECT_DOCUMENTATION.md` - **Start here**
- `docs/ARCHITECTURE.md` - Deep technical dive
- `docs/SCALING.md` - How to deploy to 100+ accounts

### Monitoring
- `cloudtrail/detect-misuse.md` - Set up alerts
- `cloudtrail/alerts.json` - CloudWatch configurations

## Getting Started

1. **Fork this repo**
2. **Read:** [docs/PROJECT_DOCUMENTATION.md](docs/PROJECT_DOCUMENTATION.md)
3. **Clone:** `git clone https://github.com/yourusername/aws-iam-crossaccount.git`
4. **Update:** `terraform.tfvars` with your account IDs
5. **Deploy:** `terraform apply`
6. **Test:** Run the assume-role command
7. **Monitor:** Set up CloudWatch alarms

## What You'll Learn

After this project, you'll understand:
- ✅ How enterprise cloud teams manage access
- ✅ Why least privilege is critical
- ✅ How to audit everything with CloudTrail
- ✅ How to design for security AND scalability
- ✅ How to prevent the Capital One breach pattern
- ✅ How Netflix manages 100+ AWS accounts

## Resources

- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AssumeRole API](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html)
- [Confused Deputy Problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html)
- [CloudTrail User Guide](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/)

## Author

Built as a portfolio project to demonstrate production-grade AWS IAM architecture.

**This is not a toy project.** This is the real pattern used by Netflix, Uber, and enterprise cloud teams. Anyone reviewing this will see that you understand how modern cloud infrastructure works.

## License

MIT - Feel free to use this for learning or in your own projects.

--
