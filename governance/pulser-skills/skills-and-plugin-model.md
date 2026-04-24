# Pulser Vendor-Neutral Skills and Plugin Model

**Document Version**: 1.0  
**Status**: SPECIFICATION (Ready for Implementation)  
**Last Updated**: 2026-04-25  
**Related**: [CLI Skills Model](./cli-skills-model.md) | [Graph + PowerShell Skills Model](./graph-powershell-skills-model.md) | [Conversation Behavior Spec](./conversation-behavior-spec.md)

---

## 1. Purpose

Pulser is a multi-platform agent: it runs in Odoo, Azure Foundry, Microsoft 365 Copilot, CLI workflows, and Teams. Each platform has its own tool/skill interface:
- **Claude Skills** → instruction folders + scripts + resources
- **MCP tools** → typed tool discovery + calling protocol
- **M365 Copilot plugins** → OpenAPI or MCP descriptors
- **Azure Functions** → serverless skill bindings
- **Foundry tools** → runtime tool registration
- **CLI wrappers** → command-line skill invocation

To avoid duplicating logic across six adapters, Pulser uses **one canonical skill contract** that generates adapters for each platform.

**Rule**: 
```text
Skill logic lives once (in Pulser Skill Contract).
Adapters expose it to each agent surface.
No authority drift. No business logic duplication.
```

---

## 2. Taxonomy

### 2.1 Pulser Skill Contract (SSOT)

The **canonical definition** of a Pulser capability. Defines:
- Inputs, outputs, risk, RBAC, approval gates, denied actions
- Required tools/APIs/permissions
- Evidence retention, evals, audit trail
- Vendor-neutral (no platform-specific language)

**Format**: YAML + markdown (skill.yaml + SKILL.md)  
**Owner**: agent-platform/pulser/skills/  
**Exports to**: All adapters below

**Example**:
```yaml
name: pulser.bir_certificate_review
kind: pulser_skill
domain: finance_compliance
risk: medium
runtime: pulser_tool_api

inputs:
  certificate_type: ["2307", "2316", "2306"]
  attachment_id: string

outputs:
  extraction_summary: object
  confidence_tier: ["auto_approve", "low_confidence", "hard_reject"]

allowed:
  - document_intelligence.extract
  - odoo.record.read
  - evidence.write

denied:
  - bir.autonomous_filing
  - accounting.autonomous_posting

evidence:
  required: true
  retention: P7Y
```

### 2.2 Claude Skill

**What it is**: Portable task pack (instructions + scripts + resources) that Claude loads dynamically.

**Pulser use**: Repeatable task packs that combine reasoning + bounded automation.

**Examples**:
- `pulser.pr_review` — Claude reviews GitHub PRs using rules from SKILL.md
- `pulser.tax_audit_checklist` — Claude conducts PH tax audit using BIR checklist
- `pulser.odoo_test_runner` — Claude runs Odoo module tests and interprets results

**Adapter format**:
```text
.claude/skills/pulser_bir_certificate_review/
├── SKILL.md                      (human instructions)
├── claude_skill_manifest.yaml    (Claude-specific metadata)
├── prompts/
│   ├── extract_fields.md
│   ├── validate_tin.md
│   └── match_to_odoo.md
├── schemas/
│   └── bir_2307_schema.json
└── examples/
    └── sample_certificate.pdf
```

**Claude integration**: Claude loads SKILL.md as system prompt + available resources; calls Pulser Tool API for `document_intelligence.extract`, `odoo.record.read`, etc.

### 2.3 MCP Tool

**What it is**: Standard protocol for agents to discover and call tools (Anthropic + open community standard).

**Pulser use**: Primary cross-agent tool interface. Pulser exposes skills as MCP tools.

**Examples**:
- `pulser/bir.validate_tin_atc` → `{ tin: string, atc: string } → { is_valid: boolean, reason?: string }`
- `pulser/odoo.module_test` → `{ module: string, db: string } → { passed: boolean, logs: string }`
- `pulser/graph.group_membership_read` → `{ user_id: string } → { groups: array, roles: array }`

**Adapter format**:
```yaml
# mcp.yaml (derived from skill.yaml)
name: pulser-skills-mcp
tools:
  - name: pulser.bir.validate_tin_atc
    description: Validate Philippine TIN and ATC codes
    inputSchema:
      type: object
      properties:
        tin: { type: string }
        atc: { type: string }
      required: [tin, atc]
    output:
      type: object
      properties:
        is_valid: { type: boolean }
        reason: { type: string }
```

