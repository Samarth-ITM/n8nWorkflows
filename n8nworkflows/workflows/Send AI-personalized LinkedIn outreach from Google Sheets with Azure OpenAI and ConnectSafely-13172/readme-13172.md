Send AI-personalized LinkedIn outreach from Google Sheets with Azure OpenAI and ConnectSafely

https://n8nworkflows.xyz/workflows/send-ai-personalized-linkedin-outreach-from-google-sheets-with-azure-openai-and-connectsafely-13172


# Send AI-personalized LinkedIn outreach from Google Sheets with Azure OpenAI and ConnectSafely

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Send AI-personalized LinkedIn outreach from Google Sheets with Azure OpenAI and ConnectSafely  
**Workflow name (in JSON):** LinkedIn Outreach Automation- Connectsafely  
**Purpose:** Automate daily B2B LinkedIn outreach by pulling prospects from Google Sheets, generating a personalized first-touch message with Azure OpenAI, saving it back to the sheet, validating it, fetching LinkedIn profile identifiers via ConnectSafely, sending the message via ConnectSafely, and recording send status/URNs back into the sheet.

### 1.1 Scheduling & Start
Runs on a daily schedule (configured as 5 PM via cron).

### 1.2 Data Retrieval (Google Sheets)
Loads prospect rows from a specific spreadsheet/sheet (“Automation result”).

### 1.3 AI Message Generation (Azure OpenAI via LangChain Agent)
For each prospect, checks whether a LinkedIn message is missing; if missing, generates a short personalized outreach message using an agent backed by Azure OpenAI GPT-4o-mini.

### 1.4 Persistence (Write-back to Sheets)
Saves the generated LinkedIn message into Google Sheets, matching by “Person Name”, then re-fetches sheet data.

### 1.5 Message Validation, Profile Lookup & Send (ConnectSafely)
Validates the message is not empty, fetches LinkedIn profile data (HTTP request to ConnectSafely API), extracts URNs/identifiers, sends the message using the ConnectSafely LinkedIn node, then updates the sheet with send status and profile URN.

---

## 2. Block-by-Block Analysis

### Block A — Documentation & Warnings (Sticky Notes)
**Overview:** Provides human-facing explanations, setup instructions, and credential warnings. No execution impact.  
**Nodes involved:**  
- Main Overview  
- Section: Data Retrieval  
- Section: AI Message Generation  
- Section: Message Validation & Sending  
- Warning: Azure OpenAI  
- Warning: ConnectSafely  

**Node details (all Sticky Notes):**
- **Type/role:** `stickyNote` for documentation.
- **Failure modes:** None (non-executing).
- **Connections:** None.

---

### Block B — Scheduling & Data Retrieval
**Overview:** Triggers daily and reads prospect rows from Google Sheets.  
**Nodes involved:**  
- Daily Schedule Trigger (5 PM)  
- Get Prospects from Sheet  

#### Daily Schedule Trigger (5 PM)
- **Type/role:** `scheduleTrigger` entry point.
- **Configuration:** Cron expression `0 0 17 * * *` (runs daily at 17:00:00).
- **Outputs:** To **Get Prospects from Sheet**.
- **Edge cases/failures:** Timezone depends on n8n instance settings; cron may not match intended local time.

#### Get Prospects from Sheet
- **Type/role:** `googleSheets` read rows from sheet.
- **Configuration choices:**
  - Targets Spreadsheet ID `1figNpIMxgqp1L47tYWStpY1yB4F9txn605MKX_5x5rI`
  - Sheet/tab: “Automation result” (gid `987670951`)
  - `executeOnce: true` (node-level setting in JSON)
  - `onError: continueRegularOutput` (workflow continues even if this node errors)
- **Outputs:** To **Check if Message Exists**.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases/failures:**
  - OAuth expired / missing permissions → auth error.
  - Sheet renamed / columns changed → downstream expressions fail (missing keys like `Person Name`).
  - `continueRegularOutput` can mask failures (may output empty items and silently skip work).

---

### Block C — Per-Prospect Filtering & AI Generation Loop
**Overview:** Filters prospects that don’t yet have a LinkedIn message, batches them, and generates a personalized message using an AI agent.  
**Nodes involved:**  
- Check if Message Exists  
- Batch Process Prospects  
- Generate Personalized Message with AI  
- Azure OpenAI GPT-4o-mini  

