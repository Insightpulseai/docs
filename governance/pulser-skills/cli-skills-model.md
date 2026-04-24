# Pulser CLI Skills Model

**Version**: 1.0  
**Status**: Active  
**Last Updated**: 2026-04-25

---

## Executive Summary

CLI tools are powerful but dangerous when exposed directly to agents. Pulser wraps CLI commands behind **typed, governed interfaces** called **CLI skills**. This prevents unrestricted shell access while allowing agents to safely accomplish bounded tasks.

**Core Rule**: 
```
CLI tool ≠ skill directly
CLI tool + contract + policy + sandbox + evidence = skill
```

This document defines the Pulser CLI skills architecture, initial skill set, risk classification, and implementation patterns.

---

## 1. Why CLI Skills Matter

### The Problem
- **Raw CLI access**: Agents get full shell authority → potential for data exfiltration, unauthorized deployments, accidental destruction
- **No auditability**: Shell commands are invisible to governance
- **No RBAC**: All users run all commands the same way
- **No approval gates**: Destructive operations happen without human review

### The Solution
- **Typed contracts**: Inputs/outputs are schema-validated
- **Allowlisting**: Only authorized repos, environments, resources
- **Command adapters**: Agents cannot invoke arbitrary commands; only pre-defined adapters
- **Evidence tracking**: All skill invocations are logged + auditable
- **Risk-based policies**: Read-only skills auto-allowed; destructive skills require approval

---

## 2. Core Architecture

### Layers

| Layer | Component | Example | Purpose |
|-------|-----------|---------|---------|
| **1. Raw CLI** | Binary/executable | `gh`, `az`, `odoo-bin`, `pac` | Actual tool |
| **2. Skill Wrapper** | Typed interface | `pulser.github.pr_status` | Task abstraction |
| **3. Contract** | JSON/YAML schema | `inputs`, `outputs`, `denied` | Input/output validation |
| **4. Adapter** | Python/JS code | `github_cli.py` | CLI invocation logic |
| **5. Policy** | MCP/RBAC rules | `cli-skill-policy.yaml` | Who can run what |
| **6. Sandbox** | Execution environment | `local`, `devcontainer`, `CI` | Where it runs |
| **7. Evidence** | Logs + artifacts | `skill-run.log`, `skill-run.json` | Auditability |

### Data Flow

```
Agent Request
    ↓
Skill Input Validation (schema)
    ↓
RBAC Check (policy)
    ↓
Allowlist Check (repo/env/resource)
    ↓
Command Adapter (build CLI command)
    ↓
Sandbox Execution (isolated env)
    ↓
Output Capture + Sanitization
    ↓
Schema Validation (output)
    ↓
Evidence Logging
    ↓
Agent Response
```

---

## 3. Supported CLI Families

| CLI | Purpose | Pulser Skills | Risk | Mode |
|-----|---------|---------------|------|------|
| **gh** (GitHub CLI) | GitHub ops | `pulser.github.pr_status` `pulser.github.check_branch` | Low | Read-only |
| **az** (Azure CLI) | Azure ops | `pulser.azure.resource_inventory` `pulser.azure.env_status` | Low-Medium | Read + preview |
| **azd** (Azure Developer CLI) | Deployment | `pulser.azure.deploy_preview` | High | Deploy nonprod only |
| **databricks** | Data warehouse | `pulser.databricks.sql_status` `pulser.databricks.job_status` | Medium | Read + SQL |
| **pac** (Power Platform CLI) | Power Platform | `pulser.powerplatform.solution_validate` | Medium | Validate/export |
| **teamsapp** / **atk** (App Teams Toolkit) | M365 agents | `pulser.m365.agent_package` `pulser.m365.provision_sandbox` | Medium | Dev/sandbox only |
| **odoo-bin** | Odoo ERP | `pulser.odoo.module_test` `pulser.odoo.module_install` | High | Dev/test only |
| **docker compose** | Local stack | `pulser.local.stack_health` | Low | Local/dev only |
| **pytest** | Agent testing | `pulser.agentplatform.test` | Low | Agent-platform only |

