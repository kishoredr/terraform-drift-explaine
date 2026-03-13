# IaC Hardening Suggestions Catalogue

After delivering a plan analysis, recommend 1–3 suggestions from this catalogue
based on what issues were found. Match suggestions to the specific problem class.

---

## Category 1: Lifecycle Protection

### 1.1 Prevent Accidental Destruction
**When to suggest**: Plan shows a `-` or `-/+` on a stateful resource (RDS, S3, EKS, DynamoDB)

```hcl
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

> This causes `terraform destroy` or any plan that would destroy this resource to fail
> with an explicit error, forcing intentional override.

---

### 1.2 Create Before Destroy
**When to suggest**: Plan shows `-/+` and downtime is the concern

```hcl
resource "aws_elasticache_replication_group" "main" {
  # ...
  lifecycle {
    create_before_destroy = true
  }
}
```

> Note: Not always possible — resources with unique names or fixed endpoints cannot
> have two instances simultaneously.

---

### 1.3 Ignore Externally Managed Attributes
**When to suggest**: Drift caused by autoscaling, secrets rotation, or external automation

```hcl
resource "aws_autoscaling_group" "main" {
  # ...
  lifecycle {
    ignore_changes = [
      desired_capacity,  # managed by autoscaling policies
      tag,               # managed by AWS Config / tagging automation
    ]
  }
}
```

---

## Category 2: Precondition & Validation

### 2.1 Precondition Checks
**When to suggest**: Plan has changes that should only proceed if certain conditions are met

```hcl
resource "aws_db_instance" "main" {
  # ...

  lifecycle {
    precondition {
      condition     = var.environment != "prod" || var.multi_az == true
      error_message = "Production RDS must have multi_az enabled."
    }
  }
}
```

> Terraform 1.2+. Fails the plan with a clear message before any apply happens.

---

### 2.2 Variable Validation
**When to suggest**: Risk from incorrect variable values (wrong environment, wrong size)

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

variable "db_instance_class" {
  type        = string

  validation {
    condition     = !startswith(var.db_instance_class, "db.t2.") || var.environment != "prod"
    error_message = "Do not use db.t2 instance classes in production."
  }
}
```

---

## Category 3: CI/CD Plan Gating

### 3.1 Saved Plan Files (Prevent Plan-Apply Drift)
**When to suggest**: Team applies plans without re-running them; risk that infrastructure changed between plan and apply

```bash
# In CI pipeline:
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json

# Review tfplan.json before applying
# Then apply the exact saved plan:
terraform apply tfplan.binary
```

> The saved `.binary` plan is cryptographically tied to the state at plan time.
> `terraform apply tfplan.binary` cannot diverge from what was reviewed.

---

### 3.2 tfsec / Checkov Static Analysis
**When to suggest**: IAM or security-related changes found in the plan

```yaml
# GitHub Actions example
- name: Run tfsec
  uses: aquasecurity/tfsec-action@v1.0.0
  with:
    soft_fail: false

- name: Run Checkov
  uses: bridgecrewio/checkov-action@master
  with:
    directory: .
    framework: terraform
```

---

### 3.3 Atlantis / Terraform Cloud for PR-Based Plans
**When to suggest**: Team is running `terraform plan` and `apply` manually; no review gate

> Consider adopting [Atlantis](https://www.runatlantis.io/) or Terraform Cloud to:
> - Auto-run `terraform plan` on every PR
> - Require plan approval before `terraform apply`
> - Store plan output as PR comment for audit trail

---

## Category 4: State Safety

### 4.1 Remote State with Locking
**When to suggest**: No evidence of remote state, or state locking not mentioned

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tfstate-bucket"
    key            = "prod/terraform.tfstate"
    region         = "eu-central-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # DynamoDB table for locking
  }
}
```

> Without locking, concurrent applies from two engineers can corrupt state.

---

### 4.2 State Backup Before High-Risk Applies
**When to suggest**: CRITICAL changes are present in the plan

```bash
#!/bin/bash
# pre-apply-backup.sh — run before any high-risk terraform apply
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
WORKSPACE=$(terraform workspace show)
terraform state pull > "tfstate-backup-${WORKSPACE}-${TIMESTAMP}.tfstate"
echo "State backed up to: tfstate-backup-${WORKSPACE}-${TIMESTAMP}.tfstate"
```

---

## Category 5: Drift Detection

### 5.1 Scheduled Drift Detection
**When to suggest**: Drift was caused by manual console edits or external automation going undetected

```yaml
# GitHub Actions — run terraform plan nightly and alert on drift
name: Drift Detection
on:
  schedule:
    - cron: '0 6 * * *'  # 6am UTC daily

jobs:
  drift-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Terraform Plan
        run: terraform plan -detailed-exitcode
        # exit code 2 = changes detected = drift
      - name: Alert on Drift
        if: failure()
        run: |
          # Send Slack/PagerDuty alert
          echo "Drift detected in production infrastructure"
```

---

### 5.2 AWS Config Rules
**When to suggest**: Drift from external automation or manual changes

> Enable AWS Config with rules such as:
> - `required-tags` — alert when resources lack required tags
> - `restricted-ssh` — alert when SSH is opened to `0.0.0.0/0`
> - `rds-instance-deletion-protection-enabled`
> - `eks-cluster-secrets-encrypted`

---

## Suggestion Selection Guide

| Problem Found | Suggest |
|---|---|
| Stateful resource being destroyed | 1.1 `prevent_destroy` |
| Downtime from resource replacement | 1.2 `create_before_destroy` |
| Drift from autoscaling/automation | 1.3 `ignore_changes` |
| Wrong variable value risk | 2.2 Variable validation |
| Security group / IAM changes | 3.2 tfsec/Checkov |
| Manual apply workflow | 3.1 Saved plan files OR 3.3 Atlantis |
| No state locking evidence | 4.1 Remote state with locking |
| CRITICAL changes present | 4.2 State backup script |
| Console-edit drift detected | 5.1 Scheduled drift detection |
