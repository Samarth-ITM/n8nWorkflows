Vet new Form.io leads with ZeroBounce and sync qualified contacts to Pipedrive

https://n8nworkflows.xyz/workflows/vet-new-form-io-leads-with-zerobounce-and-sync-qualified-contacts-to-pipedrive-13008


# Vet new Form.io leads with ZeroBounce and sync qualified contacts to Pipedrive

## 1. Workflow Overview

**Workflow name:** Vet new Form.io leads with ZeroBounce and sync qualified contacts to Pipedrive

**Purpose:**  
Automatically process new **Form.io** form submissions, verify whether the submitted email is usable using **ZeroBounce** (validation + AI scoring for catch-all), then:
- **Accept** good leads: log them to a Google Sheet and **create a Person in Pipedrive**
- **Reject** bad/uncertain leads: log them to a Google Sheet with a clear rejection reason

**Primary use cases:**
- Prevent bad emails (invalid, suppressed, risky) from entering Pipedrive
- Save ZeroBounce credits by checking credits before making paid calls
- Keep a full audit trail of accepted/rejected leads in Google Sheets

### 1.1 Logical Blocks
1. **Input Reception & Basic Verification** (Form.io trigger → check email present)
2. **Credit Check (Validation)** (ensure ZeroBounce credits exist before validation)
3. **Stage 1: Email Validation** (ZeroBounce validation → status routing)
4. **Credit Check (Scoring) + Stage 2: AI Scoring** (only for catch-all emails)
5. **Outputs** (Google Sheets Accepted/Rejected + Pipedrive sync for accepted)

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Basic Verification
**Overview:** Receives new Form.io submissions and ensures an email was provided. If missing, it is immediately routed to “Rejected (not validated)” with a reason.  
**Nodes involved:** `Form.io Trigger`, `Has email?`, `Email missing`, `Add to Rejected not validated`

#### Node: Form.io Trigger
- **Type / role:** Form.io Trigger (`n8n-nodes-base.formIoTrigger`) — entry point webhook-like trigger for Form.io “create” events.
- **Configuration:**
  - Events: `create`
  - Form ID: `696513d375f70a96eef5dd2a`
  - Project ID: `6965134b4e1fa06a10411955`
- **Key data used downstream:** `item.json.name`, `item.json.email`
- **Connections:** → `Has email?`
- **Requirements:** Valid Form.io credentials.
- **Failure/edge cases:**
  - Credential/auth failure, webhook registration issues
  - Form schema changes (field renamed from `email` or `name`) will break expressions

#### Node: Has email?
- **Type / role:** IF (`n8n-nodes-base.if`) — checks if `$json.email` is not empty.
- **Configuration choices:**
  - Condition: `email` **notEmpty**
- **Connections:**
  - **True:** → `Check credits for validation`
  - **False:** → `Email missing`
- **Edge cases:**
  - Email field present but whitespace-only may pass/fail depending on Form.io payload formatting (here it checks “notEmpty” but not “trim”)

#### Node: Email missing
- **Type / role:** Set (`n8n-nodes-base.set`) — assigns a human-readable rejection reason and an override value used by the “not validated” rejection logger.
- **Configuration:**
  - Sets `reason = "Email missing"`
  - Sets `email_override = {{ $('Form.io Trigger').item.json.name }}`
    - Note: this is likely a **mistake** (it stores *name* as email fallback). It might be intended to store the submitted email or a placeholder string.
- **Connections:** → `Add to Rejected not validated`
- **Edge cases:**
  - If name is also missing, `email_override` becomes empty; Google Sheets match/update keyed on Email may behave unexpectedly.

#### Node: Add to Rejected not validated
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — logs rejections that did not reach ZeroBounce (email missing or insufficient credits for validation).
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Document: “ZeroBounce Form.io Pipedrive” (`1C-V2jrev3Ep7G532VHIrEbFlBF9rLeVgyhV-SXA4-GQ`)
  - Sheet: `Rejected` (gid=0)
  - Matching column: `Email`
  - Writes minimal columns: Name, Email (from `email_override`), Rejected At (`$now`), Reject Reason (`$json.reason`)