**Explicitly NOT supported**:
- Generic `run_shell_command` (no arbitrary shell)
- Direct `pip install` (use `pip list` or managed requirements only)
- Direct database access (use `sqlalchemy` client, not `psql` shell)
- Production Terraform/Bicep apply (use preview/plan only)
- AWS CLI (not in initial Wave 1)

---

## 4. Risk Classification

### Read-Only (Auto-Allowed)
Examples: `gh pr view`, `az resource list`, `databricks catalogs list`, `docker ps`

**Policy**: Auto-allowed if:
- Scoped to allowlisted repos/resources
- No side effects on system state
- Output is sanitized for secrets

**Evidence**: Logged; no approval required

---

### Local Build / Test
Examples: `odoo-bin --test-enable`, `pytest`, `npm test`, `docker compose up -d`

**Policy**: Allowed in:
- Local development environment
- CI runner (GitHub Actions)
- Devcontainer only

**Denied**:
- On production infrastructure
- Concurrent with other deployments
- Without health check post-execution

**Evidence**: Full command, stdout/stderr, exit code, build artifacts

---

### Draft Write
Examples: Create evidence file, generate PR body, draft solution package, prepare deployment manifest

**Policy**:
- File created in temp/draft area only
- Output reviewable before commit
- No direct git push

**Approval**: Human review of draft before promotion

**Evidence**: Draft file, SHA, author, timestamp

---

### Deployment (Non-Prod)
Examples: `azd up --environment dev`, `docker compose up` (staging), Odoo module install (test DB)

**Policy**:
- Dev/staging/sandbox only
- Locked to specific environment
- Health check before + after
- Rollback available

**Approval**: Auto-allowed if health checks pass; human override available

**Evidence**: Deployment plan, execution log, health check results, rollback markers

---

### Deployment (Production)
Examples: `azd up --environment prod`, `pac solution import` (production Dataverse)

**Policy**: **BLOCKED by default**

**Approval**: Requires:
- Explicit ADR/RFC merged
- Human approval from appropriate group
- Change control ticket linked
- Pre-deployment validation passed
- Communication to stakeholders

**Evidence**: Full audit trail, approver identity, deployment log, rollback proof

---

### Admin / Destructive
Examples: `azd down`, RBAC policy changes, `databricks workspace delete`, `docker system prune -a`

**Policy**: **BLOCKED by default**; never auto-allowed

**Approval**: Requires:
- Emergency procedure OR
- Explicit human approval + 24h notice
- Change control + management sign-off
- Post-incident review

**Evidence**: Comprehensive audit trail, all approvals, detailed logs

---

## 5. Skill Contract Schema

All Pulser CLI skills must conform to this schema:

```yaml
# pulser/skills/cli/github_pr_status.yaml

name: pulser.github.pr_status
kind: cli_skill
version: "1.0"
runtime: local_or_ci
command_adapter: github_cli
risk: low
mode: read_only

# Human-readable description
description: >
  Reads GitHub pull request metadata including state, mergeability, CI checks,
  branches, and linked issues. Safe for any authorized user. No side effects.

# Input schema (JSON Schema)
inputs:
  repo:
    type: string
    required: true
    description: Repository in owner/name format
    allowlist:
      - Insightpulse-ai/docs
      - Insightpulse-ai/marketplace-publishing
      - Insightpulse-ai/platform
      - Insightpulse-ai/agent-platform
      - Insightpulse-ai/odoo
      - Insightpulse-ai/data-intelligence
      - Insightpulse-ai/.github

  pr_number:
    type: integer
    required: true
    description: Pull request number
    minimum: 1
    maximum: 999999

  include_checks:
    type: boolean
    required: false
    default: true
    description: Include CI check status

# Output schema (JSON Schema)
outputs:
  state:
    type: string
    enum: ["OPEN", "CLOSED", "MERGED"]
    description: PR state

  mergeable:
    type: string
    enum: ["MERGEABLE", "CONFLICTING", "UNKNOWN"]
    description: Mergeability status

  checks:
    type: array
    items:
      type: object
      properties:
        name:
          type: string
        status:
          type: string
          enum: ["success", "failure", "pending", "skipped"]
        url:
          type: string
    description: CI check results (if include_checks=true)

  head_ref:
    type: string
    description: Source branch name

  base_ref:
    type: string
    description: Target branch name

  title:
    type: string
    description: PR title

  author:
    type: string
    description: PR author GitHub handle

  created_at:
    type: string
    format: date-time

  updated_at:
    type: string
    format: date-time

  review_decision:
    type: string
    enum: ["APPROVED", "CHANGES_REQUESTED", "COMMENTED", "PENDING"]
    description: Current review decision

  linked_issues:
    type: array
    items:
      type: object
      properties:
        number:
          type: integer
        title:
          type: string
    description: Issues linked to this PR

# Denied actions (never allowed even if requested)
denied:
  - merge
  - close
  - rebase
  - squash
  - delete_branch
  - admin_override
  - force_push
  - edit_body_as_author  # always shows original body

# Required evidence
evidence:
  required: true
  fields:
    - skill_name
    - repo
    - pr_number
    - timestamp
    - requested_by_user
    - outputs_hash

# Approval requirements
approval:
  required: false
  note: "Read-only skill; no approval required"

# Policy tags
policy:
  risk_class: read_only
  rbac_required: false
  sandboxes: ["local", "ci", "devcontainer"]
  denied_sandboxes: ["production"]
  requires_vpc: false
  secrets_in_output: false
  audit_required: true

# Timeout and retry
execution:
  timeout_seconds: 30
  retry_count: 2
  retry_delay_seconds: 5
```

