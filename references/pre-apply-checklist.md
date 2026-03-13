# Pre-Apply Checklist Item Bank

Select only the items relevant to the specific plan being reviewed.
Do not include items for resource types not present in the plan.

---

## Always Include

- [ ] Back up the current state file before applying:
  ```bash
  terraform state pull > terraform-backup-$(date +%Y%m%d-%H%M%S).tfstate
  ```
- [ ] Confirm you are in the correct Terraform workspace:
  ```bash
  terraform workspace show
  ```
- [ ] Confirm the correct AWS/GCP/Azure account/project is targeted:
  ```bash
  aws sts get-caller-identity        # AWS
  gcloud config get-value project    # GCP
  az account show                    # Azure
  ```

---

## Database Resources (RDS, Aurora, Cloud SQL, Azure SQL)

- [ ] Take a manual DB snapshot before applying:
  ```bash
  # AWS RDS
  aws rds create-db-snapshot \
    --db-instance-identifier <identifier> \
    --db-snapshot-identifier pre-tf-apply-$(date +%Y%m%d)

  # Aurora cluster
  aws rds create-db-cluster-snapshot \
    --db-cluster-identifier <cluster-id> \
    --db-cluster-snapshot-identifier pre-tf-apply-$(date +%Y%m%d)
  ```
- [ ] Verify `skip_final_snapshot = false` is set in Terraform before applying any destroy
- [ ] Confirm `deletion_protection = true` is set on production databases
- [ ] Notify application teams of expected maintenance window
- [ ] If `multi_az` is being enabled/disabled: expect a failover event; verify replica lag is zero first
- [ ] If parameter group is changing: confirm which parameters require a DB reboot to take effect

---

## EKS / Kubernetes

- [ ] Verify cluster is healthy before applying:
  ```bash
  kubectl get nodes
  kubectl get pods --all-namespaces | grep -v Running
  ```
- [ ] If node group is being replaced: cordon and drain existing nodes first:
  ```bash
  kubectl cordon <node>
  kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
  ```
- [ ] Confirm PodDisruptionBudgets will not block the drain:
  ```bash
  kubectl get pdb --all-namespaces
  ```
- [ ] Check that `aws-node`, `kube-proxy`, and CoreDNS are running with sufficient replicas to survive node drain
- [ ] If EKS version is changing: review AWS EKS upgrade documentation for the specific version pair
- [ ] Verify IRSA / service account annotations will not be disrupted by node group replacement

---

## Networking (VPC, Subnets, Security Groups, Route Tables)

- [ ] If removing a security group rule: confirm no active connections depend on it:
  ```bash
  aws ec2 describe-network-interfaces --filters "Name=group-id,Values=<sg-id>"
  ```
- [ ] If changing route table associations: verify fallback routing exists
- [ ] If VPC or subnet is being replaced: all resources inside (EC2, RDS, EKS) will need re-provisioning
- [ ] Notify security team if WAF, NACLs, or security group rules are changing

---

## Load Balancers (ALB, NLB, CloudFront)

- [ ] If ALB is being replaced: DNS name will change; update all CNAME/Alias records
- [ ] If HTTPS listener is being modified: verify certificate ARN is valid and not expired:
  ```bash
  aws acm describe-certificate --certificate-arn <arn>
  ```
- [ ] If target group health check settings are changing: verify current instance health
- [ ] If CloudFront distribution is being updated: propagation takes 15–30 minutes; warn users

---

## IAM

- [ ] If a role policy attachment is being removed: identify all services/apps using this role:
  ```bash
  aws iam get-role --role-name <role-name>
  aws iam list-attached-role-policies --role-name <role-name>
  ```
- [ ] If an IAM policy document is changing: review the diff carefully for privilege escalation or regression
- [ ] If a role trust policy is changing: confirm the new principal can still assume the role
- [ ] Run IAM Access Analyzer to verify no unintended public access is introduced

---

## KMS Keys

- [ ] NEVER apply KMS key deletion without verifying nothing depends on it:
  ```bash
  aws kms list-grants --key-id <key-id>
  aws kms get-key-rotation-status --key-id <key-id>
  ```
- [ ] KMS key deletion has a mandatory 7–30 day waiting period; verify `deletion_window_in_days` is set
- [ ] If key policy is changing: confirm all key users and key admins are still present

---

## S3

- [ ] If bucket is being destroyed: verify objects are either deleted or backed up
- [ ] If bucket policy is changing: check for accidental public access:
  ```bash
  aws s3api get-bucket-policy-status --bucket <bucket-name>
  ```
- [ ] If versioning is being disabled: understand that existing versions are not deleted but cannot be restored via versioning

---

## ElastiCache (Redis / Memcached)

- [ ] If cluster is being replaced: warm up application caches after apply; expect cache-miss spike
- [ ] Notify application teams of cache flush (especially for session stores)
- [ ] Verify application has fallback behavior for cache miss (e.g., DB fallback)

---

## DNS (Route53, Cloud DNS)

- [ ] If records are being deleted: check if any active traffic routes through them:
  ```bash
  dig <record-name> @8.8.8.8
  ```
- [ ] If TTL is being lowered before a cutover: apply the low TTL plan first, wait for propagation, then do the actual cutover
- [ ] For production domains: notify users of expected DNS propagation delay (up to 48h for high TTL)

---

## Lambda

- [ ] If environment variables are changing: verify secrets are not being cleared or overwritten
- [ ] If memory or timeout is changing: test in staging first; Lambda costs scale with both
- [ ] If VPC configuration is changing: cold start latency may increase significantly

---

## ECS

- [ ] If service is being replaced: verify minimum healthy percent and maximum percent settings allow rolling update:
  ```bash
  aws ecs describe-services --cluster <cluster> --services <service>
  ```
- [ ] Confirm new task definition is stable in staging before applying to production

---

## General Production Checklist

- [ ] Apply during a low-traffic window if any HIGH/CRITICAL changes are present
- [ ] Have a rollback plan documented before applying
- [ ] Assign an engineer to monitor dashboards/alarms during and 30 minutes after apply
- [ ] Set up a war room channel or incident bridge in case issues arise
- [ ] Run `terraform plan` one final time immediately before applying to catch any new drift
