Sync LinkedIn profile visitors into HubSpot CRM leads with ConnectSafely.ai and Apify

https://n8nworkflows.xyz/workflows/sync-linkedin-profile-visitors-into-hubspot-crm-leads-with-connectsafely-ai-and-apify-12309


# Sync LinkedIn profile visitors into HubSpot CRM leads with ConnectSafely.ai and Apify

## 1. Workflow Overview

**Purpose:** Automatically fetch LinkedIn profile visitors (last 7 days) via ConnectSafely.ai, enrich each visitor using an Apify actor (LinkedIn profile enrichment), then **create/update HubSpot contacts** only when an email address is present.

**Primary use cases:**
- Sales/recruiting teams converting LinkedIn visitors into CRM leads automatically
- Weekly lead capture with optional manual runs for testing
- Enrichment + CRM sync with simple eligibility gating (email exists)

### 1.1 Trigger & Execution Entry Points
- Weekly schedule trigger for production runs
- Manual trigger for testing (“Execute workflow”)

### 1.2 Fetch Visitors (ConnectSafely.ai)
- HTTP POST to ConnectSafely.ai endpoint to retrieve visitors (past 7 days, fetch all)
- Split the returned visitors array into individual items

### 1.3 Per-Visitor Loop + Enrichment (Apify)
- Loop over each visitor record (batching/iteration)
- Run an Apify actor per visitor using the visitor’s LinkedIn URL as input

### 1.4 Validation & HubSpot CRM Sync
- Check enrichment output for existence of email
- If eligible: create or update HubSpot contact with mapped profile fields
- If not eligible: skip (routed to a NoOp placeholder)

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Start
**Overview:** Starts the workflow either on a weekly schedule or manually for testing. Both entry points converge into the same HTTP fetch step.

**Nodes involved:**
- `Schedule Trigger`
- `When clicking ‘Execute workflow’`

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Config (interpreted):** Runs on an interval of **weeks** (weekly cadence).
- **Connections:** Outputs to `HTTP Request`.
- **Version notes:** TypeVersion `1.2`.
- **Edge cases / failures:**
- Schedule misconfiguration (timezone expectations, cadence not as intended).
- Workflow not active → schedule won’t run.

#### Node: When clicking ‘Execute workflow’
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual entry point for testing.
- **Config:** No parameters.
- **Connections:** Outputs to `HTTP Request`.
- **Edge cases:** None; only runs when manually executed in the editor.

---

### Block 2 — Fetch Visitors from ConnectSafely.ai & Split Array
**Overview:** Calls ConnectSafely.ai to retrieve LinkedIn profile visitors and splits the `visitors` array into individual items for downstream per-visitor processing.

**Nodes involved:**
- `HTTP Request`
- `📤 Split Comments Array`

#### Node: HTTP Request
- **Type / role:** `n8n-nodes-base.httpRequest` — calls ConnectSafely.ai API.
- **Config (interpreted):**
  - **Method:** `POST`
  - **URL:** `https://api.connectsafely.ai/linkedin/profile/visitors`
  - **Body (JSON):** `{"timeRange":"past_7_days","start":0,"fetchAll":true}`
  - **Auth:** Bearer token via **Generic Credential Type** (`httpBearerAuth`)
- **Key variables/expressions:** None in body; static JSON.
- **Connections:** Outputs to `📤 Split Comments Array`.
- **Credentials:** `Karan ConnectSafely AI` (Bearer token).
- **Version notes:** TypeVersion `4.2`.
- **Edge cases / failures:**
  - 401/403 if bearer token invalid/expired.
  - API rate limits / throttling.
  - Response shape changes (if `visitors` field missing/renamed, downstream split fails).
  - Large payloads when `fetchAll: true` (execution time/memory).

#### Node: 📤 Split Comments Array
- **Type / role:** `n8n-nodes-base.splitOut` — splits an array field into individual items.
- **Config (interpreted):**
  - **Field to split out:** `visitors`
  - Produces one n8n item per visitor.
- **Connections:** Outputs to `Loop Over Items`.
- **Version notes:** TypeVersion `1`.
- **Sticky note:** Contains “STEP 3: SPLIT DATA…” (see summary table for full text).
- **Edge cases / failures:**
  - If `visitors` is not an array (null/object), the node can error or output zero items.
  - If API returns empty array → downstream loop receives no items.

---

### Block 3 — Loop Per Visitor + Enrich via Apify
**Overview:** Iterates over each visitor item and enriches it by running an Apify actor, passing the visitor’s LinkedIn profile URL.

