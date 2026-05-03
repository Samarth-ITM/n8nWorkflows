Send WhatsApp follow-up when a new lead submits a form with MoltFlow and log to Sheets

https://n8nworkflows.xyz/workflows/send-whatsapp-follow-up-when-a-new-lead-submits-a-form-with-moltflow-and-log-to-sheets-13480


# Send WhatsApp follow-up when a new lead submits a form with MoltFlow and log to Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow triggers on a form submission (any tool that can POST to a webhook), extracts lead details (name, phone, email, interest), sends an immediate personalized WhatsApp follow-up via **MoltFlow (WaiFlow API v2)**, and then logs the lead to **Google Sheets** for CRM tracking.

**Target use cases:**
- Lead generation forms (Typeform, Jotform, Google Forms, custom forms)
- Instant WhatsApp outreach to improve conversion
- Simple CRM logging in Sheets

### 1.1 Input Reception (Webhook Trigger)
Receives the form submission payload via HTTP POST.

### 1.2 Data Normalization & Message Preparation (Code)
Maps common field names to standardized fields, sanitizes phone numbers, and builds the WhatsApp message. Marks invalid submissions as `skip`.

### 1.3 Validation / Routing (IF)
Allows only valid leads (non-skipped) to continue to WhatsApp + logging.

### 1.4 WhatsApp Sending (HTTP Request to MoltFlow)
Sends a WhatsApp message using MoltFlow’s API with header-based authentication.

### 1.5 CRM Logging (Google Sheets Append)
Appends the lead data into a “Leads” sheet, including a “WhatsApp Sent” status.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Webhook Trigger)

**Overview:**  
Accepts inbound form submissions via a webhook endpoint (`POST /form-lead`). This is the single entry point for the workflow.

**Nodes involved:**
- Form Submission Webhook

#### Node: Form Submission Webhook
- **Type / Role:** `Webhook` — workflow trigger and HTTP endpoint.
- **Key configuration (interpreted):**
  - HTTP Method: `POST`
  - Path: `form-lead` (full URL depends on your n8n instance)
- **Input / Output:**
  - **Input:** external HTTP POST request (form submission payload)
  - **Output:** passes the received request data to “Parse Form Data”
- **Version-specific notes:** Node typeVersion `2` (standard webhook behavior).
- **Edge cases / failures:**
  - Incorrect method (GET instead of POST) → no trigger
  - Form tool not sending JSON or sending unexpected structure (affects next Code node parsing)
  - Large payloads/timeouts depending on n8n hosting limits

---

### Block 2 — Data Normalization & Message Preparation (Code)

**Overview:**  
Transforms raw form payload into normalized lead fields, creates a personalized WhatsApp message, and produces a `skip` flag when the phone number is missing/invalid.

**Nodes involved:**
- Parse Form Data

#### Node: Parse Form Data
- **Type / Role:** `Code` — data parsing, mapping, validation, message templating.
- **Key configuration choices:**
  - Mode: `runOnceForAllItems` (expects a single incoming webhook event; uses the first item)
  - Hardcoded constant: `SESSION_ID = 'YOUR_SESSION_ID'` (must be replaced)
  - Message template with placeholders:
    - `Hi {{name}}!... about {{interest}} ...`
- **Key logic / variables:**
  - Payload source:
    - `const body = $input.first().json.body || $input.first().json;`
    - Supports webhook payloads either directly in JSON or nested in `body`.
  - Field mapping (fallbacks):
    - `name`: `body.name || body.full_name || body.first_name || 'there'`
    - `phone`: `body.phone || body.phone_number || body.mobile || ''` then sanitized to digits only
    - `email`: `body.email || ''`
    - `interest`: `body.interest || body.subject || body.service || body.message || 'your inquiry'`
  - Phone validation:
    - If no phone or `< 7` digits → output `{ skip: true, reason: 'No valid phone number', ... }`
  - WhatsApp identifiers:
    - `chat_id = phone + '@c.us'` (WhatsApp-style chat ID used by MoltFlow)
  - Output fields include:
    - `session_id`, `chat_id`, `message`, `name`, `phone`, `email`, `interest`, `submitted_at`, `skip`
- **Input / Output:**
  - **Input:** Webhook JSON
  - **Output:** Single normalized item to “Valid Lead?”
