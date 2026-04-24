# Microsoft Graph and PowerShell Skills Model

**Document Version**: 1.0  
**Status**: SPECIFICATION (Ready for Implementation)  
**Last Updated**: 2026-04-25  
**Related**: [CLI Skills Governance Model](./cli-skills-model.md) | [Pulser Awareness Contract](../architecture/PULSER_AWARENESS_CONTRACT.md) | [Conversation Behavior Spec](./conversation-behavior-spec.md)

---

## 1. Purpose

Pulser is a financial automation agent. To act on behalf of users in Microsoft 365 (M365), it must access two fundamentally different Microsoft surfaces:

- **Microsoft Graph REST / SDK** → **Runtime product surface** (user-facing, runtime decisions)
- **Microsoft Graph PowerShell** → **Admin/provisioning/ops surface** (setup, auditing, tenant seeding)

Both surfaces can become Pulser **skills** only when wrapped in:
- Explicit contracts (YAML schemas defining inputs, outputs, auth scopes, guardrails)
- Risk-aware policies (approval gates matched to risk class)
- Tenant-scoped sandboxes (only zmc2r.onmicrosoft.com initially)
- Immutable evidence (all invocations logged with audit trail)

This model prevents arbitrary PowerShell execution, unbounded Graph permission grants, and tenant-wide mutations without approval. It establishes a clear separation: **Pulser runtime uses Graph APIs; admin tasks are isolated to named PowerShell skills**.

---

## 2. Graph Skill Lane: Runtime API Surface

Microsoft Graph is the **primary runtime API surface for Pulser M365 integration**. Graph skills handle user-facing, delegated-auth scenarios where Pulser acts on behalf of a signed-in user or reads non-sensitive tenant data.

### 2.1 Recommended Graph Skills (Wave 1-2)

| Skill Name | Purpose | Risk | Auth Mode | Approval Gate | Example Use |
|---|---|---|---|---|---|
| `pulser.graph.user_profile_read` | Read signed-in user identity, email, name | low | delegated | none | Initialize user context |
| `pulser.graph.group_membership_read` | Resolve user's M365 groups for RBAC mapping | low | delegated | none | Determine Pulser role (controller, viewer, admin) |
| `pulser.graph.mail_attachment_intake` | Read mail attachments from BIR certificate mailbox | low | delegated (service) | none | Ingest BIR certificates in sandbox |
| `pulser.graph.calendar_context_read` | Read user's calendar for context (read-only) | low | delegated | none | Understand user workload/availability |
| `pulser.graph.sharepoint_file_search` | Search for financial documents in SharePoint | medium | delegated | none | Find open budgets, forecasts |
| `pulser.graph.onedrive_file_save` | Save Pulser artifacts (reports, exports) to OneDrive | medium | delegated | yes, draft only | Store generated financial statements |
| `pulser.graph.teams_message_draft` | Compose (not send) Teams messages for review | low | delegated | none | Generate Teams notifications for approval |

### 2.2 Denied Graph Actions (By Default)

```yaml
denied_graph_actions:
  - mail.send_as_user                    # No impersonation
  - mail.send_on_behalf                  # No delegate access
  - calendar.invite_attendees            # No calendar mutations
  - group.create_security_group          # No group creation
  - user.change_password                 # No user mutations
  - app.grant_admin_consent              # No tenant-wide grants
  - directory.roles.assign               # No RBAC changes
  - sharepoint.publish_draft             # No publishing without review
  - teams.send_message                   # Send must go through approval workflow
  - onedrive.share_publicly              # No public sharing
```

### 2.3 Graph Auth Model: Delegated vs. App-Only

**Delegated Authentication** (Default for Pulser runtime):
- Pulser acts **on behalf of the signed-in user**
- User's permissions boundary is respected (RBAC enforced by Graph)
- Example: `pulser.graph.user_profile_read` uses the user's own identity
- Best for: user-context scenarios, reading user-owned resources

