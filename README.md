# EKS Cluster Provisioning with Terraform and GitHub Actions

A production-ready solution for deploying and managing an Amazon Elastic Kubernetes Service (EKS) cluster on AWS, fully automated through Terraform and a GitHub Actions CI/CD pipeline.

![EKS- GitHub Actions- Terraform](assets/Presentation1.gif)

---

## About the Project

This project provisions a secure, scalable EKS cluster on AWS using Infrastructure as Code principles. The entire lifecycle - from cluster creation to teardown - is handled through a GitHub Actions workflow, removing the need for any manual CLI operations. The architecture follows AWS and Terraform best practices, covering networking, IAM, node group configuration, and add-on management.

The project is structured around two layers - a reusable Terraform module (`module/`) that defines all AWS resources, and an environment-specific configuration layer (`eks/`) that drives the module with values defined in a `.tfvars` file. This separation makes it easy to replicate the setup across multiple environments (dev, staging, production) by simply swapping the variables file.

### Key Features

**Infrastructure as Code** - All AWS resources are defined in Terraform, ensuring deployments are consistent, version-controlled, and repeatable across environments.

**Modular Architecture** - The codebase is split into a reusable `module/` layer (VPC, IAM, EKS) and an environment-specific `eks/` layer, making it straightforward to extend or adapt for other environments.

**GitHub Actions CI/CD** - A manually triggered workflow supports `plan`, `apply`, and `destroy` actions with a configurable `.tfvars` input, giving full control over when and how infrastructure changes are applied. A secondary workflow automatically runs `terraform validate` and `fmt` checks on every push to catch errors early.

**Remote State with Locking** - Terraform state is stored in an encrypted S3 bucket. DynamoDB is used for state locking, preventing race conditions during concurrent runs.

**Mixed Node Groups** - The cluster runs both On-Demand and Spot instance node groups with autoscaling configured, balancing reliability and cost efficiency.

**Private Networking** - A dedicated VPC is provisioned with public and private subnets across three availability zones, NAT gateways, and route tables that keep worker nodes off the public internet.

**OIDC Provider** - An OpenID Connect provider is attached to the cluster, enabling IAM Roles for Service Accounts (IRSA) for fine-grained, pod-level AWS permissions.

**Managed Add-ons** - Core EKS add-ons (VPC CNI, CoreDNS, kube-proxy, AWS EBS CSI Driver) are provisioned and versioned as part of the Terraform configuration.

---

## Architecture Overview

```
GitHub Actions
      │
      ▼
terraform init ──────────────────► S3 Bucket (Remote State)
      │                                  │
      ▼                            DynamoDB (State Lock)
terraform plan/apply
      │
      ▼
  AWS eu-west-1
  ┌─────────────────────────────────────┐
  │  VPC (10.16.0.0/16)                 │
  │  ┌─────────────┐  ┌──────────────┐  │
  │  │ Public       │  │ Private      │  │
  │  │ Subnets x3   │  │ Subnets x3   │  │
  │  │ (IGW)        │  │ (NAT Gateway)│  │
  │  └─────────────┘  └──────────────┘  │
  │                                     │
  │  EKS Cluster v1.33                  │
  │  ┌──────────────┐ ┌──────────────┐  │
  │  │ On-Demand    │ │ Spot         │  │
  │  │ Node Group   │ │ Node Group   │  │
  │  │ (t3.medium)  │ │ (c5/m5/t3)  │  │
  │  └──────────────┘ └──────────────┘  │
  └─────────────────────────────────────┘
```

---

## Directory Structure

```
.
├── .github/
│   └── workflows/
│       ├── terraform.yml            # Main workflow: plan / apply / destroy
│       └── validate-terraform.yml   # Auto-validates on every push
├── eks/
│   ├── backend.tf                   # S3 + DynamoDB remote state config
│   ├── main.tf                      # Root module calling eks/module
│   ├── variables.tf                 # Input variable declarations
│   └── dev.tfvars                   # Environment-specific values
├── module/
│   ├── eks.tf                       # EKS cluster and node groups
│   ├── vpc.tf                       # VPC, subnets, gateways, route tables
│   ├── iam.tf                       # Cluster and node group IAM roles
│   ├── gather.tf                    # Data sources (TLS cert, OIDC policy)
│   └── variables.tf                 # Module input variables
├── assets/
│   └── Presentation1.gif
├── Jenkinsfile                      # Alternative Jenkins pipeline
└── README.md
```

---

## Infrastructure Details

