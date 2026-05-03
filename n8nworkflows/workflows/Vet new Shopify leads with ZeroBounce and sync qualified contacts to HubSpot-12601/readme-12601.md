Vet new Shopify leads with ZeroBounce and sync qualified contacts to HubSpot

https://n8nworkflows.xyz/workflows/vet-new-shopify-leads-with-zerobounce-and-sync-qualified-contacts-to-hubspot-12601


# Vet new Shopify leads with ZeroBounce and sync qualified contacts to HubSpot

## 1. Workflow Overview

**Title:** Vet new Shopify leads with ZeroBounce and sync qualified contacts to HubSpot

**Purpose:**  
When a new Shopify customer is created, this workflow verifies that an email exists, checks available ZeroBounce credits, validates the email via ZeroBounce, and (when needed) applies ZeroBounce AI Scoring for catch-all emails. Qualified leads are written to an **Accepted** Google Sheet and synced to **HubSpot**. Rejected or unprocessed leads are written to a **Rejected** Google Sheet with a clear rejection reason.

**Target use cases**
- Preventing low-quality/invalid emails from entering HubSpot
- Handling catch-all domains via AI scoring instead of blanket acceptance/rejection
- Auditing decisions via Google Sheets logs (Accepted / Rejected)

### 1.1 Trigger & basic verification
Starts on Shopify `customers/create`, then checks if `email` is present.

### 1.2 Stage 1: Credit check + ZeroBounce validation
Checks ZeroBounce credits before validation; validates email; routes based on `status` (valid / catch-all / other).

### 1.3 Stage 2: Credit check + AI Scoring for catch-all emails
If catch-all, checks credits then scores email and routes by score thresholds.

### 1.4 Output: Logging + HubSpot sync
Accepted leads → Google Sheets “Accepted” → create HubSpot contact.  
Rejected leads → Google Sheets “Rejected” with reason (missing email, invalid, low/medium score, insufficient credits).

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Verification
**Overview:** Receives Shopify customer creation events and ensures an email exists before spending ZeroBounce credits.  
**Nodes involved:** `Shopify customer created`, `Has email?`, `Email missing`

#### Node: Shopify customer created
- **Type / role:** `Shopify Trigger` — webhook trigger for new customers.
- **Configuration:**
  - Topic: **customers/create**
  - Auth: **OAuth2** (Shopify Access Token account)
- **Outputs:** Shopify customer payload (includes `id`, `email`, `first_name`, `last_name`, `addresses`, etc.).
- **Connections:** → `Has email?`
- **Failure modes / edge cases:**
  - OAuth token revoked/expired
  - Shopify webhook delivery failures
  - Customer record may have missing/empty `email` (handled next)

#### Node: Has email?
- **Type / role:** `IF` — gates the workflow if email is missing.
- **Condition:** `{{$json.email}}` **not empty**
- **Outputs:**
  - **true** → `Check credits for validation`
  - **false** → `Email missing`
- **Failure modes / edge cases:**
  - Email can exist but be malformed; handled later by ZeroBounce validation

#### Node: Email missing
- **Type / role:** `Set` — assigns a rejection reason.
- **Configuration:** sets field `reason = "Email missing"`
- **Connections:** → `Add to Rejected not validated`
- **Edge cases:** None (simple enrichment node)

---

### Block 2 — Stage 1: Credit Check & Email Validation
**Overview:** Prevents validation calls if credits are insufficient; validates email; routes outcomes by ZeroBounce `status`.  
**Nodes involved:** `Check credits for validation`, `Validate email`, `Status`, `Valid`, `Not valid`, `Not enough credits for validation`

#### Node: Check credits for validation
- **Type / role:** `ZeroBounce` (resource: **account**) — verifies credit availability.
- **Configuration choices:**
  - `creditsRequired: 1`
  - **onError:** `continueErrorOutput` (important: failure goes to a secondary output)