### 2.4 Microsoft 365 Copilot Plugin

**What it is**: Declarative agent plugin that describes MCP servers or REST APIs via OpenAPI for M365 Copilot integration.

**Pulser use**: Expose Pulser skills to Teams agents, Microsoft 365 Copilot, Power Automate.

**Examples**:
- Plugin exposes `pulser/graph.group_membership_read` to Teams agents
- Plugin describes OpenAPI endpoint for `pulser/vat_filing_evidence`
- Declarative agent calls plugin → plugin calls MCP → Pulser Tool API

**Adapter format**:
```yaml
# openapi.yaml (derived from skill.yaml)
openapi: 3.0.0
info:
  title: Pulser Skills API
  version: 1.0.0

paths:
  /skills/bir/validate:
    post:
      summary: Validate BIR certificate fields
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/BirCertificate'
      responses:
        '200':
          description: Validation result
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ValidationResult'
```

### 2.5 Azure Function MCP Tool

**What it is**: Azure Function exposing a bounded capability as an MCP tool via bindings.

**Pulser use**: Narrow, fast, serverless skills that don't require Odoo/Graph/Complex state.

**Examples**:
- `pulser.vat.compute` → validate VAT calculations
- `pulser.bir.validate_tin_atc` → check TIN/ATC format
- `pulser.evidence.hash_file` → compute file hash for audit trail
- `pulser.graph.group_membership_read` → fast Graph query for RBAC

**Adapter format**:
```json
{
  "name": "pulser-vat-compute",
  "runtime": "node",
  "bindings": [
    {
      "name": "mcpTool",
      "type": "mcpTool",
      "direction": "in",
      "toolName": "pulser.vat.compute"
    }
  ],
  "scriptFile": "index.js"
}
```

**Best candidates**: Fast, pure functions. **Avoid**: Long-running tasks, Odoo module tests, DI training.

### 2.6 Foundry Tool

**What it is**: Tool registered in Azure AI Foundry agent runtime and available to Foundry agents.

**Pulser use**: Runtime capability availability for Foundry-hosted Pulser agents.

**Examples**:
- Foundry agent calls `pulser.bir_certificate_review`
- Foundry agent calls `pulser.vat_filing_evidence`
- Foundry agent invokes `pulser.odoo.module_test`

**Adapter format**:
```yaml
# foundry_tool.yaml (derived from skill.yaml)
name: pulser.bir_certificate_review
type: tool
runtime_environment: foundry

function:
  name: bir_certificate_review
  description: Review inbound PH BIR certificates
  input_schema:
    type: object
    properties:
      certificate_type:
        type: string
        enum: ["2307", "2316", "2306"]
  output_schema:
    type: object
    properties:
      confidence_tier:
        type: string
        enum: ["auto_approve", "low_confidence", "hard_reject"]

endpoints:
  - https://pulser-api.insightpulseai.com/tools/bir/certificate-review
```

### 2.7 CLI Skill

**What it is**: Command-line wrapper around a Pulser skill with typed inputs and no arbitrary shell execution.

**Pulser use**: DevOps/admin automation (GitHub, Azure, Odoo, PowerShell tasks).

**Examples**:
- `pulser github pr-status --repo Insightpulse-ai/agent-platform --pr 42`
- `pulser odoo module-test --module ipai_pulser_chat --db odoo_dev`
- `pulser powershell e5-sandbox-rbac-seed --dry-run true`

**Adapter format**:
```bash
#!/usr/bin/env python3
# cli/pulser_github_pr_status.py

@click.command()
@click.option('--repo', required=True, help='GitHub repo (owner/name)')
@click.option('--pr', required=True, type=int, help='PR number')
def pr_status(repo, pr):
    """Get GitHub PR status via pulser.github.pr_status skill."""
    # Calls Pulser Tool API with typed inputs
    # No arbitrary shell; no os.system()
    pass
```

---

## 3. Canonical Skill Package Layout

All Pulser skills live under `agent-platform/pulser/skills/`:

