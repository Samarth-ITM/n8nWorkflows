Send package and visitor alerts with Slack, SMS, email, Google Sheets and Claude

https://n8nworkflows.xyz/workflows/send-package-and-visitor-alerts-with-slack--sms--email--google-sheets-and-claude-12430


# Send package and visitor alerts with Slack, SMS, email, Google Sheets and Claude

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow receives incoming **package** or **visitor arrival** events via an HTTP webhook, **normalizes** the incoming payload, uses **Claude (Anthropic)** to **classify urgency** (“high” vs “regular”) with a structured output parser, then **routes notifications**:
- Tenant gets **email** (always) and **SMS** (only for high urgency).
- Property management gets a **Slack** alert (only on high urgency in the current wiring).
- The event is **logged to Google Sheets**, and Claude suggests **follow-up actions** based on elapsed time rules.

**Typical use cases:** property management/front desk automation, concierge package intake, visitor check-in alerts, audit logging, and follow-up recommendations.

### Logical blocks
1.1 **Input Reception & Configuration** (Webhook → config variables → normalize payload)  
1.2 **AI Urgency Classification (Claude + Structured Parser)**  
1.3 **Notification Routing & Communications** (Switch → email/SMS/Slack)  
1.4 **Logging & Follow-up Suggestion** (Merge → Google Sheets → AI follow-up)

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Configuration

**Overview:**  
Accepts an inbound POST webhook, sets workflow-level configuration values (channel IDs, feature flags), and normalizes incoming fields into a consistent schema used throughout the workflow.

**Nodes involved:**
- Package/Visitor Webhook
- Workflow Configuration
- Normalize Data

#### Node: Package/Visitor Webhook
- **Type / Role:** `Webhook` trigger; entry point for external systems.
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: `package-visitor-webhook`
  - Response mode: **Last node** (workflow response will come from the last executed node).
- **Inputs/Outputs:**
  - No input (trigger).
  - Output → **Workflow Configuration**
- **Key data expectations:**
  - Expects payload under `body` (e.g., `body.tenant`, `body.email`, etc.).
- **Edge cases / failures:**
  - If the caller sends fields outside `body`, the normalization expressions may resolve to `undefined`.
  - Response depends on the final node; if downstream fails, webhook call likely returns error.
- **Version notes:** Type version **2.1**.

#### Node: Workflow Configuration
- **Type / Role:** `Set`; defines reusable configuration constants.
- **Configuration (interpreted):**
  - Adds:
    - `managementSlackChannel` (string placeholder for Slack channel ID)
    - `smsEnabled` = `true` (feature flag; **not used downstream** in this workflow)
    - `highUrgencyThreshold` = `"perishable,high-value"` (**not used downstream**; classification is driven by Claude prompt instead)
  - “Include other fields” enabled, so it preserves incoming webhook fields.
- **Inputs/Outputs:**
  - Input ← Webhook
  - Output → **Normalize Data**
- **Edge cases / failures:**
  - Placeholders must be replaced; otherwise Slack node may fail when trying to use the channel ID.
- **Version notes:** Type version **3.4**.

#### Node: Normalize Data
- **Type / Role:** `Set`; maps incoming webhook payload variations into a single consistent schema.
- **Configuration (interpreted):** Creates/overwrites fields:
  - `tenantName` = `body.tenant` OR `body.tenantName`
  - `tenantEmail` = `body.tenantEmail` OR `body.email`
  - `tenantPhone` = `body.tenantPhone` OR `body.phone`
  - `building` = `body.building`
  - `packageType` = `body.packageType` OR `body.type`
  - `arrivalTime` = `body.arrivalTime` OR `body.timestamp` OR `now`
  - `description` = `body.description` OR empty string
  - `isVisitor` = `body.isVisitor` OR `false`
  - Preserves other fields.
- **Inputs/Outputs:**
  - Input ← Workflow Configuration
  - Output → **Classify Urgency**
- **Key expressions:** uses n8n expressions like `{{ $json.body.tenant || $json.body.tenantName }}`.
- **Edge cases / failures:**
  - If `body` is missing, expressions referencing `$json.body.*` can error or evaluate to `undefined` depending on runtime behavior; safer would be optional chaining patterns (not used here).