- **Connections:**
  - **Success (main output 0)** → `Validate email`
  - **Error output** → `Not enough credits for validation`
- **Failure modes / edge cases:**
  - Invalid API key / auth failure
  - Network timeout
  - ZeroBounce service errors
  - Because `continueErrorOutput` is enabled, failures should not stop the workflow but will route to rejection

#### Node: Validate email
- **Type / role:** `ZeroBounce` (default validation endpoint) — validates deliverability and returns status/sub_status and diagnostics.
- **Configuration:**
  - Email: `{{ $('Shopify customer created').item.json.email }}`
  - Timeout: **10 seconds**
- **Connections:** → `Status`
- **Key outputs used later:**
  - `status` (e.g., `valid`, `catch-all`, etc.)
  - `domain`, `mx_found`, `mx_record`, `free_email`, `sub_status`, `did_you_mean`, `smtp_provider`, `domain_age_days`, `account`, `address`
- **Failure modes / edge cases:**
  - Timeout (10s) may be tight during API slowness
  - Unexpected/missing fields if API errors are returned
  - If Shopify payload changes, expression referencing the trigger node could break

#### Node: Status
- **Type / role:** `Switch` — routes based on `{{$json.status}}`
- **Rules:**
  - Output **Valid** if status equals `"valid"`
  - Output **Catch-all** if status equals `"catch-all"`
  - Fallback output renamed to **Other**
- **Connections:**
  - **Valid** → `Valid`
  - **Catch-all** → `Check credits for scoring`
  - **Other** → `Not valid` (covers invalid, unknown, disposable, spamtrap, etc.)
- **Failure modes / edge cases:**
  - Case sensitivity: rule expects lowercase strings exactly (`valid`, `catch-all`)
  - Any new/unknown status values will fall into **Other**

#### Node: Valid
- **Type / role:** `Set` — assigns acceptance reason.
- **Configuration:** `reason = "Valid"`
- **Connections:** → `Add to Accepted`

#### Node: Not valid
- **Type / role:** `Set` — assigns rejection reason.
- **Configuration:** `reason = "Not valid"`
- **Connections:** → `Add to Rejected`

#### Node: Not enough credits for validation
- **Type / role:** `Set` — assigns rejection reason for credit shortage.
- **Configuration:** `reason = "Not enough credits for validation"`
- **Connections:** → `Add to Rejected not validated`
- **Edge cases:** The workflow logs fewer ZeroBounce fields here because validation was not performed.

---

### Block 3 — Stage 2: Credit Check & AI Email Scoring (Catch-all path)
**Overview:** For catch-all emails, checks credits before calling scoring, then categorizes by score thresholds.  
**Nodes involved:** `Check credits for scoring`, `Score email`, `Filter by score`, `High score`, `Medium score`, `Low score`, `Not enough credits for scoring`

#### Node: Check credits for scoring
- **Type / role:** `ZeroBounce` (resource: **account**) — verifies credits before scoring.
- **Configuration:**
  - `creditsRequired: 1`
  - **onError:** `continueErrorOutput`
- **Connections:**
  - **Success** → `Score email`
  - **Error output** → `Not enough credits for scoring`
- **Failure modes / edge cases:** same patterns as validation credit check.

#### Node: Score email
- **Type / role:** `ZeroBounce` (resource: **scoring**) — AI scoring for email quality.
- **Configuration:**
  - Email: `{{ $('Shopify customer created').item.json.email }}`
- **Outputs used later:** `score` (numeric)
- **Connections:** → `Filter by score`
- **Failure modes / edge cases:**
  - If scoring returns non-numeric score or missing `score`, downstream comparisons may misbehave (though switch is set to loose validation)

#### Node: Filter by score
- **Type / role:** `Switch` — routes by numeric score.
- **Rules:**
  - **high** if `score >= 9`
  - **medium** if `score >= 3`
  - **low** if `score < 3`