```text
agent-platform/pulser/skills/
├── bir_certificate_review/
│   ├── skill.yaml                    # SSOT: canonical contract
│   ├── SKILL.md                      # Human-readable instructions
│   ├── prompts/
│   │   ├── extract_fields.md
│   │   ├── validate_tin_atc.md
│   │   └── match_to_odoo.md
│   ├── schemas/
│   │   ├── bir_2307_schema.json
│   │   ├── bir_2316_schema.json
│   │   └── bir_2306_schema.json
│   ├── examples/
│   │   ├── sample_2307.pdf
│   │   └── sample_validation_output.json
│   ├── evals/
│   │   ├── extraction_accuracy.yaml
│   │   └── confidence_tier_accuracy.yaml
│   └── adapters/
│       ├── mcp.yaml
│       ├── openapi.yaml
│       ├── azure_function.json
│       ├── foundry_tool.yaml
│       └── claude_skill_manifest.yaml
│
├── github_pr_status/
│   ├── skill.yaml
│   ├── SKILL.md
│   ├── examples/
│   └── adapters/
│       ├── mcp.yaml
│       ├── cli.py
│       └── foundry_tool.yaml
│
├── odoo_module_test/
├── graph_group_membership_read/
├── powershell_e5_sandbox_rbac_seed/
├── m365_agent_package_validate/
├── vat_filing_evidence/
├── office_generate_workbook/
└── README.md
```

### 3.1 Contract Schema: `skill.yaml`

```yaml
# Metadata
name: pulser.bir_certificate_review          # Global skill name
kind: pulser_skill                            # Always "pulser_skill"
domain: finance_compliance                    # Domain: finance_compliance, devops, identity, …
version: "1.0.0"
stable: true                                  # Production-ready

# Description
description: >
  Reviews inbound Philippine BIR 2307, 2316, and 2306 certificates.
  Extracts fields via Document Intelligence, validates TIN/ATC/dates/amounts,
  matches to Odoo records, assigns confidence tier, and prepares audit evidence.

# Scope
runtime: pulser_tool_api                      # Where skill runs: pulser_tool_api, foundry, mcp, azure_function, cli
risk: medium                                  # low | medium | high | critical
risk_rationale: "Reads financial docs; validates tax data; no autonomous filing"

# Inputs
inputs:
  certificate_type:
    type: enum
    values: ["2307", "2316", "2306"]
    required: true
    description: "BIR certificate form type"
  
  attachment_id:
    type: string
    required: true
    description: "Odoo ir.attachment ID or file path"
  
  odoo_record_ref:
    type: string
    required: false
    description: "Odoo model.id reference (account.move, res.partner, …)"
  
  confidence_threshold:
    type: float
    required: false
    default: 0.85
    description: "Minimum confidence for auto_approve"

# Outputs
outputs:
  extraction_summary:
    type: object
    description: "Extracted fields (tin, atc, amounts, dates, etc.)"
    properties:
      tin: { type: string }
      atc: { type: string }
      date_issued: { type: string, format: "date" }
      withholding_amount: { type: number }
  
  validation_findings:
    type: array
    items:
      type: object
      properties:
        check: { type: string }
        status: { type: string, enum: ["pass", "warn", "fail"] }
        reason: { type: string }
  
  confidence_tier:
    type: enum
    values: ["auto_approve", "low_confidence", "hard_reject"]
    description: "Readiness for approval"
  
  evidence_ref:
    type: string
    description: "Reference to immutable evidence log"
  
  odoo_matches:
    type: array
    items:
      type: object
      properties:
        model: { type: string }
        record_id: { type: integer }
        match_confidence: { type: number }

# Tool Capabilities
tooling:
  allowed:
    - document_intelligence.extract      # Azure Document Intelligence
    - odoo.record.search                 # Odoo ORM read
    - odoo.record.read
    - azure_ai_search.query              # For context retrieval
    - evidence.write                     # Audit evidence log
  
  requires_approval:
    - odoo.draft.writeback               # Write draft fields to Odoo
  
  denied:
    - bir.autonomous_filing              # Do not file taxes
    - bir.autonomous_payment             # Do not make payments
    - odoo.accounting_post               # Do not post accounting
    - odoo.record.delete

# RBAC & Approval
rbac:
  allowed_groups:
    - sg-pulser-bir-processors           # Can invoke
    - sg-pulser-tax-reviewers            # Can invoke + review
    - sg-pulser-bir-approvers            # Can invoke + approve
    - sg-pulser-controllers              # Can invoke + approve

approval_required: false                 # None for read/extraction
approval_required_if:
  - odoo.draft.writeback: true            # Write to Odoo requires approval

# Evidence & Audit
evidence:
  required: true
  retention: P7Y                          # 7-year record retention (Philippines)
  log_fields:
    - user_id
    - certificate_type
    - tin
    - extraction_confidence
    - validation_status
    - odoo_record_matches

# Evals
evals:
  required: true
  criteria:
    - extraction_accuracy              # Does DI extract correct fields?
    - tin_atc_validation_accuracy      # Does validator catch bad TINs?
    - odoo_match_relevance             # Does record matching return correct Odoo records?
    - confidence_tier_calibration      # Is confidence_tier prediction well-calibrated?

# Agent distribution
platforms:
  - claude_skill                 # Export as Claude Skill
  - mcp_tool                     # Expose via MCP
  - m365_copilot_plugin          # M365 Copilot plugin
  - foundry_tool                 # Foundry tool registry
  - azure_function               # Optional: Azure Function MCP binding
  - cli_wrapper                  # Optional: CLI command
```

