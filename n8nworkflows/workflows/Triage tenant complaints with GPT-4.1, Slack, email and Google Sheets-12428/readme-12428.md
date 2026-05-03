Triage tenant complaints with GPT-4.1, Slack, email and Google Sheets

https://n8nworkflows.xyz/workflows/triage-tenant-complaints-with-gpt-4-1--slack--email-and-google-sheets-12428


# Triage tenant complaints with GPT-4.1, Slack, email and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Triage tenant complaints with GPT-4.1, Slack, email and Google Sheets  
**Workflow name (JSON):** Automated Tenant Complaint Management and Escalation System

This workflow receives tenant complaints via an HTTP webhook, normalizes the incoming fields, uses an OpenAI chat model (GPT‑4.1 mini) through LangChain Agent nodes to classify urgency/category, routes complaints by urgency, notifies stakeholders (Slack for High, email acknowledgment for Medium/Low), then consolidates outcomes and logs the complaint into Google Sheets. Finally, it generates resolution recommendations using AI.

### 1.1 Input Reception
Receives complaint submissions from a tenant form/portal via webhook.

### 1.2 Configuration & Data Normalization
Injects environment-like configuration (Slack channel, manager email, Google Sheet identifiers) and maps webhook payload into consistent fields.

### 1.3 AI Classification (Urgency + Category)
Uses GPT model via LangChain Agent to produce a structured text classification (Urgency/Category/Reason).

### 1.4 Routing & Notifications
Routes based on detected urgency string:
- **High:** Slack notification to property manager channel
- **Medium/Low:** Generate and send acknowledgment email to tenant  
Additionally, **Medium** branch creates task metadata (but is not connected to the merge/logging in this JSON—see edge cases).

### 1.5 Consolidation, Logging & Resolution Suggestions
Merges branches (as wired), logs to Google Sheets (append or update), then generates recommended resolution steps via AI.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input Reception
**Overview:** Accepts complaint submissions via an HTTP POST endpoint and starts the workflow execution.  
**Nodes involved:** Tenant Complaint Webhook

#### Node: Tenant Complaint Webhook
- **Type / role:** `n8n-nodes-base.webhook` — Entry trigger receiving external HTTP requests.
- **Key configuration:**
  - **Method:** POST
  - **Path:** `tenant-complaint` (endpoint becomes `/webhook/tenant-complaint` or `/webhook-test/tenant-complaint` depending on mode)
  - **Response:** `lastNode` (responds with the output of the last executed node in the run)
- **Expected input:** JSON body with fields like `name`, `unit`, `complaintType`, `description`, `timestamp`, `email` under `body`.
- **Outputs:** Passes request data to **Workflow Configuration**.
- **Edge cases / failures:**
  - Missing/invalid body fields will later cause empty strings or failed expressions (e.g., `$json.body.name`).
  - With `responseMode=lastNode`, if later branches do not converge correctly or error, the webhook response may error/timeout.

---

### Block 2.2 — Configuration & Data Normalization
**Overview:** Adds workflow-level configuration placeholders and maps the incoming webhook payload into normalized fields used downstream.  
**Nodes involved:** Workflow Configuration, Normalize Complaint Data

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — Injects configuration values and keeps existing fields.
- **Configuration choices:**
  - Adds fields:
    - `slackChannel` (placeholder Slack channel ID)
    - `propertyManagerEmail` (placeholder from-address)
    - `googleSheetId` (placeholder document ID)
    - `googleSheetName` (placeholder tab name)
  - **Include other fields:** true (preserves webhook payload and metadata)
- **Connections:**
  - Input: Tenant Complaint Webhook
  - Output: Normalize Complaint Data
- **Edge cases / failures:**
  - Placeholders must be replaced; otherwise Slack/Email/Sheets nodes will fail or write to invalid destinations.