| Component | Details |
|---|---|
| EKS Version | 1.33 |
| Region | eu-west-1 (Dublin) |
| VPC CIDR | 10.16.0.0/16 |
| Availability Zones | eu-west-1a, eu-west-1b, eu-west-1c |
| Public Subnets | 3 × /20 subnets |
| Private Subnets | 3 × /20 subnets |
| On-Demand Instances | t3.medium |
| Spot Instances | c5.large, c5.xlarge, m5.large, m5.xlarge, t3.large, t3.xlarge, t3.medium |
| EKS Add-ons | vpc-cni, coredns, kube-proxy, aws-ebs-csi-driver |
| Terraform State | S3 + DynamoDB locking |

---

## How to Use This Project

### Prerequisites

Before running the workflows, ensure the following are in place:

- An active AWS account
- An IAM user with programmatic access and the following permissions:
  - `AmazonEKSClusterPolicy`
  - `AmazonEKSWorkerNodePolicy`
  - `AmazonEC2FullAccess`
  - `IAMFullAccess`
  - `AmazonS3FullAccess`
  - `AmazonDynamoDBFullAccess`
  - `AmazonVPCFullAccess`
- AWS Access Key ID and Secret Access Key for that IAM user

---

### Step 1 - Fork or clone this repository

Fork this repo to your own GitHub account or upload the files directly. Make sure the folder structure is preserved exactly as shown above.

---

### Step 2 - Create AWS backend resources

These two resources must exist in AWS **before** running the workflow, as Terraform needs them to store state.

**Create the S3 bucket:**
1. Go to AWS Console → S3 → Create bucket
2. Bucket name: match exactly what is in `eks/backend.tf` (e.g. `your-name-eks-tf-state`)
3. Region: `eu-west-1`
4. Enable **versioning** and **server-side encryption (SSE-S3)**
5. Keep Block all public access **ON**

**Create the DynamoDB table:**
1. Go to AWS Console → DynamoDB → Create table
2. Table name: `Lock-Files`
3. Partition key: `LockID` (type: String)
4. Leave all other settings as default

---

### Step 3 - Update configuration files

**`eks/backend.tf`** - update the S3 bucket name to match what you created:
```hcl
bucket = "your-name-eks-tf-state"
region = "eu-west-1"
```

**`eks/main.tf`** - update the org label to your own identifier:
```hcl
locals {
  org = "your-initials"
}
```

**`eks/dev.tfvars`** - review and adjust region, instance types, and node group sizing as needed.

---

### Step 4 - Create GitHub Actions environment

1. Go to your repo → **Settings → Environments**
2. Click **New environment**
3. Name it exactly: `production`
4. Save

---

### Step 5 - Add AWS credentials as GitHub Secrets

1. Go to **Settings → Secrets and variables → Actions**
2. Click **New repository secret** and add:

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | Your IAM user secret key |

> Make sure there are no extra spaces when pasting the values.

---

### Step 6 - Trigger the workflow

1. Go to the **Actions** tab in your repo
2. Click **EKS-Creation-Using-Terraform** in the left sidebar
3. Click **Run workflow**
4. Fill in the inputs:

| Input | Value |
|---|---|
| `tfvars_file` | `dev.tfvars` |
| `action` | `plan` (always run this first) |

5. Click the green **Run workflow** button
6. Watch the live logs - each step will show what Terraform is doing
7. Once `plan` looks correct, re-run with `action: apply` to provision the cluster

---

### Step 7 - Verify in AWS Console

After a successful `apply` (takes 10–15 minutes), verify the following in AWS:

- **EKS** → your cluster should show as `Active`
- **VPC** → new VPC with 6 subnets (3 public, 3 private)
- **EC2** → worker nodes running in the private subnets
- **S3** → `terraform.tfstate` file present in your bucket
- **DynamoDB** → `Lock-Files` table showing the lock record

---

### Step 8 - Connect to the cluster (optional)

If you want to interact with the cluster using `kubectl`:

```bash
aws eks update-kubeconfig --region eu-west-1 --name dev-your-initials-eks-cluster
kubectl get nodes
kubectl get pods -A
```

---

### Step 9 - Destroy when done

To avoid unnecessary AWS charges, always destroy the infrastructure after use:

1. Go to **Actions → EKS-Creation-Using-Terraform → Run workflow**
2. Set `action` to `destroy`
3. Click **Run workflow**

> ⚠️ Running this cluster costs approximately $0.20–0.30/hr. Always destroy after use.

---

### Alternative: Jenkins Pipeline

A `Jenkinsfile` is also included if you prefer to run this through a Jenkins server instead of GitHub Actions. It supports the same `plan`, `apply`, and `destroy` actions via build parameters, and uses the `aws-creds` Jenkins credential for AWS authentication.

---

## License

This project is licensed under the Apache 2.0 License. See the [LICENSE](LICENSE) file for details.