- **Version-specific notes:** Node typeVersion `2`.
- **Edge cases / failures:**
  - If the incoming webhook payload is not JSON (e.g., form posts URL-encoded) and n8n doesn’t parse it as expected, `body.*` fields may be undefined → results in `skip: true`
  - `SESSION_ID` left as `YOUR_SESSION_ID` → MoltFlow API likely rejects requests (auth/session error)
  - Phone normalization removes `+` and formatting; if your provider needs E.164 with country code, you must ensure forms collect country code or add logic to prefix it

---

### Block 3 — Validation / Routing (IF)

**Overview:**  
Routes execution only for valid leads. If `skip` is true, the workflow stops on the false branch (currently unused).

**Nodes involved:**
- Valid Lead?

#### Node: Valid Lead?
- **Type / Role:** `IF` — conditional branching.
- **Configuration (interpreted):**
  - Condition: Boolean equals
  - Expression: `{{ $json.skip }}` must equal `false`
- **Input / Output:**
  - **Input:** Output of “Parse Form Data”
  - **True output:** to “Send WhatsApp Follow-up”
  - **False output:** not connected (workflow ends for invalid leads)
- **Version-specific notes:** Node typeVersion `2`.
- **Edge cases / failures:**
  - If `skip` is missing (unexpected code output), strict boolean comparison may fail; ensure Code node always outputs `skip`
  - No handling/logging for skipped leads (could be added to log invalid submissions)

---

### Block 4 — WhatsApp Sending (HTTP Request to MoltFlow)

**Overview:**  
Sends a WhatsApp follow-up message using MoltFlow’s API endpoint with header-based API key authentication.

**Nodes involved:**
- Send WhatsApp Follow-up

#### Node: Send WhatsApp Follow-up
- **Type / Role:** `HTTP Request` — calls MoltFlow (WaiFlow) API v2.
- **Configuration choices (interpreted):**
  - Method: `POST`
  - URL: `https://apiv2.waiflow.app/api/v2/messages/send`
  - Authentication: Generic Credential → `httpHeaderAuth`
    - Credential name: “MoltFlow API Key”
    - Expected header (per sticky note): `X-API-Key: <your_key>`
  - Body:
    - Sent as JSON (n8n “specifyBody: json”, “sendBody: true”)
    - Expression builds payload:
      - `session_id: $json.session_id`
      - `chat_id: $json.chat_id`
      - `message: $json.message`
    - Note: It is explicitly stringified: `JSON.stringify({...})`
- **Input / Output:**
  - **Input:** Validated lead item from IF true branch
  - **Output:** Response passed to “Log Lead to Sheet”
- **Version-specific notes:** Node typeVersion `4.2`.
- **Edge cases / failures:**
  - Missing/incorrect API key header → 401/403 from MoltFlow
  - Wrong/expired `session_id` → message send fails
  - Invalid `chat_id` formatting or missing country code → message not delivered
  - API downtime / rate limiting → request errors; consider adding retries or error workflow
  - Body formatting: since it is `JSON.stringify`, ensure the node is truly sending it as JSON; if MoltFlow expects an object and n8n sends a string, remove `JSON.stringify(...)` and pass the object directly

---

### Block 5 — CRM Logging (Google Sheets Append)

**Overview:**  
Appends the lead details into Google Sheets for tracking, including a “WhatsApp Sent” indicator.

**Nodes involved:**
- Log Lead to Sheet

#### Node: Log Lead to Sheet
- **Type / Role:** `Google Sheets` — append a new row to a sheet.
- **Configuration choices (interpreted):**
  - Operation: `append`
  - Document: `YOUR_GOOGLE_SHEET_URL` (must be replaced with real spreadsheet)
  - Sheet name: `Leads`
  - Column mapping: “defineBelow” with explicit field mapping:
    - Date → `$('Parse Form Data').first().json.submitted_at`
    - Name → `$('Parse Form Data').first().json.name`
    - Email → `$('Parse Form Data').first().json.email`
    - Phone → `$('Parse Form Data').first().json.phone`
    - Interest → `$('Parse Form Data').first().json.interest`
    - WhatsApp Sent → `Yes`
