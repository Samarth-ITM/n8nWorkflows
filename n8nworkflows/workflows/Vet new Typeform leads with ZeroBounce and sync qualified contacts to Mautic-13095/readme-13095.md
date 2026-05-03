Vet new Typeform leads with ZeroBounce and sync qualified contacts to Mautic

https://n8nworkflows.xyz/workflows/vet-new-typeform-leads-with-zerobounce-and-sync-qualified-contacts-to-mautic-13095


# Vet new Typeform leads with ZeroBounce and sync qualified contacts to Mautic

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow captures new **Typeform** form submissions, verifies that an email is present, then uses **ZeroBounce** to (1) validate the email and (2) score “catch-all” emails. Only high-quality leads are synced to **Mautic**. All outcomes (accepted/rejected) are logged to **Google Sheets** for auditing and review.

**Target use cases:**
- Reduce CRM pollution by filtering invalid/low-quality emails before creating contacts.
- Handle “catch-all” domains with a scoring strategy instead of a hard reject.
- Avoid API failures by checking ZeroBounce credits before making billable requests.
- Maintain a reviewable trail of accepted/rejected leads.

### Logical blocks
**1.1 Trigger & Email Presence Gate**  
Typeform submission → check that an email was provided.

**1.2 Credit Check (Validation) + ZeroBounce Validation**  
Ensure credits → validate email → route based on status.

**1.3 Credit Check (Scoring) + ZeroBounce Scoring for Catch-all**  
If catch-all → ensure credits → score → route by score bands.

**1.4 Outputs (Google Sheets + Mautic)**  
Accepted leads → log to “Accepted” sheet → create/update Mautic contact.  
Rejected leads → log to “Rejected” sheet (with reason).

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Email Presence Gate

**Overview:**  
Receives a new Typeform submission and blocks processing if the email answer is empty, logging it as rejected for review.

**Nodes involved:**
- Typeform Trigger
- Has email?
- Email missing
- Add to Rejected not validated

#### Node: **Typeform Trigger**
- **Type:** Typeform Trigger (`n8n-nodes-base.typeformTrigger`)
- **Role:** Entry point; fires on new form submissions.
- **Configuration (interpreted):**
  - Watches **Form ID**: `Myr169jg`
  - Uses Typeform API credentials (access token/OAuth).
- **Key data used downstream:**
  - `item.json["What is your email address?"]`
  - `item.json["What is your first name?"]`
  - `item.json["What is your last name?"]`
- **Outputs:** To **Has email?**
- **Edge cases / failures:**
  - Typeform credential revoked/expired.
  - Form question labels changed → expressions break (fields not found).
  - Multiple email fields or different wording → must update expressions.

#### Node: **Has email?**
- **Type:** IF (`n8n-nodes-base.if`)
- **Role:** Checks that the email answer is not empty.
- **Configuration:**
  - Condition: `{{$json['What is your email address?']}}` **notEmpty**
- **Routing:**
  - **True** → Check credits for validation
  - **False** → Email missing
- **Edge cases:**
  - Email field exists but contains whitespace only (Typeform usually prevents this, but not guaranteed).
  - If Typeform question name changes, condition fails unexpectedly.

#### Node: **Email missing**
- **Type:** Set (`n8n-nodes-base.set`)
- **Role:** Creates a rejection payload when email is missing.
- **Configuration:**
  - Sets `reason = "Email missing"`
  - Sets `email_override` to:  
    `{{$('Typeform Trigger').item.json['What is your first name?']}} {{$('Typeform Trigger').item.json['What is your last name?']}}`
    - Note: this is used as a placeholder “Email” for logging; it’s not an actual email.
- **Outputs:** To **Add to Rejected not validated**
- **Edge cases:**
  - Produces a non-email value in `email_override`; Google Sheets row matching uses “Email” column, so it may collide if multiple people share same name.

