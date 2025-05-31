# AWS AMI Bakery

Build custom Amazon Machine Images (AMIs) using **AWS Image Builder** and **Sceptre**.

---

## Overview

This project provisions all necessary AWS Image Builder resources using CloudFormation via Sceptre. It supports automated image versioning, cross-account AMI sharing, and streamlined image pipeline creation.

### Why are all resources deployed in a single stack?

When using versioned resources like Components and Recipes, CloudFormation cannot delete old versions if other stacks still reference them. For example:

1. Component is updated → CloudFormation tries to delete the old version.
2. The old Component is still referenced by a Recipe → deletion fails.
3. The same issue occurs when the Recipe is updated and still referenced by the Pipeline.

By deploying all the following resources in a single stack, CloudFormation can handle dependencies gracefully during stack updates:

- **Image Builder Component**
- **Image Recipe**
- **Image Pipeline**
- **Infrastructure Configuration**
- **Distribution Configuration**

---

## Features

- **Automated versioning** using timestamp (no manual version management)
- **Custom Sceptre hooks**:
  - Upload build and hardening components to S3 automatically
  - Start the image pipeline on update (uncomment the after_launch hook if you want to use this)
- **KMS CMK** for secure AMI encryption and sharing within the AWS Organization
- **Pre-checks and Guardrails**:
  - `yamllint` for validating YAML structure
  - `flake8` for Python code style compliance
  - `bandit` for Python security issues
  - `checkov` for static security analysis of IaC (CloudFormation templates)

---

## Pre-requisites

Ensure the deploying IAM entity has permissions to manage:

- EC2
- S3
- KMS
- Image Builder
- CloudFormation
- IAM

---

### First Time Deployment
Change these values in `sceptre/config/config.yaml`

``` yaml
# Replace these account IDs to ones you want to share these images to
account_ids_to_share_ami: 123456789011, 123456789012, 123456789013

# Manually create an EC2 Key Pair to reference
key_pair: ami-builder

# Select a name for the S3 Bucket and versioning state
bucket_name: image-builder-artifacts
bucket_versioning: Enabled

# Organization ID for KMS to share the AMIs to other accounts
org_id: o-123456
```

Also:

- Update the `SubnetId` in each `image-creation-stack.yaml`. The current value is specific to a personal environment.

---

## Deployment Options

### Pipeline Deployment

Pipeline deployment can be used if the AWS Account is configured with:

- An **OpenID Connect (OIDC) provider**
- An IAM **role** with permissions to deploy all necessary resources

See: [Use IAM Roles to Connect GitHub Actions to AWS](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/)

Create GitHub Actions variables:
- `AWS_REGION`
  - e.g. `ap-southeast-2`
- `IAM_ROLE`
  - This is the arn of the IAM Role that will be assumed to run the Pipeline.
  - e.g. `arn:aws:iam::123456789012:role/ExampleRole`


**To deploy:**

1. Create a feature branch from `main`
2. Open a pull request to `main`
3. Wait for Linting and Security tools to run
4. Merge when checks pass
5. Manually run image `workflow`

---

### Local Deployment

To deploy resources locally, use the following Sceptre commands:

```shell
# Setup Virtual Environment in the directory of the repo
python -m venv .venv

# Activate Virtual Environment
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run pre-checks
yamllint sceptre
flake8 sceptre
bandit -r sceptre
checkov -d sceptre

# Launch image stacks (update paths as needed)
sceptre launch -y images/linux/amazon-2023/infra/image-creation-stack.yaml
sceptre launch -y images/linux/rhel-9/infra/image-creation-stack.yaml
sceptre launch -y images/windows/server-2022/infra/image-creation-stack.yaml
sceptre launch -y images/windows/server-2025/infra/image-creation-stack.yaml
```