---

## 6. Command Adapter Pattern

Adapters are language-specific implementations that translate skill inputs → CLI command → parsed output.

### Example: GitHub CLI Adapter (Python)

```python
# pulser/skills/adapters/github_cli.py

import subprocess
import json
from typing import dict, list
from dataclasses import dataclass

@dataclass
class GitHubPRSkill:
    """Adapter for pulser.github.pr_status"""
    
    def __init__(self):
        self.gh_bin = "gh"  # Assumes `gh` in PATH
    
    def invoke(self, repo: str, pr_number: int, include_checks: bool = True) -> dict:
        """
        Invoke: gh pr view <pr_number> --repo <repo> --json state,mergeability,...
        """
        fields = "state,mergeability,checks,headRefName,baseRefName,title,author,createdAt,updatedAt,reviewDecision,labels"
        
        cmd = [
            self.gh_bin, "pr", "view",
            str(pr_number),
            "--repo", repo,
            "--json", fields,
        ]
        
        try:
            result = subprocess.run(cmd, capture_output=True, text=True, timeout=30)
            if result.returncode != 0:
                raise RuntimeError(f"gh pr view failed: {result.stderr}")
            
            data = json.loads(result.stdout)
            
            # Transform output to match contract schema
            output = {
                "state": data.get("state", "").upper(),
                "mergeable": data.get("mergeability", "UNKNOWN").upper(),
                "head_ref": data.get("headRefName"),
                "base_ref": data.get("baseRefName"),
                "title": data.get("title"),
                "author": data.get("author", {}).get("login"),
                "created_at": data.get("createdAt"),
                "updated_at": data.get("updatedAt"),
                "review_decision": (data.get("reviewDecision") or "PENDING").upper(),
                "checks": self._extract_checks(data.get("checks", [])) if include_checks else [],
                "linked_issues": [],  # TODO: extract from body
            }
            
            return output
            
        except subprocess.TimeoutExpired:
            raise RuntimeError(f"gh pr view timed out after 30s")
    
    def _extract_checks(self, checks: list) -> list:
        """Transform gh check output to contract schema"""
        return [
            {
                "name": check.get("name"),
                "status": check.get("conclusion", "pending").lower(),
                "url": check.get("url"),
            }
            for check in checks
        ]
```

### Example: Odoo CLI Adapter (Python)