#### Node: **Add to Rejected not validated**
- **Type:** Google Sheets (`n8n-nodes-base.googleSheets`)
- **Role:** Logs rejections where full ZeroBounce details are unavailable (no validation/scoring ran).
- **Configuration:**
  - Operation: **appendOrUpdate**
  - Match column: **Email**
  - Target document: `ZeroBounce Jotform ActiveCampaign` (same spreadsheet ID used elsewhere)
  - Target sheet: **Rejected**
  - Writes: Email (from `email_override`), First/Last name, Rejected At (`$now`), Reject Reason (`$json.reason`)
- **Inputs:** From **Email missing** or **Not enough credits for validation**
- **Edge cases / failures:**
  - Google OAuth expired / missing permissions.
  - Sheet headers mismatch (appendOrUpdate mapping may fail or write incorrect columns).
  - If “Email” is not unique (e.g., placeholder), updates may overwrite prior rows.

---

### 2.2 Credit Check (Validation) + ZeroBounce Validation

**Overview:**  
Checks ZeroBounce credits before validating the email. If credit check fails, rejects without validation. If validation succeeds, routes by validation status (valid / catch-all / other invalid).

**Nodes involved:**
- Check credits for validation
- Validate email
- Status
- Valid
- Not valid
- Not enough credits for validation

#### Node: **Check credits for validation**
- **Type:** ZeroBounce (`@zerobounce/n8n-nodes-zerobounce.zeroBounce`)
- **Role:** Guardrail to ensure at least 1 credit exists before validating.
- **Configuration:**
  - Resource: **account**
  - Option: **creditsRequired = 1**
  - **onError:** `continueErrorOutput` (important)
    - This makes the node emit an error output instead of stopping the workflow.
- **Routing:**
  - **Success output (main index 0)** → Validate email
  - **Error output (main index 1)** → Not enough credits for validation
- **Edge cases / failures:**
  - ZeroBounce API key invalid.
  - Network/timeouts.
  - API rate limits.
  - If `continueErrorOutput` is removed, the workflow will fail hard instead of gracefully rejecting.

#### Node: **Validate email**
- **Type:** ZeroBounce (`@zerobounce/n8n-nodes-zerobounce.zeroBounce`)
- **Role:** Performs ZeroBounce email validation.
- **Configuration:**
  - Email: `{{$('Typeform Trigger').item.json['What is your email address?']}}`
  - Timeout: 10 seconds
- **Outputs:** To **Status**
- **Edge cases / failures:**
  - Email expression missing due to Typeform label changes.
  - Timeout too low for occasional slow responses.
  - Validation returns statuses like `do_not_mail`, `invalid`, `spamtrap`, etc. (these fall into “Other” output later).

#### Node: **Status**
- **Type:** Switch (`n8n-nodes-base.switch`)
- **Role:** Routes based on `$json.status` from ZeroBounce validation result.
- **Configuration:**
  - Rule 1 → Output **Valid** if `$json.status == "valid"`
  - Rule 2 → Output **Catch-all** if `$json.status == "catch-all"`
  - Fallback output renamed to **Other**
- **Routing:**
  - Valid → Valid (Set node)
  - Catch-all → Check credits for scoring
  - Other → Not valid
- **Edge cases:**
  - Status values are case-sensitive here; if API returns unexpected casing, routing fails into fallback.

#### Node: **Valid**
- **Type:** Set
- **Role:** Annotates accepted reason for “valid” emails.
- **Configuration:** `reason = "Valid"`
- **Outputs:** To **Add to Accepted**

#### Node: **Not valid**
- **Type:** Set
- **Role:** Annotates rejection reason for anything not routed to valid/catch-all.
- **Configuration:** `reason = "Not valid"`
- **Outputs:** To **Add to Rejected**
- **Edge cases:**
  - “Other” includes a wide range of statuses; if you want different handling (e.g., `do_not_mail` vs `invalid`), add more switch rules.

#### Node: **Not enough credits for validation**
- **Type:** Set
- **Role:** Annotates rejection reason when validation credit check fails.
- **Configuration:**
  - `reason = "Not enough credits for validation"`
  - `email_override = {{$('Form.io Trigger').item.json.email}}`
    - **Potential configuration bug:** there is **no “Form.io Trigger” node** in this workflow. This expression will fail at runtime unless such a node exists or is renamed. It should likely reference **Typeform Trigger** instead.