**Nodes involved:**
- `Loop Over Items`
- `Run an Actor and get dataset`

#### Node: Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — controls iteration/batching.
- **Config (interpreted):**
  - Uses default options (batch size not explicitly set in JSON).
  - In this workflow, the iteration path uses the **second output** (index 1) to proceed to enrichment.
- **Connections:**
  - Input from `📤 Split Comments Array`
  - Output (index 1) → `Run an Actor and get dataset`
  - Output (index 0) unused (commonly “done/no items” path)
- **Version notes:** TypeVersion `3`.
- **Edge cases / failures:**
  - If batch size default is too small/large, could impact runtime.
  - Misunderstanding outputs: if wired incorrectly, may never process items or may loop incorrectly.

#### Node: Run an Actor and get dataset
- **Type / role:** `@apify/n8n-nodes-apify.apify` — runs an Apify actor and returns its dataset results.
- **Config (interpreted):**
  - **Operation:** “Run actor and get dataset”
  - **Actor:** `UMdANQyqx3b2JVuxg` (configured via Apify Console URL reference)
  - **Custom body sent to actor:**
    - JSON with `linkedin` set from the current item: `{{ $json.navigationUrl }}`
- **Key expressions/variables:**
  - `{{ $json.navigationUrl }}` must exist on each visitor item from ConnectSafely.ai.
- **Connections:** Outputs to `Check if Contact is eligible to add into CRM`.
- **Credentials:** `Karan Apify Account`.
- **Version notes:** TypeVersion `1`.
- **Edge cases / failures:**
  - Apify token/permissions issues.
  - Actor runtime errors (LinkedIn page blocked, captcha, scraping limits).
  - Missing/invalid `navigationUrl` → actor input invalid → enrichment fails.
  - Dataset schema mismatch with later field mapping (expects `04_Email`, etc.).

---

### Block 4 — Validate Eligibility & HubSpot CRM Sync
**Overview:** Ensures enriched output includes an email address before creating/updating a HubSpot contact. If no email, execution is routed to a placeholder node.

**Nodes involved:**
- `Check if Contact is eligible to add into CRM`
- `Create or update a contact`
- `Replace Me`

#### Node: Check if Contact is eligible to add into CRM
- **Type / role:** `n8n-nodes-base.if` — conditional gate.
- **Config (interpreted):**
  - Condition: **string exists** for `{{ $json['04_Email'] }}`
  - If email exists → “true” branch; else → “false” branch
- **Key expressions/variables:**
  - `$json['04_Email']` (note the numeric prefix; must match Apify dataset field names exactly)
- **Connections:**
  - True → `Create or update a contact`
  - False → `Replace Me`
- **Version notes:** TypeVersion `2.2`.
- **Edge cases / failures:**
  - If Apify output doesn’t contain `04_Email` (different naming), all items go false.
  - If email exists but is malformed, HubSpot may reject or create bad data (no validation here beyond existence).

#### Node: Create or update a contact
- **Type / role:** `n8n-nodes-base.hubspot` — upsert contact in HubSpot.
- **Config (interpreted):**
  - **Auth:** HubSpot **app token**
  - **Email (primary key):** `{{ $json['04_Email'] }}`
  - **Additional fields mapped:**
    - First name: `02_First_name`
    - Last name: `03_Last_name`
    - Job title: `07_Title`
    - LinkedIn URL: `06_Linkedin_url`
    - Company name: `16_Company_name`
    - City: `14_City`
    - Country: `15_Country`
    - Street address: `13_Current_address`
    - Lead status: `NEW` (static)
- **Connections:** Outputs to `Replace Me`.
- **Credentials:** `HubSpot App Token account`.
- **Version notes:** TypeVersion `2.1`.
- **Edge cases / failures:**
  - 401/403 if app token invalid or missing scopes (contacts write).
  - Property name mismatches in HubSpot portal (e.g., `linkedinUrl`, `leadStatus`) if not available → HubSpot API error.
  - Rate limiting when processing many visitors.
  - Duplicate handling: “create or update” depends on HubSpot matching by email; if email reused/shared, may overwrite.