#### Node: Normalize Complaint Data
- **Type / role:** `n8n-nodes-base.set` — Creates consistent field names for downstream usage.
- **Configuration choices (expressions):**
  - `tenantName = {{$json.body.name}}`
  - `unit = {{$json.body.unit}}`
  - `complaintType = {{$json.body.complaintType}}`
  - `description = {{$json.body.description}}`
  - `timestamp = {{$json.body.timestamp || $now.toISO()}}` (fallback to current time)
  - `tenantEmail = {{$json.body.email}}`
  - **Include other fields:** true
- **Connections:**
  - Input: Workflow Configuration
  - Output: AI Complaint Classifier
- **Edge cases / failures:**
  - If webhook payload is not shaped as `{ body: {...} }`, expressions evaluate to `null`/empty, reducing AI quality and potentially breaking email sending (`toEmail`).

---

### Block 2.3 — AI Classification (Urgency + Category)
**Overview:** Builds a prompt from normalized fields and asks GPT to output a strict formatted classification.  
**Nodes involved:** AI Complaint Classifier, OpenAI Chat Model

#### Node: AI Complaint Classifier
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LangChain “Agent” node used here as an LLM-driven text processor/classifier.
- **Configuration choices:**
  - **Input text:**  
    `Tenant: {{tenantName}} ... Description: {{description}}`
  - **System message:** Enforces:
    - Urgency: High/Medium/Low
    - Category: Maintenance/Noise/Neighbor/Safety/Billing/Other
    - Output format:
      ```
      Urgency: ...
      Category: ...
      Reason: ...
      ```
  - **Prompt type:** “define” (custom system message)
- **Model binding / dependencies:**
  - Requires an attached language model via `ai_languageModel` connection from **OpenAI Chat Model**.
- **Connections:**
  - Input: Normalize Complaint Data
  - Output: Route by Urgency
  - AI language model input: OpenAI Chat Model
- **Outputs:**
  - Produces an `output` field (typical for these Agent nodes) containing the formatted classification text. Downstream nodes reference `{{$json.output}}`.
- **Edge cases / failures:**
  - If the model deviates from exact format, the switch routing (string contains checks) may fail.
  - OpenAI API errors (quota, auth, timeouts) will stop routing.

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Provides the chat model to LangChain Agent nodes.
- **Configuration choices:**
  - **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI API credential (“n8n free OpenAI API credits”).
- **Connections:**
  - Output (ai_languageModel): AI Complaint Classifier
- **Edge cases / failures:**
  - Credential misconfiguration, depleted credits, or model access restrictions.

---

### Block 2.4 — Classification Routing & Notifications
**Overview:** Routes based on urgency (by searching the classifier’s output text) and triggers appropriate actions: Slack for High, email acknowledgment for Medium/Low (as wired), plus task data creation for Medium (but not merged downstream).  
**Nodes involved:** Route by Urgency, Notify Property Manager (High Priority), AI Email Generator, OpenAI Chat Model for Email, Send Acknowledgment Email, Create Task Data (Medium Priority)

#### Node: Route by Urgency
- **Type / role:** `n8n-nodes-base.switch` — Branching logic.
- **Configuration choices:**
  - Evaluates **contains** against `leftValue = {{$json}}` and rightValue like `"Urgency: High"`, `"Urgency: Medium"`, `"Urgency: Low"`.
  - Produces three named outputs: **High**, **Medium**, **Low** (renamed outputs enabled).
- **Connections:**
  - Input: AI Complaint Classifier
  - Outputs:
    - High → Notify Property Manager (High Priority)
    - Medium → AI Email Generator
    - Low → Create Task Data (Medium Priority) *(note: this is likely a wiring error; see edge cases)*
- **Edge cases / failures:**
  - `leftValue = {{$json}}` stringifies an object; behavior depends on n8n coercion. It often becomes `"[object Object]"`, which would **never contain** `"Urgency: High"`. In many workflows, this should be `{{$json.output}}`.
  - If the classifier returns slightly different wording (e.g., `Urgency - High`), routing fails.
  - No default branch defined; unmatched items may be dropped.

#### Node: Notify Property Manager (High Priority)
- **Type / role:** `n8n-nodes-base.slack` — Sends Slack message for urgent cases.
- **Configuration choices:**
  - Sends a formatted message including tenant/unit/type/timestamp/description and `AI Classification: {{$json.output}}`.
  - **Channel selection:** uses Channel ID from configuration:  
    `{{$('Workflow Configuration').first().json.slackChannel}}`