- **Outputs:** To **Add to Rejected not validated**
- **Edge cases / failures:**
  - Expression evaluation error due to missing node reference.

---

### 2.3 Credit Check (Scoring) + ZeroBounce Scoring for Catch-all

**Overview:**  
Only for catch-all emails: checks credits, then requests a ZeroBounce “scoring” result and routes to accepted/rejected based on numeric score thresholds.

**Nodes involved:**
- Check credits for scoring
- Score email
- Filter by score
- High score
- Medium score
- Low score
- Not enough credits for scoring

#### Node: **Check credits for scoring**
- **Type:** ZeroBounce
- **Role:** Prevents scoring calls when credits are insufficient.
- **Configuration:**
  - Resource: **account**
  - creditsRequired: 1
  - onError: `continueErrorOutput`
- **Routing:**
  - Success → Score email
  - Error → Not enough credits for scoring

#### Node: **Score email**
- **Type:** ZeroBounce
- **Role:** Requests email quality scoring for catch-all emails.
- **Configuration:**
  - Resource: **scoring**
  - Email: `{{$('Typeform Trigger').item.json['What is your email address?']}}`
- **Outputs:** To **Filter by score**
- **Edge cases:**
  - Scoring response might not include `score` or it may be a string; downstream uses “loose” validation to reduce failures.

#### Node: **Filter by score**
- **Type:** Switch
- **Role:** Categorizes leads into high/medium/low based on `$json.score`.
- **Configuration (thresholds):**
  - **high** if score >= 9
  - **medium** if score >= 3
  - **low** if score < 3
  - Uses **loose type validation**, which helps if `score` comes as a string.
- **Routing:**
  - high → High score
  - medium → Medium score
  - low → Low score
- **Edge cases:**
  - Overlap handling: medium rule is “>= 3” and comes after high; so 9+ matches high first, which is correct.
  - If score is null/undefined, it will hit fallback (configured as output index 2), which may not be connected (risk of silent drop). Consider connecting fallback to rejection.

#### Node: **High score**
- **Type:** Set
- **Role:** Annotates acceptance reason for catch-all but high-scoring.
- **Configuration:** `reason = "High score"`
- **Outputs:** To **Add to Accepted**

#### Node: **Medium score**
- **Type:** Set
- **Role:** Annotates rejection reason.
- **Configuration:** `reason = "Medium score"`
- **Outputs:** To **Add to Rejected**

#### Node: **Low score**
- **Type:** Set
- **Role:** Annotates rejection reason.
- **Configuration:** `reason = "Low score"`
- **Outputs:** To **Add to Rejected**

#### Node: **Not enough credits for scoring**
- **Type:** Set
- **Role:** Annotates rejection reason when scoring credit check fails.
- **Configuration:** `reason = "Not enough credits for scoring"`
- **Outputs:** To **Add to Rejected**
- **Edge cases:**
  - This path still logs validation details if validation happened and node references are present.

---

### 2.4 Outputs (Google Sheets + Mautic)

**Overview:**  
All accepted leads are logged to an “Accepted” Google Sheet and then synced to Mautic. All rejected leads are logged to a “Rejected” sheet with diagnostic details (status, sub_status, domain, MX, score when available).

**Nodes involved:**
- Add to Accepted
- Create a Mautic contact
- Add to Rejected
- Add to Rejected not validated

#### Node: **Add to Accepted**
- **Type:** Google Sheets
- **Role:** Append/update accepted lead row with validation/scoring metadata.
- **Configuration:**
  - Operation: **appendOrUpdate**
  - Match on column: **Email**
  - Writes:
    - Email, First Name, Last Name from Typeform
    - Accepted At = `$now`
    - Accepted Reason = `$json.reason` (from Valid/High score Set nodes)
    - Many fields from **Validate email** output (domain, status, mx_found, mx_record, free_email, sub_status, did_you_mean, smtp_provider, domain_age_days, account)
    - ZB Score: `{{$node["Score email"] ? $('Score email').item.json.score : ""}}`
      - Uses a conditional presence check to avoid failing when scoring didn’t run.
  - Target sheet: **Accepted** in the spreadsheet