### 3.2 Human Instructions: `SKILL.md`

Claude-compatible human-readable skill documentation.

```markdown
# Pulser BIR Certificate Review

**Skill Name**: pulser.bir_certificate_review  
**Domain**: Finance Compliance  
**Risk**: medium  
**Status**: Stable  

## What This Skill Does

Receives a Philippine BIR 2307, 2316, or 2306 withholding certificate (image, PDF, or email attachment), extracts all fields using Azure Document Intelligence, validates fields against BIR rules and Odoo records, and prepares a review evidence log.

**Does NOT**: File taxes, make payments, or autonomously post accounting entries.

## Workflow

1. **Receive certificate** → Input: `certificate_type` (2307/2316/2306), `attachment_id`
2. **Extract fields** → Document Intelligence extracts: TIN, ATC, date issued, withholding amount, payer details
3. **Validate TIN/ATC** → Check format, query BIR registry (if available)
4. **Validate amounts** → Check withholding percentage matches PH tax rules
5. **Match to Odoo** → Find corresponding Odoo purchase invoice, vendor, or payment
6. **Assign confidence** → Confidence tier: auto_approve | low_confidence | hard_reject
7. **Prepare evidence** → Immutable audit log with all extraction + validation results
8. **Request approval** → If writing to Odoo, request approval before writeback

## Boundaries

- ❌ Do not file taxes autonomously
- ❌ Do not make payments
- ❌ Do not post accounting entries
- ❌ Do not delete records
- ✅ Do extract, validate, and prepare evidence only
- ✅ Use Odoo as the ERP source of truth
- ✅ Use only approved tool calls

## Example

**Input**:
```json
{
  "certificate_type": "2307",
  "attachment_id": "ir_attachment_12345",
  "odoo_record_ref": "account.move.678"
}
```

**Output**:
```json
{
  "extraction_summary": {
    "tin": "000-000-000-001",
    "atc": "VAT000-001",
    "date_issued": "2026-04-20",
    "withholding_amount": 5000.00
  },
  "validation_findings": [
    { "check": "TIN format", "status": "pass" },
    { "check": "ATC valid", "status": "pass" },
    { "check": "Withholding percentage", "status": "pass" }
  ],
  "confidence_tier": "auto_approve",
  "evidence_ref": "evidence-2307-20260425-001",
  "odoo_matches": [
    {
      "model": "account.move",
      "record_id": 678,
      "match_confidence": 0.98
    }
  ]
}
```

## Error Handling

- `low_confidence` → Requires human review before Odoo writeback
- `hard_reject` → Extraction failed; certificate may be unreadable or malformed

## Approval Requirements

- Extraction & validation: Auto-allowed (read-only)
- Write to Odoo draft fields: Approval required (one approver)
- Post to accounting: Approval required (two signers) + change control ticket

## Related Skills

- `pulser.odoo.module_test` — Test certificate extraction after code changes
- `pulser.graph.group_membership_read` — Determine who can approve
- `pulser.evidence.hash_file` — Audit trail hashing
```

---

## 4. Adapter Rules

### 4.1 Claude Skill Export

**When**: Task requires reasoning + decision-making. Claude reads SKILL.md and calls Pulser Tool API.

**Rule**: Claude never executes arbitrary scripts; Claude calls only Pulser Tool API endpoints with typed inputs/outputs.