- **Connections:** terminal
- **Failure/edge cases:**
  - OAuth token expired / insufficient permissions
  - If `Email` is blank or duplicated, `appendOrUpdate` may overwrite unexpected rows

---

### Block 2 — Credit Check (Validation)
**Overview:** Prevents the validation call if ZeroBounce credits are insufficient. Uses a node error path to route to rejection logging instead of failing the workflow.  
**Nodes involved:** `Check credits for validation`, `Not enough credits for validation`

#### Node: Check credits for validation
- **Type / role:** ZeroBounce (`@zerobounce/n8n-nodes-zerobounce.zeroBounce`) — checks account credits.
- **Configuration choices:**
  - Resource: `account`
  - Credits required: `1`
  - **onError:** `continueErrorOutput` (critical design choice)
- **Connections:**
  - **Success output:** → `Validate email`
  - **Error output:** → `Not enough credits for validation`
- **Version-specific:** typeVersion 1 (ZeroBounce community node).
- **Failure/edge cases:**
  - Network/timeouts/API errors route to the error output (treated same as “not enough credits”)
  - If the account endpoint is down, you may reject valid leads unnecessarily (false negatives)

#### Node: Not enough credits for validation
- **Type / role:** Set — prepares rejection reason and supplies a usable email field for logging.
- **Configuration:**
  - `reason = "Not enough credits for validation"`
  - `email_override = {{ $('Form.io Trigger').item.json.email }}`
- **Connections:** → `Add to Rejected not validated`
- **Edge cases:**
  - If the trigger email is empty and you got here anyway (unexpected), logging may still be ambiguous

---

### Block 3 — Stage 1: Email Validation
**Overview:** Validates the email via ZeroBounce and routes based on status: valid → accept, catch-all → score, other → reject.  
**Nodes involved:** `Validate email`, `Status`, `Valid`, `Not valid`

#### Node: Validate email
- **Type / role:** ZeroBounce — performs email validation.
- **Configuration:**
  - Email: `{{ $('Form.io Trigger').item.json.email }}`
  - Timeout: 10 seconds
- **Connections:** → `Status`
- **Failure/edge cases:**
  - If this node errors (no explicit `onError`), the workflow execution fails unless n8n global error handling is set.
  - Timeout at 10s may be too low during latency spikes.

#### Node: Status
- **Type / role:** Switch (`n8n-nodes-base.switch`) — routes by `$json.status` from ZeroBounce validation response.
- **Routing rules:**
  - If `status == "valid"` → output **Valid** → `Valid` node
  - If `status == "catch-all"` → output **Catch-all** → `Check credits for scoring`
  - Fallback **Other** → `Not valid`
- **Connections:**
  - Output 1 (Valid) → `Valid`
  - Output 2 (Catch-all) → `Check credits for scoring`
  - Fallback (Other) → `Not valid`
- **Edge cases:**
  - ZeroBounce also returns statuses like `invalid`, `do_not_mail`, `spamtrap`, `abuse`, `unknown` etc. All are treated as “Not valid” here (rejected).

#### Node: Valid
- **Type / role:** Set — annotates acceptance reason for logging.
- **Configuration:** `reason = "Valid"`
- **Connections:** → `Add to Accepted`

#### Node: Not valid
- **Type / role:** Set — annotates rejection reason.
- **Configuration:** `reason = "Not valid"`
- **Connections:** → `Add to Rejected`
- **Edge cases:**
  - This collapses multiple failure modes into one bucket; you still log `ZB Status/Sub Status` into the Rejected sheet for later analysis.

---

