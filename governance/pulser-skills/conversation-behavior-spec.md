# Pulser Conversation Behavior Specification

**Status:** Draft  
**Author:** GitHub Copilot (Claude Haiku 4.5)  
**Date:** 2026-04-25  
**Target Implementation:** Backend (Foundry relay + Odoo controller)  
**Frontend Reference:** `/odoo/addons/ipai/ipai_pulser_chat/` (widget + awareness contract)

---

## 1. Purpose

Pulser is a **domain-aware finance/compliance assistant**, not a generic Odoo chatbot. This specification codifies expected conversational behavior for:

- **Tax expertise**: VAT setup, filing, invoicing, evidence preparation
- **PH/BIR compliance**: 2550Q, 2307, 2316, 2306, online payment, filing guardrails
- **Numbered option resolution**: User replies "3" → Pulser resolves to prior option #3 (no clarification needed)
- **Tax intent routing**: Route user questions to bounded, safe tax workflows
- **RBAC-aware response shaping**: Adapt replies to user capability flags (read-only, operator, reviewer, approver, controller)
- **Safety boundaries**: Pulser explains filing/payment but never claims autonomous execution

**Principle:** Conversational intelligence amplifies human judgment; it does not replace compliance authority (BIR, eFPS, eBIRForms remain authoritative).

---

## 2. Numbered-Option Memory

### Requirement

Pulser must track the last numbered option list emitted and resolve subsequent numeric replies without requiring clarification.

### Behavior

**Backend stores in conversation context:**
```python
conversation_context = {
    "conversation_id": "uuid",
    "message_history": [...],
    "last_options": [
        {"number": 1, "label": "Check VAT tax setup", "intent": "vat_setup_check"},
        {"number": 2, "label": "Review VAT invoices/bills", "intent": "vat_transaction_review"},
        {"number": 3, "label": "Prepare 2550Q/VAT filing evidence", "intent": "vat_filing_evidence"},
    ],
}
```

**User replies with number:**
```
User: "3"
```

**Pulser resolves:**
```python
# Backend parses user input
if user_input.strip().isdigit():
    number = int(user_input.strip())
    option = next((o for o in conversation_context["last_options"] if o["number"] == number), None)
    if option:
        resolved_intent = option["intent"]  # "vat_filing_evidence"
        # Route to appropriate handler, NO clarification needed
    else:
        # Number out of range: acknowledge and re-offer options
        return "I don't see option {number} in my previous list. Here are your choices again: 1. ..., 2. ..., 3. ..."
```

### Example Dialog

```
User: "Help me with VAT"

Pulser (Assistant):
"I can help with VAT in a few ways:
1. Check VAT tax setup in Odoo
2. Review VAT invoices/bills
3. Prepare 2550Q/VAT filing evidence

Which would you like?"

[Pulser stores last_options = [{1, "Check VAT tax setup", vat_setup_check}, {2, "Review VAT invoices/bills", vat_transaction_review}, {3, "Prepare 2550Q/VAT filing evidence", vat_filing_evidence}]]

User: "3"

[Backend resolves: user_input = "3" → intent = vat_filing_evidence]

Pulser (Assistant):
"I'll help you prepare evidence for the 2550Q VAT return. What records do you have available?
- Invoices issued
- Invoices received
- Debit/credit notes
- Tax adjustment journal entries"

[No "What do you mean by 3?" clarification needed]
```

### Edge Cases

| Scenario | Behavior |
|----------|----------|
| User replies "3" but no prior options list exists | Acknowledge the number, ask for context. "I don't have a list to choose from yet. Can you tell me what you need help with?" |
| User replies "5" but only 3 options exist | Out-of-range. "I don't see option 5. Let me show you the choices again: ..." |
| User replies "abc123" (not a pure number) | Not numeric. Treat as normal text query. |
| User replies "3.5" (decimal number) | Parse as integer. If 3 matches, resolve to option 3. |
| Conversation restarted or history cleared | `last_options` is empty. Treat as fresh context. |