- **Input / Output:**
  - **Input:** Output from “Send WhatsApp Follow-up”
  - **Output:** final node (no outgoing connections)
- **Version-specific notes:** Node typeVersion `4.5`.
- **Edge cases / failures:**
  - OAuth not connected or expired → auth errors
  - Sheet name “Leads” not found → append fails
  - Column headers must match exactly (“Date”, “Name”, etc.) depending on how the node maps columns
  - Currently logs “WhatsApp Sent: Yes” even if the WhatsApp API call returned an error but didn’t stop execution (depends on n8n error settings). For stricter accuracy, map from HTTP response status or enable “Continue On Fail” logic explicitly.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / workflow description | — | — | ## Form → WhatsApp Lead Follow-up<br>Instantly reach new leads on WhatsApp when they submit a form (Typeform, JotForm, Google Forms, or any webhook-enabled form) using [MoltFlow](https://molt.waiflow.app).<br><br>**How it works:**<br>1. A form submission triggers this webhook<br>2. Contact info is extracted (name, phone, interest)<br>3. A personalized WhatsApp message is sent<br>4. Lead is logged to Google Sheets for CRM tracking |
| Sticky Note1 | Sticky Note | Setup instructions | — | — | ## Setup (5 min)<br>1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp<br>2. Activate this workflow — copy the webhook URL<br>3. Configure your form tool to POST to this webhook on submission<br>4. Map your form field names in the Parse Form node<br>5. Set `YOUR_SESSION_ID` in the Parse Form node<br>6. (Optional) Connect Google Sheets to log leads<br>7. Add MoltFlow API Key: Header Auth → `X-API-Key` |
| Form Submission Webhook | Webhook | Receives form submissions | — | Parse Form Data | ## Form → WhatsApp Lead Follow-up<br>Instantly reach new leads on WhatsApp when they submit a form (Typeform, JotForm, Google Forms, or any webhook-enabled form) using [MoltFlow](https://molt.waiflow.app).<br><br>**How it works:**<br>1. A form submission triggers this webhook<br>2. Contact info is extracted (name, phone, interest)<br>3. A personalized WhatsApp message is sent<br>4. Lead is logged to Google Sheets for CRM tracking |
| Parse Form Data | Code | Normalize payload, validate phone, build message | Form Submission Webhook | Valid Lead? | ## Setup (5 min)<br>1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp<br>2. Activate this workflow — copy the webhook URL<br>3. Configure your form tool to POST to this webhook on submission<br>4. Map your form field names in the Parse Form node<br>5. Set `YOUR_SESSION_ID` in the Parse Form node<br>6. (Optional) Connect Google Sheets to log leads<br>7. Add MoltFlow API Key: Header Auth → `X-API-Key` |
| Valid Lead? | IF | Gate: only proceed when `skip=false` | Parse Form Data | Send WhatsApp Follow-up (true) | ## Form → WhatsApp Lead Follow-up<br>Instantly reach new leads on WhatsApp when they submit a form (Typeform, JotForm, Google Forms, or any webhook-enabled form) using [MoltFlow](https://molt.waiflow.app).<br><br>**How it works:**<br>1. A form submission triggers this webhook<br>2. Contact info is extracted (name, phone, interest)<br>3. A personalized WhatsApp message is sent<br>4. Lead is logged to Google Sheets for CRM tracking |
| Send WhatsApp Follow-up | HTTP Request | Send WhatsApp message via MoltFlow API | Valid Lead? (true) | Log Lead to Sheet | ## Setup (5 min)<br>1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp<br>2. Activate this workflow — copy the webhook URL<br>3. Configure your form tool to POST to this webhook on submission<br>4. Map your form field names in the Parse Form node<br>5. Set `YOUR_SESSION_ID` in the Parse Form node<br>6. (Optional) Connect Google Sheets to log leads<br>7. Add MoltFlow API Key: Header Auth → `X-API-Key` |
| Log Lead to Sheet | Google Sheets | Append lead to Google Sheets | Send WhatsApp Follow-up | — | ## Setup (5 min)<br>1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp<br>2. Activate this workflow — copy the webhook URL<br>3. Configure your form tool to POST to this webhook on submission<br>4. Map your form field names in the Parse Form node<br>5. Set `YOUR_SESSION_ID` in the Parse Form node<br>6. (Optional) Connect Google Sheets to log leads<br>7. Add MoltFlow API Key: Header Auth → `X-API-Key` |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Send WhatsApp Follow-up When a New Lead Submits a Form with MoltFlow*