- **Connections:**
  - Input: Route by Urgency (High)
  - Output: Consolidate All Branches (index 0)
- **Edge cases / failures:**
  - Slack credential/token not shown in JSON (must exist in n8n); missing scopes or invalid channel ID causes failure.
  - If `timestamp` or `output` missing, message still sends but is incomplete.

#### Node: AI Email Generator
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Generates tenant acknowledgment email body.
- **Configuration choices:**
  - **Input text:** includes tenant/unit/type/description plus `Urgency: {{$json.output}}` (note: this passes the entire classifier output text, not a parsed urgency value).
  - **System message:** instructs email tone and SLA by urgency; returns **email body only**.
- **Model binding:** requires OpenAI model via **OpenAI Chat Model for Email** connection.
- **Connections:**
  - Input: Route by Urgency (Medium)
  - Output: Send Acknowledgment Email
  - AI language model input: OpenAI Chat Model for Email
- **Edge cases / failures:**
  - If tenant email is missing, sending step fails later.
  - If classifier output is long/unstructured, urgency inference for SLA may be inconsistent.

#### Node: OpenAI Chat Model for Email
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Supplies model to the email generator agent.
- **Configuration:** model `gpt-4.1-mini`
- **Credentials:** same OpenAI credential as other model nodes.
- **Connections:** ai_languageModel → AI Email Generator
- **Edge cases:** same OpenAI-related failures.

#### Node: Send Acknowledgment Email
- **Type / role:** `n8n-nodes-base.emailSend` — Sends the generated email to tenant.
- **Configuration choices:**
  - **To:** `{{$json.tenantEmail}}`
  - **From:** `{{$('Workflow Configuration').first().json.propertyManagerEmail}}`
  - **Subject:** `Complaint Acknowledgment - Unit {{ $json.unit }}`
  - **Body:** `{{$json.output}}` (email body generated by AI Email Generator)
  - **Format:** text
- **Connections:**
  - Input: AI Email Generator
  - Output: Consolidate All Branches (index 1)
- **Edge cases / failures:**
  - Email credentials are not shown in JSON; node requires SMTP or another configured email transport in n8n.
  - Invalid `fromEmail` (placeholder or domain restrictions) can cause send rejection.
  - `tenantEmail` missing/invalid will fail.

#### Node: Create Task Data (Medium Priority)
- **Type / role:** `n8n-nodes-base.set` — Prepares task metadata for follow-up.
- **Configuration choices:**
  - `taskTitle = "Review Complaint - Unit {{unit}}"`
  - `taskDescription` includes tenant name, description, classification
  - `taskPriority = "Medium"`
  - `taskStatus = "Pending Review"`
  - `taskDueDate = {{$now.plus(2, 'days').toISO()}}`
  - Includes other fields: true
- **Connections:**
  - Input: Route by Urgency (Low output in this JSON)
  - Output: **None** (not connected)
- **Edge cases / failures:**
  - As wired, **Low urgency complaints do not get an email** and **do not get merged/logged through this branch**, because this node doesn’t connect to the merge.
  - Likely intended to be used for **Medium**, not Low, and then sent to a task system (Asana/Trello/Jira) or merged for logging.

---

### Block 2.5 — Consolidation, Logging & Resolution Suggestions
**Overview:** Consolidates certain branches, writes the complaint to Google Sheets (append or update), then uses AI to suggest resolution steps.  
**Nodes involved:** Consolidate All Branches, Log to Google Sheets, AI Resolution Suggester, OpenAI Chat Model for Resolutions

#### Node: Consolidate All Branches
- **Type / role:** `n8n-nodes-base.merge` — Combines execution paths before logging.
- **Configuration choices:**
  - **Mode:** combine
  - **Combine by:** combineAll
- **Connections:**
  - Inputs:
    - From Notify Property Manager (High Priority)
    - From Send Acknowledgment Email
  - Output: Log to Google Sheets