### Block 4 — Credit Check (Scoring) + Stage 2: AI Email Scoring (Catch-all only)
**Overview:** For catch-all emails only, checks credits and then requests ZeroBounce AI scoring. Routes high scores to acceptance; medium/low to rejection.  
**Nodes involved:** `Check credits for scoring`, `Not enough credits for scoring`, `Score email`, `Filter by score`, `High score`, `Medium score`, `Low score`

#### Node: Check credits for scoring
- **Type / role:** ZeroBounce account check — prevents scoring without credits.
- **Configuration:**
  - Resource: `account`
  - Credits required: `1`
  - onError: `continueErrorOutput`
- **Connections:**
  - **Success output:** → `Score email`
  - **Error output:** → `Not enough credits for scoring`
- **Failure/edge cases:**
  - Same as validation credit check: API errors become “not enough credits” suppression.

#### Node: Not enough credits for scoring
- **Type / role:** Set — rejection reason for scoring step.
- **Configuration:** `reason = "Not enough credits for scoring"`
- **Connections:** → `Add to Rejected`
- **Edge cases:**
  - `Add to Rejected` expects data from `Validate email` for many fields. If scoring credits fail, `Validate email` data exists (because scoring only happens after validation catch-all), so this is safe.

#### Node: Score email
- **Type / role:** ZeroBounce scoring (`resource: scoring`) — returns `score` for the email.
- **Configuration:**
  - Email: `{{ $('Form.io Trigger').item.json.email }}`
  - Resource: `scoring`
- **Connections:** → `Filter by score`
- **Failure/edge cases:**
  - If scoring API fails, node has no `onError` path configured; execution may fail.

#### Node: Filter by score
- **Type / role:** Switch — buckets by numeric score.
- **Routing rules:**
  - `score >= 9` → `high`
  - `score >= 3` → `medium`  (this also captures 9+, but the first rule takes precedence)
  - `score < 3` → `low`
- **Connections:**
  - high → `High score`
  - medium → `Medium score`
  - low → `Low score`
- **Edge cases:**
  - If `score` missing/null/non-numeric, it will fall into fallback (configured fallbackOutput index 2). In this workflow fallback is set to `2` (which corresponds to one of the outputs depending on n8n internals). This is risky; better to set explicit fallback output name.

#### Node: High score / Medium score / Low score
- **Type / role:** Set nodes — assign `reason`:
  - High score: `reason = "High score"` → `Add to Accepted`
  - Medium score: `reason = "Medium score"` → `Add to Rejected`
  - Low score: `reason = "Low score"` → `Add to Rejected`

---

### Block 5 — Outputs (Google Sheets + Pipedrive)
**Overview:** Writes accepted and rejected outcomes to two Google Sheets tabs. For accepted leads, also creates a Person in Pipedrive.  
**Nodes involved:** `Add to Accepted`, `Add to Rejected`, `Create a person in Pipedrive`

#### Node: Add to Accepted
- **Type / role:** Google Sheets — audit log for accepted leads.
- **Configuration:**
  - Operation: `appendOrUpdate`
  - Document: “ZeroBounce Form.io Pipedrive”
  - Sheet: `Accepted` (gid `490801678`)
  - Matching column: `Email`
  - Writes:
    - Form: Name, Email
    - ZeroBounce validation fields: domain, status, account, mx_found, mx_record, free_email, sub_status, did_you_mean, smtp_provider, domain_age_days
    - Score: `{{ $node["Score email"] ? $('Score email').item.json.score : "" }}`
    - Accepted At: `{{$now}}`
    - Accepted Reason: `{{$json.reason}}`
- **Connections:** → `Create a person in Pipedrive`
- **Edge cases:**
  - If the lead is “Valid” (no scoring), score field becomes blank as intended.
  - If Google Sheets schema headers don’t match, append/update may fail or write empties.

