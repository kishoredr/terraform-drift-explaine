# terraform-drift-explainer

> A Claude skill that transforms raw `terraform plan` output into a risk-tiered,
> human-readable brief with a clear Go / No-Go verdict.

---

## What This Skill Does

Terraform plan output is dense and unforgiving. A 300-line diff mixing harmless
metadata updates with a silent `# forces replacement` on your production RDS instance
looks identical at a glance. This skill gives Claude the specialized knowledge to:

- **Parse** every resource change and identify the exact change type
- **Classify** each change by severity (🔴 CRITICAL → 🟢 LOW) using a curated risk matrix
- **Diagnose** root causes (manual edits, provider upgrades, state desync, external automation)
- **Generate** a tailored pre-apply checklist with exact CLI commands
- **Deliver** a Go / Conditional Go / No-Go verdict with clear rationale
- **Suggest** IaC hardening to prevent the same class of issue from recurring

---

## When It Triggers

Claude will activate this skill automatically when you:

- Paste `terraform plan` output into the chat
- Paste `terraform show` or state drift output
- Ask "is it safe to apply this plan?"
- Ask "why is Terraform trying to replace my [resource]?"
- Ask "what does `forces replacement` mean here?"
- Share a `.tfplan` file
- Ask about unexpected infrastructure changes

---

## File Structure

```
terraform-drift-explainer/
├── SKILL.md                          # Main skill instructions (Claude reads this)
└── references/
    ├── change-type-glossary.md       # Terraform symbol definitions (+, -, ~, -/+, <=)
    ├── severity-matrix.md            # Risk classification table (50+ resource types)
    ├── drift-causes.md               # Root cause pattern library (8 patterns)
    ├── pre-apply-checklist.md        # Checklist item bank by resource type
    └── iac-hardening.md              # IaC hardening suggestions catalogue
```

---

## Output Format

Every analysis produces a structured brief:

```
📋 Terraform Plan Analysis

Summary
  [2-sentence executive summary]

Risk Register
  🔴 module.database.aws_db_instance.main
    Action:     destroy-and-replace
    Risk:       CRITICAL — DATA_LOSS
    What:       The RDS instance will be destroyed and recreated...
    Why:        db_subnet_group_name forces replacement (Pattern: manual console edit)
    Confidence: High

  🟠 aws_security_group_rule.allow_https
    Action:     destroy
    Risk:       HIGH — SECURITY
    ...

⚠️ Replacement Warnings
  [Exact attribute, data loss risk, safer alternative]

✅ Pre-Apply Checklist
  [ ] Take RDS snapshot before applying: aws rds create-db-snapshot ...
  [ ] Back up state file: terraform state pull > backup.tfstate
  ...

Verdict: NO-GO
Reason:  Production RDS instance will be destroyed without a final snapshot.
Before applying:
  - Set skip_final_snapshot = false or take a manual snapshot first
  - Confirm replacement is intentional and not caused by subnet group drift

🔧 IaC Hardening Suggestions
  1. Add lifecycle { prevent_destroy = true } to aws_db_instance.main
  2. Add scheduled drift detection to your CI pipeline
```

---

## Installation

### Option A: Claude Skills Directory

Copy the `terraform-drift-explainer/` folder into your Claude skills directory:

```bash
cp -r terraform-drift-explainer/ ~/.claude/skills/
```

### Option B: Manual

Place `SKILL.md` and the `references/` folder anywhere Claude can read them and
configure your Claude setup to include this skill.

---

## Coverage

### Providers
- **AWS** (primary) — 50+ resource types with specific guidance
- **Google Cloud** — GKE, Cloud SQL, GCS
- **Azure** — SQL Server, AKS
- **Kubernetes** — CRD and cluster-scoped resources
- Generic patterns applicable to any provider

### Root Cause Patterns
1. Manual console edits
2. Provider version upgrades
3. External automation / third-party mutation
4. State desync
5. Argument rename / schema change
6. Missing `lifecycle` block
7. Workspace / variable mismatch
8. Sensitive value changed externally (secrets rotation)

---

## Contributing

PRs welcome for:
- New resource types in the severity matrix
- Additional root cause patterns
- GCP / Azure provider-specific gotchas
- New IaC hardening patterns

Please include the resource type, the specific change scenario, and a brief
explanation of the failure mode when adding to the severity matrix.

---

## License

MIT
