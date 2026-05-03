Vet Jotform leads with ZeroBounce and sync qualified contacts to ActiveCampaign

https://n8nworkflows.xyz/workflows/vet-jotform-leads-with-zerobounce-and-sync-qualified-contacts-to-activecampaign-13018


# Vet Jotform leads with ZeroBounce and sync qualified contacts to ActiveCampaign

## 1. Workflow Overview

**Workflow name:** Vet new Jotform leads with ZeroBounce and sync qualified contacts to ActiveCampaign  
**Purpose:** When a new Jotform submission arrives, the workflow verifies the presence of an email, checks available ZeroBounce credits, validates the email, and (only for “catch-all” emails) scores it. Qualified leads are synced to **ActiveCampaign** and logged to an **Accepted** Google Sheet; rejected/suppressed leads are logged to a **Rejected** Google Sheet with a clear reason.

### 1.1 Logical blocks
1. **Trigger & Email Presence Gate**: Receive Jotform submission; reject immediately if email is missing.  
2. **Stage 1 – Credit Check (Validation) + Email Validation**: Ensure enough ZeroBounce credits; validate email; route by status.  
3. **Stage 2 – Credit Check (Scoring) + Email Scoring**: For “catch-all” only, ensure credits; score email; route by score threshold.  
4. **Outputs**:  
   - **Accepted path** → append/update Google Sheet (Accepted) → create/update ActiveCampaign contact  
   - **Rejected paths** → append/update Google Sheet (Rejected), with variants for “not validated” cases (missing email / insufficient credits)