#### Node: Add to Rejected
- **Type / role:** Google Sheets — audit log for rejected leads (includes validation fields, plus score if present).
- **Configuration:**
  - Operation: `appendOrUpdate`
  - Sheet: `Rejected` (gid=0)
  - Matching column: `Email`
  - Writes:
    - Name, Email, Rejected At (`$now`), Reject Reason (`$json.reason`)
    - Validation fields similar to Accepted
    - Score uses: `{{ $node["Score email"] ? $('Score email').item.json.score : "" }}`
- **Connections:** terminal
- **Edge cases:**
  - Uses `{{ $node["Validate email"] ? ... : "" }}` only for `did_you_mean` but assumes other Validate email fields exist; generally true except if you later route here before validation.

#### Node: Create a person in Pipedrive
- **Type / role:** Pipedrive (`n8n-nodes-base.pipedrive`) — creates a Person record for accepted leads.
- **Configuration:**
  - Resource: `person`
  - Name: `{{ $('Form.io Trigger').item.json.name }}`
  - Email: array with one element: `{{ $('Form.io Trigger').item.json.email }}`
- **Connections:** terminal
- **Failure/edge cases:**
  - Duplicate people: depending on Pipedrive settings, this may create duplicates rather than update an existing Person.
  - API key/auth errors; rate limiting.
  - If email is blank (should not happen on accepted path), Pipedrive may reject or create with empty email.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form.io Trigger | Form.io Trigger | Entry point: receives new form submissions | — | Has email? | ## ⚡ Trigger & Verification |
| Has email? | IF | Guards workflow: ensure email exists | Form.io Trigger | Check credits for validation; Email missing | ## ⚡ Trigger & Verification |
| Check credits for validation | ZeroBounce | Prevent validation call if no credits / API issue | Has email? | Validate email; Not enough credits for validation | ## ✔️ Stage 1: Email Validation |
| Validate email | ZeroBounce | Email validation (status/sub_status/mx/etc.) | Check credits for validation | Status | ## ✔️ Stage 1: Email Validation |
| Status | Switch | Route by validation `status` | Validate email | Valid; Check credits for scoring; Not valid | ## ✔️ Stage 1: Email Validation |
| Valid | Set | Mark accepted reason “Valid” | Status | Add to Accepted | ## 📤 Output Results |
| Check credits for scoring | ZeroBounce | Prevent scoring call if no credits / API issue | Status | Score email; Not enough credits for scoring | ## 🎯 Stage 2: AI Email Scoring |
| Score email | ZeroBounce | AI scoring for catch-all emails | Check credits for scoring | Filter by score | ## 🎯 Stage 2: AI Email Scoring |
| Filter by score | Switch | Bucket score into high/medium/low | Score email | High score; Medium score; Low score | ## 🎯 Stage 2: AI Email Scoring |
| High score | Set | Mark accepted reason “High score” | Filter by score | Add to Accepted | ## 📤 Output Results |
| Medium score | Set | Mark rejected reason “Medium score” | Filter by score | Add to Rejected | ## 📤 Output Results |
| Low score | Set | Mark rejected reason “Low score” | Filter by score | Add to Rejected | ## 📤 Output Results |
| Not valid | Set | Mark rejected reason “Not valid” | Status | Add to Rejected | ## 📤 Output Results |
| Not enough credits for scoring | Set | Mark rejected reason “Not enough credits for scoring” | Check credits for scoring | Add to Rejected | ## 📤 Output Results |
| Not enough credits for validation | Set | Mark rejected reason “Not enough credits for validation”, set `email_override` | Check credits for validation | Add to Rejected not validated | ## 📤 Output Results |
| Email missing | Set | Mark rejected reason “Email missing”, set `email_override` | Has email? | Add to Rejected not validated | ## 📤 Output Results |
| Add to Accepted | Google Sheets | Log accepted lead + ZB details | Valid / High score | Create a person in Pipedrive | ## 📤 Output Results |
| Create a person in Pipedrive | Pipedrive | Create Person for accepted leads | Add to Accepted | — | ## 📤 Output Results |
| Add to Rejected | Google Sheets | Log rejected lead + ZB details (+score if any) | Not valid / Medium score / Low score / Not enough credits for scoring | — | ## 📤 Output Results |
| Add to Rejected not validated | Google Sheets | Log rejection without ZB validation | Email missing / Not enough credits for validation | — | ## 📤 Output Results |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## Form.io to Pipedrive: Advanced ZeroBounce Lead Vetting… + link: https://www.zerobounce.net |
| Sticky Note3 | Sticky Note | Section header | — | — | ## ⚡ Trigger & Verification |
| Sticky Note4 | Sticky Note | Section header | — | — | ## ✔️ Stage 1: Email Validation |
| Sticky Note5 | Sticky Note | Section header | — | — | ## 🎯 Stage 2: AI Email Scoring |
| Sticky Note6 | Sticky Note | Section header | — | — | ## 📤 Output Results |
| Sticky Note | Sticky Note | Branding image | — | — | ![ZeroBounce Logo](https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zb-logo-purple.svg) |
| Sticky Note7 | Sticky Note | Setup requirements + headers + links | — | — | Link: https://www.zerobounce.net/members/API |
| Sticky Note8 | Sticky Note | Key benefits | — | — |  |
| Sticky Note9 | Sticky Note | Nodes used + links | — | — | Links: https://n8n.io/integrations/zerobounce etc. |
| Sticky Note10 | Sticky Note | How it works (step list) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1. Form.io credential: connect to your Form.io project with API key/token in n8n.
   2. ZeroBounce credential: create a ZeroBounce API key (link in Notes) and add it as **ZeroBounce account** credentials.
   3. Google Sheets OAuth2 credential: authenticate and ensure access to the target spreadsheet.
   4. Pipedrive credential: configure **Pipedrive API key** (or supported auth method) with permission to create People.