- **Version notes:** Type version **3.4**.

---

### 2.2 AI Urgency Classification (Claude + Structured Parser)

**Overview:**  
Uses Claude (Anthropic chat model) to classify urgency into **high** or **regular** and produce a required **reason**, enforced through a structured output parser.

**Nodes involved:**
- Classify Urgency
- Anthropic Chat Model
- Urgency Output Parser

#### Node: Anthropic Chat Model
- **Type / Role:** `lmChatAnthropic`; provides the LLM backend for the chain.
- **Configuration (interpreted):**
  - Model: `claude-sonnet-4-5-20250929` (as selected in node)
  - Default options (no special tuning shown)
- **Inputs/Outputs:**
  - Output (AI language model connection) → **Classify Urgency** (`ai_languageModel`)
- **Credentials:** requires Anthropic credentials in n8n (not embedded in JSON).
- **Edge cases / failures:**
  - Auth/permission errors, model availability changes, rate limits, timeouts.
- **Version notes:** Type version **1.3**.

#### Node: Urgency Output Parser
- **Type / Role:** `outputParserStructured`; enforces a JSON schema output for the LLM response.
- **Configuration (interpreted):**
  - Manual JSON schema requiring:
    - `urgency`: enum `"high"` or `"regular"`
    - `reason`: string
- **Inputs/Outputs:**
  - Output parser connection → **Classify Urgency** (`ai_outputParser`)
- **Edge cases / failures:**
  - If the model responds outside the schema, parsing fails and the workflow stops unless error handling is configured elsewhere.
- **Version notes:** Type version **1.3**.

#### Node: Classify Urgency
- **Type / Role:** `chainLlm`; prompts the model and returns structured fields.
- **Configuration (interpreted):**
  - Prompt text includes:
    - Type, Description, Is Visitor, Arrival Time
  - “Define” prompt mode
  - Output parser enabled (so final output should include `urgency` and `reason`)
- **Inputs/Outputs:**
  - Main input ← Normalize Data
  - AI model input ← Anthropic Chat Model
  - Output parser input ← Urgency Output Parser
  - Main output → **Route by Urgency**
- **Key variables used:** `{{ $json.packageType }}`, `{{ $json.description }}`, `{{ $json.isVisitor }}`, `{{ $json.arrivalTime }}`
- **Edge cases / failures:**
  - Missing/empty fields can reduce classification quality.
  - Parser failures if Claude returns non-conforming structure.
- **Version notes:** Type version **1.7**.

---

### 2.3 Notification Routing & Communications

**Overview:**  
Routes the event by `urgency`. High urgency sends tenant email + tenant SMS + management Slack. Regular urgency sends tenant email only. Both branches are merged for logging.

**Nodes involved:**
- Route by Urgency
- Email Tenant - High Urgency
- SMS Tenant - High Urgency
- Notify Management
- Email Tenant - Regular
- Merge Notification Branches

#### Node: Route by Urgency
- **Type / Role:** `Switch`; branching logic.
- **Configuration (interpreted):**
  - Rule 1 “High Urgency”: if `{{ $json.urgency }}` equals `"high"`
  - Rule 2 “Regular”: if `{{ $json.urgency }}` equals `"regular"`
  - Outputs are renamed accordingly.
- **Inputs/Outputs:**
  - Input ← Classify Urgency
  - Output 0 (“High Urgency”) → Email Tenant - High Urgency AND Notify Management (two parallel outputs)
  - Output 1 (“Regular”) → Email Tenant - Regular
- **Edge cases / failures:**
  - If `urgency` is missing or not exactly `"high"`/`"regular"`, **no rule matches** and nothing downstream runs (no default branch configured).
- **Version notes:** Type version **3.3**.

#### Node: Email Tenant - High Urgency
- **Type / Role:** `Email Send`; urgent tenant email notification.
- **Configuration (interpreted):**
  - To: `{{ $json.tenantEmail }}`
  - From: placeholder sender email
  - Subject: “🚨 Urgent: … Arrival at …”
  - HTML includes building, type, arrival time, description, and urgency reason.