#### Check if Message Exists
- **Type/role:** `if` node to detect missing message.
- **Condition:** `LinkedIn message` **is empty** (`={{ $json["LinkedIn message"] }}` with operator `empty`).
- **Outputs/connections:** Only the **true** output is wired to **Batch Process Prospects** (so only empty-message prospects proceed).
- **Edge cases/failures:**
  - If the sheet column name differs (extra spaces/case), `$json["LinkedIn message"]` will be `undefined`; “empty” will typically evaluate as true → unintended generation.
  - If message contains whitespace, it might not be treated as empty (depending on n8n string checks).

#### Batch Process Prospects
- **Type/role:** `splitInBatches` to process items in batches/looping.
- **Configuration:** Default options (batch size not explicitly set in JSON).
- **Connections:** From **Check if Message Exists** → (batch output) → **Generate Personalized Message with AI**.
- **Important wiring detail:** In the workflow JSON, the connection uses the *second* “main” output path from SplitInBatches, which is atypical. This can cause unexpected looping behavior depending on n8n version and node semantics.
- **Edge cases/failures:**
  - Miswired output can result in zero items processed or endless looping if a “continue” path is incorrectly used.

#### Generate Personalized Message with AI
- **Type/role:** `@n8n/n8n-nodes-langchain.agent` (LangChain agent) to generate text.
- **Model binding:** Receives its language model via the `ai_languageModel` connection from **Azure OpenAI GPT-4o-mini**.
- **Prompt (user text):** Builds a JSON-like “Prospect Data” block using expressions:
  - `Person Name`, `Company name`, `Current role`, `Industry`, `Company size`, `Loacation` (note typo), `Key Focus Area`.
- **System message:** Sets brand voice and constraints:
  - Friendly, observant, not salesy, 50–85 words, no links, no emojis, soft CTA.
  - Represents “LeadMe” with link: https://leadme.cloud/agencies
- **Output:** Sent to **Save Generated Message to Sheet**.
- **Edge cases/failures:**
  - Missing fields (e.g., typo column `Loacation`) reduces personalization.
  - Model may output extra whitespace/newlines; downstream “empty” checks may pass/fail unexpectedly.
  - If Azure OpenAI credentials/model deployment mismatch → agent fails.

#### Azure OpenAI GPT-4o-mini
- **Type/role:** `lmChatAzureOpenAi` language model provider node.
- **Configuration:** Model set to `gpt-4o-mini` (must exist in your Azure OpenAI deployment or mapping).
- **Credentials:** Azure OpenAI API credentials.
- **Connections:** Provides `ai_languageModel` input to **Generate Personalized Message with AI**.
- **Edge cases/failures:**
  - Azure endpoint/deployment/model name mismatch (common Azure OpenAI issue).
  - 429 rate limits; timeouts; network blocks.

---

### Block D — Save Message & Re-fetch Sheet
**Overview:** Persists the generated message to Google Sheets and then re-reads the sheet before validation/sending.  
**Nodes involved:**  
- Save Generated Message to Sheet  
- Fetch Updated Prospect Data  

#### Save Generated Message to Sheet
- **Type/role:** `googleSheets` append-or-update to store generated message.
- **Operation:** `appendOrUpdate`
- **Matching:** `matchingColumns: ["Person Name"]`
- **Columns written:**
  - `Person Name` = `{{ $('Loop Over Items1').item.json['Person Name'] }}`
  - `LinkedIn message` = `{{ $json.output }}`
- **Critical issue (expression dependency):**
  - References a node named **`Loop Over Items1`**, which does **not exist** in this workflow JSON. The batching node is named **Batch Process Prospects**.
  - This will likely cause an expression error or write wrong/empty “Person Name”, preventing correct updates.
- **Error behavior:** `onError: continueRegularOutput` (can silently fail and still proceed).
- **Outputs:** To **Fetch Updated Prospect Data**.
- **Edge cases/failures:**
  - If matching column “Person Name” is not unique, multiple rows may be updated unpredictably.
  - Expression failure due to missing node reference (`Loop Over Items1`) is highly likely.