- **Outputs:** To **Create a Mautic contact**
- **Edge cases / failures:**
  - If “Validate email” didn’t run (shouldn’t happen on accepted path), expressions referencing it could fail.
  - Schema/headers must match. appendOrUpdate requires consistent header names and matching column availability.

#### Node: **Create a Mautic contact**
- **Type:** Mautic (`n8n-nodes-base.mautic`)
- **Role:** Creates (or updates, depending on Mautic node behavior) a contact in Mautic for accepted leads.
- **Configuration:**
  - Email, First Name, Last Name from Typeform fields.
  - Additional fields: empty.
- **Inputs:** From **Add to Accepted**
- **Edge cases / failures:**
  - Mautic credentials invalid / endpoint unreachable.
  - If Mautic has required fields beyond these, creation may fail.
  - Duplicate handling depends on Mautic API/settings; ensure it updates by email if desired.

#### Node: **Add to Rejected**
- **Type:** Google Sheets
- **Role:** Append/update rejected lead row with validation/scoring metadata and reason.
- **Configuration:**
  - Operation: **appendOrUpdate**
  - Match column: **Email**
  - Writes:
    - Email, First Name, Last Name from Typeform
    - Rejected At = `$now`
    - Reject Reason = `$json.reason` (from Not valid / Medium / Low / Not enough credits for scoring)
    - Validation metadata from **Validate email**
    - ZB Score: same conditional logic as Accepted
    - `ZB Did You Mean` uses: `{{$node["Validate email"] ? $('Validate email').item.json.did_you_mean : ""}}` (defensive)
- **Inputs:** From rejection reason Set nodes.
- **Edge cases:**
  - If a rejection happens before validation (e.g., credit check fail for validation), this node is not used; instead “Rejected not validated” is used.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Typeform Trigger | Typeform Trigger | Entry point on new form submission | — | Has email? | ## ⚡ Trigger & Verification |