```python
# pulser/skills/adapters/odoo_cli.py

import subprocess
import json
from typing import dict, list

@dataclass
class OdooModuleTestSkill:
    """Adapter for pulser.odoo.module_test"""
    
    def __init__(self, db_name: str = "odoo_dev", db_host: str = "localhost"):
        self.odoo_bin = "odoo-bin"
        self.db_name = db_name
        self.db_host = db_host
    
    def invoke(self, modules: list, db_name: str = None) -> dict:
        """
        Invoke: odoo-bin -d <db> -m <modules> --test-enable --stop-after-init
        """
        db = db_name or self.db_name
        
        # Allowlist: only test modules in ipai/
        allowed_modules = [
            "ipai_pulser_chat",
            "ipai_demo_finops",
            "ipai_tax_intelligence",
            # ... others
        ]
        
        for mod in modules:
            if mod not in allowed_modules:
                raise ValueError(f"Module '{mod}' not in allowlist")
        
        cmd = [
            self.odoo_bin,
            "-d", db,
            "-m", ",".join(modules),
            "--test-enable",
            "--stop-after-init",
        ]
        
        try:
            result = subprocess.run(cmd, capture_output=True, text=True, timeout=120)
            
            return {
                "db": db,
                "modules_tested": modules,
                "exit_code": result.returncode,
                "status": "passed" if result.returncode == 0 else "failed",
                "stdout": result.stdout[-500:],  # Last 500 chars
                "stderr": result.stderr[-500:],  # Last 500 chars
                "passed": result.returncode == 0,
            }
            
        except subprocess.TimeoutExpired:
            raise RuntimeError(f"odoo-bin test timed out after 120s")
```

---

## 7. Initial Skill Set (Wave 1)

### GitHub Skills (`pulser.github.*`)

#### pulser.github.pr_status
- **Risk**: Low
- **Inputs**: repo, pr_number, include_checks
- **Outputs**: state, mergeable, checks, author, created_at, linked_issues
- **Approval**: None
- **Use**: Pre-merge validation, CI status check
- **Denied**: merge, close, rebase

#### pulser.github.check_branch
- **Risk**: Low
- **Inputs**: repo, branch
- **Outputs**: exists, protection_rules, latest_commit, last_push_date
- **Approval**: None
- **Use**: Branch validation before work
- **Denied**: delete, force_push, unprotect

---

### Azure Skills (`pulser.azure.*`)

#### pulser.azure.resource_inventory
- **Risk**: Low
- **Inputs**: subscription_id, resource_group (optional), resource_type (optional)
- **Outputs**: resources[], names, IDs, types, locations, tags
- **Approval**: None
- **Use**: Infrastructure discovery, health check
- **Denied**: modify, delete, update_tags

#### pulser.azure.env_status
- **Risk**: Low
- **Inputs**: environment (dev/staging/prod)
- **Outputs**: resource_health[], connectivity[], performance[]
- **Approval**: None
- **Use**: Pre-deployment health check
- **Denied**: None (read-only)

#### pulser.azure.deploy_preview
- **Risk**: Medium (non-prod only)
- **Inputs**: environment (dev/staging only), template_file, parameters_file
- **Outputs**: proposed_resources[], estimated_cost, deployment_plan
- **Approval**: Auto-approved if health checks pass
- **Use**: Preview deployment before apply
- **Denied**: deploy, apply_destructive_changes

---

### Odoo Skills (`pulser.odoo.*`)

#### pulser.odoo.module_test
- **Risk**: Low (dev DB only)
- **Inputs**: modules[], db_name (dev only)
- **Outputs**: modules_tested, status (passed/failed), exit_code, logs
- **Approval**: None
- **Use**: Local test before PR
- **Denied**: modify_db, change_settings

#### pulser.odoo.module_install
- **Risk**: Medium (test/dev only)
- **Inputs**: modules[], db_name (dev only), skip_dependencies
- **Outputs**: install_result, installed_modules, errors
- **Approval**: Auto if health check passes
- **Use**: Module installation during testing
- **Denied**: production, uninstall, modify_data

---

### Power Platform Skills (`pulser.powerplatform.*`)

#### pulser.powerplatform.solution_validate
- **Risk**: Low
- **Inputs**: solution_file, target_environment (dev/sandbox only)
- **Outputs**: validation_result (pass/fail), warnings[], errors[]
- **Approval**: None
- **Use**: Solution validation before export
- **Denied**: import_production, delete_solution, modify_environment

---

### M365 Skills (`pulser.m365.*`)