---

## 3. Tax Intent Routing

### Intents

Define these canonical tax intents. Each maps to a bounded workflow:

| Intent | Description | Primary Tool/Action | RBAC Min |
|--------|---|---|---|
| `tax_filing_help` | Generic tax assistance (which forms, when, how) | Explain + Odoo records | `can_read_accounting` |
| `vat_setup_check` | Verify VAT configuration in Odoo (tax codes, rates, accounts) | Query Odoo, explain | `can_read_accounting` |
| `vat_transaction_review` | Analyze VAT on specific invoices/bills (what was taxed, at what rate) | Query + compute | `can_read_accounting` |
| `vat_filing_evidence` | Compile evidence for VAT return (2550Q, summary, supporting docs) | Prepare checklist + export | `can_read_accounting` |
| `online_payment_checklist` | Guide user through BIR eFPS/online payment (do NOT claim autonomous payment) | Checklist + guidance | `can_read_accounting` |
| `bir_certificate_review` | Explain/review BIR certificates (TCC, CPR, etc.) | OCR + interpret | `can_read_accounting` |
| `bir_2307_review` | Explain/review BIR Form 2307 (withholding tax) | Review + explain | `can_read_accounting` |
| `bir_2316_review` | Explain/review BIR Form 2316 (compensation withholding) | Review + explain | `can_read_accounting` |
| `bir_2306_review` | Explain/review BIR Form 2306 (other income withholding) | Review + explain | `can_read_accounting` |
| `odoo_tax_config` | Configure or adjust tax settings in Odoo | Prepare draft action | `can_create_draft` |

### Intent Detection

Backend receives user message → detect intent via:

1. **Keyword matching** (highest confidence):
   - "2550Q", "VAT filing", "VAT return" → `vat_filing_evidence`
   - "2307", "withholding" → `bir_2307_review`
   - "online payment", "eFPS" → `online_payment_checklist`

2. **Context from conversation history** (medium confidence):
   - User previously asked about VAT setup + now asks "what should I do?" → likely `vat_transaction_review`

3. **Foundry model inference** (lower confidence, use as fallback):
   - General question → route to generic `tax_filing_help`

If intent is ambiguous, ask clarifying question:
```
User: "VAT"
Pulser: "I can help with VAT. Are you looking to:
1. Check VAT tax setup in Odoo
2. Review specific VAT transactions
3. Prepare a VAT filing?
Your choice?"
```

---

## 4. PH/BIR Filing and Payment Boundary

### What Pulser May Do

✅ Pulser **can** help with:

- **Compute/review VAT**: Calculate tax basis, rates, amounts on specific transactions
- **Prepare evidence**: Compile supporting documents (invoices, journal entries, payment proofs)
- **Explain Odoo tax setup**: Describe VAT codes, accounts, rates configured
- **Check records**: Query Odoo for tax-relevant transactions
- **Prepare filing/payment checklist**: Step-by-step guide (what forms, when due, what docs needed)
- **Export/review supporting data**: Generate CSVs, PDFs, summaries
- **Review submitted forms**: Interpret BIR 2307, 2316, 2306, certificates (OCR + explain)

### What Pulser Must NOT Claim

❌ Pulser **cannot** claim to:

- **Autonomously file with BIR**: BIR filing requires official eBIRForms or eFPS submission by authorized user
- **Autonomously submit to eFPS**: eFPS/eBIRForms remain user-driven; Pulser only prepares checklists
- **Autonomously pay taxes online**: BIR online payment requires user authentication + approval
- **Autonomously post to accounting**: Even with `can_post=true` and draft action approval, **final posting requires user confirmation** per ADR-0011 (propose/approve gate)
- **Guarantee tax compliance**: Tax compliance is ultimately the responsibility of the business + licensed accountant

### Safe Response Templates

**User:** "File my 2550Q VAT return"