#### Fetch Updated Prospect Data
- **Type/role:** `googleSheets` read (re-fetch) sheet data.
- **Configuration:** Same document + sheet as earlier.
- **Notable setting:** `executeOnce: true`.
- **Outputs:** To **Validate Message Not Empty**.
- **Edge cases/failures:**
  - This re-fetch returns **all rows** unless filtered; downstream validation may process the wrong row(s) unless constrained elsewhere (it is not constrained in this JSON).

---

### Block E — Validation, Profile Lookup, Send, and Status Update
**Overview:** Ensures messages exist, fetches profile identifiers, sends LinkedIn message via ConnectSafely, and writes send status back to Google Sheets.  
**Nodes involved:**  
- Validate Message Not Empty  
- Get LinkedIn Profile Data  
- Extract Profile URN and Identifiers  
- Send LinkedIn Message via ConnectSafely  
- Update Sheet with Send Status  

#### Validate Message Not Empty
- **Type/role:** `if` node to check presence of LinkedIn message before attempting send.
- **Condition:** `LinkedIn message` **notEmpty** using `={{ $json["LinkedIn message"] }}`
- **Odd configuration detail:** A `rightValue` of `"ADDED in AIMFOX"` is present, but for `notEmpty` this value is not meaningful (the operator ignores it).
- **Outputs:** True branch → **Get LinkedIn Profile Data**.
- **Edge cases/failures:**
  - If Fetch Updated Prospect Data returns many rows, this will pass for any row with a message and attempt to send for all of them.
  - If messages contain only whitespace, they may still count as not empty.

#### Get LinkedIn Profile Data
- **Type/role:** `httpRequest` to ConnectSafely API endpoint for profile lookup.
- **Request:**
  - `POST https://api.connectsafely.ai/linkedin/profile`
  - Header: `Authorization: Bearer YOUR_TOKEN_HERE` (placeholder, not a real token)
  - Body parameter: `profileId = "vedant-badwaniya-4a0465333"` (hardcoded)
- **Outputs:** To **Extract Profile URN and Identifiers**.
- **Edge cases/failures:**
  - As-is, it will always look up the same hardcoded profile, not the prospect’s profile.
  - Placeholder token guarantees 401 unless replaced.
  - If API response structure differs, the next Code node may produce nulls.

#### Extract Profile URN and Identifiers
- **Type/role:** `code` node to normalize identifiers.
- **Logic:**
  - Reads `item.profile.entityUrn` → `profileUrn`
  - Uses `item.profileId` or `profile.publicIdentifier` → `profileId`
  - Builds `profileUrl` from `publicIdentifier`
  - Outputs structured JSON fields: `success`, `profileUrn`, `profileId`, `publicIdentifier`, `profileUrl`, plus optional name/headline/location.
- **Outputs:** To **Send LinkedIn Message via ConnectSafely**.
- **Edge cases/failures:**
  - If `items[0].json.profile` is missing, outputs null URN/IDs; sending will fail.
  - Assumes single-item input; if multiple items come in, it ignores all but the first.

#### Send LinkedIn Message via ConnectSafely
- **Type/role:** `n8n-nodes-connectsafely-ai.connectSafelyLinkedIn` send message operation.
- **Configuration (current JSON):**
  - `operation: sendMessage`
  - `accountId: 695ce64a09c18d6bbbe90ed0`
  - `recipientProfileId: {{ $json.profileId }}`
  - `recipientProfileUrn: {{ $json.profileUrn }}`
  - **message:** hardcoded `Hii testt vedant`
  - **subject:** hardcoded `test`
- **Outputs:** To **Update Sheet with Send Status**.
- **Credentials:** ConnectSafely API credential.
- **Edge cases/failures:**
  - Hardcoded message bypasses AI-generated text; likely not intended.
  - If URN/ID invalid → ConnectSafely send fails.
  - Rate limits / permission issues / LinkedIn restrictions.

#### Update Sheet with Send Status
- **Type/role:** `googleSheets` append-or-update to store send results.
- **Operation:** `appendOrUpdate`
- **Matching:** `matchingColumns: ["Linkedin Url"]`
- **Columns written:**
  - `Profile urn` = `{{ $json.recipientProfileUrn }}`
  - `Linkedin Url` = `{{ $('Code in JavaScript').item.json.profileUrl }}`
  - `message sent` = `{{ $json.success }}`