#### pulser.m365.agent_package
- **Risk**: Low (sandbox only)
- **Inputs**: agent_app_dir, environment (dev/sandbox only)
- **Outputs**: package_result, zip_file, manifest_validation
- **Approval**: None
- **Use**: Package Teams agent before provisioning
- **Denied**: publish_production, deploy_live

#### pulser.m365.provision_sandbox
- **Risk**: Medium (E5 sandbox only)
- **Inputs**: agent_app_dir, sandbox_id (zmc2r only)
- **Outputs**: provision_result, app_id, dev_tunnel_url
- **Approval**: Auto if validation passes
- **Use**: Provision agent in E5 sandbox
- **Denied**: production_tenant, cross_tenant, delete_app

---

### Databricks Skills (`pulser.databricks.*`)

#### pulser.databricks.sql_status
- **Risk**: Low
- **Inputs**: workspace_url, warehouse_id
- **Outputs**: warehouse_status, job_status[], query_status[]
- **Approval**: None
- **Use**: Warehouse health check
- **Denied**: stop_warehouse, delete_job, modify_policy

---

### Local Skills (`pulser.local.*`)

#### pulser.local.stack_health
- **Risk**: Low
- **Inputs**: None (localhost only)
- **Outputs**: containers[], services[], health_status, connectivity[]
- **Approval**: None
- **Use**: Local stack health check
- **Denied**: restart, recreate, prune

---

## 8. Policy Document

See: [cli-skill-policy.yaml](cli-skill-policy.yaml) (separate file)

Key rules:
- All CLI skills require **typed input + output validation**
- Allowlists are **per-skill, not per-user**
- Read-only skills auto-allowed if scoped
- Write operations require health check + audit
- Destructive operations require human approval
- Production operations are **blocked by default**
- Evidence is **required for all invocations**

---

## 9. Guardrails

### DO NOT

```yaml
# ❌ FORBIDDEN: Generic shell execution
skill: run_shell_command
inputs:
  command: "anything"
```

```yaml
# ❌ FORBIDDEN: Unrestricted pip install
skill: install_python_package
inputs:
  package_name: "any_package"
```

```yaml
# ❌ FORBIDDEN: Direct Databricks Connect
skill: databricks_connect
inputs:
  code: "arbitrary python"
```

```yaml
# ❌ FORBIDDEN: Production apply without approval
skill: azd_up
inputs:
  environment: "production"
  auto_approved: true
```

### DO

```yaml
# ✅ GOOD: Bounded GitHub skill
skill: pulser.github.pr_status
inputs:
  repo: "Insightpulse-ai/agent-platform"  # Allowlisted
  pr_number: 42
```

```yaml
# ✅ GOOD: Typed Odoo skill
skill: pulser.odoo.module_test
inputs:
  modules: ["ipai_pulser_chat"]  # Allowlisted
  db_name: "odoo_dev"  # Dev only
```

```yaml
# ✅ GOOD: Read-only Azure skill
skill: pulser.azure.resource_inventory
inputs:
  subscription_id: "..."
  # No modify/delete capability
```

```yaml
# ✅ GOOD: Approval-gated deployment
skill: pulser.azure.deploy_preview
inputs:
  environment: "staging"  # Non-prod
  template_file: "..."
approval_required: true
```

---

## 10. Evidence Requirements

All CLI skill invocations must produce **auditable evidence**:

```json
{
  "skill_run_id": "skill-run-2026-04-25-pr-status-001",
  "skill_name": "pulser.github.pr_status",
  "timestamp": "2026-04-25T03:30:00Z",
  "requested_by": "agent-id-123",
  "requested_by_user": "alice@insightpulseai.com",
  "environment": "local",
  "inputs": {
    "repo": "Insightpulse-ai/agent-platform",
    "pr_number": 42
  },
  "outputs_hash": "sha256:abc123...",
  "exit_code": 0,
  "duration_seconds": 2.5,
  "status": "success",
  "errors": [],
  "warnings": [],
  "cli_command_sanitized": "gh pr view 42 --repo Insightpulse-ai/agent-platform --json state,mergeability,...",
  "approval_chain": [],
  "rollback_available": false
}
```