- **Edge cases / failures:**
  - Any branch not connected here will never be logged (currently: the “Low → Create Task Data” path).
  - `combineAll` can produce nested/merged structures depending on runtime item counts; ensure Google Sheets mappings still resolve.

#### Node: Log to Google Sheets
- **Type / role:** `n8n-nodes-base.googleSheets` — Persists complaint record.
- **Operation:** `appendOrUpdate`
- **Key configuration:**
  - **Document ID:** `{{$('Workflow Configuration').first().json.googleSheetId}}`
  - **Sheet name:** `{{$('Workflow Configuration').first().json.googleSheetName}}`
  - **Mapping (define below):**
    - Timestamp, Tenant Name, Unit, Complaint Type, Description, Urgency, Status
  - **Status:** hard-coded `"Logged"`
  - **Matching columns:** `Timestamp` (used to decide update vs append)
- **Credentials:** Google Sheets OAuth2 credential (“Google Sheets account 3”).
- **Connections:**
  - Input: Consolidate All Branches
  - Output: AI Resolution Suggester
- **Edge cases / failures:**
  - Using `Timestamp` as matching key is risky: collisions or formatting changes can overwrite prior rows.
  - If sheet headers differ from schema, mapping may fail.
  - OAuth token expiry/permissions errors.

#### Node: AI Resolution Suggester
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Generates actionable resolution steps for property managers.
- **Configuration choices:**
  - Input includes complaint type, description, urgency (`{{$json.output}}`), and unit.
  - System message asks for steps, resources, prevention tips with a specific markdown-like format.
- **Model binding:** via OpenAI Chat Model for Resolutions.
- **Connections:**
  - Input: Log to Google Sheets
  - AI language model input: OpenAI Chat Model for Resolutions
  - Output: **None** (end of workflow)
- **Edge cases / failures:**
  - The `Urgency` field passed is `{{$json.output}}` which may be the classifier output text, not only “High/Medium/Low”; still workable but noisy.
  - Output is not delivered to Slack/email/sheet in this JSON; it is generated but not stored unless execution result is inspected.

#### Node: OpenAI Chat Model for Resolutions
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration:** model `gpt-4.1-mini`
- **Credentials:** same OpenAI credential.
- **Connections:** ai_languageModel → AI Resolution Suggester
- **Edge cases:** OpenAI credential/quota/model access.

---

### Sticky Notes (contextual documentation within canvas)
- **Sticky Note (Main / Setup):** describes overall workflow purpose and setup steps.
- **Sticky Note1:** “Webhook Trigger”
- **Sticky Note2:** “AI logi” (truncated, likely “AI logic”)
- **Sticky Note3:** “Classification and Routing”
- **Sticky Note4:** “Log and Resolve”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Tenant Complaint Webhook | Webhook | Receive tenant complaint via HTTP POST | — | Workflow Configuration | ## Webhook Trigger |
| Workflow Configuration | Set | Store Slack/Email/Sheets config placeholders | Tenant Complaint Webhook | Normalize Complaint Data | ## Main … (purpose + setup steps) |
| Normalize Complaint Data | Set | Map webhook payload into normalized fields | Workflow Configuration | AI Complaint Classifier | ## AI logi |
| AI Complaint Classifier | LangChain Agent | Classify urgency/category/reason | Normalize Complaint Data | Route by Urgency | ## AI logi |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM provider for classifier agent | — | AI Complaint Classifier (ai_languageModel) | ## AI logi |
| Route by Urgency | Switch | Branch by urgency string match | AI Complaint Classifier | Notify Property Manager (High Priority); AI Email Generator; Create Task Data (Medium Priority) | ## Classification and Routing |
| Notify Property Manager (High Priority) | Slack | Alert property managers for high urgency | Route by Urgency (High) | Consolidate All Branches | ## Classification and Routing |
| AI Email Generator | LangChain Agent | Generate acknowledgment email body | Route by Urgency (Medium) | Send Acknowledgment Email | ## Classification and Routing |
| OpenAI Chat Model for Email | OpenAI Chat Model (LangChain) | LLM provider for email generator | — | AI Email Generator (ai_languageModel) | ## Classification and Routing |
| Send Acknowledgment Email | Email Send | Send acknowledgment email to tenant | AI Email Generator | Consolidate All Branches | ## Classification and Routing |
| Create Task Data (Medium Priority) | Set | Prepare follow-up task metadata | Route by Urgency (Low) | — | ## Classification and Routing |
| Consolidate All Branches | Merge | Combine routed outcomes before logging | Notify Property Manager (High Priority); Send Acknowledgment Email | Log to Google Sheets | ## Log and Resolve |
| Log to Google Sheets | Google Sheets | Append/update complaint record | Consolidate All Branches | AI Resolution Suggester | ## Log and Resolve |
| AI Resolution Suggester | LangChain Agent | Generate recommended resolution steps | Log to Google Sheets | — | ## Log and Resolve |
| OpenAI Chat Model for Resolutions | OpenAI Chat Model (LangChain) | LLM provider for resolution suggester | — | AI Resolution Suggester (ai_languageModel) | ## Log and Resolve |
| Sticky Note | Sticky Note | Canvas documentation | — | — | ## Main … (purpose + setup steps) |
| Sticky Note1 | Sticky Note | Canvas documentation | — | — | ## Webhook Trigger |
| Sticky Note2 | Sticky Note | Canvas documentation | — | — | ## AI logi |
| Sticky Note3 | Sticky Note | Canvas documentation | — | — | ## Classification and Routing |
| Sticky Note4 | Sticky Note | Canvas documentation | — | — | ## Log and Resolve |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named “Automated Tenant Complaint Management and Escalation System”.