- **Important behavior note:** In many switch implementations, rules are evaluated top-to-bottom and the first match wins; with this ordering, `>=9` should route to high before it can match `>=3`.
- **Connections:**
  - high → `High score`
  - medium → `Medium score`
  - low → `Low score`
- **Failure modes / edge cases:**
  - `score` null/undefined could route to fallback (configured fallback output `2`), but nothing is connected to fallback—this could silently drop items. Consider adding a fallback route to rejection.

#### Node: High score
- **Type / role:** `Set` — assigns acceptance reason.
- **Configuration:** currently sets `reason = "Medium score"` (this appears to be a configuration mistake; should likely be `"High score"`).
- **Connections:** → `Add to Accepted`

#### Node: Medium score
- **Type / role:** `Set` — assigns rejection reason.
- **Configuration:** `reason = "Medium score"`
- **Connections:** → `Add to Rejected`

#### Node: Low score
- **Type / role:** `Set` — assigns rejection reason.
- **Configuration:** `reason = "Low score"`
- **Connections:** → `Add to Rejected`

#### Node: Not enough credits for scoring
- **Type / role:** `Set` — assigns rejection reason.
- **Configuration:** `reason = "Not enough credits for scoring"`
- **Connections:** → `Add to Rejected`

---

### Block 4 — Output: Google Sheets logging + HubSpot sync
**Overview:** Writes accepted/rejected decisions to Google Sheets; accepted leads also create a HubSpot contact.  
**Nodes involved:** `Add to Accepted`, `Create Hubspot contact`, `Add to Rejected`, `Add to Rejected not validated`

#### Node: Add to Accepted
- **Type / role:** `Google Sheets` — append or update in “Accepted” worksheet.
- **Operation:** `appendOrUpdate`
- **Matching column:** `ID` (Shopify customer ID)
- **Document:** “ZeroBounce Validation” spreadsheet
- **Sheet:** “Accepted”
- **Data mapping highlights:**
  - Uses Shopify fields: `id`, `email`, `first_name`, `last_name`
  - Uses validation fields: `domain`, `status`, `sub_status`, `mx_found`, etc.
  - Uses scoring if present: `{{ $node["Score email"] ? $('Score email').item.json.score : "" }}`
  - Uses workflow decision: `Accepted Reason = {{$json.reason}}`
  - Timestamp: `Accepted At = {{$now}}`
- **Connections:** → `Create Hubspot contact`
- **Failure modes / edge cases:**
  - Google OAuth expired
  - Sheet schema mismatch (missing headers or renamed columns)
  - If `addresses[0]` doesn’t exist, HubSpot node may fail later (see below)

#### Node: Create Hubspot contact
- **Type / role:** `HubSpot` — creates a contact (CRM sync).
- **Authentication:** OAuth2
- **Configuration:**
  - Email: `{{ $('Validate email').item.json.address }}`
  - Additional fields:
    - `firstName` / `lastName` from Shopify customer
    - `countryRegionCode = {{ $('Shopify customer created').item.json.addresses[0].country_code }}`
- **Connections:** none (end node)
- **Failure modes / edge cases:**
  - HubSpot OAuth expired / missing scopes
  - Contact already exists: depending on node behavior/version, create may error or upsert; verify expected behavior in your HubSpot node configuration/version
  - `addresses[0]` can be missing → expression evaluation error. Consider guarding with a ternary (e.g., `addresses?.[0]?.country_code || ""`).

#### Node: Add to Rejected
- **Type / role:** `Google Sheets` — append or update in “Rejected” worksheet with full ZB diagnostics when available.
- **Operation:** `appendOrUpdate`
- **Matching column:** `ID`
- **Sheet:** “Rejected”
- **Mapping highlights:**
  - Includes `Reject Reason = {{$json.reason}}`
  - Includes ZeroBounce validation diagnostics
  - Includes ZB Score if available (guarded similarly)