| Has email? | IF | Gate: email present? | Typeform Trigger | Check credits for validation; Email missing | ## ⚡ Trigger & Verification |
| Check credits for validation | ZeroBounce | Guardrail: ensure credits before validation | Has email? | Validate email; Not enough credits for validation | ## ✔️ Stage 1: Email Validation |
| Validate email | ZeroBounce | Validate email status/metadata | Check credits for validation | Status | ## ✔️ Stage 1: Email Validation |
| Status | Switch | Route by validation status | Validate email | Valid; Check credits for scoring; Not valid | ## ✔️ Stage 1: Email Validation |
| Valid | Set | Mark accepted reason (“Valid”) | Status | Add to Accepted | ## 📤 Output Results |
| Not valid | Set | Mark rejected reason (“Not valid”) | Status | Add to Rejected | ## 📤 Output Results |
| Check credits for scoring | ZeroBounce | Guardrail: ensure credits before scoring | Status | Score email; Not enough credits for scoring | ## 🎯 Stage 2: Email Scoring |
| Score email | ZeroBounce | Score catch-all emails | Check credits for scoring | Filter by score | ## 🎯 Stage 2: Email Scoring |
| Filter by score | Switch | Route by score thresholds | Score email | High score; Medium score; Low score | ## 🎯 Stage 2: Email Scoring |
| High score | Set | Mark accepted reason (“High score”) | Filter by score | Add to Accepted | ## 📤 Output Results |
| Medium score | Set | Mark rejected reason (“Medium score”) | Filter by score | Add to Rejected | ## 📤 Output Results |
| Low score | Set | Mark rejected reason (“Low score”) | Filter by score | Add to Rejected | ## 📤 Output Results |
| Not enough credits for scoring | Set | Mark rejected reason (scoring credits insufficient) | Check credits for scoring | Add to Rejected | ## 📤 Output Results |
| Not enough credits for validation | Set | Mark rejected reason (validation credits insufficient) + override email | Check credits for validation | Add to Rejected not validated | ## 📤 Output Results |
| Email missing | Set | Mark rejected reason (missing email) + placeholder | Has email? | Add to Rejected not validated | ## 📤 Output Results |
| Add to Accepted | Google Sheets | Log accepted lead row | Valid; High score | Create a Mautic contact | ## 📤 Output Results |
| Create a Mautic contact | Mautic | Sync accepted lead into Mautic | Add to Accepted | — | ## 📤 Output Results |
| Add to Rejected | Google Sheets | Log rejected lead row with diagnostics | Not valid; Medium score; Low score; Not enough credits for scoring | — | ## 📤 Output Results |
| Add to Rejected not validated | Google Sheets | Log rejected lead row without ZB diagnostics | Email missing; Not enough credits for validation | — | ## 📤 Output Results |
| Sticky Note2 | Sticky Note | Documentation (workflow description) | — | — | ## Typeform to Mautic: Advanced ZeroBounce Lead Vetting\n\n**This workflow automates the transition of new Typeform submissions into Mautic, using [ZeroBounce](https://www.zerobounce.net) for multi-layer validation (Validation + Scoring).**\n\n**Results are also output to Google Sheets for review.** |
| Sticky Note3 | Sticky Note | Section header | — | — | ## ⚡ Trigger & Verification |
| Sticky Note4 | Sticky Note | Section header | — | — | ## ✔️ Stage 1: Email Validation |
| Sticky Note5 | Sticky Note | Section header | — | — | ## 🎯 Stage 2: Email Scoring |
| Sticky Note6 | Sticky Note | Section header | — | — | ## 📤 Output Results |
| Sticky Note | Sticky Note | Branding image | — | — | ![ZeroBounce Logo](https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zb-logo-purple.svg) |
| Sticky Note7 | Sticky Note | Requirements and sheet headers | — | — | ### 📋 Setup Requirements\n- **Typeform:** Connect via Access Token or OAuth2 to receive \"Form Submission\" events.\n- **ZeroBounce:** Connect via API Key. *[Create one here](https://www.zerobounce.net/members/API)*.\n- **Mautic:** Connect via credentials or OAuth2 to create/update people for high-quality leads.\n- **Google Sheets:** Connect via OAuth2 to append/update rows in a Google Sheets worksheet. Alternatively, swap these nodes out with any other data storage node e.g. **n8n Data Table** or **Microsoft Excel**. The sheets/tables can be created with the headers:\n    - *Accepted* columns:\n`Email,First Name,Last Name,Accepted At,Accepted Reason,ZB Status,ZB Sub Status,ZB Free Email,ZB Did You Mean,ZB Account,ZB Domain,ZB Domain Age Days,ZB SMTP Provider,ZB MX Found,ZB MX Record,ZB Score`\n    - *Rejected* columns\n`Email,First Name,Last Name,Rejected At,Rejected Reason,ZB Status,ZB Sub Status,ZB Free Email,ZB Did You Mean,ZB Account,ZB Domain,ZB Domain Age Days,ZB SMTP Provider,ZB MX Found,ZB MX Record,ZB Score` |
| Sticky Note8 | Sticky Note | Benefits summary | — | — | ### 💡 Key Benefits\n- **✨ Zero-Waste Sync:** Only \"Valid\" or \"High-Scoring\" leads reach your CRM.\n- **🛡️ Credit Safety:** Internal checks ensure you never trigger an API call without credits.\n- **📊 Detailed Suppressions:** Every rejected lead is categorized by reason (e.g. Email Missing, Invalid, Low Score, or Insufficient credits). |
| Sticky Note9 | Sticky Note | Integrations referenced | — | — | ### 🧩 Nodes used in this workflow\n- [ZeroBounce](https://n8n.io/integrations/zerobounce)\n- [Typeform Trigger](https://n8n.io/integrations/typeform-trigger)\n- [Mautic](https://n8n.io/integrations/mautic)\n- [Google Sheets](https://n8n.io/integrations/google-sheets) (or alternative e.g. [Data Table](https://n8n.io/integrations/data-table)) |
| Sticky Note10 | Sticky Note | How-it-works description | — | — | ### 🚀 How it Works\n1. **⚡ Trigger & Verification:** Activates on new **Typeform** submissions and first checks if an email address is provided.\n    - **Email present:** Proceeds to credit check and validation\n    - **Email missing:** Customer details are added to *Rejected* output for review with reason `'Email missing'`.\n2. **💳 Credit Management:** Before each ZeroBounce call, the workflow checks your account for sufficient credits to prevent node failures. \n    - **Success:** Proceeds to Stage 1 (Validation).\n    - **Failure:** Customer details are added to *Rejected* output for retry with reason `'Not enough credits'`.\n3. **✔️ Stage 1: Email Validation:** Validates the email address with ZeroBounce. \n    - **Valid:** Adds the result to *Accepted* output and creates a person in **Mautic**.\n    - **Invalid:** Logs to a **Google Sheet** with the validation results and a reason for rejection for review.\n    - **Catch-all:** Proceeds to Stage 2 (Scoring).\n4.  **🎯 Stage 2: Email Scoring:** For \"Catch-all\" emails, the workflow requests **ZeroBounce Scoring**.\n    - **High Score (>=9):** Syncs to **Mautic**.\n    - **Medium/Low Score:** Suppressed and added to a **Google Sheet** with the assigned score for review.\n5. **📤 Output Results:**\n    - **Accepted:** Output validation and scoring results to *Accepted* and **Mautic**\n    - **Rejected** Output validation and scoring results to *Rejected*. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   `Vet new Typeform leads with ZeroBounce and sync qualified contacts to Mautic`