2. **Add “Webhook” node**
   - Name: **Tenant Complaint Webhook**
   - Method: **POST**
   - Path: **tenant-complaint**
   - Response mode: **Last node**
   - Save to generate the test/production URL.

3. **Add “Set” node**
   - Name: **Workflow Configuration**
   - Enable **Include Other Fields**
   - Add string fields:
     - `slackChannel` = (your Slack channel ID, e.g. `C0123...`)
     - `propertyManagerEmail` = (from address you control)
     - `googleSheetId` = (Spreadsheet ID from URL)
     - `googleSheetName` = (tab name, e.g. `Complaints Log`)
   - Connect: **Tenant Complaint Webhook → Workflow Configuration**

4. **Add “Set” node**
   - Name: **Normalize Complaint Data**
   - Enable **Include Other Fields**
   - Add fields (as expressions):
     - `tenantName` = `{{$json.body.name}}`
     - `unit` = `{{$json.body.unit}}`
     - `complaintType` = `{{$json.body.complaintType}}`
     - `description` = `{{$json.body.description}}`
     - `timestamp` = `{{$json.body.timestamp || $now.toISO()}}`
     - `tenantEmail` = `{{$json.body.email}}`
   - Connect: **Workflow Configuration → Normalize Complaint Data**

5. **Add “OpenAI Chat Model” (LangChain) node**
   - Name: **OpenAI Chat Model**
   - Model: **gpt-4.1-mini**
   - Credentials: configure **OpenAI API** (API key / organization as required).

6. **Add “AI Agent” (LangChain Agent) node**
   - Name: **AI Complaint Classifier**
   - Text: build the multi-line complaint summary (tenant/unit/type/description).
   - System message: include the strict format instructions (Urgency/Category/Reason).
   - Connect:
     - Main: **Normalize Complaint Data → AI Complaint Classifier**
     - AI language model: **OpenAI Chat Model → AI Complaint Classifier** (ai_languageModel port)

7. **Add “Switch” node**
   - Name: **Route by Urgency**
   - Create 3 rules (string contains):
     - High: contains `Urgency: High`
     - Medium: contains `Urgency: Medium`
     - Low: contains `Urgency: Low`
   - Important: set the value being tested to the classifier output (recommended):
     - Use `{{$json.output}}` rather than `{{$json}}`
   - Connect: **AI Complaint Classifier → Route by Urgency**