- **Failure modes / edge cases:**
  - If validation did not run, references to `Validate email` may fail; this node uses expressions like `$('Validate email').item.json...` frequently, which can be unsafe if reached from “not validated” paths (this is why the workflow uses a separate node below).

#### Node: Add to Rejected not validated
- **Type / role:** `Google Sheets` — logs rejections when validation did not happen (missing email / insufficient validation credits).
- **Operation:** `appendOrUpdate`
- **Matching column:** `ID`
- **Sheet:** “Rejected”
- **Mapping:** Only Shopify basics + `Reject Reason`, avoids ZeroBounce fields.
- **Failure modes / edge cases:** Standard Google Sheets OAuth/schema issues.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Shopify customer created | Shopify Trigger | Entry point: fires on new Shopify customer creation | — | Has email? | ## ⚡ Trigger & Verification |
| Has email? | IF | Checks whether the Shopify event includes a non-empty email | Shopify customer created | Check credits for validation; Email missing | ## ⚡ Trigger & Verification |
| Check credits for validation | ZeroBounce | Ensures credits before validation API call | Has email? (true) | Validate email; Not enough credits for validation | ## ✔️ Stage 1: Email Validation |
| Validate email | ZeroBounce | Validates email deliverability and returns status + diagnostics | Check credits for validation | Status | ## ✔️ Stage 1: Email Validation |
| Status | Switch | Routes by ZeroBounce validation status (valid/catch-all/other) | Validate email | Valid; Check credits for scoring; Not valid | ## ✔️ Stage 1: Email Validation |
| Check credits for scoring | ZeroBounce | Ensures credits before AI scoring call | Status (Catch-all) | Score email; Not enough credits for scoring | ## 🎯 Stage 2: AI Email Scoring |
| Score email | ZeroBounce | AI scoring for catch-all emails | Check credits for scoring | Filter by score | ## 🎯 Stage 2: AI Email Scoring |
| Filter by score | Switch | Routes by score thresholds (high/medium/low) | Score email | High score; Medium score; Low score | ## 🎯 Stage 2: AI Email Scoring |
| Valid | Set | Sets acceptance reason “Valid” | Status (Valid) | Add to Accepted | ## 📤 Output Results |
| High score | Set | Sets acceptance reason (currently mis-set to “Medium score”) | Filter by score (high) | Add to Accepted | ## 📤 Output Results |
| Medium score | Set | Sets rejection reason “Medium score” | Filter by score (medium) | Add to Rejected | ## 📤 Output Results |
| Low score | Set | Sets rejection reason “Low score” | Filter by score (low) | Add to Rejected | ## 📤 Output Results |
| Not valid | Set | Sets rejection reason “Not valid” | Status (Other) | Add to Rejected | ## 📤 Output Results |
| Not enough credits for scoring | Set | Sets rejection reason “Not enough credits for scoring” | Check credits for scoring (error) | Add to Rejected | ## 📤 Output Results |
| Not enough credits for validation | Set | Sets rejection reason “Not enough credits for validation” | Check credits for validation (error) | Add to Rejected not validated | ## 📤 Output Results |
| Email missing | Set | Sets rejection reason “Email missing” | Has email? (false) | Add to Rejected not validated | ## 📤 Output Results |
| Add to Accepted | Google Sheets | Logs accepted leads to “Accepted” sheet (appendOrUpdate by ID) | Valid / High score | Create Hubspot contact | ## 📤 Output Results |
| Create Hubspot contact | HubSpot | Creates HubSpot contact for accepted leads | Add to Accepted | — | ## 📤 Output Results |
| Add to Rejected | Google Sheets | Logs rejected leads (with ZB diagnostics when available) | Not valid / Medium score / Low score / Not enough credits for scoring | — | ## 📤 Output Results |
| Add to Rejected not validated | Google Sheets | Logs rejected leads when validation didn’t run | Email missing / Not enough credits for validation | — | ## 📤 Output Results |
| Sticky Note2 | Sticky Note | Documentation / description | — | — | ## Shopify to HubSpot: Advanced ZeroBounce Lead Vetting **This workflow automates the transition of new Shopify customers into HubSpot, using [ZeroBounce](https://www.zerobounce.net) for multi-layer validation (Validation + AI Scoring).** **Results are also output to Google Sheets for review.** |
| Sticky Note3 | Sticky Note | Section label | — | — | ## ⚡ Trigger & Verification |
| Sticky Note4 | Sticky Note | Section label | — | — | ## ✔️ Stage 1: Email Validation |
| Sticky Note5 | Sticky Note | Section label | — | — | ## 🎯 Stage 2: AI Email Scoring |
| Sticky Note6 | Sticky Note | Section label | — | — | ## 📤 Output Results |
| Sticky Note | Sticky Note | Branding image | — | — | ![ZeroBounce Logo](https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zerobounce-logo.svg) |
| Sticky Note7 | Sticky Note | Setup requirements + sheet headers | — | — | ### 📋 Setup Requirements - **Shopify:** Connect via OAuth2 to watch for "Customer Created" events (Topic: `customers/create`). - **ZeroBounce:** Connect via API Key. *[Create one here](https://www.zerobounce.net/members/API)*. - **HubSpot:** Connect via OAuth2 to create/update contacts for high-quality leads. - **Google Sheets:** Connect via OAuth2 to append/update rows in a Google Sheets worksheet. Alternatively, swap these nodes out with any other data storage node e.g. **n8n Data Table** or **Microsoft Excel**. The sheets/tables can be created with the headers: - *Accepted* columns: `ID,Email,First Name,Last Name,Accepted At,Accepted Reason,ZB Status,ZB Sub Status,ZB Free Email,ZB Did You Mean,ZB Account,ZB Domain,ZB Domain Age Days,ZB SMTP Provider,ZB MX Found,ZB MX Record,ZB Score` - *Rejected* columns `ID,Email,First Name,Last Name,Rejected At,Rejected Reason,ZB Status,ZB Sub Status,ZB Free Email,ZB Did You Mean,ZB Account,ZB Domain,ZB Domain Age Days,ZB SMTP Provider,ZB MX Found,ZB MX Record,ZB Score` |
| Sticky Note8 | Sticky Note | Benefits summary | — | — | ### 💡 Key Benefits - **✨ Zero-Waste Sync:** Only "Valid" or "High-Scoring" leads reach your CRM. - **🛡️ Credit Safety:** Internal checks ensure you never trigger an API call without credits. - **📊 Detailed Suppressions:** Every rejected lead is categorized by reason (e.g. Email Missing, Invalid, Low Score, or Insufficient credits). |
| Sticky Note9 | Sticky Note | Integrations list | — | — | ### 🧩 Nodes used in this workflow - [ZeroBounce](https://n8n.io/integrations/zerobounce) - [Shopify](https://n8n.io/integrations/shopify) - [Hubspot](https://n8n.io/integrations/hubspot) - [Google Sheets](https://n8n.io/integrations/google-sheets) (or alternative e.g. [Data Table](https://n8n.io/integrations/data-table)) |
| Sticky Note10 | Sticky Note | How-it-works narrative | — | — | ### 🚀 How it Works 1. **⚡ Trigger & Verification:** Activates on new **Shopify** customers and first checks if an email address is provided. - **Email present:** Proceeds to credit check and validation - **Email missing:** Customer details are added to *Rejected* output for review with reason `'Email missing'`. 2. **💳 Credit Management:** Before each ZeroBounce call, the workflow checks your account for sufficient credits to prevent node failures. - **Success:** Proceeds to Stage 1 (Validation). - **Failure:** Customer details are added to *Rejected* output for retry with reason `'Not enough credits'`. 3. **✔️ Stage 1: Email Validation:** Validates the email address with ZeroBounce. - **Valid:** Adds the result to *Accepted* output and creates a contact in **HubSpot**. - **Invalid:** Logs to an **n8n Data Table** with the validation results and a reason for rejection for review. - **Catch-all:** Proceeds to Stage 2 (Scoring). 4. **🎯 Stage 2: AI Email Scoring:** For "Catch-all" emails, the workflow requests **ZeroBounce AI Scoring**. - **High Score (>=9):** Syncs to **HubSpot**. - **Medium/Low Score:** Suppressed and added to an **n8n Data Table** with the assigned score for review. 5. **📤 Output Results:** - **Accepted:** Output validation and scoring results to *Accepted* and **Hubspot** - **Rejected** Output validation and scoring results to *Rejected*. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger: Shopify**
   1. Add node: **Shopify Trigger**
   2. Event/Topic: `customers/create`
   3. Authentication: **OAuth2**
   4. Connect Shopify OAuth2 credentials (App + scopes needed for customers webhooks/events)
   5. Save to generate webhook

