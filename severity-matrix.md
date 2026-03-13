# Severity Matrix

Full risk classification table for Terraform resource changes.
Classify every changed resource against this table before producing the brief.

---

## 🔴 CRITICAL

These changes can cause **data loss**, **extended downtime**, or **irreversible destruction**
of production infrastructure. Require explicit acknowledgement + mitigation plan before applying.

| Resource Type | Trigger | Failure Mode | Notes |
|---|---|---|---|
| `aws_db_instance` | `-` or `-/+` | DATA_LOSS | RDS deletion without snapshot = permanent data loss |
| `aws_rds_cluster` | `-` or `-/+` | DATA_LOSS | Aurora cluster; check `skip_final_snapshot = false` |
| `aws_dynamodb_table` | `-` or `-/+` | DATA_LOSS | Table deletion destroys all data |
| `aws_s3_bucket` | `-` | DATA_LOSS | Bucket deletion; objects may or may not be deleted |
| `aws_elasticache_replication_group` | `-/+` | DATA_LOSS + DOWNTIME | Redis replacement wipes cache; apps may fail on cold start |
| `aws_elasticache_cluster` | `-/+` | DATA_LOSS + DOWNTIME | Memcached cluster replacement |
| `aws_eks_node_group` | `-/+` | DOWNTIME | Node group replacement drains and replaces all nodes |
| `aws_eks_cluster` | `-/+` | DOWNTIME | Control plane replacement; workloads unavailable |
| `aws_elasticsearch_domain` / `aws_opensearch_domain` | `-/+` | DATA_LOSS + DOWNTIME | Index data can be lost |
| `aws_ebs_volume` | `-` | DATA_LOSS | Volume deletion; ensure snapshot exists |
| `aws_efs_file_system` | `-` | DATA_LOSS | EFS deletion |
| `aws_kms_key` | `-` | SECURITY + DATA_LOSS | KMS key deletion makes encrypted data permanently unreadable |
| `aws_cognito_user_pool` | `-` | DATA_LOSS | User pool deletion removes all user accounts |
| `google_sql_database_instance` | `-/+` | DATA_LOSS | Cloud SQL instance replacement |
| `google_container_cluster` | `-/+` | DOWNTIME | GKE cluster replacement |
| `azurerm_sql_server` / `azurerm_mssql_server` | `-/+` | DATA_LOSS | SQL Server replacement |
| Any resource | `-/+` with `deletion_protection` warning | DATA_LOSS | Deletion protection was bypassed |

---

## 🟠 HIGH

These changes can cause **partial outages**, **access control regressions**, or
**breaking changes** to dependent systems. Require careful review before applying.

| Resource Type | Trigger | Failure Mode | Notes |
|---|---|---|---|
| `aws_security_group_rule` | `-` | SECURITY | Removing an ingress/egress rule may block live traffic |
| `aws_security_group` | `-/+` | SECURITY + DOWNTIME | SG replacement changes SG ID; all references break |
| `aws_iam_role_policy_attachment` | `-` | SECURITY | Removing a policy may break services relying on that permission |
| `aws_iam_policy` | `~` on `policy` document | SECURITY | Policy change may expand or restrict permissions unexpectedly |
| `aws_lb_listener` / `aws_alb_listener` | `-/+` or `-` | DOWNTIME | Load balancer listener change disrupts routing |
| `aws_lb` / `aws_alb` | `-/+` | DOWNTIME + BREAKING_CHANGE | ALB replacement changes DNS name and ARN |
| `aws_route53_record` | `-` | BREAKING_CHANGE | DNS record removal breaks routing for dependent services |
| `aws_cloudfront_distribution` | `-/+` | DOWNTIME | Distribution recreation can take 15–30 minutes |
| `aws_acm_certificate` | `-` | BREAKING_CHANGE | Certificate deletion breaks HTTPS for associated resources |
| `aws_ssm_parameter` | `-` | BREAKING_CHANGE | Apps reading this parameter at runtime will fail |
| `aws_secretsmanager_secret` | `-` | BREAKING_CHANGE + SECURITY | Secret deletion; apps fail to retrieve credentials |
| `aws_autoscaling_group` | `-/+` | DOWNTIME | ASG replacement terminates all current instances |
| `aws_launch_template` | `-/+` | DOWNTIME | New ASG instances get new launch template; rolling update may cause downtime |
| `aws_ecs_service` | `-/+` | DOWNTIME | ECS service replacement; tasks killed and restarted |
| `aws_ecs_task_definition` | `~` with CPU/memory change | DOWNTIME | ECS tasks replaced with new definition |
| `aws_vpc` | `-/+` | DOWNTIME | VPC replacement breaks all subnet, SG, and routing resources |
| `aws_subnet` | `-/+` | DOWNTIME | Subnet replacement may cause ENI and instance failure |
| `aws_internet_gateway` | `-` | DOWNTIME | Removes public internet access for entire VPC |
| `aws_kms_key` | `~` on `key_usage` or rotation | SECURITY | Key policy changes affect all encrypted resources |
| `kubernetes_*` CRD or cluster-scoped | `-/+` | BREAKING_CHANGE | CRD replacements can orphan all custom resources |

---

## 🟡 MEDIUM

Changes that are unlikely to cause outages but deserve a second look before production applies.

| Resource Type | Trigger | Notes |
|---|---|---|
| Any resource | Tag-only `~` changes | Low risk but verify tagging strategy is intentional |
| `aws_autoscaling_group` | `~` on `desired_capacity` | Scaling change; may terminate running instances if scaling down |
| `aws_cloudwatch_metric_alarm` | `~` on `threshold` or `comparison_operator` | Alert sensitivity changes; may miss or over-fire |
| `aws_wafv2_web_acl` | `~` | WAF rule changes; verify no valid traffic is now blocked |
| `aws_lambda_function` | `~` on `memory_size`, `timeout`, `environment` | Config change; test impact on cold starts and downstream timeouts |
| `aws_api_gateway_*` | `~` | API config change; may affect clients on cached stages |
| `aws_s3_bucket_policy` | `~` | Bucket policy change; verify access is not accidentally widened |
| `aws_iam_role` | `~` on `assume_role_policy` | Trust policy change; verify service can still assume the role |
| `aws_db_parameter_group` | `~` | DB parameter changes; some require reboot to take effect |
| `aws_elasticache_parameter_group` | `~` | Cache parameter changes; some require cluster reboot |

---

## 🟢 LOW

Routine changes. Verify they're intentional, then proceed.

| Resource Type | Trigger | Notes |
|---|---|---|
| Any | `<=` data source read | No infrastructure change |
| Any | `+` new resource creation (non-stateful) | Verify naming and placement |
| `aws_cloudwatch_log_group` | Any `~` | Retention/tag changes; no traffic impact |
| `aws_iam_role` | `+` new role | New role; verify it's scoped correctly |
| `null_resource` / `terraform_data` | Any | Local/provisioner state; not real infrastructure |
| `random_*` | `~` | Random value change; only matters if used in resource names that force replacement |
| Any | Output value changes | Outputs are informational only |
| Provider version | `~` | Provider version bump; check changelogs for breaking changes separately |

---

## Modifier: Production vs. Non-Production

Upgrade severity by one level if the workspace/module path indicates production:
- Path contains: `prod`, `production`, `prd`, `main`, `live`
- Workspace named: `prod`, `production`
- Variable `environment = "prod"`

Example: A MEDIUM tag change in prod should be treated with the same care as HIGH.