8. **High urgency path (Slack)**
   - Add **Slack** node named **Notify Property Manager (High Priority)**
   - Operation: send message to a channel
   - Channel ID: `{{$('Workflow Configuration').first().json.slackChannel}}`
   - Message: include complaint fields + `{{$json.output}}`
   - Credentials: configure Slack OAuth/token with permission to post in the channel.
   - Connect: **Route by Urgency (High) → Notify Property Manager (High Priority)**

9. **Medium/Low acknowledgment email path**
   - Add **OpenAI Chat Model** node named **OpenAI Chat Model for Email** (gpt-4.1-mini, same OpenAI creds).
   - Add **LangChain Agent** node named **AI Email Generator**
     - System message: email guidelines and “Return ONLY the email body text”.
     - Text: include tenant/unit/type/description and urgency context.
     - Connect ai_languageModel: **OpenAI Chat Model for Email → AI Email Generator**
   - Add **Email Send** node named **Send Acknowledgment Email**
     - To: `{{$json.tenantEmail}}`
     - From: `{{$('Workflow Configuration').first().json.propertyManagerEmail}}`
     - Subject: `Complaint Acknowledgment - Unit {{$json.unit}}`
     - Body: `{{$json.output}}`
     - Configure email credentials (SMTP or provider-supported email node/credentials in your n8n).
   - Connect: **AI Email Generator → Send Acknowledgment Email**
   - Connect: **Route by Urgency (Medium) → AI Email Generator**
   - (If you want Low to also get an email, connect **Low → AI Email Generator** as well.)

10. **Optional task metadata (Medium follow-up)**
   - Add **Set** node named **Create Task Data (Medium Priority)**
   - Fields:
     - taskTitle, taskDescription, taskPriority=Medium, taskStatus=Pending Review, taskDueDate=now+2 days
   - Connect it to the appropriate branch (recommended: **Medium**, not Low), then either:
     - send to a task tool node (Asana/Trello/Jira), and/or
     - merge into logging.

11. **Merge branches**
   - Add **Merge** node named **Consolidate All Branches**
   - Mode: **Combine**
   - Combine by: **Combine All**
   - Connect:
     - **Notify Property Manager (High Priority) → Consolidate All Branches**
     - **Send Acknowledgment Email → Consolidate All Branches**
     - (Optionally also connect task branch if you want it logged.)

12. **Log to Google Sheets**
   - Add **Google Sheets** node named **Log to Google Sheets**
   - Credentials: Google Sheets OAuth2 (ensure spreadsheet access)
   - Operation: **Append or Update**
   - Document ID: `{{$('Workflow Configuration').first().json.googleSheetId}}`
   - Sheet name: `{{$('Workflow Configuration').first().json.googleSheetName}}`
   - Map columns: Timestamp, Tenant Name, Unit, Complaint Type, Description, Urgency, Status
   - Matching column: Timestamp (or replace with a more stable unique ID).
   - Connect: **Consolidate All Branches → Log to Google Sheets**

13. **AI resolution suggestions**
   - Add **OpenAI Chat Model** node named **OpenAI Chat Model for Resolutions** (gpt-4.1-mini).
   - Add **LangChain Agent** node named **AI Resolution Suggester**
     - System message: request steps/resources/prevention tips.
     - Text: include complaint details and urgency.
     - Connect ai_languageModel: **OpenAI Chat Model for Resolutions → AI Resolution Suggester**
   - Connect: **Log to Google Sheets → AI Resolution Suggester**
   - (Optional) Add a Slack/email or Google Sheets update step to store the resolution suggestions.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically triages tenant complaints using AI and routes high-priority issues to property managers immediately while logging all complaints for reporting. Medium and low-priority issues are acknowledged and scheduled for follow-up…” | Sticky note: Main |
| Setup notes: connect form to webhook; add credentials for Slack/Email/Google Sheets/AI; customize prompts; test High/Medium/Low; adjust follow-up logic. | Sticky note: Main |
| Section headers present on canvas: “Webhook Trigger”, “AI logi”, “Classification and Routing”, “Log and Resolve”. | Sticky notes on canvas |