2) **Add email presence gate**
   1. Add node: **IF**
   2. Condition: `email` → **not empty**
      - Expression: `{{$json.email}}`
   3. Connect: `Shopify customer created` → `Has email?`

3) **Stage 1 credit check (validation)**
   1. Add node: **ZeroBounce**
   2. Resource: **Account**
   3. Option: **Credits required = 1**
   4. Enable **On Error: Continue (send to error output)** (called `continueErrorOutput` in JSON)
   5. Connect: `Has email? (true)` → `Check credits for validation`

4) **Validate email**
   1. Add node: **ZeroBounce**
   2. Operation: **Validate Email** (default validation)
   3. Email field expression: `{{ $('Shopify customer created').item.json.email }}`
   4. Options: Timeout `10` seconds
   5. Connect: `Check credits for validation (success)` → `Validate email`

5) **Handle insufficient credits for validation**
   1. Add node: **Set** named `Not enough credits for validation`
   2. Set field `reason` to `"Not enough credits for validation"`
   3. Connect: `Check credits for validation (error output)` → this Set node

6) **Route by validation status**
   1. Add node: **Switch** named `Status`
   2. Add rules:
      - Output “Valid” if `{{$json.status}}` equals `valid`
      - Output “Catch-all” if `{{$json.status}}` equals `catch-all`
      - Fallback output name: `Other`
   3. Connect: `Validate email` → `Status`