- **Inputs/Outputs:**
  - Input ← Route by Urgency (High)
  - Output → SMS Tenant - High Urgency
- **Credentials:** depends on how n8n email is configured (SMTP or email service in node settings/environment).
- **Edge cases / failures:**
  - Invalid/missing tenant email.
  - SMTP/auth errors; HTML rendering issues are minor.
- **Version notes:** Type version **2.1**.

#### Node: SMS Tenant - High Urgency
- **Type / Role:** `sms77`; sends SMS for urgent items.
- **Configuration (interpreted):**
  - To: `{{ $json.tenantPhone }}`
  - Message: short urgent message with building and package type.
- **Inputs/Outputs:**
  - Input ← Email Tenant - High Urgency
  - Output → Merge Notification Branches (input index 0)
- **Credentials:** requires sms77 credentials (not shown in JSON).
- **Edge cases / failures:**
  - Missing/invalid phone number; country code formatting issues.
  - SMS provider quota/rate limits.
- **Version notes:** Type version **1**.

#### Node: Notify Management
- **Type / Role:** `Slack`; posts an alert to management.
- **Configuration (interpreted):**
  - Channel: by ID from configuration:  
    `{{ $('Workflow Configuration').first().json.managementSlackChannel }}`
  - Message includes tenant, building, type, arrival time, description, and urgency reason.
  - Text also states whether tenant notifications were “email and SMS” vs “email” (based on urgency).
- **Inputs/Outputs:**
  - Input ← Route by Urgency (High)
  - **No outgoing connection** in this workflow (Slack branch does not merge back).
- **Important wiring note:** Management is only notified for **High Urgency** because Slack is only connected from the “High Urgency” switch output.
- **Edge cases / failures:**
  - Invalid Slack credentials or channel ID, missing scopes, channel not found.
  - Expression failure if Workflow Configuration node produced placeholder/empty ID.
- **Version notes:** Type version **2.3**.

#### Node: Email Tenant - Regular
- **Type / Role:** `Email Send`; non-urgent tenant email.
- **Configuration (interpreted):**
  - To: `{{ $json.tenantEmail }}`
  - From: placeholder sender email
  - Subject: “… Arrival Notification - {{ building }}”
  - HTML includes building, type, arrival time, description (no urgency reason).
- **Inputs/Outputs:**
  - Input ← Route by Urgency (Regular)
  - Output → Merge Notification Branches (input index 1)
- **Edge cases / failures:** same as urgent email (email validity, SMTP/auth).
- **Version notes:** Type version **2.1**.

#### Node: Merge Notification Branches
- **Type / Role:** `Merge`; recombines high-urgency (email→SMS) and regular (email) paths for unified logging.
- **Configuration (interpreted):**
  - Mode: **Combine**
  - Combine by: **combineAll**
- **Inputs/Outputs:**
  - Input 0 ← SMS Tenant - High Urgency
  - Input 1 ← Email Tenant - Regular
  - Output → Log to Google Sheets
- **Edge cases / failures:**
  - With “combineAll”, behavior depends on item counts/timing; typically OK here because each execution produces one item on exactly one branch. If both branches ever produced items in the same execution, you may get combined arrays/merged structures that differ from expectations.
- **Version notes:** Type version **3.2**.

---

### 2.4 Logging & Follow-up Suggestion

**Overview:**  
Logs the event to Google Sheets (append or update keyed on Timestamp), then asks Claude to recommend follow-up actions based on urgency and elapsed time rules.

**Nodes involved:**
- Log to Google Sheets
- Suggest Follow-up
- Anthropic Chat Model 2

#### Node: Log to Google Sheets
- **Type / Role:** `Google Sheets`; persistence/audit log.
- **Configuration (interpreted):**
  - Operation: **Append or Update**
  - Target: spreadsheet `documentId` placeholder; `sheetName` placeholder
  - Mapping mode: define columns:
    - Timestamp = `arrivalTime`
    - Tenant Name, Building, Type, Is Visitor, Urgency, Description
    - Notification Sent = `now`
  - Matching column: **Timestamp** (used to update if same timestamp exists)