- **Critical issue (node reference mismatch):**
  - References **`Code in JavaScript`**, but the actual code node is named **Extract Profile URN and Identifiers**. This expression will fail unless renamed.
- **Edge cases/failures:**
  - Matching by “Linkedin Url” requires that exact column value to already exist and match; but here it attempts to *set* Linkedin Url from the profile lookup. If the sheet row’s Linkedin Url is empty, update may not target the right row (it may append instead).
  - `message sent` is configured as type string in schema; may store boolean as string.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Overview | stickyNote | Documentation / overview |  |  | ### How it works… (full setup/customization text as provided) |
| Section: Data Retrieval | stickyNote | Documentation header |  |  | ## Data Retrieval… |
| Section: AI Message Generation | stickyNote | Documentation header |  |  | ## AI Message Generation… |
| Section: Message Validation & Sending | stickyNote | Documentation header |  |  | ## Message Validation & LinkedIn Sending… |
| Warning: Azure OpenAI | stickyNote | Credential warning |  |  | ⚠️ **Azure OpenAI Credentials Required**… |
| Warning: ConnectSafely | stickyNote | Credential warning |  |  | ⚠️ **ConnectSafely API Required**… |
| Daily Schedule Trigger (5 PM) | scheduleTrigger | Daily trigger |  | Get Prospects from Sheet |  |
| Get Prospects from Sheet | googleSheets | Read prospects | Daily Schedule Trigger (5 PM) | Check if Message Exists |  |
| Check if Message Exists | if | Filter prospects needing message | Get Prospects from Sheet | Batch Process Prospects |  |
| Batch Process Prospects | splitInBatches | Batch/loop prospects | Check if Message Exists | Generate Personalized Message with AI |  |
| Azure OpenAI GPT-4o-mini | lmChatAzureOpenAi | LLM provider |  | Generate Personalized Message with AI (ai_languageModel) |  |
| Generate Personalized Message with AI | langchain agent | Produce outreach text | Batch Process Prospects; Azure OpenAI GPT-4o-mini | Save Generated Message to Sheet |  |
| Save Generated Message to Sheet | googleSheets | Write AI message back | Generate Personalized Message with AI | Fetch Updated Prospect Data |  |
| Fetch Updated Prospect Data | googleSheets | Re-read sheet data | Save Generated Message to Sheet | Validate Message Not Empty |  |
| Validate Message Not Empty | if | Ensure message exists before send | Fetch Updated Prospect Data | Get LinkedIn Profile Data |  |
| Get LinkedIn Profile Data | httpRequest | Retrieve LinkedIn profile identifiers | Validate Message Not Empty | Extract Profile URN and Identifiers |  |
| Extract Profile URN and Identifiers | code | Normalize URN/ID/url fields | Get LinkedIn Profile Data | Send LinkedIn Message via ConnectSafely |  |
| Send LinkedIn Message via ConnectSafely | connectsafelyLinkedIn | Send LinkedIn message | Extract Profile URN and Identifiers | Update Sheet with Send Status |  |
| Update Sheet with Send Status | googleSheets | Persist send status + URN | Send LinkedIn Message via ConnectSafely |  |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
   - Name it: `LinkedIn Outreach Automation- Connectsafely` (or your preferred name).

2) **Add trigger**
   - Add node: **Schedule Trigger**
   - Set cron to `0 0 17 * * *` (adjust timezone in n8n settings if needed).
   - Connect to the next step.

3) **Google Sheets: read prospects**
   - Add node: **Google Sheets**
   - Credentials: connect Google Sheets OAuth2.
   - Select spreadsheet: your prospect spreadsheet.
   - Select sheet/tab: `Automation result`.
   - Operation: “Read” (or the equivalent “Get Many” depending on UI version).
   - Connect: Trigger → Sheets Read.

4) **Filter: only prospects missing a message**
   - Add node: **IF**
   - Condition: String → `LinkedIn message` → **is empty**.
     - Expression: `{{ $json["LinkedIn message"] }}`
   - Connect: Sheets Read → IF
   - Use the **true** output for the next step.

5) **Batch processing**
   - Add node: **Split In Batches**
   - Set an explicit batch size (recommended) like 10–50 to control rate.
   - Connect: IF (true) → Split In Batches.