**Adapter structure**:
```yaml
# adapters/claude_skill_manifest.yaml
name: pulser_bir_certificate_review
type: task_pack

instructions: |
  Load SKILL.md from parent directory as your system instructions.
  Use only the approved tool calls: document_intelligence.extract, odoo.record.search, evidence.write.
  Do not execute shell commands or arbitrary Python.
  Always return evidence_ref with your response.

tools_available:
  - pulser_tool_api/document_intelligence.extract
  - pulser_tool_api/odoo.record.search
  - pulser_tool_api/odoo.record.read
  - pulser_tool_api/evidence.write

resources:
  - schemas/bir_2307_schema.json
  - examples/sample_2307.pdf
  - prompts/extract_fields.md
  - prompts/validate_tin_atc.md
  - prompts/match_to_odoo.md
```

### 4.2 MCP Tool Adapter

**When**: Tool needs to be discoverable and callable by any MCP agent (Claude, Foundry, custom agents).

**Rule**: Skill contract generates MCP tool schema automatically. No business logic duplication.

**Adapter structure**:
```yaml
# adapters/mcp.yaml
name: pulser-skills-mcp
protocol: mcp
version: 1.0

tools:
  - name: pulser.bir.certificate_review
    description: "Review PH BIR certificate; extract, validate, match to Odoo"
    
    input_schema:
      type: object
      properties:
        certificate_type:
          type: string
          enum: ["2307", "2316", "2306"]
        attachment_id:
          type: string
      required: ["certificate_type", "attachment_id"]
    
    result_schema:
      type: object
      properties:
        extraction_summary:
          type: object
        confidence_tier:
          type: string
        evidence_ref:
          type: string

schema_generator: 
  source: ../skill.yaml
  mapping:
    inputs → input_schema
    outputs → result_schema
    allowed_tools → mcp_resources
    denied → explicit_denial_check
```

### 4.3 Microsoft 365 Copilot Plugin

**When**: Skill needs to be available to Teams agents or M365 Copilot declarative agents.

**Rule**: Plugin describes MCP servers or OpenAPI endpoints for M365 Copilot integration.

**Adapter structure**:
```yaml
# adapters/openapi.yaml
openapi: 3.0.0
info:
  title: Pulser Skills API
  version: 1.0.0
  description: Pulser finops agent skills for M365 Copilot

servers:
  - url: https://pulser-api.insightpulseai.com
    description: Pulser Tool API

paths:
  /tools/bir/certificate-review:
    post:
      summary: Review BIR certificate
      description: Extract, validate, and match BIR certificate to Odoo records
      
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                certificate_type: { enum: ["2307", "2316", "2306"] }
                attachment_id: { type: string }
              required: ["certificate_type", "attachment_id"]
      
      responses:
        '200':
          description: Certificate review result
          content:
            application/json:
              schema:
                type: object
                properties:
                  confidence_tier: { enum: ["auto_approve", "low_confidence", "hard_reject"] }
                  evidence_ref: { type: string }

# Pulser security policy
x-pulser-policy:
  approval_required: false
  evidence_required: true
  denied_actions:
    - bir.autonomous_filing
    - accounting.autonomous_posting
```

### 4.4 Azure Function MCP Binding

**When**: Skill is narrow, fast, and serverless-friendly. Avoid long-running tasks, Odoo tests, DI training.

**Best candidates**: `pulser.vat.compute`, `pulser.bir.validate_tin_atc`, `pulser.graph.group_membership_read`

**Adapter structure**:
```json
{
  "name": "pulser-bir-validate-tin-atc",
  "runtime": "python",
  "version": "1.0",
  "bindings": [
    {
      "name": "mcpTool",
      "type": "mcpTool",
      "direction": "in",
      "toolName": "pulser.bir.validate_tin_atc",
      "operationId": "validate_tin_atc"
    }
  ],
  "handler": "index.main",
  "timeout": 30
}
```

### 4.5 Foundry Tool Registration

**When**: Skill runs inside Foundry agent runtime.

**Rule**: Foundry loads tool definition from registry; calls Pulser Tool API endpoint.

**Adapter structure**:
```yaml
# adapters/foundry_tool.yaml
name: pulser.bir_certificate_review
type: tool
runtime: foundry
version: 1.0.0

capabilities:
  - read_attachments
  - query_odoo
  - write_evidence

function:
  name: bir_certificate_review
  handler: https://pulser-api.insightpulseai.com/tools/bir/certificate-review
  
  description: >
    Review Philippine BIR 2307/2316/2306 certificates.
    Extracts fields, validates, matches to Odoo, assigns confidence tier.
  
  input_schema:
    $ref: ../skill.yaml#/inputs
  
  output_schema:
    $ref: ../skill.yaml#/outputs

approval_gates:
  - odoo.draft.writeback: "require_approval=true"

audit:
  evidence_required: true
  retention_days: 2555  # 7 years
```