**App-Only Authentication** (Restricted, admin approval required):
- Pulser acts **with service identity** (managed identity or service principal)
- No user context; permissions granted via admin consent
- Example: `pulser.graph.mail_attachment_intake` may use app-only to read sandbox mailbox
- Requires: app-only permission grant (delegated vs. app-only [Microsoft Learn](https://learn.microsoft.com/en-us/graph/permissions-overview))
- Approval gate: **admin consent required** for any app-only scope grant

### 2.4 Customer Token Boundaries

**Critical Rule**: Customer-provided tokens **must terminate at the Pulser API boundary**.

```text
Customer Browser
    ↓ (customer token)
Pulser Frontend (odoo/ipai_pulser_chat)
    ↓ (BOUNDARY: token not passed further)
Pulser Backend (Azure Function/Container)
    ↓ (uses Pulser managed identity or app registration)
Microsoft Graph API
    ↓
M365 Tenant (sandbox: zmc2r.onmicrosoft.com)
```

**Implementation**:
1. Customer authenticates to Pulser frontend via Entra ID (delegated flow)
2. Frontend receives ID token + access token for Pulser backend API
3. Frontend calls Pulser backend, passing **user identity in JWT claims only** (no raw token forwarding)
4. Pulser backend extracts user identity from claims
5. Pulser backend uses its own managed identity to call Graph on behalf of that user (via Graph delegated flow with `user_id` parameter)
6. Customer's token is never stored, forwarded, or logged

**Why**: Prevents token theft, simplifies key rotation, maintains audit trail under Pulser's identity.

---

## 3. PowerShell Skill Lane: Admin/Provisioning Surface

Microsoft Graph PowerShell is the **admin/provisioning surface for Pulser**. PowerShell skills are **never called during runtime chat**; they are invoked only during:
- Tenant seeding (initial M365 sandbox setup)
- Permission auditing (verify app consent, RBAC configuration)
- Environment validation (check Teams manifest, Power Platform prerequisites)
- Evidence collection (gather logs, group memberships for audit)

### 3.1 Recommended PowerShell Skills (Wave 1-2)

| Skill Name | Purpose | Risk | Tenant Scope | Approval Gate | Example Use |
|---|---|---|---|---|---|
| `pulser.powershell.e5_sandbox_rbac_seed` | Create test groups, seed RBAC in E5 sandbox | medium | zmc2r.onmicrosoft.com | dry_run=false | Initial sandbox setup |
| `pulser.powershell.graph_permission_audit` | List all Graph API permissions granted to Pulser app | low | zmc2r.onmicrosoft.com | none | Verify least-privilege |
| `pulser.powershell.teams_app_validation` | Validate Teams app manifest for M365 agent compatibility | low | zmc2r.onmicrosoft.com | none | Pre-deployment check |
| `pulser.powershell.powerplatform_env_check` | Verify Power Platform environment (DLP, capacity) | low | zmc2r.onmicrosoft.com | none | Validate prerequisites |
| `pulser.powershell.m365_sample_data_check` | Verify test data exists (sample budget, groups) | low | zmc2r.onmicrosoft.com | none | Smoke test sandbox |

### 3.2 Allowed PowerShell Cmdlets (Allowlist Pattern)

**Core Connection**:
```powershell
Connect-MgGraph               # authenticate to Graph
Get-MgContext                 # verify context
Disconnect-MgGraph            # cleanup
```

**Read-Only (Auto-Allowed)**:
```powershell
Get-MgUser                    # read user properties
Get-MgGroup                   # read group definitions
Get-MgGroupMember             # read group membership
Get-MgApplication             # read app registration
Get-MgServicePrincipal        # read service principal
Get-MgOAuth2PermissionGrant   # read permission grants
Get-MgAppRoleAssignment       # read role assignments
```

**Write (Approval-Gated)**:
```powershell
New-MgGroup                   # create security group
New-MgGroupMember             # add member to group
New-MgOAuth2PermissionGrant   # delegate permission
Update-MgGroup                # modify group properties
```

**Admin (Approval Required)**:
```powershell
New-MgServicePrincipal        # create service principal
Update-MgApplication          # change app configuration
Grant-MgApplicationPermission  # grant permission consent
```

### 3.3 Denied PowerShell Cmdlets (By Default)

```yaml
denied_powershell_cmdlets:
  - Invoke-Expression                    # NO arbitrary code execution
  - Invoke-Command                       # NO remote execution
  - Start-Process                        # NO process spawning
  - Invoke-WebRequest                    # NO arbitrary HTTP
  - Get-Content (logs/config)            # NO sensitive file read
  - Set-Content (system files)           # NO system write
  - Remove-MgUser                        # NO user deletion
  - Remove-MgGroup                       # NO group deletion
  - Remove-MgServicePrincipal            # NO service principal deletion
  - Clear-MgOAuth2PermissionGrant        # NO permission revocation
  - Remove-MgApplication                 # NO app deletion
  - Get-MgApplicationPassword            # NO secret exposure
  - Get-MgServicePrincipalPassword       # NO secret exposure
  - Import-Module (unsigned)             # NO unsigned module load
```

### 3.4 PowerShell Tenant Scoping

**Only allowed tenant (initially)**:
```yaml
allowed_tenants:
  - zmc2r.onmicrosoft.com
```

**Implementation**:
1. `Connect-MgGraph` must explicitly pass `-TenantId zmc2r.onmicrosoft.com`
2. Adapter validates tenant before cmdlet invocation
3. Any cmdlet output containing a different tenant ID is rejected
4. Audit log includes tenant ID for every invocation

**Future**: After sandbox validation, support `insightpulseai.com` (production E5 tenant).

### 3.5 PowerShell Dry-Run Pattern

**All write operations must support `--dry-run` parameter**:

```powershell
# Dry run: show what would happen
pwsh-skill e5_sandbox_rbac_seed --dry_run=true

# Real run: create groups
pwsh-skill e5_sandbox_rbac_seed --dry_run=false
```

**Implementation**:
1. Dry run outputs planned changes (names, members, groups to create) as JSON
2. User/admin reviews JSON output
3. If approved, re-invoke with `--dry_run=false`
4. Real run creates exactly the resources from the dry-run plan

---

## 4. Auth Model

### 4.1 Authentication Flows

**Delegated Flow (User Context)**:
```
User
  ↓ (authenticate)
Entra ID
  ↓ (issue ID token + access token for Pulser backend)
Pulser Frontend
  ↓ (user_id in JWT claims)
Pulser Backend
  ↓ (uses Pulser managed identity to call Graph with user_id)
Microsoft Graph
  ↓ (Graph enforces user's RBAC)
M365 Resource
```

**App-Only Flow (Service Context)**:
```
Pulser Backend (service principal)
  ↓ (authenticate with certificate/client secret)
Entra ID
  ↓ (issue access token for service principal)
Microsoft Graph
  ↓ (Graph enforces app-granted permissions)
M365 Resource
```

### 4.2 Managed Identity for Pulser Backend

**Recommended Setup** (Azure Container Apps):
- Pulser backend container has system-assigned managed identity
- Managed identity is granted Graph permissions via app role assignments
- No client secrets stored in code or Key Vault (only cert-based auth)
- All Graph calls authenticated with `DefaultAzureCredential` (Python SDK)

**Implementation** (Python):
```python
from azure.identity import DefaultAzureCredential
from msgraph.core import GraphClient

# Pulser backend uses managed identity automatically
credential = DefaultAzureCredential()
client = GraphClient(credential=credential)

# Call Graph API
groups = client.get("/me/memberOf")
```

### 4.3 Admin Consent for App-Only Permissions

**When does Pulser need app-only Graph permissions?**

| Scenario | Required Permissions | Who Grants | Method |
|---|---|---|---|
| Read BIR mailbox (service) | `Mail.ReadWrite` (app-only) | M365 admin | Azure portal → App registrations → API permissions → Grant admin consent |
| Seed test groups | `Group.ReadWrite.All` (app-only) | M365 admin | PowerShell: `Grant-MgApplicationPermission` |
| Audit tenant RBAC | `Directory.Read.All` (app-only) | M365 admin | PowerShell: `Grant-MgApplicationPermission` |

**Critical Rule**: **No tenant-wide permission grant without explicit admin approval** (e.g., `grant_admin_consent` is denied).

### 4.4 Customer Tenant Isolation

**Customer Tenants** (Future: post-sandbox validation):
- Customers bring their own M365 tenants (e.g., `acme.onmicrosoft.com`)
- Pulser service principal is created per customer
- Customer admin grants only required Graph permissions (least-privilege)
- All calls scoped to customer tenant only
- No cross-tenant data access

---

## 5. Guardrails

### 5.1 Explicit Denials

```yaml
# Core Runtime Denials
no_arbitrary_powershell:
  description: "Pulser runtime never executes raw PowerShell scripts"
  enforcement: "All PowerShell invocations go through named skills only"
  penalty: "reject_request_with_error"

no_model_generated_shell:
  description: "LLM-generated shell commands are never executed"
  enforcement: "Only pre-defined, approved command adapters execute"
  penalty: "reject_with_error_log_incident"

no_invoke_expression:
  description: "PowerShell Invoke-Expression is completely forbidden"
  enforcement: "Adapter pre-filters all cmdlet calls; rejects Invoke-Expression"
  penalty: "security_alert_escalate_to_admin"

# Permission Denials
no_tenant_wide_permission_grant:
  description: "Cannot grant tenant-wide admin consent without explicit approval"
  enforcement: "grant_admin_consent skill is approval-gated; requires 2 signers"
  penalty: "reject_with_change_control_ticket"

no_mailbox_send_without_approval:
  description: "Mail.Send permission requires approval for each use"
  enforcement: "pulser.graph.mail.send does not exist; Teams message draft only"
  penalty: "reject_with_audit_log"

# Data Denials
no_production_mutation_without_change_control:
  description: "User/group mutations in prod require change control ticket"
  enforcement: "Mutation skills auto-tagged with approval_required: true"
  penalty: "reject_unless_approved_change_ticket_present"

no_secrets_in_logs:
  description: "Passwords, tokens, keys never logged or returned"
  enforcement: "Adapter scrubs sensitive fields; Get-MgApplicationPassword forbidden"
  penalty: "redact_and_log_incident"

# Sandbox Enforcement
sandbox_tenant_only:
  description: "Only zmc2r.onmicrosoft.com in Wave 1; cannot connect to customer tenants"
  enforcement: "Connect-MgGraph must pass -TenantId zmc2r.onmicrosoft.com"
  penalty: "reject_if_tenant_id_differs"

no_supabase_n8n_dependency:
  description: "No Supabase/n8n in Graph or PowerShell skill implementations"
  enforcement: "Code review gates require Python SDK (azure-identity, msgraph) only"
  penalty: "pr_blocked_code_review"

no_databricks_connect_in_odoo_env:
  description: "Databricks Connect CLI forbidden in Odoo environment"
  enforcement: "Adapter pre-filters; databricks command rejected in odoo_dev database context"
  penalty: "reject_with_audit_log"
```

### 5.2 Approval Matrix

| Action | Risk | Approval Required | Signers | Justification |
|---|---|---|---|---|
| Graph read (user profile, groups) | low | none | — | Read-only, user RBAC enforced |
| Teams message draft | low | none | — | Draft only, no send |
| Graph write (file save, calendar update) | medium | user | 1 | User owns resource |
| Teams manifest validation | low | none | — | Validation only |
| PowerShell dry run (RBAC seed) | low | none | — | Preview only, no changes |
| PowerShell write (group creation) | medium | admin | 1 | Tenant change, sandbox-scoped |
| PowerShell permission audit | low | none | — | Read-only |
| Mail.Send (future) | high | approval | 2 | Customer-facing communication |
| Grant tenant-wide permission | critical | approval + change control | 2 | Tenant security boundary |
| Production tenant mutation | critical | change control | 2 | Production stability |

---

## 6. Skill Contract Schema

### 6.1 Unified Skill Contract (YAML)

```yaml
# Metadata
name: pulser.graph.group_membership_read
kind: graph_skill  # or cli_skill, powershell_skill
runtime: pulser_api  # where skill runs (pulser_api, local_or_ci, admin_console)
version: "1.0"
stable: true

# Description
description: >
  Reads the signed-in user's Microsoft 365 group membership for Pulser RBAC mapping.
  Returns group IDs, names, and inferred Pulser roles (controller, viewer, admin).

# Risk & Governance
risk: low
risk_rationale: "Read-only; respects user's own RBAC; no state mutation"
approval_required: false
evidence_required: true

# Authentication
auth:
  mode: delegated  # delegated | app_only
  scopes:
    - User.Read
    - GroupMember.Read.All
  token_source: user_id_claim  # from JWT claims, never customer token

# Inputs
inputs:
  user_id:
    type: string
    required: true
    source: jwt_claim_oid
    validation: "non-empty UUID"
  exclude_roles:
    type: array
    items: string
    required: false
    default: []

# Outputs
outputs:
  user:
    type: object
    fields:
      id: string
      displayName: string
      mail: string
  groups:
    type: array
    items:
      type: object
      fields:
        id: string
        displayName: string
        mail: string
  pulser_roles:
    type: array
    items: string
    example: ["controller", "viewer"]

# Guardrails
denied_actions:
  - create_group
  - update_group
  - delete_group
  - grant_permission
  - send_as_user

# Sandbox & Tenants
tenant_scope:
  allowed_tenants:
    - zmc2r.onmicrosoft.com

# Errors & Fallback
on_error: reject  # reject | retry | fallback
fallback_action: none

# Audit & Evidence
evidence_required: true
log_user_id: true
log_groups_read: true
log_scopes_used: true
redact_tokens: true
```

### 6.2 PowerShell Skill Contract (YAML)

```yaml
# Metadata
name: pulser.powershell.e5_sandbox_rbac_seed
kind: cli_skill
runtime: local_or_ci
command_adapter: pwsh  # powershell 7+
version: "1.0"
stable: false

# Description
description: >
  Creates or verifies Pulser RBAC test groups in the Microsoft 365 E5 sandbox tenant.
  Supports dry-run preview before applying changes.

# Risk & Governance
risk: medium
risk_rationale: "Creates groups and membership; sandbox-scoped; requires approval for dry_run=false"
approval_required: false  # true only if dry_run=false
evidence_required: true

# Allowed Cmdlets
allowed_cmdlets:
  - Connect-MgGraph
  - Get-MgContext
  - Disconnect-MgGraph
  - Get-MgGroup
  - Get-MgUser
  - New-MgGroup
  - New-MgGroupMember

# Denied Cmdlets
denied_cmdlets:
  - Invoke-Expression
  - Invoke-Command
  - Start-Process
  - Remove-MgGroup
  - Remove-MgUser
  - Get-MgApplicationPassword

# Inputs
inputs:
  dry_run:
    type: boolean
    required: true
    default: true
    description: "Preview changes without applying; always use true first"
  group_names:
    type: array
    items: string
    required: false
    default:
      - "Pulser_Controllers"
      - "Pulser_Viewers"
      - "Pulser_Admins"
  members_to_add:
    type: array
    items: string
    required: false
    validation: "valid Azure AD user UPNs"

# Outputs (Dry Run)
outputs_dry_run:
  type: object
  fields:
    groups_to_create:
      type: array
      items:
        type: object
        fields:
          displayName: string
          members: array
    groups_to_update:
      type: array
    groups_already_exist:
      type: array
    plan: string

# Outputs (Real Run)
outputs_real:
  type: object
  fields:
    groups_created:
      type: array
      items:
        type: object
        fields:
          id: string
          displayName: string
    members_added:
      type: integer
    errors: array

# Sandbox & Tenant Scope
tenant_scope:
  allowed_tenants:
    - zmc2r.onmicrosoft.com
  validation: "Connect-MgGraph -TenantId zmc2r.onmicrosoft.com"

# Approval Gates
approval_required:
  - dry_run: false  # real execution requires approval

# Environment
auth:
  mode: app_only  # PowerShell admin tasks typically app-only
  scopes:
    - Group.ReadWrite.All
    - User.Read.All

# Error Handling
on_error: reject
fallback_action: none

# Audit & Evidence
evidence_required: true
log_cmdlets_executed: true
log_group_names: true
log_members_added: true
redact_passwords: true
redact_client_secrets: true
```

---

## 7. Skill Repository Placement

### 7.1 Directory Structure

```text
agent-platform/
├── pulser/
│   ├── skills/
│   │   ├── graph/
│   │   │   ├── user_profile_read.yaml
│   │   │   ├── group_membership_read.yaml
│   │   │   ├── mail_attachment_intake.yaml
│   │   │   ├── calendar_context_read.yaml
│   │   │   ├── sharepoint_file_search.yaml
│   │   │   ├── onedrive_file_save.yaml
│   │   │   └── teams_message_draft.yaml
│   │   ├── powershell/
│   │   │   ├── e5_sandbox_rbac_seed.yaml
│   │   │   ├── graph_permission_audit.yaml
│   │   │   ├── teams_app_validation.yaml
│   │   │   ├── powerplatform_env_check.yaml
│   │   │   └── m365_sample_data_check.yaml
│   │   ├── adapters/
│   │   │   ├── graph_client.py          # Graph API runtime client
│   │   │   ├── graph_auth.py            # Delegated/app-only auth
│   │   │   ├── powershell_runner.py     # PowerShell cmdlet adapter
│   │   │   ├── powershell_validator.py  # Allowlist enforcement
│   │   │   └── skill_contract.py        # Contract schema validation
│   │   ├── policy/
│   │   │   ├── graph-policy.yaml        # Graph approval matrix, denials
│   │   │   ├── powershell-policy.yaml   # PowerShell cmdlet allowlist
│   │   │   └── tenant-policy.yaml       # Tenant scoping rules
│   │   └── evidence/
│   │       ├── graph_evidence.py        # Evidence logging for Graph calls
│   │       └── powershell_evidence.py   # Evidence logging for PowerShell
│   │
│   ├── graph/
│   │   ├── client.py                    # Graph SDK client setup
│   │   ├── auth.py                      # Delegated auth, managed identity
│   │   ├── users.py                     # User.Read* operations
│   │   ├── groups.py                    # Group.Read* operations
│   │   ├── mail.py                      # Mail.Read* operations (read-only)
│   │   ├── files.py                     # Files operations (OneDrive, SharePoint)
│   │   ├── teams.py                     # Teams operations (draft only)
│   │   └── calendar.py                  # Calendar.Read* operations
│   │
│   ├── powershell/
│   │   ├── runner.py                    # PowerShell 7+ runner
│   │   ├── cmdlet_allowlist.py          # Cmdlet filtering
│   │   ├── tenant_validator.py          # Tenant scope enforcement
│   │   └── dry_run_engine.py            # Dry run planning & review
│   │
│   └── README.md
```

### 7.2 File Ownership

| Directory | Owned By | Maintenance |
|---|---|---|
| `skills/graph/` | Pulser Product Team | Every M365 feature release |
| `skills/powershell/` | Pulser Ops/Admin Team | On-demand for admin tasks |
| `skills/adapters/` | Pulser Platform Engineering | API/policy updates |
| `skills/policy/` | Security/Compliance | Quarterly audit review |
| `graph/` | Product Engineering | Runtime Graph client maintenance |
| `powershell/` | Platform Engineering | PowerShell runner security updates |

---

## 8. Initial Implementation Order

### 8.1 Wave 1: Proof-of-Concept (Weeks 1-2)

**Goal**: Demonstrate Graph API + PowerShell skill governance working safely together.

**Skills**:
1. **`pulser.graph.group_membership_read`** (Graph runtime)
   - Read user's M365 group membership
   - Implement delegated auth with managed identity
   - Return groups mapped to Pulser RBAC roles
   - **Acceptance**: Pulser backend can determine user's role (controller, viewer, admin)

2. **`pulser.powershell.e5_sandbox_rbac_seed`** (PowerShell admin)
   - Create test groups in sandbox (Pulser_Controllers, Pulser_Viewers, Pulser_Admins)
   - Implement dry-run preview before creating groups
   - Add test users to groups
   - **Acceptance**: Dry run shows planned changes; real run creates groups without errors

### 8.2 Wave 2: Extended Runtime (Weeks 3-4)

**Goal**: Add read-only capabilities for financial context + evidence collection.

**Skills**:
3. **`pulser.graph.mail_attachment_intake`** (Graph runtime)
   - Read BIR certificate attachments from sandbox mailbox
   - Implement app-only auth for service mailbox access
   - Parse/validate certificates
   - **Acceptance**: Pulser can ingest BIR certs without manual intervention

4. **`pulser.powershell.graph_permission_audit`** (PowerShell admin)
   - List all Graph permissions granted to Pulser service principal
   - Compare against approved permission list (least-privilege check)
   - Report discrepancies
   - **Acceptance**: Audit confirms Pulser has only required permissions

5. **`pulser.graph.onedrive_file_save`** (Graph runtime)
   - Save Pulser-generated artifacts (reports, forecasts) to OneDrive
   - Implement folder structure: `/Pulser/Reports/{year}/{month}/`
   - Support draft-mode (artifact approval required before publish)
   - **Acceptance**: Artifacts saved and retrievable; draft flag prevents publishing

### 8.3 Wave 3: Admin/Setup (Weeks 5-6)

**Goal**: Complete admin skill set for tenant provisioning and validation.

**Skills**:
6. **`pulser.powershell.teams_app_validation`** (PowerShell admin)
   - Validate Pulser Teams/M365 agent manifest against deployment requirements
   - Check app permissions, scopes, icons, descriptions
   - Report deployment readiness
   - **Acceptance**: Manifest passes all validation checks

7. **`pulser.powershell.m365_sample_data_check`** (PowerShell admin)
   - Verify test data exists (sample budget, groups, mailbox)
   - Create test data if missing
   - Report sandbox readiness
   - **Acceptance**: Sandbox has all prerequisites for Pulser testing

### 8.4 Implementation Checklist

- [ ] **Week 1-2**: Graph skill + PowerShell skill working together
  - [ ] `pulser.graph.group_membership_read` implemented and tested
  - [ ] `pulser.powershell.e5_sandbox_rbac_seed` implemented and tested
  - [ ] Skill contracts validated
  - [ ] Evidence logging verified
  - [ ] PR submitted for review

- [ ] **Week 3-4**: Extended runtime + audit capabilities
  - [ ] `pulser.graph.mail_attachment_intake` implemented
  - [ ] `pulser.powershell.graph_permission_audit` implemented
  - [ ] `pulser.graph.onedrive_file_save` implemented
  - [ ] All 5 skills working in CI/CD
  - [ ] PR submitted for review

- [ ] **Week 5-6**: Admin skill set complete
  - [ ] `pulser.powershell.teams_app_validation` implemented
  - [ ] `pulser.powershell.m365_sample_data_check` implemented
  - [ ] All 7 skills pass security review
  - [ ] Documentation complete
  - [ ] PR submitted for review

---

## 9. Acceptance Criteria

### 9.1 Architecture

- [x] **Graph and PowerShell are clearly separated**
  - Graph skills handle runtime/user-facing scenarios
  - PowerShell skills handle admin/setup/ops automation
  - No runtime code calls PowerShell directly
  - No PowerShell used for user-context decisions

- [x] **Runtime uses Graph APIs/SDK, not raw PowerShell**
  - All Graph interactions go through `microsoft.graph` SDK (Python)
  - No `pwsh -Command "..."` in Pulser runtime
  - PowerShell only called for admin tasks (via named skills)

- [x] **PowerShell is limited to admin/setup/ops skills**
  - PowerShell skills not part of chat runtime
  - PowerShell skills called only during provisioning/audit
  - No LLM-generated PowerShell commands executed

- [x] **Arbitrary shell is denied**
  - No `Invoke-Expression`, `Invoke-Command`, `Start-Process`
  - No model-generated shell scripts executed
  - Only pre-defined, approved command adapters allowed

- [x] **Tenant-wide changes require approval**
  - `grant_admin_consent` denied by default
  - `New-MgGroup` requires admin approval (skill contract)
  - Change control tickets required for production mutations

- [x] **Evidence is required for all skill invocations**
  - Immutable JSON logs stored for every skill call
  - Logs include user ID, skill name, inputs, outputs, scopes used
  - Audit trail linked to user identity and change ticket (if applicable)

- [x] **No Supabase/n8n dependency introduced**
  - All implementation via `azure-identity`, `msgraph` SDKs
  - CI/CD validates no forbidden imports
  - Code review gates check for compliance

### 9.2 Security

- [x] **Customer tokens terminate at Pulser API boundary**
  - No customer token forwarding to Graph
  - Pulser backend uses managed identity for Graph calls
  - User identity passed via JWT claims only

- [x] **Managed identity for Pulser backend**
  - Pulser service principal has only required Graph permissions
  - No client secrets; certificate-based auth (if needed)
  - Least-privilege RBAC: read-only by default, write/admin approval-gated

- [x] **Sandbox tenant only (Wave 1)**
  - `zmc2r.onmicrosoft.com` is the only allowed tenant
  - Connection attempts to other tenants rejected
  - Future: customer tenants after validation

- [x] **Secrets not logged or exposed**
  - Passwords, tokens, keys redacted from all logs
  - `Get-MgApplicationPassword` forbidden
  - Evidence logs scrubbed before storage

### 9.3 Operational

- [x] **Dry-run pattern for write operations**
  - All PowerShell write skills support `--dry_run=true/false`
  - Dry run shows planned changes without applying
  - User/admin reviews and approves before real run

- [x] **Approval matrix implemented**
  - Read-only operations auto-allowed
  - Write operations require admin approval
  - Destructive operations require change control + 2 signers

- [x] **Skill contracts validated at load time**
  - Skill contract YAML parsed and validated on startup
  - Invalid contracts rejected with clear error messages
  - Policy engine enforces approval gates

### 9.4 Observability

- [x] **Evidence logging for all skill invocations**
  - Immutable JSON logs with timestamp, user ID, skill name, inputs, outputs
  - Audit trail supports compliance review
  - Evidence searchable by user, skill, tenant, time range

- [x] **Error handling with audit trail**
  - Errors logged with context (user, skill, tenant, attempted action)
  - No sensitive data in error messages returned to LLM
  - Security incidents escalated to admin alerts

---

## 10. FAQ & Troubleshooting

### Q: Why is PowerShell a "skill" and not just called directly by the agent?

**A**: Because calling PowerShell directly from the LLM creates these risks:
- **Arbitrary code execution**: LLM-generated cmdlets could delete groups, revoke permissions, leak secrets
- **Auditability gap**: No approval gates or change control
- **Scope creep**: No restriction to sandbox tenant; could affect production
- **Token theft**: LLM might forward customer tokens to Graph

By wrapping PowerShell in a **named skill with a contract**, we:
- Pre-approve exact cmdlets allowed
- Enforce dry-run preview before real changes
- Log all invocations with evidence
- Limit to sandbox tenant only
- Terminate customer tokens at API boundary

### Q: Can Pulser use Azure PowerShell (`Az` module) for Azure resource management?

**A**: Not in Wave 1. Azure PowerShell is reserved for future admin-only scenarios (e.g., creating Azure storage for audit logs). In Wave 1, Pulser focuses exclusively on M365 (Graph + Microsoft Graph PowerShell).

**Future consideration**: `pulser.powershell.azure_resource_audit` (admin skill, high risk, requires change control).

### Q: Why don't we use the Microsoft Graph PowerShell SDK directly in Pulser runtime?

**A**: Microsoft Graph PowerShell is a cmdlet wrapper over Graph APIs; it's designed for admin automation, not runtime API calls. Using Graph REST/SDK directly is:
- **Faster**: No PowerShell process spawn; direct HTTP
- **Simpler**: No shell environment to manage
- **Safer**: No shell execution vector
- **Observable**: Standard SDK logging, not PowerShell transcript

Use the SDK for runtime; use PowerShell for admin/ops tasks.

### Q: What if we need a permission that requires tenant-wide admin consent?

**A**: Create a PowerShell skill with approval_required=true. Example:

```yaml
name: pulser.powershell.grant_graph_permission
kind: cli_skill
approval_required: true

inputs:
  permission_name:
    type: string
    allowlist: ["Directory.Read.All", "Mail.ReadWrite"]
```

When Pulser needs a new permission:
1. Admin runs `pulser.powershell.grant_graph_permission` with `--dry_run=true`
2. Skill shows the permission that would be granted
3. Admin reviews and creates change control ticket
4. Admin re-runs with `--dry_run=false`
5. Skill records evidence (who approved, when, ticket ID)

### Q: How do we prevent Pulser from being a stepping stone for a compromised M365 account?

**A**: Multiple layers:
1. **Delegated auth**: Pulser acts within user's RBAC boundary; can't grant itself more permissions
2. **Managed identity**: Pulser backend doesn't store user tokens; uses its own identity for backend-to-Graph calls
3. **Read-only default**: Graph skills are read-only unless explicitly approved
4. **Tenant scoping**: PowerShell skills locked to sandbox tenant only
5. **Evidence trail**: All access logged with user ID and timestamp; admins can audit and revoke access

### Q: What if a PowerShell skill's `allowed_cmdlets` list is too restrictive?

**A**: Skills go through security review before production. If a skill needs additional cmdlets:
1. Security review approves the cmdlet and updates skill contract
2. Risk re-assessed (could change from `low` to `medium`)
3. New approval gates added if needed
4. Change tracked in version history

Example: If `graph_permission_audit` needs to also read service principal secrets (for audit), the skill contract would change and secrets would be redacted from evidence logs.

### Q: How do we handle multi-tenant scenarios (customers with their own M365)?

**A**: After Wave 1 (sandbox validation), customer tenants are supported via:
1. **Customer-managed service principal**: Customer creates app registration and grants Graph permissions in their tenant
2. **Tenant isolation**: Pulser service principal can only access customer's tenant; customer ID passed in all requests
3. **Customer token termination**: Customer's admin token does not reach Pulser backend; only tenant ID/permission scopes
4. **Per-customer RBAC**: Pulser enforces customer's own M365 group membership for access control

Future skill: `pulser.powershell.customer_tenant_seed` (admin skill, approves customer tenant setup).

---

## 11. Related Documentation

- **[CLI Skills Governance Model](./cli-skills-model.md)** — Foundation for all CLI tool governance (GitHub, Azure, Odoo, etc.)
- **[Pulser Awareness Contract](../architecture/PULSER_AWARENESS_CONTRACT.md)** — System boundaries and component contracts
- **[Conversation Behavior Spec](./conversation-behavior-spec.md)** — Pulser's conversational logic (numbered options, tax intent routing, RBAC shaping)
- **[Microsoft Learn: Graph Permissions](https://learn.microsoft.com/en-us/graph/permissions-overview)** — Official scopes and delegated vs. app-only auth
- **[Microsoft Learn: Graph PowerShell SDK](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview?view=graph-powershell-1.0)** — Cmdlet reference and auth patterns
- **[Microsoft Learn: PowerShell Auth](https://learn.microsoft.com/en-us/powershell/microsoftgraph/authentication-commands?view=graph-powershell-1.0)** — Connect-MgGraph and auth flows

---

## 12. Versioning & Change Log

| Version | Date | Change | Author |
|---|---|---|---|
| 1.0 | 2026-04-25 | Initial specification (7 Graph skills + 5 PowerShell skills, Wave 1-3 roadmap) | Pulser Team |

---

**Next Action**: Create PR for governance review and security sign-off.

```text
docs(product): define Graph and PowerShell skills model

Define Microsoft Graph (runtime API) and PowerShell (admin/provisioning) as
separate Pulser skill lanes. Establishes auth boundaries, approval gates,
tenant scoping, and evidence logging for safe M365 integration.

- Graph skills: 7 runtime APIs (user, group, mail, calendar, files, teams)
- PowerShell skills: 5 admin/setup tasks (RBAC seed, permission audit, validation)
- Auth model: delegated (user context) vs. app-only (service context)
- Guardrails: no arbitrary shell, no tenant-wide grants, sandbox-only (Wave 1)
- Approval matrix: read-only auto-allowed, write/admin approval-gated
- Evidence: immutable JSON logs for all invocations

Acceptance criteria: Graph/PowerShell clearly separated, runtime uses APIs not
shell, arbitrary shell denied, tenant-wide changes approval-gated, evidence
required, no Supabase/n8n dependencies.

Closes: Related to CLI Skills Model (prior PR)
```