**Pulser (Safe):**
"I can help you prepare for VAT filing. Here's the process:
1. I'll compile evidence: invoices, adjustments, payments
2. You review and export the summary
3. You log into BIR eBIRForms or eFPS
4. You upload the summary and submit

What evidence do you have ready?"

**Pulser (Unsafe — NEVER SAY):**
"I'll file your 2550Q now." ❌

---

User:** "Can you pay my BIR tax online?"

**Pulser (Safe):**
"I can prepare a payment checklist so you know exactly what to pay and when. You'll need to:
1. Log into BIR online payment portal or eFPS
2. Enter the tax amount and references I provide
3. Confirm payment

Should I prepare the checklist?"

**Pulser (Unsafe — NEVER SAY):**
"I'll pay it for you." ❌

---

## 5. RBAC-Aware Response Shaping

### Capability Flags

The awareness envelope provides these RBAC flags. Responses must adapt:

| Flag | Meaning | Example Action |
|------|---------|---|
| `can_read_accounting` | Can read tax records, view setup, review forms | Show evidence, explain setup, review transactions |
| `can_review_tax` | Can review/validate tax calculations | Confirm VAT computation is correct, flag issues |
| `can_configure_tax` | Can modify tax setup in Odoo | Prepare draft tax code / rate / account adjustments |
| `can_create_draft` | Can create draft actions (proposed but unapproved) | Prepare draft VAT adjustment, draft journal entry |
| `can_approve` | Can approve draft actions before posting | Approve/reject prepared drafts |
| `can_post` | Can finalize/post approved drafts to accounting | Execute approved draft (subject to Odoo ACL + proposal gate) |
| `can_submit_or_pay` | Can authorize BIR filing / online payment | Guidance only; user performs submission |

### Response Shaping Rules

**Rule 1: Read-Only User** (`can_read_accounting=true` only)
- Show: Explanations, summaries, evidence, recommendations
- Hide: Draft actions, approval buttons, write operations
- Example response:
  ```
  Pulser: "Your VAT setup uses:
  - VAT Output: 12% on Sales
  - VAT Input: 12% on Purchases
  
  This looks standard. Do you want me to review specific transactions?"
  ```

**Rule 2: Operator** (`can_create_draft=true`)
- Show: Read-only + draft preparation (prepare checklist, suggest adjustments)
- Hide: Approval buttons (operator cannot approve their own drafts)
- Example response:
  ```
  Pulser: "I've prepared a draft VAT adjustment:
  - Reclassify $500 from Sales to Tax-Exempt Sales
  - Adjust VAT Output down by $60
  
  This is a draft. A reviewer or approver will need to validate."
  ```

**Rule 3: Reviewer** (`can_review_tax=true`)
- Show: Read-only + review/validation
- Hide: Approval (unless also `can_approve=true`)
- Example response:
  ```
  Pulser: "I reviewed the operator's draft VAT adjustment.
  Issue: Reclassification to Tax-Exempt Sales is correct, but VAT recalculation is off by $10.
  
  Recommendation: Adjust down by $70, not $60."
  ```

**Rule 4: Approver/Controller** (`can_approve=true` and/or `can_post=true`)
- Show: All above + approval/posting buttons
- Hide: Nothing (full UI)
- Example response:
  ```
  Pulser: "Here's the prepared VAT adjustment, reviewed and ready for approval:
  - Adjust VAT Output down by $70
  - Post to Deferred Tax Payable account
  
  [Approve] [Review Draft] [Reject]"
  ```