### 4.6 CLI Wrapper

**When**: Skill invoked from command line (DevOps, admin, testing).

**Rule**: No arbitrary shell; typed inputs only. Calls Pulser Tool API.

**Adapter structure**:
```python
# adapters/cli.py
#!/usr/bin/env python3

import click
import requests
import json

@click.group()
def pulser():
    """Pulser finops agent CLI"""
    pass

@pulser.group()
def bir():
    """BIR certificate operations"""
    pass

@bir.command()
@click.option('--cert-type', type=click.Choice(['2307', '2316', '2306']), required=True)
@click.option('--attachment-id', required=True)
@click.option('--odoo-record', required=False)
def certificate_review(cert_type, attachment_id, odoo_record):
    """Review BIR certificate"""
    payload = {
        "certificate_type": cert_type,
        "attachment_id": attachment_id,
    }
    if odoo_record:
        payload["odoo_record_ref"] = odoo_record
    
    response = requests.post(
        "https://pulser-api.insightpulseai.com/tools/bir/certificate-review",
        json=payload,
        headers={"Authorization": "Bearer <token>"}
    )
    
    print(json.dumps(response.json(), indent=2))

if __name__ == "__main__":
    pulser()
```

**Usage**:
```bash
$ pulser bir certificate-review --cert-type 2307 --attachment-id ir_attachment_12345
{
  "confidence_tier": "auto_approve",
  "evidence_ref": "evidence-2307-20260425-001"
}
```

---

## 5. Repo Placement & Ownership

| Directory | Owner | Responsibility |
|---|---|---|
| `agent-platform/pulser/skills/` | Pulser Product | Skill contracts (SSOT), SKILL.md, adapters |
| `agent-platform/pulser/tools/` | Pulser Platform Eng | Tool API runtime (skills loader, policy engine, evidence logging) |
| `docs/product/pulser-finance-compliance/` | Product | Strategy, governance, acceptance criteria |
| `odoo/addons/ipai_pulser_chat/` | Odoo Integration | Pulser widget, Odoo runtime integration |
| `platform/contracts/` | Enterprise Arch | Tool API contracts (OpenAPI, MCP schemas) |
| `marketplace-publishing/` | GTM | Packaged skill descriptions for customers |

**Skill Ownership Rule**: If you modify `skill.yaml`, all adapters (mcp.yaml, openapi.yaml, foundry_tool.yaml, claude_skill_manifest.yaml) are **auto-derived**. No manual sync required.

---

## 6. Initial Skills (Wave 1-3)

### Wave 1: Proof-of-Concept (Weeks 1-2)

| Skill | Adapter(s) | Purpose | Risk |
|---|---|---|---|
| `pulser.github.pr_status` | CLI, MCP, Foundry | Get PR status (checks, mergeable) | low |
| `pulser.odoo.module_test` | CLI, Foundry | Run Odoo module tests | low |
| `pulser.graph.group_membership_read` | MCP, Foundry, Azure Function | Read user's M365 groups for RBAC | low |

### Wave 2: Runtime Skills (Weeks 3-4)

| Skill | Adapter(s) | Purpose | Risk |
|---|---|---|---|
| `pulser.vat_filing_evidence` | Foundry, Claude Skill, MCP | Prepare VAT filing evidence log | medium |
| `pulser.bir_certificate_review` | Foundry, Claude Skill, MCP | Review BIR certificates (extract, validate, match) | medium |
| `pulser.office.generate_workbook` | Foundry, Azure Function, MCP | Generate Excel reports from Odoo data | medium |

### Wave 3: Admin Skills (Weeks 5-6)

| Skill | Adapter(s) | Purpose | Risk |
|---|---|---|---|
| `pulser.powershell.e5_sandbox_rbac_seed` | CLI, Foundry | Create sandbox M365 groups | medium |
| `pulser.m365_agent_package_validation` | CLI, Foundry, MCP | Validate Teams/M365 agent manifests | low |
| `pulser.azure.resource_inventory` | CLI, Foundry, MCP | Query Azure resource inventory | medium |

---

## 7. Guardrails

### 7.1 Global Denials