2. **Add trigger**
   1. Add node **Form.io Trigger**.
   2. Configure:
      - Events: `create`
      - Project ID: your Form.io project
      - Form ID: your Form
   3. Select Form.io credentials.

3. **Add email presence check**
   1. Add **IF** node named **Has email?**
   2. Condition: `{{$json.email}}` → **notEmpty**
   3. Connect: `Form.io Trigger` → `Has email?`

4. **If email missing path**
   1. Add **Set** node named **Email missing**
      - Set `reason` = `Email missing`
      - Set `email_override` = `{{ $('Form.io Trigger').item.json.name }}` (consider changing to a safer value like the submitted email or `"Missing Email"`).
   2. Add **Google Sheets** node **Add to Rejected not validated**
      - Operation: **Append or Update**
      - Spreadsheet: your file
      - Sheet: `Rejected`
      - Matching column: `Email`
      - Map:
        - Name = `{{ $('Form.io Trigger').item.json.name }}`
        - Email = `{{ $json.email_override }}`
        - Rejected At = `{{ $now }}`
        - Reject Reason = `{{ $json.reason }}`
   3. Connect: `Has email? (false)` → `Email missing` → `Add to Rejected not validated`

5. **Check ZeroBounce credits (validation)**
   1. Add ZeroBounce node **Check credits for validation**
      - Resource: **account**
      - Option: **creditsRequired = 1**
      - Error handling: set **On Error** to **Continue (error output)** (named in JSON as `continueErrorOutput`)
   2. Connect: `Has email? (true)` → `Check credits for validation`

6. **If insufficient credits for validation**
   1. Add **Set** node **Not enough credits for validation**
      - `reason` = `Not enough credits for validation`
      - `email_override` = `{{ $('Form.io Trigger').item.json.email }}`
   2. Connect: `Check credits for validation (error)` → `Not enough credits for validation` → `Add to Rejected not validated`

7. **Stage 1 validation**
   1. Add ZeroBounce node **Validate email**
      - Resource: default validation
      - Email: `{{ $('Form.io Trigger').item.json.email }}`
      - Timeout: 10 seconds
   2. Connect: `Check credits for validation (success)` → `Validate email`