6) **Azure OpenAI model node**
   - Add node: **Azure OpenAI Chat Model** (LangChain)
   - Credentials: configure Azure OpenAI:
     - Endpoint, API key, API version
     - Deployment/model mapping for `gpt-4o-mini` (must exist in your Azure resource)
   - Connect its **AI language model** output to the agent node in the next step.

7) **AI agent to generate the message**
   - Add node: **AI Agent (LangChain Agent)**
   - Set **System message** to the brand rules (as in the workflow).
   - Set **User prompt/text** to include prospect fields (Person Name, company, role, etc.).
   - Ensure it outputs **only** the message text.
   - Connect: Split In Batches → Agent
   - Connect: Azure OpenAI Model → Agent (via the `ai_languageModel` connection).

8) **Write generated message back to Google Sheets**
   - Add node: **Google Sheets**
   - Operation: **Append or Update**
   - Matching column: `Person Name` (or a better unique key if you have one, e.g., Linkedin Url).
   - Map fields:
     - `Person Name` = `{{ $json["Person Name"] }}` (from the same item being processed)
     - `LinkedIn message` = `{{ $json.output }}` (or the agent’s actual output field in your version)
   - Important: do **not** reference non-existent nodes (fix the original workflow’s `Loop Over Items1` reference).
   - Connect: Agent → Sheets Update.

9) **(Optional but recommended) Avoid re-reading the entire sheet**
   - Instead of re-fetching all rows, pass along the same item and validate/send directly.
   - If you still want re-fetch:
     - Add **Google Sheets Read** node again and filter down to the same prospect (e.g., by Person Name or Linkedin Url).

10) **Validate message not empty**
   - Add node: **IF**
   - Condition: `{{ $json["LinkedIn message"] }}` **is not empty**
   - Connect from either:
     - The updated item stream, or
     - Your re-fetch + filter step.

11) **Get LinkedIn profile data (ConnectSafely)**
   - Preferred approach: use a ConnectSafely-native node if available for profile lookup; otherwise HTTP.
   - Add node: **HTTP Request**
     - Method: POST
     - URL: `https://api.connectsafely.ai/linkedin/profile`
     - Auth: use a secure credential or header with a real bearer token (do not hardcode).
     - Body: set `profileId` dynamically from sheet data (e.g., from `Linkedin Url` parsed to publicIdentifier).
   - Fix the original workflow issue: remove hardcoded `vedant-badwaniya-4a0465333`.

12) **Extract identifiers**
   - Add node: **Code**
   - Use code similar to the provided node to output `profileUrn`, `profileId`, `profileUrl`, etc.
   - Connect: HTTP Request → Code.

13) **Send LinkedIn message via ConnectSafely**
   - Add node: **ConnectSafely LinkedIn**
   - Credentials: ConnectSafely API key credential.
   - Operation: `sendMessage`
   - Map:
     - `recipientProfileId` = `{{ $json.profileId }}`
     - `recipientProfileUrn` = `{{ $json.profileUrn }}`
     - `message` = the AI-generated message (ensure you pass it through; do not hardcode).
   - Connect: Code → ConnectSafely Send.

14) **Update sheet with send status**
   - Add node: **Google Sheets** (Append or Update)
   - Matching column: choose a stable unique key (recommended: `Linkedin Url` or an internal ID).
   - Set:
     - `Profile urn` = `{{ $json.recipientProfileUrn }}`
     - `message sent` = `{{ $json.success }}`
   - If you need `profileUrl`, reference the correct code node output (fix the original workflow’s bad reference to `Code in JavaScript`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| LeadMe referenced as the sender’s context in the AI system message | https://leadme.cloud/agencies |
| Azure OpenAI must be configured with access to GPT-4o-mini | Mentioned in “Warning: Azure OpenAI” sticky note |
| ConnectSafely API credentials required to send LinkedIn messages | Mentioned in “Warning: ConnectSafely” sticky note |
| Key implementation issues to fix before production: non-existent node references (`Loop Over Items1`, `Code in JavaScript`), hardcoded ConnectSafely bearer token, hardcoded profileId, hardcoded message text | Affects saving, profile lookup, sending, and sheet update correctness |