7) **Valid path**
   1. Add node: **Set** named `Valid`, set `reason = "Valid"`
   2. Connect: `Status (Valid)` → `Valid`

8) **Non-valid path**
   1. Add node: **Set** named `Not valid`, set `reason = "Not valid"`
   2. Connect: `Status (Other)` → `Not valid`

9) **Catch-all path: credit check (scoring)**
   1. Add node: **ZeroBounce** named `Check credits for scoring`
   2. Resource: **Account**
   3. Credits required: `1`
   4. Enable **continue on error output**
   5. Connect: `Status (Catch-all)` → `Check credits for scoring`

10) **AI Scoring**
   1. Add node: **ZeroBounce** named `Score email`
   2. Resource/operation: **Scoring**
   3. Email expression: `{{ $('Shopify customer created').item.json.email }}`
   4. Connect: `Check credits for scoring (success)` → `Score email`

11) **Handle insufficient credits for scoring**
   1. Add node: **Set** named `Not enough credits for scoring`
   2. Set `reason = "Not enough credits for scoring"`
   3. Connect: `Check credits for scoring (error output)` → this Set node

12) **Score threshold routing**
   1. Add node: **Switch** named `Filter by score`
   2. Rules (in this order):
      - high: `{{$json.score}} >= 9`
      - medium: `{{$json.score}} >= 3`
      - low: `{{$json.score}} < 3`
   3. Connect: `Score email` → `Filter by score`

