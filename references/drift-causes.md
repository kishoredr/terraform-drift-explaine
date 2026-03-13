# Drift Root Cause Pattern Library

Use this reference to diagnose *why* drift or unexpected plan changes are occurring.
For each CRITICAL or HIGH change, pick the most likely root cause from this library
and state your confidence level (High / Medium / Low).

---

## Pattern 1: Manual Console Edit

**Symptoms**
- Attribute in plan differs from code, but code hasn't changed recently
- Single attribute mismatch (e.g., instance type, tag value, security group rule)
- No recent git changes to the `.tf` file for that resource

**Common culprits**
- EC2 instance type changed via Console
- Security group rule added/removed in Console
- RDS parameter changed via Console
- S3 bucket policy edited in Console

**Diagnosis signal**: Run `git log -p -- path/to/resource.tf` — if no recent changes,
it's almost certainly a manual edit.

**Remediation**
```bash
# Option A: Accept console state, update TF code to match
terraform show -json | jq '.values.root_module.resources[] | select(.address == "RESOURCE")'
# Then update the .tf file to match

# Option B: Revert console change, let Terraform restore intended state
# Apply the plan after verifying this is safe
```

---

## Pattern 2: Provider Version Upgrade

**Symptoms**
- Plan shows changes after bumping `required_providers` version
- New attributes appear with default values
- Existing attributes renamed or split into sub-blocks
- `# forces replacement` on attributes you didn't change

**Common culprits**
- `hashicorp/aws` 4.x → 5.x: major breaking changes to S3, VPC, SGs
- `hashicorp/kubernetes` 2.x: label/annotation handling changed
- `hashicorp/google` 4.x → 5.x: breaking changes to compute resources

**Diagnosis signal**: Check `CHANGELOG.md` of the provider. Look for "Breaking Changes"
or "Deprecated" sections between your old and new version.

**Remediation**
```bash
# Pin to previous working version temporarily
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.67"  # pin until migration is done
    }
  }
}
```

---

## Pattern 3: External Automation / Third-Party Mutation

**Symptoms**
- Resources show drift without any Terraform changes
- The changed values look like they came from an automated process
- Common in: ASG desired count, ECS task count, security group rules, tags

**Common culprits**
- AWS Auto Scaling adjusting `desired_capacity` → TF wants to reset it
- Kubernetes Cluster Autoscaler changing node group sizes
- AWS Config remediation adding/removing tags
- CI/CD pipeline injecting environment variables or tags
- AWS Security Hub adding tags for compliance

**Diagnosis signal**: Check CloudTrail for API calls on the resource in the last 24h.
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=RESOURCE_ID \
  --start-time 2024-01-01T00:00:00Z
```

**Remediation**
```hcl
# Ignore externally managed attributes
resource "aws_autoscaling_group" "main" {
  lifecycle {
    ignore_changes = [desired_capacity, tag]
  }
}
```

---

## Pattern 4: State Desync (Resource Exists Outside State)

**Symptoms**
- Terraform plans to create a resource that already exists in the cloud
- Error: `already exists`, `duplicate`, `conflict`
- Or: Terraform plans to destroy a resource that doesn't exist

**Common culprits**
- Resource was created manually and never imported
- `terraform import` was run but missed a child resource
- State file was partially lost or restored from an old backup
- Module was refactored and resource address changed without `terraform state mv`

**Diagnosis signal**:
```bash
# Check if resource exists in state
terraform state list | grep RESOURCE_NAME

# Check if resource exists in cloud but not state
aws <service> describe-<resource> --name RESOURCE_NAME
```

**Remediation**
```bash
# Import existing resource into state
terraform import aws_db_instance.main db-instance-identifier

# Move resource to new address after refactor
terraform state mv module.old.aws_vpc.main module.new.aws_vpc.main
```

---

## Pattern 5: Argument Rename / Schema Change

**Symptoms**
- Attribute in plan has a new name but same value
- Old attribute appears as being removed, new one added
- Common after provider major version bumps

**Common culprits**
- `aws` provider 4→5: `aws_security_group` inline rules → standalone `aws_vpc_security_group_*_rule`
- `aws` provider: `aws_s3_bucket` ACL, versioning, logging split into separate resources
- `google` provider: `google_container_cluster` node config restructured

**Diagnosis signal**: Provider CHANGELOG shows "argument renamed" or "moved to separate resource".

**Remediation**: Migrate to new resource schema. Do not try to force the old attribute
name — the provider will reject it or silently ignore it.

---

## Pattern 6: Missing `lifecycle` Block on Replacement-Sensitive Resource

**Symptoms**
- Plan shows `-/+` (destroy-replace) for a resource where you expected in-place update
- The replacement trigger is an argument that logically shouldn't require recreation

**Common culprits**
- RDS: `db_subnet_group_name`, `availability_zone`, `multi_az` without `apply_immediately`
- EKS node group: `launch_template` version, `instance_types`, `subnet_ids`
- ElastiCache: `subnet_group_name`, `parameter_group_name`
- SG: inline `ingress`/`egress` blocks mixed with standalone rules

**Remediation**
```hcl
resource "aws_db_instance" "main" {
  # ... config ...

  lifecycle {
    create_before_destroy = true   # provision new first, then destroy old
    prevent_destroy       = true   # block accidental terraform destroy
    ignore_changes        = [snapshot_identifier]  # ignore rotation-managed attributes
  }
}
```

---

## Pattern 7: Workspace / Variable Mismatch

**Symptoms**
- Plan looks correct in dev but shows major changes in prod
- Variables passed via `.tfvars` or environment differ from expected
- `terraform.workspace` interpolation resolving to wrong value

**Diagnosis signal**:
```bash
terraform workspace show        # confirm active workspace
terraform console               # interactive: type var.environment to inspect
cat terraform.tfvars            # verify var file in use
env | grep TF_VAR_              # check environment variable overrides
```

**Remediation**: Always run `terraform workspace select <name>` before planning in
multi-workspace setups. Use `-var-file=prod.tfvars` explicitly in CI.

---

## Pattern 8: Sensitive Value Changed Externally (Secrets Rotation)

**Symptoms**
- Resource shows `~` for a sensitive attribute shown as `(sensitive value)`
- Common in: RDS `password`, Lambda `environment`, SSM parameters

**Common culprits**
- AWS Secrets Manager rotation updated the master password
- Manual password rotation
- Vault dynamic credential lease expired

**Remediation**
```hcl
resource "aws_db_instance" "main" {
  lifecycle {
    ignore_changes = [password]  # let secrets manager own this
  }
}
```

---

## Confidence Level Guide

| Confidence | When to use |
|---|---|
| **High** | You can see the exact matching pattern — the evidence is unambiguous |
| **Medium** | Pattern fits but you cannot confirm without CloudTrail/git history |
| **Low** | Multiple patterns could explain the drift; recommend the user investigates before applying |