2. **Add node: Webhook**
   - Node name: **Form Submission Webhook**
   - HTTP Method: **POST**
   - Path: **form-lead**
   - Activate workflow later to obtain the Production webhook URL.

3. **Add node: Code**
   - Node name: **Parse Form Data**
   - Mode: **Run Once for All Items**
   - Paste logic that:
     - Sets `SESSION_ID` (replace with your MoltFlow session)
     - Reads payload from `$input.first().json.body || $input.first().json`
     - Maps `name`, `phone`, `email`, `interest`
     - Sanitizes phone to digits only
     - If invalid phone, outputs `{ skip: true, reason: ... }`
     - Otherwise outputs `{ session_id, chat_id: phone+'@c.us', message, ... , skip:false }`
   - **Important required change:** set `SESSION_ID` from `YOUR_SESSION_ID` to your real session id.

4. **Connect nodes**
   - Connect **Form Submission Webhook → Parse Form Data**

5. **Add node: IF**
   - Node name: **Valid Lead?**
   - Condition type: Boolean
   - Left value (expression): `{{ $json.skip }}`
   - Operation: `equals`
   - Right value: `false`

6. **Connect nodes**
   - Connect **Parse Form Data → Valid Lead?**

7. **Add node: HTTP Request**
   - Node name: **Send WhatsApp Follow-up**
   - Method: **POST**
   - URL: `https://apiv2.waiflow.app/api/v2/messages/send`
   - Authentication: **Header Auth** (generic HTTP header auth credential)
     - Create credential (name: **MoltFlow API Key**)
     - Header name: `X-API-Key`
     - Value: your MoltFlow API key
   - Body type: JSON
   - Body fields:
     - `session_id` = `{{ $json.session_id }}`
     - `chat_id` = `{{ $json.chat_id }}`
     - `message` = `{{ $json.message }}`
   - (If you reproduce exactly as provided, it uses `JSON.stringify(...)`; preferred is to send an object directly unless MoltFlow requires a string.)

8. **Connect nodes**
   - Connect **Valid Lead? (true output) → Send WhatsApp Follow-up**

9. **Add node: Google Sheets**
   - Node name: **Log Lead to Sheet**
   - Credentials: **Google Sheets OAuth2**
     - Connect the Google account that owns/has access to the target spreadsheet
   - Operation: **Append**
   - Document: select by URL and paste your Spreadsheet URL (replace `YOUR_GOOGLE_SHEET_URL`)
   - Sheet: **Leads** (create this tab if it doesn’t exist)
   - Map columns explicitly (ensure headers exist in the first row):
     - Date → `{{ $('Parse Form Data').first().json.submitted_at }}`
     - Name → `{{ $('Parse Form Data').first().json.name }}`
     - Email → `{{ $('Parse Form Data').first().json.email }}`
     - Phone → `{{ $('Parse Form Data').first().json.phone }}`
     - Interest → `{{ $('Parse Form Data').first().json.interest }}`
     - WhatsApp Sent → `Yes`

10. **Connect nodes**
   - Connect **Send WhatsApp Follow-up → Log Lead to Sheet**

11. **Activate and configure your form**
   - Activate the workflow and copy the **Production Webhook URL** from the Webhook node.
   - In your form provider, set the “on submission” webhook to **POST** to that URL.
   - Ensure your form sends fields that match (or adjust mapping in Code node):
     - `name/full_name/first_name`
     - `phone/phone_number/mobile`
     - `email`
     - `interest/subject/service/message`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| MoltFlow service referenced as the WhatsApp sending layer | https://molt.waiflow.app |
| WhatsApp message sending endpoint used | `https://apiv2.waiflow.app/api/v2/messages/send` |
| Authentication requirement stated in workflow notes | Header Auth with `X-API-Key` |
| Setup note: obtain webhook URL by activating workflow | Applies to Webhook node in n8n |
| Replace placeholders before production use | `YOUR_SESSION_ID`, `YOUR_GOOGLE_SHEET_URL` |