13) **Create reason markers for each score band**
   1. Add **Set** `High score` with `reason = "High score"` (recommended fix; the provided workflow sets `"Medium score"` by mistake)
   2. Add **Set** `Medium score` with `reason = "Medium score"`
   3. Add **Set** `Low score` with `reason = "Low score"`
   4. Connect:
      - `Filter by score (high)` → `High score`
      - `Filter by score (medium)` → `Medium score`
      - `Filter by score (low)` → `Low score`

14) **Rejected because missing email**
   1. Add **Set** `Email missing` with `reason = "Email missing"`
   2. Connect: `Has email? (false)` → `Email missing`

15) **Google Sheets: Accepted logging**
   1. Add node: **Google Sheets** named `Add to Accepted`
   2. Auth: **Google Sheets OAuth2**
   3. Operation: **Append or Update**
   4. Spreadsheet: create/select “ZeroBounce Validation”
   5. Sheet: create/select “Accepted”
   6. Matching column: `ID`
   7. Map columns (at minimum): ID, Email, First/Last Name, Accepted At, Accepted Reason, ZB Status, ZB Score
      - Use `{{$now}}` for timestamps
      - Use `{{$json.reason}}` for decision reason
      - Use ZeroBounce fields from `Validate email`
      - For score, guard with: `{{ $node["Score email"] ? $('Score email').item.json.score : "" }}`
   8. Connect: `Valid` → `Add to Accepted` and `High score` → `Add to Accepted`

16) **HubSpot: create contact**
   1. Add node: **HubSpot** named `Create Hubspot contact`
   2. Auth: **HubSpot OAuth2**
   3. Operation: **Create Contact**
   4. Email: `{{ $('Validate email').item.json.address }}`
   5. Additional fields: firstName/lastName from Shopify; optionally country code
      - Consider safe expression for country: `{{ $('Shopify customer created').item.json.addresses?.[0]?.country_code || "" }}`
   6. Connect: `Add to Accepted` → `Create Hubspot contact`

17) **Google Sheets: Rejected logging (validated/scored paths)**
   1. Add node: **Google Sheets** named `Add to Rejected`
   2. Operation: **Append or Update**, match on `ID`
   3. Sheet: “Rejected”
   4. Map: ID, Email, First/Last Name, Rejected At (`{{$now}}`), Reject Reason (`{{$json.reason}}`) + ZeroBounce diagnostics + optional score.
   5. Connect: `Not valid` / `Medium score` / `Low score` / `Not enough credits for scoring` → `Add to Rejected`

18) **Google Sheets: Rejected logging (not validated paths)**
   1. Add node: **Google Sheets** named `Add to Rejected not validated`
   2. Operation: **Append or Update**, match on `ID`
   3. Sheet: “Rejected”
   4. Map only fields that exist without validation (ID, Email, names, timestamp, reason)
   5. Connect:
      - `Email missing` → `Add to Rejected not validated`
      - `Not enough credits for validation` → `Add to Rejected not validated`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ZeroBounce website | https://www.zerobounce.net |
| ZeroBounce API key creation | https://www.zerobounce.net/members/API |
| ZeroBounce n8n integration | https://n8n.io/integrations/zerobounce |
| Shopify n8n integration | https://n8n.io/integrations/shopify |
| HubSpot n8n integration | https://n8n.io/integrations/hubspot |
| Google Sheets n8n integration | https://n8n.io/integrations/google-sheets |
| Alternative storage: Data Table | https://n8n.io/integrations/data-table |
| Branding image used in canvas | https://raw.githubusercontent.com/zerobounce/n8n-nodes-zerobounce/main/icons/zerobounce-logo.svg |

**Notable implementation observations (actionable):**
- The `High score` Set node’s reason is currently **misconfigured** to `"Medium score"`; change it to `"High score"` to keep logs accurate.
- `Filter by score` has an unconnected fallback output; consider routing fallback to rejection to avoid silent drops.
- HubSpot country code expression can fail if `addresses[0]` is missing; use a safe/optional expression.