#### Node: Replace Me
- **Type / role:** `n8n-nodes-base.noOp` — placeholder/sink node.
- **Config:** No settings.
- **Connections:** Outputs to `Loop Over Items` (index 0 input), acting as the “continue loop” step after either branch.
- **Purpose in this workflow:** Provides a convergence point after HubSpot sync or skip, then triggers the next loop iteration.
- **Version notes:** TypeVersion `1`.
- **Edge cases / failures:**
  - If removed without rewiring, the loop may not continue properly.
  - Naming suggests it may be intended to be replaced with logging, notifications, or metrics.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Weekly automated entry point | — | HTTP Request | ## 1. Trigger<br>Runs weekly on schedule or manually for testing. |
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Manual entry point for testing | — | HTTP Request | ## 1. Trigger<br>Runs weekly on schedule or manually for testing. |
| HTTP Request | n8n-nodes-base.httpRequest | Fetch LinkedIn visitors from ConnectSafely.ai | Schedule Trigger; When clicking ‘Execute workflow’ | 📤 Split Comments Array | ## 2. Fetch Visitors<br>Retrieves LinkedIn profile visitors via ConnectSafely.ai API and splits into individual records. |
| 📤 Split Comments Array | n8n-nodes-base.splitOut | Split `visitors` array into individual visitor items | HTTP Request | Loop Over Items | 📝 STEP 3: SPLIT DATA<br><br>✅ What this does:<br>- Takes the array of comments from previous step<br>- Splits each comment into a separate item<br>- Prepares data for loop processing<br><br>🔧 Technical Details:<br>- Splits 'comments' field into individual items<br>- Each item becomes a separate execution flow<br>- Enables processing one commenter at a time<br><br>💡 Example:<br>Input: {comments: [user1, user2, user3]}<br>Output: 3 separate items, one per user |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterate over visitors (batch/loop control) | 📤 Split Comments Array; Replace Me | Run an Actor and get dataset (output index 1) | ## 2. Fetch Visitors<br>Retrieves LinkedIn profile visitors via ConnectSafely.ai API and splits into individual records. |
| Run an Actor and get dataset | @apify/n8n-nodes-apify.apify | Enrich visitor via Apify actor and return dataset | Loop Over Items | Check if Contact is eligible to add into CRM | ## 3. Enrich and Validate<br>Enriches visitor data via Apify and validates email exists before CRM sync. |
| Check if Contact is eligible to add into CRM | n8n-nodes-base.if | Gate: only sync if email exists | Run an Actor and get dataset | Create or update a contact (true); Replace Me (false) | ## 3. Enrich and Validate<br>Enriches visitor data via Apify and validates email exists before CRM sync. |
| Create or update a contact | n8n-nodes-base.hubspot | Upsert HubSpot contact from enriched fields | Check if Contact is eligible to add into CRM | Replace Me | ## 4. CRM Sync<br>Creates or updates contact in HubSpot with enriched profile data. |
| Replace Me | n8n-nodes-base.noOp | Placeholder convergence + loop continuation | Create or update a contact; Check if Contact is eligible to add into CRM (false) | Loop Over Items | ## 4. CRM Sync<br>Creates or updates contact in HubSpot with enriched profile data. |
| Workflow Overview | n8n-nodes-base.stickyNote | Documentation / context | — | — | @[youtube](pWa6g_ln1pY)<br><br>## Sync LinkedIn Profile Visitors to HubSpot CRM<br><br>Automatically capture LinkedIn profile visitors and add them as contacts in HubSpot CRM with enriched data.<br><br>### Who is this for?<br>Sales professionals, recruiters, and marketers who want to convert LinkedIn profile visitors into actionable CRM leads without manual data entry.<br><br>### How it works<br>1. **Scheduled trigger** runs weekly (or manually) to fetch recent profile visitors<br>2. **ConnectSafely.ai API** retrieves visitors from the past 7 days<br>3. **Apify actor** enriches each visitor with detailed LinkedIn profile data<br>4. **Email validation** filters contacts with valid email addresses<br>5. **HubSpot integration** creates or updates contacts with full profile information<br><br>### Setup steps<br>1. Create a [ConnectSafely.ai](https://connectsafely.ai) account and get your API key<br>2. Set up an [Apify](https://apify.com) account for LinkedIn profile enrichment<br>3. Configure HubSpot credentials with contact creation permissions<br>4. Update the HTTP Request node with your ConnectSafely.ai Bearer token<br>5. Configure the Apify node with your actor ID for LinkedIn scraping<br><br>### Customization<br>- Adjust the schedule trigger frequency based on your needs<br>- Modify the time range in the API request (past_7_days, past_30_days)<br>- Add additional CRM fields in the HubSpot node<br>- Replace HubSpot with your preferred CRM (Salesforce, Zoho, Pipedrive) |
| Section 1 - Trigger | n8n-nodes-base.stickyNote | Section label | — | — | (This note labels the Trigger block.) |
| Section 2 - Fetch Visitors | n8n-nodes-base.stickyNote | Section label | — | — | (This note labels the Fetch Visitors block.) |
| Section 3 - Enrich and Validate | n8n-nodes-base.stickyNote | Section label | — | — | (This note labels the Enrich/Validate block.) |
| Section 4 - CRM Sync | n8n-nodes-base.stickyNote | Section label | — | — | (This note labels the CRM Sync block.) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger nodes**
   1. Add **Schedule Trigger**
      - Interval: **Every 1 week** (weekly).
   2. Add **Manual Trigger** (name it “When clicking ‘Execute workflow’”).