```yaml
global_guardrails:
  # Execution
  - no_arbitrary_shell
  - no_arbitrary_powershell
  - no_model_generated_code_execution_without_sandbox
  - no_skill_installs_from_untrusted_sources
  
  # Authority
  - no_autonomous_bir_filing
  - no_autonomous_tax_payment
  - no_autonomous_accounting_posting
  - no_customer_email_send_without_approval
  - no_tenant_wide_admin_consent_without_approval
  
  # Vendor
  - no_supabase_runtime_dependency
  - no_n8n_runtime_dependency
  - no_databricks_connect_in_odoo_env
  
  # Authority Drift
  - customer_token_terminates_at_api_boundary
  - no_token_forwarding_to_third_party_apis
  - odoo_is_erp_source_of_truth
  - foundry_is_agent_runtime_source_of_truth
```

### 7.2 Security Rule

```text
A Claude Skill, Azure plugin, MCP tool, or CLI wrapper can only call
capabilities declared in the Pulser skill contract and allowed by 
MCP/RBAC policy. No exceptions.
```

**Enforcement**:
1. Skill contract lists `allowed` tools only
2. Adapters validate inputs against schema
3. Policy engine enforces RBAC + approval gates
4. All invocations logged with evidence trail

---

## 8. Acceptance Criteria

### 8.1 Architecture

- [x] **Pulser Skill Contract is the SSOT**
  - One skill.yaml + SKILL.md per skill
  - All adapters derived from contract
  - No duplicate business logic across platforms

- [x] **Claude Skills are export artifacts, not first-class citizens**
  - Claude Skill is a convenient export format
  - Claude still calls Pulser Tool API
  - No Claude-specific business logic in Pulser

- [x] **Azure/M365 plugins are adapter/distribution artifacts**
  - M365 plugin = OpenAPI descriptor or MCP endpoint
  - No separate logic; all backed by Pulser Skill Contract
  - M365 plugin calls Pulser Tool API

- [x] **No business logic duplication**
  - Extraction logic written once (in Pulser Tool API)
  - Adapters only change transport/interface
  - Evals written once; shared across all platforms

### 8.2 Governance

- [x] **No unrestricted shell or tenant-admin execution**
  - CLI wrappers have typed inputs only
  - PowerShell skills have cmdlet allowlist
  - No Invoke-Expression, no Start-Process, no arbitrary shell

- [x] **No authority drift from Odoo/Foundry/M365 boundaries**
  - Odoo remains ERP source of truth
  - Foundry remains agent runtime source of truth
  - M365 Graph permissions enforced by delegated/app-only auth

- [x] **Evidence required for all skill invocations**
  - Immutable JSON logs
  - Retention policy enforced (7 years for tax skills)
  - Searchable by user, skill, tenant, time range

- [x] **Approval gates matched to risk**
  - low = auto-allowed
  - medium = ≥1 approval
  - high/critical = ≥2 approval + change control

### 8.3 Operational

- [x] **All skills have evals**
  - Extraction accuracy
  - Validation accuracy
  - Confidence tier calibration
  - Output relevance

- [x] **All skills emit evidence**
  - User ID, skill name, inputs, outputs
  - Execution time, error messages
  - Approval gate status

- [x] **Skill contracts validate at load time**
  - Missing required fields → error
  - Invalid schema → error
  - Unapproved adapters → error

---

## 9. Compliance Mapping

### Pulser Skills → Anthropic Claude Skills