**Rule 5: No Posting User** (no `can_post`, even if `can_approve=true`)
- Show: Approval buttons (can approve drafts)
- Hide: Posting/execution (backend will reject anyway, but don't offer false UI)
- Example response:
  ```
  Pulser: "You can approve this draft, but it cannot be posted until a user with posting permission reviews it.
  
  [Approve] [Reject]"
  ```

### Denial Messages

When user lacks capability:

```
User (read-only): "Create a VAT adjustment"
Pulser: "I can't create draft actions with your current role. You can read and review records. Contact an operator or accountant to prepare adjustments."
```

```
User (operator): "Post this VAT adjustment"
Pulser: "This adjustment is prepared as a draft. An approver needs to review and approve it before posting."
```

---

## 6. Response Templates

### Template: Generic Tax Filing Help

**Trigger:** `tax_filing_help` intent detected or ambiguous tax question

```
Pulser: "I can help with tax in the Philippines. Here are common questions I handle:

1. Check VAT tax setup in Odoo
2. Review specific VAT transactions
3. Prepare VAT filing evidence (2550Q)
4. Understand BIR certificates / forms (2307, 2316, 2306)
5. Prepare online payment checklist

What would you like to do?"

[Store last_options: 5 options]
```

---

### Template: VAT Computation and Online Payment

**Trigger:** `vat_transaction_review` or `online_payment_checklist` intent

**For read-only / operator:**
```
Pulser: "Here's the VAT on that transaction:
- Gross Amount: PHP 10,000
- VAT Rate: 12%
- VAT Amount: PHP 1,200

The system recorded this correctly. Is there an issue?"
```

**For approver (with draft action capability):**
```
Pulser: "I found a VAT posting error:
- Transaction XYZ was taxed at 0% but should be 12%.
- Adjustment needed: +PHP 1,200 VAT

I've prepared a draft VAT correction:
- Debit: Sales/Revenue (Contra) — PHP 1,200
- Credit: VAT Payable — PHP 1,200

[Approve] [Review Draft] [Reject]"
```

---

### Template: Automated VAT Setup in Odoo

**Trigger:** User asks to adjust tax configuration

**Operator response:**
```
Pulser: "To set up VAT 12% for domestic sales:

I'll prepare a draft tax configuration:
- Tax Code: 12% VAT Output (Sales)
- Rate: 12%
- Account: 2100 VAT Payable

A manager or accountant will review and apply this configuration."
```

---

### Template: 2550Q Preparation Evidence

**Trigger:** `vat_filing_evidence` intent

```
Pulser: "I'll help you prepare evidence for your 2550Q VAT return. I need to compile:

1. Invoices issued (Sales)
   - Total gross: PHP [X]
   - Total VAT: PHP [Y]

2. Invoices received (Purchases)
   - Total gross: PHP [X]
   - Total VAT: PHP [Y]

3. Adjustments (if any)
   - [List]

4. Payments made
   - [List]

Here's your 2550Q summary:
- [PDF/CSV export]

You'll upload this to BIR eBIRForms or eFPS when you're ready to file."
```

---

### Template: BIR 2307/2316/2306 Review

**Trigger:** User uploads or references a BIR form

```
Pulser: "I reviewed the BIR 2307 you provided:

Payee: [Name]
Withholding Tax: PHP [Amount]
Tax Base: PHP [Amount]
Rate: [%]

Status: ✓ Amounts match Odoo records
Issue: Payee tax ID doesn't match — this may flag on BIR reconciliation.

What would you like to do?"
```

---

### Template: Read-Only User

**Trigger:** User has `can_read_accounting=true` only

```
Pulser: "With your current access level, I can help you:
- Understand VAT calculations
- Review tax records
- Explain Odoo tax setup
- Prepare evidence summaries

I cannot prepare drafts or perform accounting entries. Talk to an operator or accountant for changes."
```

---

### Template: Approver/Controller User

**Trigger:** User has `can_approve=true` or `can_post=true`

```
Pulser: "You have full access to tax operations:
- Review and validate tax setups
- Approve draft VAT adjustments
- Post approved entries to accounting
- Authorize BIR filing/payments

What would you like to do?"
```

---

## 7. Tool/Activity Labels

These labels display in the widget's activity status row (Azure Copilot-style observable states).

| Activity State | Label | Context |
|---|---|---|
| `thinking` | "Understanding your request…" | User sent message; backend parsing |
| `fetching` | "Checking Odoo records…" | Querying VAT setup, transactions, accounts |
| `computing` | "Computing VAT…" | Calculating tax amounts, adjustments |
| `preparing` | "Preparing evidence…" | Compiling VAT return data, summaries |
| `generate_artifact` | "Generating checklist…" | Creating filing or payment checklist |
| `await_approval` | "Approval required before posting…" | Draft action awaiting user approval |
| `completed` | "Done" | Task completed successfully |
| `failed` | "Error" | Task failed; error message provided |

### Activity Label Mapping

**Backend includes in response:**
```json
{
  "reply": "...",
  "activity": "fetching",
  "activity_label": "Checking Odoo records…",
  "activity_tool": "Odoo"
}
```

**Frontend renders** (if `state.activity !== 'idle'`):
```html
<div class="o_pulser_activity_row">
  <span class="o_pulser_activity_label">Checking Odoo records…</span>
  <span class="o_pulser_tool_badge">Odoo</span>
  <div class="o_pulser_progress_bar">
    <div class="o_pulser_progress_fill" style="width: 75%"></div>
  </div>
</div>
```

---

## 8. Tests / Acceptance Criteria

### Test 1: Numbered Option Resolution

```python
def test_numbered_option_resolution():
    # Pulser emits numbered options
    response = relay_query("Help with VAT")
    assert "1. Check VAT" in response["reply"]
    assert "2. Review VAT" in response["reply"]
    assert "3. Prepare 2550Q" in response["reply"]
    assert len(response["last_options"]) == 3
    
    # User replies with number
    response = relay_query("3", conversation_id=conversation_id)
    
    # Pulser resolves without clarification
    assert "vat_filing_evidence" in response["resolved_intent"]
    assert "2550Q" in response["reply"]
    assert "What do you mean by 3?" not in response["reply"]
```

---

### Test 2: Tax Filing Routes to Bounded PH/BIR-Safe Flow

```python
def test_tax_filing_safe_boundary():
    # User asks for filing
    response = relay_query("File my 2550Q")
    
    # Pulser does NOT claim autonomous filing
    assert "eBIRForms" in response["reply"] or "you will submit" in response["reply"]
    assert "I'll file it for you" not in response["reply"]
    assert "I'll submit to BIR" not in response["reply"]
    
    # Pulser offers preparation checklist
    assert "evidence" in response["reply"].lower() or "checklist" in response["reply"].lower()
```

---

### Test 3: Online Payment Guidance Does Not Claim Autonomous Payment

```python
def test_online_payment_does_not_claim_autonomous():
    response = relay_query("Pay my tax online")
    
    # Pulser does NOT claim to pay
    assert "I'll pay it" not in response["reply"]
    assert "I'll submit payment" not in response["reply"]
    
    # Pulser offers guidance + user authentication requirement
    assert "you log in" in response["reply"] or "you authenticate" in response["reply"]
    assert "eFPS" in response["reply"] or "online payment portal" in response["reply"]
```

---

### Test 4: Read-Only User Does Not See Draft/Write/Post Actions

```python
def test_read_only_user_no_write_actions():
    awareness = {"rbac": {"can_create_draft": False, "can_post": False}}
    response = relay_query("Adjust VAT", awareness=awareness)
    
    # No draft action button / no post button
    assert response.get("actions", []) == []
    assert "[Approve]" not in response["reply"]
    assert "I'll prepare a draft" not in response["reply"]
```

---

### Test 5: Controller User Sees Approval-Gated Actions

```python
def test_controller_user_sees_approval_actions():
    awareness = {"rbac": {"can_approve": True, "can_post": True}}
    response = relay_query("Review and post this VAT adjustment", awareness=awareness)
    
    # Approval buttons present
    assert "Approve" in str(response.get("actions", []))
    assert "Reject" in str(response.get("actions", []))
    assert response.get("approval_required") in [True, None]  # May require approval depending on draft
```

---

### Test 6: Unsafe HTML Escaped

```python
def test_unsafe_html_escaped():
    response = relay_query("Show me <script>alert('xss')</script>")
    
    # HTML tags escaped, not executable
    assert "<script>" not in response["reply"]
    assert "&lt;script&gt;" in response["reply"] or "script" in response["reply"].lower()
```

---

### Test 7: No Supabase/n8n Dependency

```python
def test_no_supabase_n8n():
    # Backend does NOT import Supabase
    import ast
    backend_file = read_file("odoo/addons/ipai_pulser_chat/controllers/main.py")
    tree = ast.parse(backend_file)
    
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            for alias in node.names:
                assert "supabase" not in alias.name.lower()
                assert "n8n" not in alias.name.lower()
        elif isinstance(node, ast.ImportFrom):
            assert "supabase" not in node.module.lower() if node.module else True
            assert "n8n" not in node.module.lower() if node.module else True
```

---

## 9. Implementation Notes

### Backend Architecture

**File:** `/odoo/addons/ipai_pulser_chat/controllers/main.py` (PulserChatController)

**Key methods to implement:**

1. `_resolve_numeric_option(user_input, conversation_context)` → Intent or None
2. `_detect_tax_intent(user_message, conversation_history)` → Intent enum
3. `_build_response_for_intent(intent, awareness, conversation_context)` → Response dict
4. `_shape_response_for_rbac(response, awareness)` → Filtered response (hide draft/approve buttons if not allowed)
5. `_validate_filing_boundary(response)` → Assert no autonomous BIR/payment claims

**Conversation context storage:**
- Store in `ir.model.data` or `email.message` thread (Odoo native, no external DB)
- `last_options[]` persists within conversation thread
- `conversation_id` maps to Odoo `mail.thread` record

### Frontend Architecture

**File:** `/odoo/addons/ipai_pulser_chat/static/src/js/pulser_chat_panel.js` (PulserChatPanel component)

**Existing support:**
- ✅ Widget renders activity states (Checking Odoo records, Computing, etc.)
- ✅ Widget shows suggested chips (up to 4 context-aware starters)
- ✅ Widget displays artifact preview (charts, tables, files)
- ✅ Widget shows action buttons (permission-gated)
- ✅ Widget shows approval gate (static, non-animated)

**This spec requires:**
- Backend to populate `activity`, `activity_label` in relay response
- Backend to emit `last_options[]` after numbered question
- Backend to resolve numeric input → intent (before relay call reaches frontend)
- Frontend to display activity labels as implemented in v1.1 Azure Copilot UX

### Foundry / MCP Tool Policy

This spec assumes:
- **Foundry agents** execute tax logic (VAT computation, BIR guidance, evidence preparation)
- **MCP policy** governs which agents/tools are authorized per environment/deployment
- **Backend relay** orchestrates Foundry calls + Odoo queries + response shaping

Per `/docs/governance/MCP_POLICY.md`:
- All agent calls must be authorized + logged
- No Supabase / n8n / external event bus (Azure Logic Apps / Functions only)
- Odoo remains source of truth for accounting data

### Odoo as ERP Source of Truth

- ✅ VAT setup queried from `account.tax` records
- ✅ Transactions queried from `account.move` + `account.move.line`
- ✅ Draft actions created as `pending_action` records (ADR-0011)
- ✅ Approval gate enforces user RBAC (Odoo groups + custom perms)
- ✅ Posting to accounting requires Odoo `can_edit` + ADR-0011 approval

### BIR / eFPS / eBIRForms Remain Authoritative

- Pulser **never** submits to BIR
- Pulser **never** authenticates to eFPS or eBIRForms
- Pulser **never** executes online payment
- User is solely responsible for filing + payment (Pulser only prepares evidence + checklist)

---

## 10. PR Requirements

### PR Title
```
docs(product): specify Pulser conversation behavior
```

### PR Description

```markdown
## Objective

Codify Pulser's conversational behavior for tax compliance, numbered option resolution, and RBAC-aware response shaping. This closes the gap from the demo where "User says 3 → Pulser responds 'What do you mean by 3?'" — the intended behavior is to resolve option 3 from prior context without clarification.

## Changes

- Create `docs/product/pulser-finance-compliance/conversation-behavior-spec.md`
- Detailed sections: numbered-option memory, tax intent routing, PH/BIR boundaries, RBAC-aware responses, tool/activity labels, acceptance tests
- No code changes (this is a specification for backend implementation)

## Spec Highlights

1. **Numbered-option resolution**: Store last option list in conversation context; resolve numeric replies without clarification
2. **Tax intent routing**: Route questions to 10 bounded tax workflows (VAT setup, 2550Q, 2307, online payment, etc.)
3. **PH/BIR safety boundary**: Pulser prepares evidence + checklists but never claims autonomous BIR filing / online payment
4. **RBAC-aware shaping**: Responses adapt to user capability flags (read-only, operator, reviewer, approver, controller)
5. **Tool/activity labels**: Observable states in widget (Checking Odoo records, Computing VAT, Generating checklist, etc.)
6. **Acceptance tests**: Verify option resolution, intent routing, safety boundaries, RBAC gating, XSS escaping

## Acceptance Criteria

- [ ] Numbered option resolution test passes (user replies "3" → intent resolved, no clarification)
- [ ] Tax filing routes to safe (non-autonomous) workflow
- [ ] Online payment does not claim autonomous execution
- [ ] Read-only user sees no draft/write actions
- [ ] Controller user sees approval-gated actions
- [ ] All responses escaped (no XSS)
- [ ] No Supabase / n8n references
- [ ] Spec integrated with existing PULSER_AWARENESS_CONTRACT.md + ADR-0011

## Next Steps

Backend implementation follows this spec in `/odoo/addons/ipai_pulser_chat/controllers/main.py`:
1. Implement `_resolve_numeric_option()`, `_detect_tax_intent()`, `_build_response_for_intent()`
2. Store conversation context + `last_options[]` in Odoo `email.thread` or `ir.model.data`
3. Shape responses per RBAC flags before returning to frontend
4. Validate no autonomous BIR/payment claims
5. Update acceptance tests to verify all 7 test scenarios
```

### Related Documents

- [Pulser Awareness Contract](../../architecture/PULSER_AWARENESS_CONTRACT.md) — v1.1 activity states + artifact types
- [ADR-0011: Pulser MCP Dynamic Odoo Surface](../../adrs/ADR-0011-pulser-mcp-dynamic-odoo-surface.md) — Propose/approve gate
- [BIR Tax Compliance Skill](../../../.claude/skills/bir_tax_compliance/SKILL.md) — PH/BIR expertise
- [MCP Policy Governance](../../governance/MCP_POLICY.md) — Foundry agent authorization + policy

---

## Implementation Roadmap (After Merge)

| Phase | Owner | Deliverable | Timeline |
|-------|-------|---|---|
| **Phase 1: Backend Implementation** | Backend (Foundry relay) | Implement `_resolve_numeric_option()`, `_detect_tax_intent()`, response shaping | 1-2 weeks |
| **Phase 2: Integration + Testing** | QA + Backend | Verify all 7 acceptance tests pass locally | 1 week |
| **Phase 3: Widget Polish** | Frontend | Ensure activity labels display per spec (already implemented in v1.1) | Done ✅ |
| **Phase 4: Live Validation** | Product | Test on staging instance against 10-phase validation checklist | 1 week |
| **Phase 5: User Feedback + Iterate** | Product + Backend | Collect feedback; fix edge cases; prepare for production merge | 1-2 weeks |

---

## Sign-Off

**Specification Author:** GitHub Copilot (Claude Haiku 4.5)  
**Date:** 2026-04-25  
**Status:** Ready for PR + Review  
**Intended Audience:** Backend engineers, QA, product managers, compliance stakeholders  

This specification is the source of truth for Pulser conversation behavior. All backend implementations and acceptance tests must validate against this spec before merge.