- **Inputs/Outputs:**
  - Input ← Merge Notification Branches
  - Output → Suggest Follow-up
- **Credentials:**
  - Uses `googleSheetsOAuth2Api` (configured as “Google Sheets account 3” in the workflow).
- **Edge cases / failures:**
  - Placeholders must be replaced for sheet name and document ID.
  - Matching on Timestamp can cause unintended overwrites if multiple events share the same timestamp string.
  - API quota limits, permission issues, sheet schema mismatch.
- **Version notes:** Type version **4.7**.

#### Node: Anthropic Chat Model 2
- **Type / Role:** `lmChatAnthropic`; model backend for follow-up suggestions.
- **Configuration (interpreted):**
  - Model: `claude-sonnet-4-5-20250929`
- **Inputs/Outputs:**
  - Output (AI language model connection) → Suggest Follow-up
- **Edge cases / failures:** same model-related issues as the first Anthropic model node.
- **Version notes:** Type version **1.3**.

#### Node: Suggest Follow-up
- **Type / Role:** `chainLlm`; produces a human-readable recommendation.
- **Configuration (interpreted):**
  - Prompt instructs the assistant to recommend follow-up actions if thresholds exceeded:
    - High urgency > 2 hours unclaimed
    - Regular > 24 hours
    - Visitors > 30 minutes
    - Perishables > 1 hour
  - Includes current time (`now`) in the prompt context.
  - No structured parser; returns free-form text.
- **Inputs/Outputs:**
  - Main input ← Log to Google Sheets
  - AI model input ← Anthropic Chat Model 2
  - No further nodes (end of workflow)
- **Edge cases / failures:**
  - `arrivalTime` is a string; Claude will “reason” about elapsed time but may misinterpret formats/timezones.
  - If `arrivalTime` is not parseable (or is missing), recommendation quality drops.
- **Version notes:** Type version **1.7**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Package/Visitor Webhook | n8n-nodes-base.webhook | Entry point (POST webhook) | — | Workflow Configuration | ## Webook Trigger, Configuration, Normalize Data |
| Workflow Configuration | n8n-nodes-base.set | Set configuration constants/flags | Package/Visitor Webhook | Normalize Data | ## Webook Trigger, Configuration, Normalize Data |
| Normalize Data | n8n-nodes-base.set | Normalize inbound payload to canonical fields | Workflow Configuration | Classify Urgency | ## Webook Trigger, Configuration, Normalize Data |
| Classify Urgency | @n8n/n8n-nodes-langchain.chainLlm | LLM-based urgency classification | Normalize Data; (AI) Anthropic Chat Model; (Parser) Urgency Output Parser | Route by Urgency | ## Urgency Classification |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model provider (classification) | — | Classify Urgency (ai_languageModel) | ## Urgency Classification |
| Urgency Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured output {urgency, reason} | — | Classify Urgency (ai_outputParser) | ## Urgency Classification |
| Route by Urgency | n8n-nodes-base.switch | Branch high vs regular urgency | Classify Urgency | Email Tenant - High Urgency; Notify Management; Email Tenant - Regular | ## Communication |
| Email Tenant - High Urgency | n8n-nodes-base.emailSend | Send urgent tenant email | Route by Urgency (High) | SMS Tenant - High Urgency | ## Communication |
| SMS Tenant - High Urgency | n8n-nodes-base.sms77 | Send urgent tenant SMS | Email Tenant - High Urgency | Merge Notification Branches | ## Communication |
| Notify Management | n8n-nodes-base.slack | Slack alert to management (high urgency only) | Route by Urgency (High) | — | ## Communication |
| Email Tenant - Regular | n8n-nodes-base.emailSend | Send regular tenant email | Route by Urgency (Regular) | Merge Notification Branches | ## Communication |
| Merge Notification Branches | n8n-nodes-base.merge | Merge regular/high notification paths | SMS Tenant - High Urgency; Email Tenant - Regular | Log to Google Sheets | ## Log, Follow Up |
| Log to Google Sheets | n8n-nodes-base.googleSheets | Append/update event log row | Merge Notification Branches | Suggest Follow-up | ## Log, Follow Up |
| Suggest Follow-up | @n8n/n8n-nodes-langchain.chainLlm | Generate follow-up recommendation | Log to Google Sheets; (AI) Anthropic Chat Model 2 | — | ## Log, Follow Up |
| Anthropic Chat Model 2 | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model provider (follow-up) | — | Suggest Follow-up (ai_languageModel) | ## Log, Follow Up |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## Main  This workflow automatically tracks incoming packages and visitor arrivals, notifies tenants via email/SMS, alerts property managers via Slack, and logs everything for reporting.  ## Setup  1. Connect your package/visitor system to Webhook trigger. 2. Add credentials for Slack, Email, SMS, and AI. 3. Customize AI classification prompts for package urgency. 4. Test notification routing for tenants and management. 5. Adjust logging and reporting as needed. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section header | — | — | ## Webook Trigger, Configuration, Normalize Data |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section header | — | — | ## Urgency Classification |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section header | — | — | ## Communication |
| Sticky Note4 | n8n-nodes-base.stickyNote | Section header | — | — | ## Log, Follow Up |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Webhook trigger**
   1. Add node: **Webhook**
   2. Set:
      - HTTP Method: **POST**
      - Path: `package-visitor-webhook`
      - Response Mode: **Last Node**
   3. Save to generate the production URL.

