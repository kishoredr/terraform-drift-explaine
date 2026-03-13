# Change Type Glossary

Reference for interpreting Terraform plan symbols and their precise meaning.

---

## Symbol Reference Table

| Symbol | Label | Meaning | Risk Floor |
|---|---|---|---|
| `+` | **create** | New resource will be provisioned | LOW |
| `-` | **destroy** | Existing resource will be permanently deleted | HIGH–CRITICAL |
| `~` | **update in-place** | Resource modified without recreation; no downtime from TF itself | LOW–MEDIUM |
| `-/+` | **destroy-and-replace** | Resource deleted then re-created; causes downtime and possible data loss | HIGH–CRITICAL |
| `+/-` | **create-and-destroy** | Like `-/+` but new resource created first (`create_before_destroy = true`) | MEDIUM–HIGH |
| `<=` | **read** | Data source refreshed; no infrastructure change | LOW |
| `->` | **attribute update** | Inline shorthand showing old value → new value | Depends on attribute |
| `(known after apply)` | **computed** | Final value only known post-apply; may hide breaking changes | Treat as unknown |
| `# forces replacement` | **replacement trigger** | The specific attribute causing `-/+`; always call this out explicitly | Matches parent |

---

## Reading a Resource Block

```hcl
# aws_db_instance.main must be replaced
-/+ resource "aws_db_instance" "main" {
      ~ allocated_storage       = 100 -> 200
      ~ db_subnet_group_name    = "prod-subnet" -> "prod-subnet-v2" # forces replacement
        engine                  = "postgres"
      + deletion_protection     = true
    }
```

Parse this as:
1. `-/+` → destroy-and-replace → CRITICAL candidate
2. `db_subnet_group_name` has `# forces replacement` → this attribute is the trigger
3. `allocated_storage` would have been fine as in-place (`~`) but is carried along
4. `deletion_protection` being added is LOW risk but won't prevent the recreation

---

## Common Confusions

### "Why is an update triggering a replacement?"
Not all `~` (update) attributes can be updated in-place. AWS (and other providers) require
certain parameters to cause recreation. The plan always annotates the culprit with
`# forces replacement`. If you see a `-/+` block, hunt for that annotation first.

### "Why do I see both `~` and `-/+` for the same resource?"
Terraform shows ALL attribute changes inside a `-/+` block using `~`, even though the
whole resource is being replaced. Don't read individual `~` lines as safe in this context.

### "What does `(known after apply)` hide?"
Computed values can mask critical changes. For example, a new security group's ID is
only known after creation — so any resource depending on it will also show
`(known after apply)` for that reference. In replacement chains, this can cascade.

### "`<= data source` with no changes — is that safe?"
Yes. A data source `<=` read means Terraform is refreshing its local copy of remote
state. No infrastructure is modified.

---

## Replacement-Safe Patterns

When a resource must change an attribute that forces replacement, prefer:

1. **`create_before_destroy` lifecycle rule** — new resource created, traffic migrated,
   old resource destroyed. Prevents downtime but not always possible (unique names, etc.)

2. **Blue-green via module** — provision a new resource in parallel, update all
   references, then destroy old one in a separate apply.

3. **`terraform state mv`** — if the resource is logically identical but Terraform
   tracks it under a new address, move state instead of replacing the resource.

4. **`terraform import`** — if resource already exists but isn't in state, import it
   rather than allowing TF to create (and possibly conflict with) the real thing.