2. **Add “Typeform Trigger”**
   - Node type: **Typeform Trigger**
   - Configure credentials: **Typeform account** (OAuth2 or access token).
   - Select **Form ID**: `Myr169jg` (or your own form).
   - Ensure your form has fields matching these labels (or update expressions later):
     - “What is your email address?”
     - “What is your first name?”
     - “What is your last name?”

3. **Add “Has email?” (IF)**
   - Condition: String → **notEmpty**
   - Value: `{{$json['What is your email address?']}}`
   - Connect: **Typeform Trigger → Has email?**

4. **Add “Check credits for validation” (ZeroBounce)**
   - Node type: **ZeroBounce**
   - Credentials: ZeroBounce API key
   - Resource: **account**
   - Option: set **creditsRequired = 1**
   - Error handling: set **On Error → Continue (Error Output)** (so you get a second output)
   - Connect: **Has email? (true) → Check credits for validation**

5. **Add “Validate email” (ZeroBounce)**
   - Resource: default validation endpoint (email validation)
   - Email field: `{{$('Typeform Trigger').item.json['What is your email address?']}}`
   - Timeout: 10 seconds (optional)
   - Connect: **Check credits for validation (success) → Validate email**

6. **Add “Status” (Switch)**
   - Switch on: `{{$json.status}}`
   - Rule A (rename output “Valid”): equals `"valid"`
   - Rule B (rename output “Catch-all”): equals `"catch-all"`
   - Fallback output renamed to “Other”
   - Connect: **Validate email → Status**

7. **Add acceptance reason Set node “Valid”**
   - Set: `reason = "Valid"`
   - Connect: **Status: Valid → Valid**

8. **Add scoring credit check “Check credits for scoring” (ZeroBounce)**
   - Resource: **account**
   - creditsRequired: 1
   - On Error: **Continue (Error Output)**
   - Connect: **Status: Catch-all → Check credits for scoring**

9. **Add “Score email” (ZeroBounce)**
   - Resource: **scoring**
   - Email: `{{$('Typeform Trigger').item.json['What is your email address?']}}`
   - Connect: **Check credits for scoring (success) → Score email**

10. **Add “Filter by score” (Switch)**
    - Rule high: `{{$json.score}}` **>= 9**
    - Rule medium: `{{$json.score}}` **>= 3**
    - Rule low: `{{$json.score}}` **< 3**
    - Enable loose type validation (recommended)
    - Connect: **Score email → Filter by score**

11. **Add Set nodes for reasons**
    - **High score**: `reason = "High score"`; connect from Filter by score → high
    - **Medium score**: `reason = "Medium score"`; connect from Filter by score → medium
    - **Low score**: `reason = "Low score"`; connect from Filter by score → low