2) **Add configuration constants**
   1. Add node: **Set** named “Workflow Configuration”
   2. Enable **Include Other Fields**
   3. Add fields:
      - `managementSlackChannel` (string): set to your Slack channel ID (e.g., `C0123...`)
      - `smsEnabled` (boolean): `true` (optional; not used unless you add an IF node later)
      - `highUrgencyThreshold` (string): `perishable,high-value` (optional; not used unless you incorporate it)
   4. Connect: **Webhook → Workflow Configuration**

3) **Normalize inbound data**
   1. Add node: **Set** named “Normalize Data”
   2. Enable **Include Other Fields**
   3. Add fields (expressions):
      - `tenantName`: `{{ $json.body.tenant || $json.body.tenantName }}`
      - `tenantEmail`: `{{ $json.body.tenantEmail || $json.body.email }}`
      - `tenantPhone`: `{{ $json.body.tenantPhone || $json.body.phone }}`
      - `building`: `{{ $json.body.building }}`
      - `packageType`: `{{ $json.body.packageType || $json.body.type }}`
      - `arrivalTime`: `{{ $json.body.arrivalTime || $json.body.timestamp || $now }}`
      - `description`: `{{ $json.body.description || "" }}`
      - `isVisitor`: `{{ $json.body.isVisitor || false }}`
   4. Connect: **Workflow Configuration → Normalize Data**

4) **Set up Claude urgency classification (LangChain)**
   1. Add node: **Anthropic Chat Model** (LangChain) named “Anthropic Chat Model”
      - Choose model: `claude-sonnet-4-5-20250929` (or closest available in your n8n)
      - Configure Anthropic credentials in n8n.
   2. Add node: **Structured Output Parser** named “Urgency Output Parser”
      - Schema (manual) requiring:
        - `urgency`: enum `["high","regular"]`
        - `reason`: string
   3. Add node: **Chain LLM** named “Classify Urgency”
      - Prompt text (example):
        - Include Type, Description, Is Visitor, Arrival Time from normalized fields
      - Enable output parsing (so `urgency`/`reason` are returned)
   4. Connect:
      - **Normalize Data → Classify Urgency**
      - **Anthropic Chat Model (ai_languageModel) → Classify Urgency**
      - **Urgency Output Parser (ai_outputParser) → Classify Urgency**

5) **Route by urgency**
   1. Add node: **Switch** named “Route by Urgency”
   2. Create two rules:
      - Rule “High Urgency”: `{{ $json.urgency }}` equals `"high"`
      - Rule “Regular”: `{{ $json.urgency }}` equals `"regular"`
   3. Connect: **Classify Urgency → Route by Urgency**