Evidence is stored in:
- **Local dev**: `~/.pulser/skill-runs/` (JSON logs)
- **CI/CD**: Azure Storage or GitHub Actions artifacts
- **Audit system**: Queryable via `pulser.audit.query_skill_runs`

---

## 11. Roadmap

### Wave 1 (2026-04-25 → 2026-05-15)
- ✅ Define CLI skills model (this document)
- ✅ Define initial skill set (GitHub, Azure basic, Odoo test)
- ⏳ Implement skill adapters (Python)
- ⏳ Implement skill validator (JSON schema)
- ⏳ Implement evidence logger
- ⏳ Bootstrap `pulser-devops-agent` with GitHub + Azure skills
- ⏳ Test end-to-end: agent request → skill → evidence

### Wave 2 (2026-05-15 → 2026-06-15)
- Power Platform skills (pac CLI)
- M365 agent packaging skills (teamsapp/atk)
- Databricks SQL status skill
- Approval gate implementation

### Wave 3 (2026-06-15 → 2026-07-15)
- Odoo module install/update skills (higher risk)
- Azure deployment skills (azd up/down for dev/staging)
- Cross-skill workflows (e.g., validate PR → deploy preview → run tests)

### Future (Post-Wave 3)
- AWS CLI skills (future Wave 4)
- Terraform/Bicep plan/apply (gated, non-prod only)
- Custom skill framework (allow domain teams to register skills)

---

## 12. FAQ

**Q: Can agents call arbitrary shell commands?**  
A: No. All agent CLI access goes through pre-defined, typed Pulser skills. No generic shell execution.

**Q: What if I need a CLI skill that doesn't exist?**  
A: File a feature request in the agent-platform repo. Skill addition follows change control:
1. Define contract (YAML)
2. Implement adapter (Python)
3. Write tests
4. PR review + approval
5. Merge to main; register in policy
6. Announce to agents

**Q: Can I use Databricks Connect with Pulser?**  
A: Only in a dedicated Python 3.12 environment with DBR 17.3+, isolated from Odoo. Never in the Odoo venv.

**Q: What about secrets in CLI output?**  
A: Output is scanned for common secret patterns (password=, token=, key=). Matches are redacted. Manually verify no sensitive data before committing evidence.

**Q: Can production resources be modified via skills?**  
A: No by default. Production operations require explicit human approval, change control ticket, and ADR/RFC merge.

**Q: How is evidence retained?**  
A: 90 days in hot storage (queries fast). 1 year in cold storage (archive). Immutable append-only.

**Q: Can skills be rate-limited?**  
A: Yes. Per-skill, per-user, per-environment limits. Default: 5 invocations per minute per user per skill.

---

## 13. Related Documents

- [pulser-finance-compliance/conversation-behavior-spec.md](conversation-behavior-spec.md): Conversational intelligence (numbered options, tax intent routing, RBAC response shaping)
- [PULSER_AWARENESS_CONTRACT.md](../architecture/PULSER_AWARENESS_CONTRACT.md): Backend awareness envelope (capabilities, RBAC flags)
- [agent-platform/README.md](../../agent-platform/README.md): Agent platform overview
- [docs/governance/MCP-policy.md](../../governance/MCP-policy.md): MCP governance alignment

---

## 14. Implementation Checklist

- [ ] CLI skills model documented (this file)
- [ ] Skill contract schema finalized (YAML)
- [ ] GitHub CLI adapter implemented (Python)
- [ ] Azure CLI adapter implemented (Python)
- [ ] Odoo CLI adapter implemented (Python)
- [ ] Skill validator implemented (JSON schema validation)
- [ ] Evidence logger implemented (JSON + audit storage)
- [ ] CLI skill policy document created
- [ ] Skills registered in agent-platform registry
- [ ] pulser-devops-agent bootstrap PR created
- [ ] End-to-end test: agent → skill → evidence
- [ ] Agent-platform team review + approval
- [ ] Documentation published
- [ ] Runbook created (how to add new CLI skills)

---

## Contacts & Governance

- **Owner**: Agent Platform Team
- **Reviewer**: Governance & Compliance
- **Questions**: File issues in agent-platform repo
- **Changes**: ADR required if modifying risk classes or guardrails