12. **Add rejection reason Set node “Not valid”**
    - `reason = "Not valid"`
    - Connect: **Status: Other → Not valid**

13. **Add “Not enough credits for scoring” (Set)**
    - `reason = "Not enough credits for scoring"`
    - Connect: **Check credits for scoring (error output) → Not enough credits for scoring**

14. **Add “Email missing” (Set)**
    - `reason = "Email missing"`
    - `email_override = {{$('Typeform Trigger').item.json['What is your first name?']}} {{$('Typeform Trigger').item.json['What is your last name?']}}`
    - Connect: **Has email? (false) → Email missing**

15. **Add “Not enough credits for validation” (Set)**
    - `reason = "Not enough credits for validation"`
    - IMPORTANT: set `email_override` to the Typeform email field, for example:  
      `{{$('Typeform Trigger').item.json['What is your email address?']}}`  
      (In the provided workflow JSON it references `Form.io Trigger`, which does not exist.)
    - Connect: **Check credits for validation (error output) → Not enough credits for validation**

16. **Prepare Google Sheets**
    - Create (or reuse) one spreadsheet with two sheets/tabs: **Accepted** and **Rejected**
    - Ensure headers match the sticky note lists (at minimum the columns you map).

17. **Add “Add to Accepted” (Google Sheets)**
    - Credentials: Google Sheets OAuth2
    - Document: select your spreadsheet
    - Sheet: **Accepted**
    - Operation: **appendOrUpdate**
    - Matching column: **Email**
    - Map key fields:
      - Email / First / Last from Typeform Trigger
      - Accepted At = `{{$now}}`
      - Accepted Reason = `{{$json.reason}}`
      - ZeroBounce fields from **Validate email**
      - ZB Score with conditional: `{{$node["Score email"] ? $('Score email').item.json.score : ""}}`
    - Connect: **Valid → Add to Accepted** and **High score → Add to Accepted**

18. **Add “Create a Mautic contact”**
    - Node type: **Mautic**
    - Credentials: Mautic API credentials/OAuth2
    - Configure:
      - Email = Typeform email
      - First Name / Last Name from Typeform
    - Connect: **Add to Accepted → Create a Mautic contact**

19. **Add “Add to Rejected” (Google Sheets)**
    - Sheet: **Rejected**
    - Operation: **appendOrUpdate**
    - Match column: **Email**
    - Map:
      - Email/First/Last (Typeform)
      - Rejected At = `{{$now}}`
      - Reject Reason = `{{$json.reason}}`
      - Include Validate email fields when available
      - ZB Score conditional as above
    - Connect: **Not valid → Add to Rejected**, **Medium score → Add to Rejected**, **Low score → Add to Rejected**, **Not enough credits for scoring → Add to Rejected**

20. **Add “Add to Rejected not validated” (Google Sheets)**
    - Sheet: **Rejected**
    - Operation: **appendOrUpdate**
    - Match column: **Email**
    - Map:
      - Email = `{{$json.email_override}}`
      - Rejected At = `{{$now}}`
      - Reject Reason = `{{$json.reason}}`
      - First/Last from Typeform
    - Connect: **Email missing → Add to Rejected not validated**, **Not enough credits for validation → Add to Rejected not validated**

21. **Activate workflow** after testing with a sample Typeform submission.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ZeroBounce main site | https://www.zerobounce.net |
| ZeroBounce API key creation | https://www.zerobounce.net/members/API |
| ZeroBounce n8n integration page | https://n8n.io/integrations/zerobounce |
| Typeform Trigger integration page | https://n8n.io/integrations/typeform-trigger |
| Mautic integration page | https://n8n.io/integrations/mautic |
| Google Sheets integration page | https://n8n.io/integrations/google-sheets |
| Alternative storage: n8n Data Table integration | https://n8n.io/integrations/data-table |
| Branding asset used in canvas | https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zb-logo-purple.svg |
| Important fix: “Not enough credits for validation” references a missing node (“Form.io Trigger”) | Replace expression with Typeform Trigger’s email field to avoid runtime expression errors. |