8. **Route by validation status**
   1. Add **Switch** node **Status**
      - Rule 1: `{{$json.status}} equals "valid"` → output name “Valid”
      - Rule 2: `{{$json.status}} equals "catch-all"` → output name “Catch-all”
      - Fallback output name: “Other”
   2. Connect: `Validate email` → `Status`

9. **Valid path (accept)**
   1. Add **Set** node **Valid**: `reason="Valid"`
   2. Add **Google Sheets** node **Add to Accepted**
      - Operation: Append or Update
      - Sheet: `Accepted`
      - Matching: `Email`
      - Map Name/Email from trigger; map ZB fields from **Validate email**; set `Accepted At={{$now}}`; set `Accepted Reason={{$json.reason}}`
      - Score field can be expression: `{{ $node["Score email"] ? $('Score email').item.json.score : "" }}`
   3. Add **Pipedrive** node **Create a person in Pipedrive**
      - Resource: Person
      - Name: `{{ $('Form.io Trigger').item.json.name }}`
      - Email: `{{ $('Form.io Trigger').item.json.email }}`
   4. Connect: `Status (Valid)` → `Valid` → `Add to Accepted` → `Create a person in Pipedrive`

10. **Other (invalid/suppressed/etc.) path**
   1. Add **Set** node **Not valid**: `reason="Not valid"`
   2. Add **Google Sheets** node **Add to Rejected**
      - Operation: Append or Update
      - Sheet: `Rejected`
      - Matching: `Email`
      - Map Name/Email from trigger; rejection reason from Set node; ZB fields from Validate email; score optionally from Score email if it exists.
   3. Connect: `Status (Other)` → `Not valid` → `Add to Rejected`

11. **Catch-all path → check credits (scoring)**
   1. Add ZeroBounce node **Check credits for scoring**
      - Resource: account
      - creditsRequired: 1
      - On Error: Continue (error output)
   2. Connect: `Status (Catch-all)` → `Check credits for scoring`

12. **If insufficient credits for scoring**
   1. Add **Set** node **Not enough credits for scoring**: `reason="Not enough credits for scoring"`
   2. Connect: `Check credits for scoring (error)` → `Not enough credits for scoring` → `Add to Rejected`

13. **Stage 2 scoring**
   1. Add ZeroBounce node **Score email**
      - Resource: `scoring`
      - Email: `{{ $('Form.io Trigger').item.json.email }}`
   2. Add **Switch** node **Filter by score**
      - `{{$json.score}} >= 9` → high
      - `{{$json.score}} >= 3` → medium
      - `{{$json.score}} < 3` → low
   3. Add Set nodes:
      - **High score**: `reason="High score"` → `Add to Accepted`
      - **Medium score**: `reason="Medium score"` → `Add to Rejected`
      - **Low score**: `reason="Low score"` → `Add to Rejected`
   4. Connect: `Check credits for scoring (success)` → `Score email` → `Filter by score` → (High/Medium/Low sets) → respective Google Sheets node

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Form.io to Pipedrive: Advanced ZeroBounce Lead Vetting… Results are also output to Google Sheets for review.” | Workflow intent note |
| ZeroBounce website | https://www.zerobounce.net |
| Create a ZeroBounce API key | https://www.zerobounce.net/members/API |
| Nodes used: ZeroBounce, Form.io Trigger, Pipedrive, Google Sheets (or Data Table alternative) | https://n8n.io/integrations/zerobounce ; https://n8n.io/integrations/formio-trigger ; https://n8n.io/integrations/pipedrive ; https://n8n.io/integrations/google-sheets ; https://n8n.io/integrations/data-table |
| Google Sheets headers suggested (Accepted/Rejected) | Included in “Setup Requirements” sticky note |
| Branding image | https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zb-logo-purple.svg |
| Key benefits: only good leads reach CRM; credit safety; detailed suppression reasons | From sticky note “Key Benefits” |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.