2) **Add ConnectSafely.ai fetch node**
   1. Add **HTTP Request** node.
      - Method: `POST`
      - URL: `https://api.connectsafely.ai/linkedin/profile/visitors`
      - Body Content Type: JSON
      - Body:
        - `timeRange`: `past_7_days`
        - `start`: `0`
        - `fetchAll`: `true`
      - Authentication: **Bearer Token**
        - Create credential: **HTTP Bearer Auth**
        - Paste your ConnectSafely.ai API key/token.
   2. Connect:
      - `Schedule Trigger` → `HTTP Request`
      - `Manual Trigger` → `HTTP Request`

3) **Split visitors array**
   1. Add **Split Out** node (name: “📤 Split Comments Array” if desired).
      - Field to split out: `visitors`
   2. Connect: `HTTP Request` → `Split Out`

4) **Iterate over visitors**
   1. Add **Loop Over Items / Split In Batches** node.
      - Keep defaults (or set a batch size if you expect large volume).
   2. Connect: `Split Out` → `Split In Batches`

5) **Enrich each visitor with Apify**
   1. Add **Apify** node with operation **Run actor and get dataset**.
      - Actor: select by Actor ID or paste actor console URL for `UMdANQyqx3b2JVuxg`
      - Input/body to actor (custom JSON):
        - `linkedin`: set via expression to the visitor URL field:
          - `{{ $json.navigationUrl }}`
      - Credentials:
        - Create/select **Apify API** credential (API token from Apify).
   2. Connect:
      - From `Split In Batches` **output 1** (the “items” path) → `Apify`

6) **Eligibility check (email exists)**
   1. Add **IF** node (name: “Check if Contact is eligible to add into CRM”).
      - Condition: String → **exists**
      - Left value expression: `{{ $json['04_Email'] }}`
   2. Connect: `Apify` → `IF`

7) **HubSpot upsert**
   1. Add **HubSpot** node: “Create or update a contact”.
      - Authentication: **App Token**
        - Create/select HubSpot credential with an app token that can write contacts.
      - Email field: `{{ $json['04_Email'] }}`
      - Additional fields mapping:
        - First Name: `{{ $json['02_First_name'] }}`
        - Last Name: `{{ $json['03_Last_name'] }}`
        - Job Title: `{{ $json['07_Title'] }}`
        - LinkedIn URL: `{{ $json['06_Linkedin_url'] }}`
        - Company Name: `{{ $json['16_Company_name'] }}`
        - City: `{{ $json['14_City'] }}`
        - Country: `{{ $json['15_Country'] }}`
        - Street Address: `{{ $json['13_Current_address'] }}`
        - Lead Status: `NEW`
   2. Connect:
      - `IF` (true output) → `HubSpot`

8) **Add convergence placeholder and continue loop**
   1. Add **No Operation (NoOp)** node named `Replace Me`.
   2. Connect:
      - `IF` (false output) → `Replace Me`
      - `HubSpot` → `Replace Me`
      - `Replace Me` → `Split In Batches` (to continue/advance the loop)

9) **(Optional) Add sticky notes**
   - Add sticky notes for the four sections and the overall overview content (including links).

**Important constraint:** The Apify actor output must contain the fields referenced in the IF and HubSpot nodes (e.g., `04_Email`, `02_First_name`). If your actor returns different keys, update the expressions accordingly.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| YouTube reference: `@[youtube](pWa6g_ln1pY)` | Workflow Overview sticky note |
| ConnectSafely.ai account + API key required | https://connectsafely.ai |
| Apify account + actor configuration required | https://apify.com |
| HubSpot credentials must allow contact create/update | HubSpot private app token / app token credential in n8n |