**Sticky-note stated intent:** Multi-layer vetting (Validation + Scoring) using [ZeroBounce](https://www.zerobounce.net), with results also output to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Email Presence Gate
**Overview:** Starts on new Jotform submissions and checks whether the email field is present. Missing emails are logged to Rejected without calling ZeroBounce.

**Nodes involved:**  
- Jotform Trigger  
- Has email?  
- Email missing  
- Add to Rejected not validated

#### Node: **Jotform Trigger**
- **Type / role:** `jotFormTrigger` — webhook trigger for new form submissions.
- **Configuration:** Listens to Jotform **form ID `260114799366364`**.
- **Inputs/outputs:** Entry point → outputs to **Has email?**.
- **Credentials:** JotForm API credential (“JotForm account”).
- **Failure/edge cases:**
  - Jotform credential revoked/invalid.
  - Form field names differ (workflow expects `E-mail` and `Full Name.first/last`).
  - Webhook disabled/changed in Jotform → no events.

#### Node: **Has email?**
- **Type / role:** `if` — guards downstream API calls.
- **Logic:** Checks `{{$json['E-mail']}}` **is not empty**.
- **Outputs:**
  - **True** → **Check credits for validation**
  - **False** → **Email missing**
- **Failure/edge cases:**
  - If Jotform field key isn’t exactly `E-mail`, the expression evaluates empty and causes false rejection.

#### Node: **Email missing**
- **Type / role:** `set` — builds a rejection payload for missing email.
- **Configuration choices:**
  - Sets `reason = "Email missing"`.
  - Sets `email_override = {{$json['Full Name'].first}} {{$json['Full Name'].last}}` (uses name as placeholder).
- **Outputs:** → **Add to Rejected not validated**
- **Failure/edge cases:**
  - If `Full Name.first/last` doesn’t exist, `email_override` may become blank or error depending on runtime data.

#### Node: **Add to Rejected not validated**
- **Type / role:** `googleSheets` — logs rejected leads that were *not* validated (missing email / insufficient credits).
- **Operation:** `appendOrUpdate` in Google Sheets document “ZeroBounce Jotform ActiveCampaign”, sheet “Rejected”.
- **Matching:** `Email` column is the matching key.
- **Columns written (subset):**
  - `Email = {{$json.email_override}}`
  - `First Name/Last Name` from Jotform Trigger
  - `Rejected At = {{$now}}`
  - `Reject Reason = {{$json.reason}}`
- **Credentials:** Google Sheets OAuth2.
- **Failure/edge cases:**
  - Sheet headers must exist and match expected names (at least Email/First Name/Last Name/Rejected At/Reject Reason).
  - OAuth token expired or spreadsheet permissions missing.
  - `appendOrUpdate` can overwrite rows when the same Email repeats (by design).

---

### Block 2 — Stage 1: Credit Check (Validation) + Email Validation
**Overview:** Checks ZeroBounce credits before validating. If credits exist, validates the email and routes by ZeroBounce status.

**Nodes involved:**  
- Check credits for validation  
- Validate email  
- Status  
- Valid  
- Not valid  
- Not enough credits for validation

#### Node: **Check credits for validation**
- **Type / role:** `@zerobounce/...zeroBounce` (resource: **account**) — ensures credits before validation.
- **Configuration choices:**
  - `resource = account`
  - `creditsRequired = 1`
  - **onError = continueErrorOutput** (important): failures go to the error output path rather than stopping the workflow.
- **Outputs:**
  - **Success output** → **Validate email**
  - **Error output** → **Not enough credits for validation**
- **Credentials:** ZeroBounce API key credential (“ZeroBounce account”).
- **Failure/edge cases:**
  - ZeroBounce API unreachable/timeouts.
  - Credits < 1 triggers “error” path.
  - If onError output semantics change with node version, routing must be rechecked.

#### Node: **Validate email**
- **Type / role:** ZeroBounce — performs **email validation** lookup.
- **Configuration choices:**
  - `email = {{ $('Jotform Trigger').item.json['E-mail'] }}`
  - Timeout option set to **10 seconds**.
- **Input/Output:** From credit check → outputs validation object to **Status**.
- **Failure/edge cases:**
  - If `E-mail` is malformed, ZeroBounce may return statuses like `invalid`, `do_not_mail`, etc.
  - Timeout too low for intermittent latency.
  - Expression couples node tightly to “Jotform Trigger” item pairing; if multiple items/branching breaks pairing, could misreference.

#### Node: **Status**
- **Type / role:** `switch` — routes based on `{{$json.status}}`.
- **Rules configured (renamed outputs):**
  - **Valid** when status equals `"valid"`
  - **Catch-all** when status equals `"catch-all"`
  - **Fallback output renamed to “Other”** (covers all other statuses)
- **Outputs:**
  - Valid → **Valid (Set)**
  - Catch-all → **Check credits for scoring**
  - Other → **Not valid (Set)**
- **Failure/edge cases:**
  - ZeroBounce may return `catch-all` vs `catch_all` depending on API; this workflow expects exact `"catch-all"`.
  - Many “bad” statuses (e.g., `invalid`, `do_not_mail`, `spamtrap`) fall into “Other” and are treated as Not valid.

#### Node: **Valid**
- **Type / role:** `set` — assigns acceptance reason.
- **Configuration:** Sets `reason = "Valid"`.
- **Outputs:** → **Add to Accepted**

#### Node: **Not valid**
- **Type / role:** `set` — assigns rejection reason for non-valid statuses.
- **Configuration:** Sets `reason = "Not valid"`.
- **Outputs:** → **Add to Rejected**

#### Node: **Not enough credits for validation**
- **Type / role:** `set` — rejection reason for credit shortfall before validation.
- **Configuration choices:**
  - Sets `reason = "Not enough credits for validation"`
  - Sets `email_override = {{ $('Form.io Trigger').item.json.email }}`
- **Outputs:** → **Add to Rejected not validated**
- **Important edge case (design bug):**
  - This workflow has **no “Form.io Trigger” node**. The expression `$('Form.io Trigger')...` will fail at runtime unless such a node exists.
  - It likely should reference `$('Jotform Trigger').item.json['E-mail']` instead. As-is, this path can produce an expression error or empty email_override (depends on n8n expression handling/version).

---

### Block 3 — Stage 2: Credit Check (Scoring) + Scoring Decision
**Overview:** For “catch-all” emails only, checks credits again and requests a ZeroBounce score. High scores are accepted; medium/low are rejected.

**Nodes involved:**  
- Check credits for scoring  
- Score email  
- Filter by score  
- High score  
- Medium score  
- Low score  
- Not enough credits for scoring

#### Node: **Check credits for scoring**
- **Type / role:** ZeroBounce (resource: **account**) — ensures credits before scoring.
- **Configuration:**
  - `creditsRequired = 1`
  - **onError = continueErrorOutput**
- **Outputs:**
  - Success → **Score email**
  - Error → **Not enough credits for scoring**
- **Failure/edge cases:** same as validation credit check.

#### Node: **Score email**
- **Type / role:** ZeroBounce (resource: **scoring**) — requests an email quality score.
- **Configuration:** `email = {{ $('Jotform Trigger').item.json['E-mail'] }}`
- **Outputs:** → **Filter by score**
- **Failure/edge cases:**
  - API may return missing/undefined `score`; downstream switch uses numeric comparisons.
  - If score is string, “loose” validation is enabled downstream.

#### Node: **Filter by score**
- **Type / role:** `switch` — routes by numeric `{{$json.score}}`.
- **Rules (outputs renamed):**
  - **high** if score `>= 9`
  - **medium** if score `>= 3`
  - **low** if score `< 3`
- **Outputs:**
  - high → **High score (Set)**
  - medium → **Medium score (Set)**
  - low → **Low score (Set)**
- **Edge cases / ordering note:**
  - Rules are evaluated in order. With `>=3` as “medium” after `>=9` it works as intended.
  - If score is null/empty, it will go to fallback output (configured as output index `2`), which is ambiguous; consider explicit null handling.

#### Node: **High score**
- **Type / role:** `set` — acceptance reason.
- **Configuration:** Sets `reason = "High score"`.
- **Outputs:** → **Add to Accepted**

#### Node: **Medium score**
- **Type / role:** `set` — rejection reason.
- **Configuration:** Sets `reason = "Medium score"`.
- **Outputs:** → **Add to Rejected**

#### Node: **Low score**
- **Type / role:** `set` — rejection reason.
- **Configuration:** Sets `reason = "Low score"`.
- **Outputs:** → **Add to Rejected**

#### Node: **Not enough credits for scoring**
- **Type / role:** `set` — rejection reason.
- **Configuration:** Sets `reason = "Not enough credits for scoring"`.
- **Outputs:** → **Add to Rejected**
- **Edge case:** This path logs to the “validated” rejected sheet node (includes validation fields), but scoring may not have run; the Google Sheets node handles this by conditionally leaving score blank.

---

### Block 4 — Outputs: Google Sheets + ActiveCampaign
**Overview:** Writes full decision context to Google Sheets and, for accepted leads, creates/updates the contact in ActiveCampaign.

**Nodes involved:**  
- Add to Accepted  
- Create an ActiveCampaign contact  
- Add to Rejected  
- (plus the “not validated” variant already covered)

#### Node: **Add to Accepted**
- **Type / role:** `googleSheets` — logs accepted leads with validation/scoring metadata.
- **Operation:** `appendOrUpdate` to **Accepted** sheet in the same spreadsheet.
- **Matching:** `Email` column.
- **Key expressions/fields:**
  - `Email = {{ $('Jotform Trigger').item.json['E-mail'] }}`
  - `First Name/Last Name` from Jotform
  - `Accepted Reason = {{ $json.reason }}`
  - Many ZB fields from **Validate email** output (domain, status, sub_status, mx, smtp_provider, etc.)
  - `ZB Score = {{ $node["Score email"] ? $('Score email').item.json.score : "" }}` (only set if Score email exists)
  - `Accepted At = {{$now}}`
- **Outputs:** → **Create an ActiveCampaign contact**
- **Failure/edge cases:**
  - Spreadsheet schema mismatch or renamed columns.
  - If “Validate email” didn’t run (should not happen on accepted path), expressions referencing it would fail; current routing ensures Validate email runs first.

#### Node: **Create an ActiveCampaign contact**
- **Type / role:** `activeCampaign` — creates or updates a contact.
- **Configuration choices:**
  - `email = {{$json.Email}}` (takes Email from “Add to Accepted” output row)
  - `updateIfExists = true`
  - `firstName/lastName` from Jotform Trigger.
- **Inputs/Outputs:** From **Add to Accepted**; terminal node (no outputs connected).
- **Credentials:** ActiveCampaign API credential.
- **Failure/edge cases:**
  - ActiveCampaign API key invalid / wrong account URL configuration.
  - If `{{$json.Email}}` is blank (e.g., malformed mapping), contact creation will fail.
  - Rate limiting or 429 responses for high-volume forms.

#### Node: **Add to Rejected**
- **Type / role:** `googleSheets` — logs rejected leads with validation metadata (and score if available).
- **Operation:** `appendOrUpdate` to **Rejected** sheet.
- **Matching:** `Email` column.
- **Key expressions/fields:**
  - Email/First/Last from Jotform
  - `Reject Reason = {{$json.reason}}`
  - ZB fields from **Validate email**
  - `ZB Score` is conditional on Score email node presence (same pattern as Accepted)
- **Failure/edge cases:**
  - For rejection reasons that occur before validation (e.g., “Not enough credits for scoring”), Validate email exists (since scoring only happens after validation), so OK.
  - For “Not enough credits for validation” and “Email missing”, workflow uses **Add to Rejected not validated** instead (minimal columns).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Jotform Trigger | JotForm Trigger | Entry point: receives new form submissions | — | Has email? | ## ⚡ Trigger & Verification |
| Has email? | IF | Gate: ensure email present | Jotform Trigger | Check credits for validation; Email missing | ## ⚡ Trigger & Verification |
| Check credits for validation | ZeroBounce | Prevent validation call if insufficient credits | Has email? (true) | Validate email; Not enough credits for validation | ## ✔️ Stage 1: Email Validation |
| Validate email | ZeroBounce | Validate email status (valid/catch-all/other) | Check credits for validation | Status | ## ✔️ Stage 1: Email Validation<br>![ZeroBounce Logo](https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zb-logo-purple.svg) |
| Status | Switch | Route by ZeroBounce validation status | Validate email | Valid; Check credits for scoring; Not valid | ## ✔️ Stage 1: Email Validation |
| Valid | Set | Mark accepted reason “Valid” | Status (Valid) | Add to Accepted | ## 📤 Output Results |
| Check credits for scoring | ZeroBounce | Prevent scoring call if insufficient credits | Status (Catch-all) | Score email; Not enough credits for scoring | ## 🎯 Stage 2: Email Scoring |
| Score email | ZeroBounce | Score catch-all email | Check credits for scoring | Filter by score | ## 🎯 Stage 2: Email Scoring |
| Filter by score | Switch | Route by score thresholds | Score email | High score; Medium score; Low score | ## 🎯 Stage 2: Email Scoring |
| High score | Set | Mark accepted reason “High score” | Filter by score (high) | Add to Accepted | ## 📤 Output Results |
| Medium score | Set | Mark rejected reason “Medium score” | Filter by score (medium) | Add to Rejected | ## 📤 Output Results |
| Low score | Set | Mark rejected reason “Low score” | Filter by score (low) | Add to Rejected | ## 📤 Output Results |
| Not valid | Set | Mark rejected reason “Not valid” | Status (Other) | Add to Rejected | ## 📤 Output Results |
| Not enough credits for scoring | Set | Mark rejected reason for scoring credit shortfall | Check credits for scoring (error) | Add to Rejected | ## 📤 Output Results |
| Email missing | Set | Mark rejected reason “Email missing” + placeholder email_override | Has email? (false) | Add to Rejected not validated | ## ⚡ Trigger & Verification |
| Not enough credits for validation | Set | Mark rejected reason for validation credit shortfall (has broken Form.io reference) | Check credits for validation (error) | Add to Rejected not validated | ## ✔️ Stage 1: Email Validation |
| Add to Accepted | Google Sheets | Append/update accepted lead log | Valid; High score | Create an ActiveCampaign contact | ## 📤 Output Results |
| Create an ActiveCampaign contact | ActiveCampaign | Create/update CRM contact | Add to Accepted | — | ## 📤 Output Results |
| Add to Rejected | Google Sheets | Append/update rejected lead log (validated/scored context) | Not valid; Medium score; Low score; Not enough credits for scoring | — | ## 📤 Output Results |
| Add to Rejected not validated | Google Sheets | Append/update rejected lead log (minimal fields, no validation) | Email missing; Not enough credits for validation | — | ## 📤 Output Results |
| Sticky Note2 | Sticky Note | Documentation/branding block | — | — | ## Jotform to ActiveCampaign: Advanced ZeroBounce Lead Vetting… |
| Sticky Note10 | Sticky Note | “How it Works” explanation | — | — | ### 🚀 How it Works… |
| Sticky Note8 | Sticky Note | Key benefits | — | — | ### 💡 Key Benefits… |
| Sticky Note7 | Sticky Note | Setup requirements + sheet headers | — | — | ### 📋 Setup Requirements… |
| Sticky Note9 | Sticky Note | Nodes used list + links | — | — | ### 🧩 Nodes used in this workflow… |
| Sticky Note3 | Sticky Note | Section header: Trigger & Verification | — | — | ## ⚡ Trigger & Verification |
| Sticky Note4 | Sticky Note | Section header: Stage 1 | — | — | ## ✔️ Stage 1: Email Validation |
| Sticky Note5 | Sticky Note | Section header: Stage 2 | — | — | ## 🎯 Stage 2: Email Scoring |
| Sticky Note6 | Sticky Note | Section header: Output Results | — | — | ## 📤 Output Results |
| Sticky Note | Sticky Note | ZeroBounce logo image | — | — | ![ZeroBounce Logo](https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zb-logo-purple.svg) |

---

## 4. Reproducing the Workflow from Scratch (Manual Build)

1. **Create workflow**
   - Name it: *Vet new Jotform leads with ZeroBounce and sync qualified contacts to ActiveCampaign*.

2. **Add Trigger**
   - Add node: **Jotform Trigger**
   - Configure:
     - Credential: **Jotform API Key**
     - Form: select/provide form ID **260114799366364**
   - Ensure the form includes fields named exactly:
     - `E-mail`
     - `Full Name.first` and `Full Name.last` (Jotform “Full Name” widget)

3. **Add email presence gate**
   - Add node: **IF** named **Has email?**
   - Condition: String → `notEmpty`
     - Value: `={{ $json['E-mail'] }}`
   - Connect: **Jotform Trigger → Has email?**

4. **Missing email rejection path**
   - Add node: **Set** named **Email missing**
     - Add fields:
       - `reason` = `Email missing`
       - `email_override` = `={{ $json['Full Name'].first }} {{ $json['Full Name'].last }}`
   - Add node: **Google Sheets** named **Add to Rejected not validated**
     - Credential: Google Sheets OAuth2
     - Document: your spreadsheet (create one if needed)
     - Sheet: `Rejected`
     - Operation: `Append or Update`
     - Matching column: `Email`
     - Map columns (minimum):
       - Email = `={{ $json.email_override }}`
       - First Name = `={{ $('Jotform Trigger').item.json['Full Name'].first }}`
       - Last Name = `={{ $('Jotform Trigger').item.json['Full Name'].last }}`
       - Rejected At = `={{ $now }}`
       - Reject Reason = `={{ $json.reason }}`
   - Connect: **Has email? (false) → Email missing → Add to Rejected not validated**

5. **ZeroBounce credit check for validation**
   - Add node: **ZeroBounce** named **Check credits for validation**
   - Resource: **Account**
   - Options: `creditsRequired = 1`
   - Set **On Error** to: **Continue (error output)** (`continueErrorOutput`)
   - Connect: **Has email? (true) → Check credits for validation**

6. **Handle “not enough credits for validation”**
   - Add node: **Set** named **Not enough credits for validation**
     - Fields:
       - `reason` = `Not enough credits for validation`
       - `email_override` = `={{ $('Jotform Trigger').item.json['E-mail'] }}`  *(recommended fix; do not reference “Form.io Trigger” unless you add it)*
   - Connect: **Check credits for validation (error output) → Not enough credits for validation → Add to Rejected not validated**

7. **Validate email**
   - Add node: **ZeroBounce** named **Validate email**
   - Resource: **Validation** (default validation operation)
   - Email: `={{ $('Jotform Trigger').item.json['E-mail'] }}`
   - Options: Timeout = `10` seconds
   - Connect: **Check credits for validation (success) → Validate email**

8. **Route by validation status**
   - Add node: **Switch** named **Status**
   - Value to evaluate: `={{ $json.status }}`
   - Add rules:
     - Output “Valid”: equals `valid`
     - Output “Catch-all”: equals `catch-all`
     - Fallback rename: “Other”
   - Connect: **Validate email → Status**

9. **Valid acceptance branch**
   - Add node: **Set** named **Valid** → `reason = Valid`
   - Connect: **Status (Valid) → Valid**

10. **Other statuses rejection branch**
   - Add node: **Set** named **Not valid** → `reason = Not valid`
   - Connect: **Status (Other) → Not valid**

11. **ZeroBounce credit check for scoring (catch-all only)**
   - Add node: **ZeroBounce** named **Check credits for scoring**
   - Resource: **Account**
   - Options: `creditsRequired = 1`
   - On Error: **continueErrorOutput**
   - Connect: **Status (Catch-all) → Check credits for scoring**

12. **Handle “not enough credits for scoring”**
   - Add node: **Set** named **Not enough credits for scoring** → `reason = Not enough credits for scoring`
   - Connect: **Check credits for scoring (error output) → Not enough credits for scoring**

13. **Score catch-all email**
   - Add node: **ZeroBounce** named **Score email**
   - Resource: **Scoring**
   - Email: `={{ $('Jotform Trigger').item.json['E-mail'] }}`
   - Connect: **Check credits for scoring (success) → Score email**

14. **Route by score**
   - Add node: **Switch** named **Filter by score**
   - Value: `={{ $json.score }}`
   - Rules in order:
     - `>= 9` → output “high”
     - `>= 3` → output “medium”
     - `< 3` → output “low”
   - Connect: **Score email → Filter by score**

15. **Create Set nodes for score outcomes**
   - **High score** Set: `reason = High score`
   - **Medium score** Set: `reason = Medium score`
   - **Low score** Set: `reason = Low score`
   - Connect:
     - Filter by score (high) → High score
     - Filter by score (medium) → Medium score
     - Filter by score (low) → Low score

16. **Google Sheets logging**
   - Add node: **Google Sheets** named **Add to Accepted**
     - Operation: `Append or Update`
     - Sheet: `Accepted`
     - Matching column: `Email`
     - Map fields as needed, including:
       - Email, First/Last, Accepted At (`{{$now}}`), Accepted Reason (`{{$json.reason}}`)
       - Validation fields from `Validate email` node (domain/status/sub_status/mx/smtp_provider/etc.)
       - Score (optional): `={{ $node["Score email"] ? $('Score email').item.json.score : "" }}`
   - Add node: **Google Sheets** named **Add to Rejected**
     - Operation: `Append or Update`
     - Sheet: `Rejected`
     - Matching: `Email`
     - Map fields including Reject Reason and validation fields; score conditional as above.

17. **Connect outcomes to Sheets**
   - **Valid → Add to Accepted**
   - **High score → Add to Accepted**
   - **Not valid → Add to Rejected**
   - **Medium score → Add to Rejected**
   - **Low score → Add to Rejected**
   - **Not enough credits for scoring → Add to Rejected**

18. **ActiveCampaign sync (accepted only)**
   - Add node: **ActiveCampaign** named **Create an ActiveCampaign contact**
   - Operation: Create/Update contact (enable **Update if exists**)
   - Email: `={{ $json.Email }}` (coming from Add to Accepted row output)
   - Additional fields:
     - First Name: `={{ $('Jotform Trigger').item.json['Full Name'].first }}`
     - Last Name: `={{ $('Jotform Trigger').item.json['Full Name'].last }}`
   - Connect: **Add to Accepted → Create an ActiveCampaign contact**
   - Credentials: ActiveCampaign API key (and account URL if required by your n8n credential setup).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **ZeroBounce + Jotform to ActiveCampaign** with multi-layer checks (Validation + Scoring). Results output to Google Sheets. | From sticky note: “Jotform to ActiveCampaign: Advanced ZeroBounce Lead Vetting” + [ZeroBounce](https://www.zerobounce.net) |
| Nodes used: ZeroBounce, Jotform Trigger, ActiveCampaign, Google Sheets (or Data Table alternative). | [ZeroBounce integration](https://n8n.io/integrations/zerobounce) • [Jotform Trigger](https://n8n.io/integrations/jotform-trigger) • [ActiveCampaign](https://n8n.io/integrations/activecampaign) • [Google Sheets](https://n8n.io/integrations/google-sheets) • [Data Table](https://n8n.io/integrations/data-table) |
| Setup requirements include API keys and suggested sheet headers for Accepted/Rejected. | From sticky note “Setup Requirements”; ZeroBounce API key creation: https://www.zerobounce.net/members/API |
| **Credit safety design:** checks credits before each ZeroBounce call using error-output continuation. | From “How it Works” + implemented via `continueErrorOutput` on both credit-check nodes |
| Known issue to fix: “Not enough credits for validation” references **Form.io Trigger** which is not present. | Update expression to use **Jotform Trigger** instead (recommended). |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.