**Reference**: [Anthropic Skills README](https://github.com/anthropics/skills/blob/main/README.md)

**Rule**: Pulser SKILL.md + adapters/claude_skill_manifest.yaml = valid Claude Skill export.

**Validation**: Claude skill loaded successfully and operates within boundaries.

### Pulser Skills → MCP Tools

**Reference**: [Model Context Protocol Spec](https://modelcontextprotocol.io/)

**Rule**: Pulser adapters/mcp.yaml generates valid MCP tool schemas.

**Validation**: MCP server discovery returns all expected tools.

### Pulser Skills → M365 Copilot Plugins

**Reference**: [Microsoft 365 Copilot Plugin Docs](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/overview-plugins)

**Rule**: Pulser adapters/openapi.yaml describes valid OpenAPI or MCP endpoint.

**Validation**: M365 plugin loads and agent can call skills.

---

## 10. Example: `pulser.github.pr_status` (Wave 1)

### 10.1 skill.yaml

```yaml
name: pulser.github.pr_status
kind: pulser_skill
domain: devops
risk: low
runtime: pulser_tool_api

description: "Get GitHub PR status: checks, mergeable, labels, reviewers"

inputs:
  repo:
    type: string
    required: true
    validation: "owner/repo format"
  pr_number:
    type: integer
    required: true

outputs:
  state: { type: string, enum: ["open", "closed", "merged"] }
  mergeable: { type: boolean }
  checks:
    type: array
    items:
      type: object
      properties:
        name: { type: string }
        status: { type: string, enum: ["pass", "fail", "pending"] }
  labels: { type: array, items: { type: string } }
  reviewers:
    type: array
    items:
      type: object
      properties:
        login: { type: string }
        state: { type: string, enum: ["approved", "changes_requested", "commented"] }

tooling:
  allowed:
    - github.rest_api.pull_requests.get
  denied:
    - github.merge
    - github.close
    - github.delete_branch

evidence:
  required: true
  retention: P90D

evals:
  - pr_status_accuracy
```

### 10.2 SKILL.md

```markdown
# GitHub PR Status

Get the current status of a GitHub pull request: checks, mergeable status, labels, and reviewer decisions.

**Does NOT**: Merge, close, or delete PR.

## Usage

```bash
$ pulser github pr-status --repo Insightpulse-ai/agent-platform --pr 42
{
  "state": "open",
  "mergeable": true,
  "checks": [
    { "name": "build", "status": "pass" },
    { "name": "test", "status": "pass" }
  ],
  "labels": ["docs", "priority:high"],
  "reviewers": [
    { "login": "alice", "state": "approved" }
  ]
}
```
```

### 10.3 adapters/cli.py

```python
@pulser.group()
def github():
    pass

@github.command()
@click.option('--repo', required=True)
@click.option('--pr', type=int, required=True)
def pr_status(repo, pr):
    """Get GitHub PR status"""
    # Call Pulser Tool API
    # Return JSON
    pass
```

### 10.4 adapters/mcp.yaml

```yaml
tools:
  - name: pulser.github.pr_status
    description: "Get GitHub PR status"
    input_schema:
      properties:
        repo: { type: string }
        pr_number: { type: integer }
      required: ["repo", "pr_number"]
    result_schema:
      properties:
        state: { enum: ["open", "closed", "merged"] }
        mergeable: { type: boolean }
```

---

## 11. Related Documentation

- **[CLI Skills Governance Model](./cli-skills-model.md)** — CLI tool guardrails and adapters
- **[Graph + PowerShell Skills Model](./graph-powershell-skills-model.md)** — M365-specific skills and auth
- **[Conversation Behavior Spec](./conversation-behavior-spec.md)** — Pulser LLM reasoning
- **[Anthropic Skills Guide](https://github.com/anthropics/skills/blob/main/README.md)** — Claude Skill format
- **[Model Context Protocol](https://modelcontextprotocol.io/)** — MCP tool standard
- **[Microsoft 365 Copilot Plugins](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/overview-plugins)** — M365 plugin docs
- **[Azure Functions MCP Bindings](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-mcp)** — Serverless MCP tool bindings

---

## 12. Versioning & Changelog

| Version | Date | Change | Author |
|---|---|---|---|
| 1.0 | 2026-04-25 | Initial specification (contract schema, 6 adapters, 9 initial skills, guardrails) | Pulser Team |

---

**Next Action**: Create PR for architecture review and security sign-off.

```text
docs(product): define Pulser vendor-neutral skills and plugin model

Establish one canonical Pulser Skill Contract that adapts to Claude Skills,
MCP tools, M365 Copilot plugins, Azure Functions, Foundry tools, and CLI
wrappers without duplicating logic.

- Skill Contract (SSOT): skill.yaml + SKILL.md (vendor-neutral)
- Adapters: mcp.yaml, openapi.yaml, azure_function.json, foundry_tool.yaml,
  claude_skill_manifest.yaml, cli.py (derived from contract)
- Repo layout: agent-platform/pulser/skills/<skill_name>/
- Initial skills: github.pr_status, odoo.module_test, graph.group_membership_read
- Guardrails: no arbitrary shell, no autonomous tax/payment, no token forwarding,
  no Supabase/n8n, evidence required

Acceptance criteria:
- Skill Contract is SSOT; all adapters derived
- Claude Skills are export artifacts only
- M365/Azure plugins are adapter/distribution only
- No business logic duplication
- No unrestricted shell or tenant-admin execution
- Authority boundaries (Odoo/Foundry/M365) enforced

Closes: Related to CLI Skills Model and Graph + PowerShell Skills Model PRs
```