6) **High urgency tenant email**
   1. Add node: **Email Send** named “Email Tenant - High Urgency”
   2. Set:
      - To: `{{ $json.tenantEmail }}`
      - From: your sender address
      - Subject and HTML using the normalized fields and `{{ $json.reason }}`
   3. Connect: **Route by Urgency (High) → Email Tenant - High Urgency**
   4. Configure SMTP/email credentials as required by your n8n setup.

7) **High urgency SMS**
   1. Add node: **sms77** named “SMS Tenant - High Urgency”
   2. Set:
      - To: `{{ $json.tenantPhone }}`
      - Message: include building/type and urgency wording
   3. Connect: **Email Tenant - High Urgency → SMS Tenant - High Urgency**
   4. Add sms77 credentials.

8) **High urgency Slack to management**
   1. Add node: **Slack** named “Notify Management”
   2. Operation: post message to a channel
   3. Channel selection by ID expression:
      - Channel ID: `{{ $('Workflow Configuration').first().json.managementSlackChannel }}`
   4. Text: include tenant/building/type/arrival/description and urgency reason
   5. Connect: **Route by Urgency (High) → Notify Management**
   6. Add Slack OAuth/token credentials with permission to post to the channel.

9) **Regular tenant email**
   1. Add node: **Email Send** named “Email Tenant - Regular”
   2. Set:
      - To: `{{ $json.tenantEmail }}`
      - From: your sender address
      - Subject/HTML without urgency reason
   3. Connect: **Route by Urgency (Regular) → Email Tenant - Regular**

10) **Merge branches for logging**
   1. Add node: **Merge** named “Merge Notification Branches”
   2. Mode: **Combine**, Combine By: **combineAll**
   3. Connect:
      - **SMS Tenant - High Urgency → Merge (Input 1)**
      - **Email Tenant - Regular → Merge (Input 2)**  
      (Exact input index labeling depends on UI; ensure both feed into Merge.)

11) **Log to Google Sheets**
   1. Add node: **Google Sheets** named “Log to Google Sheets”
   2. Credentials: Google Sheets OAuth2
   3. Operation: **Append or Update**
   4. Select Spreadsheet (Document ID) and Sheet Name
   5. Map columns:
      - Timestamp = `{{ $json.arrivalTime }}`
      - Tenant Name = `{{ $json.tenantName }}`
      - Building = `{{ $json.building }}`
      - Type = `{{ $json.packageType }}`
      - Is Visitor = `{{ $json.isVisitor }}`
      - Urgency = `{{ $json.urgency }}`
      - Description = `{{ $json.description }}`
      - Notification Sent = `{{ $now }}`
   6. Matching column: **Timestamp**
   7. Connect: **Merge Notification Branches → Log to Google Sheets**

12) **Follow-up suggestion (Claude)**
   1. Add node: **Anthropic Chat Model** named “Anthropic Chat Model 2” (can reuse the same credentials/model)
   2. Add node: **Chain LLM** named “Suggest Follow-up”
      - System/instruction message: rules for when to follow up (2h/24h/30m/1h) and allowed actions
      - Prompt text includes packageType, tenantName, building, urgency, arrivalTime, current time (`{{ $now }}`)
   3. Connect:
      - **Log to Google Sheets → Suggest Follow-up**
      - **Anthropic Chat Model 2 (ai_languageModel) → Suggest Follow-up**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically tracks incoming packages and visitor arrivals, notifies tenants via email/SMS, alerts property managers via Slack, and logs everything for reporting. | Sticky note “Main” |
| Setup: connect inbound system to webhook; add credentials for Slack/Email/SMS/AI; customize urgency prompt; test routing; adjust logging/reporting. | Sticky note “Main” |
| **Important wiring behavior:** Slack management notifications currently occur only for **High Urgency** events (Slack node is not connected to the Regular branch). | Workflow logic note |
| `smsEnabled` and `highUrgencyThreshold` are defined but not used; add an **IF** node before SMS (and/or influence the prompt) if you want these to control behavior. | Workflow configuration note |
| Google Sheets “Append or Update” matches by **Timestamp**; collisions can overwrite rows if timestamps repeat. | Logging design note |