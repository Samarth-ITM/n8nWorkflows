Automated Gmail classification and labeling with GPT-4, Sheets and Slack alerts

https://n8nworkflows.xyz/workflows/automated-gmail-classification-and-labeling-with-gpt-4--sheets-and-slack-alerts-11836


# Automated Gmail classification and labeling with GPT-4, Sheets and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Automated Gmail classification and labeling with GPT-4, Sheets and Slack alerts  
**Workflow name (n8n):** Classify Gmail with AI Auto-Labeler

**Purpose:**  
Automatically monitor a Gmail inbox for unread emails, extract/clean the email content, classify the email with an OpenAI chat model (category, priority, sentiment), extract entities and action items as structured JSON, apply category-specific Gmail labels, log results to Google Sheets, and escalate high-priority/urgent emails to Slack. Classification failures are logged to a separate “Error Log” sheet.

**Target use cases:**
- Auto-triage of inbound support and sales inboxes
- Prioritization and escalation of urgent emails
- Building a searchable audit trail of inbound communications in Sheets

### 1.1 Email Intake (Polling Trigger)
Monitors Gmail for unread emails and emits each matching message into the workflow.

### 1.2 Email Normalization
Extracts consistent fields (sender, subject, body, thread/message IDs) and prepares a trimmed plain-text body for AI processing.

### 1.3 AI Classification & Structured Extraction
Uses an OpenAI chat model via a LangChain Agent with a structured output parser enforcing JSON schema-like output.

### 1.4 Validation & Canonical Output
Validates and normalizes the AI response (defaults, allowed values, guaranteed arrays), then produces a stable output object for downstream routing, logging, and alerting.

### 1.5 Routing & Gmail Labeling
Routes by category into 9 label-application branches, then logs all emails to Google Sheets.

### 1.6 Escalation (Slack) + Error Logging
Sends Slack alerts for “High” priority or “Urgent” signals; logs failed classifications to a separate sheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Email Intake (Gmail polling)
**Overview:** Polls Gmail periodically for unread emails and starts the workflow per email.  
**Nodes involved:** `Gmail Check Every 2 Min`

#### Node: Gmail Check Every 2 Min
- **Type / role:** `gmailTrigger` — entry point; polls Gmail inbox.
- **Key configuration (interpreted):**
  - Filter: `readStatus = unread` (only unread messages)
  - Poll: configured as “every minute” (`pollTimes.item[0].mode = everyMinute`) despite the node name saying “Every 2 Min”.
- **Inputs / outputs:**
  - **Output:** to `Extract and Clean Email Content`
- **Version notes:** typeVersion `1.3` (older trigger versions can differ in polling config fields).
- **Edge cases / failures:**
  - Gmail OAuth credential missing/expired → auth failures.
  - Rate limits or large inbox volumes can cause skipped/delayed polling.
  - If unread state changes quickly (e.g., other automation marks read), messages may be missed.

---

### Block 2 — Email Normalization & Cleaning
**Overview:** Normalizes Gmail trigger payload into a consistent structure and extracts a clean text body for the AI prompt.  
**Nodes involved:** `Extract and Clean Email Content`

#### Node: Extract and Clean Email Content
- **Type / role:** `code` — transforms raw Gmail payload into canonical fields.
- **Key configuration choices:**
  - Extract sender from `from.value[0]` if present; fallback to `From`; default `user@example.com`.
  - Subject from `subject` or `Subject`; fallback `"No Subject"`.
  - Body:
    - Prefer `textPlain` (or `text`)
    - Else fallback to HTML (`html` or `textHtml`) and strip tags/scripts/styles to plain text.
  - Truncates body to **first 3000 characters** for AI (`bodyForAI`).
  - Derives:
    - `threadId = emailData.threadId || emailData.id`
    - `messageId = emailData.id || emailData.messageId`
  - Adds `labels`, `receivedDate` (from `internalDate` if present).
- **Expressions/variables:**
  - Pure JS mapping over `items`.
- **Inputs / outputs:**
  - **Input:** from `Gmail Check Every 2 Min`
  - **Output:** to `Classify and Extract Entities`
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - Missing fields in Gmail payload → handled by fallbacks, but `threadId/messageId` may be inaccurate if payload differs.
  - HTML stripping is simplistic; may produce noisy text for complex emails.
  - Very large emails: only first 3000 chars are analyzed (classification may miss critical info at end).

---

### Block 3 — AI Classification & Entity Extraction (LangChain)
**Overview:** Uses an OpenAI chat model with an agent prompt to classify the email and extract entities/action items, with structured output parsing.  
**Nodes involved:** `OpenAI Chat Model`, `Structured Output Parser`, `Classify and Extract Entities`

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM to the agent.
- **Key configuration choices:**
  - Model: `gpt-4.1-mini`
  - Temperature: `0.2` (more deterministic)
  - Max tokens: `1500`
- **Connections:**
  - **AI output (language model connector):** to `Classify and Extract Entities` (as `ai_languageModel`)
- **Version notes:** typeVersion `1.2` (model list/options differ across versions).
- **Edge cases / failures:**
  - OpenAI credential/API key missing or invalid.
  - Model availability changes; `gpt-4.1-mini` must exist in your account/region.
  - Token limits: long emails (even trimmed) + system prompt can still approach limits.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces a JSON-shaped output.
- **Key configuration choices:**
  - Provides a **schema example** with fields:
    - `category`, `priority`, `sentiment`
    - `entities` object containing arrays: names/companies/locations/dates/amounts/phoneNumbers/urls/keywords
    - `actionItems` array
    - `summary` string
- **Connections:**
  - **AI output parser connector:** to `Classify and Extract Entities` (as `ai_outputParser`)
- **Version notes:** typeVersion `1.3`.
- **Edge cases / failures:**
  - Parser can fail if the model emits non-JSON or partial JSON.
  - Example schema is not a strict JSON Schema; it guides formatting but does not guarantee compliance.

#### Node: Classify and Extract Entities
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — runs prompt + model + parser to produce structured classification.
- **Key configuration choices:**
  - Input text template:
    - `Sender`, `Subject`, `Body` from the cleaned email fields.
  - **System message** defines:
    - Allowed categories (9)
    - Allowed priority levels (High/Medium/Low)
    - Allowed sentiment (Positive/Neutral/Negative/Urgent)
    - Entity extraction requirements
    - Output must be valid JSON, no markdown/code blocks
    - Fallback rules: unclear → `Inquiry` and `Medium`
  - `hasOutputParser = true` so the parser is used.
- **Connections:**
  - **Main input:** from `Extract and Clean Email Content`
  - **Main output:** to `Parse and Validate AI Response`
  - **AI connectors:** receives model from `OpenAI Chat Model` and parser from `Structured Output Parser`
- **Version notes:** typeVersion `3`.
- **Edge cases / failures:**
  - Model may still output invalid JSON; parser/agent can throw.
  - Hallucinated categories/values (mitigated later by validation node).
  - Emails with little content: should fall back to Inquiry/Medium (prompted), but not guaranteed.

---

### Block 4 — Validation & Canonicalization of AI Output
**Overview:** Ensures the AI result is safe and predictable for routing/logging, enforcing allowed values and guaranteed array fields; produces a final normalized payload.  
**Nodes involved:** `Parse and Validate AI Response`

#### Node: Parse and Validate AI Response
- **Type / role:** `code` — normalizes and validates AI output; handles errors.
- **Key behaviors (interpreted):**
  - Reads AI output from `item.json.output` or uses `item.json` directly.
  - Requires `classification.category`; otherwise throws.
  - Ensures `entities` exists and contains array fields for:
    `names, companies, locations, dates, amounts, phoneNumbers, urls, keywords`
  - Ensures `actionItems` is an array (implemented using `actionKey = 'action' + 'Items'`).
  - Enforces allowed values:
    - Category must be one of: Inquiry, Support Request, Newsletter, Marketing, Personal, Urgent, Spam, Invoice/Receipt, Meeting Request  
      Otherwise coerces to `Inquiry`
    - Priority: High/Medium/Low → else `Medium`
    - Sentiment: Positive/Neutral/Negative/Urgent → else `Neutral`
  - Pulls original email fields from `$('Extract and Clean Email Content').first()` to ensure consistent IDs and metadata.
  - Outputs:
    - Original email info + `category/priority/sentiment`
    - `entities` object and `entitiesJson` string
    - `actionItems` array and `actionItemsJson` string
    - `summary`, `timestamp`
    - `classificationValid: true`
  - On exception:
    - Emits an “Unclassified” record with `classificationValid: false` and `classificationError`
    - Sets empty `actionItems` and minimal safe defaults
- **Connections:**
  - **Input:** from `Classify and Extract Entities`
  - **Outputs:** to both
    - `Check Classification Success`
    - `Check if High Priority or Urgent`
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - If upstream node naming changes, expression `$('Extract and Clean Email Content')` breaks.
  - If AI output structure differs (e.g., nested differently), the node may mark as failed.
  - `Unclassified` category is **not** part of routing switch rules; those items will go to the error logging branch, not labeling.

---

### Block 5 — Classification Success Gate + Category Routing
**Overview:** Splits flow into success (label + main log) vs failure (error log). Successful items are routed to one of 9 Gmail labeling branches.  
**Nodes involved:** `Check Classification Success`, `Route by Category`

#### Node: Check Classification Success
- **Type / role:** `if` — routes based on `classificationValid`.
- **Condition:**
  - `{{ $json.classificationValid }}` is boolean `true`
- **Connections:**
  - **True output (index 0):** to `Route by Category`
  - **False output (index 1):** to `Log Classification Errors`
- **Version notes:** typeVersion `2.2`
- **Edge cases / failures:**
  - If `classificationValid` is missing/non-boolean, strict type validation may behave unexpectedly.

#### Node: Route by Category
- **Type / role:** `switch` — routes to a label node per category.
- **Rules:** equals match on `{{ $json.category }}` for:
  1) Inquiry  
  2) Support Request  
  3) Newsletter  
  4) Marketing  
  5) Personal  
  6) Urgent  
  7) Spam  
  8) Invoice/Receipt  
  9) Meeting Request
- **Fallback:** enabled; `fallbackOutput = 3` (note: the value refers to output index; ensure it matches expectations in your n8n version). In this workflow, all 9 categories are covered; fallback should rarely be used due to upstream coercion.
- **Connections:**
  - Each output goes to the corresponding `Label: ...` Gmail node.
- **Version notes:** typeVersion `3.3`
- **Edge cases / failures:**
  - If category contains unexpected whitespace/case, equals match fails (mitigated by validation).
  - Fallback output index mismatches can route to an unintended branch in some configurations.

---

### Block 6 — Gmail Label Application (9 branches)
**Overview:** Applies a Gmail label based on the chosen category; each branch then forwards to the main Google Sheets log.  
**Nodes involved:**  
`Label: Inquiry`, `Label: Support Request`, `Label: Newsletter`, `Label: Marketing`, `Label: Personal`, `Label: Urgent`, `Label: Spam`, `Label: Invoice/Receipt`, `Label: Meeting Request`

**Important note:** In the provided JSON, the Gmail nodes are configured with operation `addLabels` but **no label IDs/names are specified**. In n8n, `addLabels` typically requires selecting labels. As-is, these nodes may do nothing or error depending on node version/UI defaults.

#### Common configuration across all “Label:” nodes
- **Type / role:** `gmail` — message operation.
- **Operation:** `addLabels`
- **Message ID expression:**
  - `={{ $('Extract and Clean Email Content').first().json.messageId }}`
- **Connections:**
  - **Input:** from `Route by Category` (specific output)
  - **Output:** to `Log All Emails`
- **Version notes:** typeVersion `2.1`
- **Edge cases / failures (common):**
  - Missing Gmail credentials / insufficient scopes (needs modify labels scope).
  - Incorrect `messageId` (must match Gmail message ID, not RFC Message-ID).
  - Labels not configured in node parameters → label operation may fail.
  - Expression coupling to node name `Extract and Clean Email Content`.

#### Node list (per branch)
- Label: Inquiry
- Label: Support Request
- Label: Newsletter
- Label: Marketing
- Label: Personal
- Label: Urgent
- Label: Spam
- Label: Invoice/Receipt
- Label: Meeting Request

---

### Block 7 — Logging (Google Sheets)
**Overview:** Appends a row for every successfully labeled email in a main sheet; logs failures to an “Error Log” sheet.  
**Nodes involved:** `Log All Emails`, `Log Classification Errors`

#### Node: Log All Emails
- **Type / role:** `googleSheets` — writes classification outcomes.
- **Operation:** `appendOrUpdate` with `useAppend = true` (effectively append).
- **Sheet:** `Sheet1` (Document ID not set in JSON; must be configured).
- **Columns mapped (interpreted):**
  - Subject, Summary, Category, Entities (JSON string), Priority, Sentiment, Timestamp
  - Message ID, Sender Name, Sender Email
  - Action Items (JSON string)
  - Labels Applied: `"{{ $json.category }}, {{ $json.priority }}"`
- **Credentials:** Google Sheets OAuth2 (placeholder in JSON).
- **Connections:**
  - **Inputs:** from all 9 label nodes
  - **No downstream outputs** shown (terminal)
- **Version notes:** typeVersion `4.7`
- **Edge cases / failures:**
  - Missing `documentId` → node cannot write until configured.
  - Sheet/tab `Sheet1` must exist.
  - Google API rate limits.
  - Large JSON strings might exceed cell limits for very entity-rich emails.

#### Node: Log Classification Errors
- **Type / role:** `googleSheets` — writes failed classification details.
- **Operation:** `appendOrUpdate` with append enabled.
- **Sheet:** `Error Log` (must exist).
- **Columns mapped:**
  - Subject, Timestamp, Message ID, Sender Email
  - Error Message (`classificationError`)
  - Email Body Preview: first 500 chars
- **Credentials:** Google Sheets OAuth2 (same placeholder)
- **Connections:**
  - **Input:** from `Check Classification Success` (false branch)
- **Version notes:** typeVersion `4.7`
- **Edge cases / failures:**
  - Missing/incorrect document ID or sheet tab.
  - If `bodyText` is empty, substring call in expression must still be valid (it is if `bodyText` is a string; validation ensures it in error path).

---

### Block 8 — Slack Escalation (High priority / urgent)
**Overview:** For items flagged as high priority or urgent, posts a detailed alert to a Slack channel with a Gmail link.  
**Nodes involved:** `Check if High Priority or Urgent`, `Send Urgent Email Alert`

#### Node: Check if High Priority or Urgent
- **Type / role:** `if` — escalation gate.
- **Condition (OR):**
  - `priority == "High"` OR
  - `category == "Urgent"` OR
  - `sentiment == "Urgent"`
- **Connections:**
  - **True output:** to `Send Urgent Email Alert`
  - **False output:** not connected (no action)
- **Version notes:** typeVersion `2.2`
- **Edge cases / failures:**
  - If fields missing or non-strings, strict validation can cause false negatives; validation node should ensure these exist.

#### Node: Send Urgent Email Alert
- **Type / role:** `slack` — posts message to a channel.
- **Key configuration choices:**
  - Mode: send to a selected channel (`select = channel`, `channelId` must be set).
  - Message template includes:
    - sender, subject, category, priority, sentiment
    - summary
    - action items as bullet list: `{{ $json.actionItems.join('\n- ') }}`
    - entity highlights (names, companies, dates, amounts)
    - Gmail thread link: `https://mail.google.com/mail/u/0/#inbox/{{ $json.threadId }}`
  - `continueOnFail = true` so Slack errors don’t fail the whole execution.
- **Connections:**
  - **Input:** from `Check if High Priority or Urgent`
- **Version notes:** typeVersion `2.3`
- **Edge cases / failures:**
  - Slack credentials missing/invalid; channel ID not set.
  - If `actionItems` is empty, `join` returns empty string (acceptable).
  - Gmail link assumes mailbox `/u/0/`; for Google Workspace multi-account setups this may not open the intended account.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Check Every 2 Min | gmailTrigger | Poll unread emails from Gmail | — | Extract and Clean Email Content | ## 1. Email Ingestion<br>Gmail inbox checked every 2 minutes, email is parsed for agent. |
| Extract and Clean Email Content | code | Normalize and clean email content for AI | Gmail Check Every 2 Min | Classify and Extract Entities | ## 2. AI Classification & Entity Extraction<br>Agent looks through classifications in "System Message" (Inquiry, Support, Newsletter, Marketing, Personal, Urgent, Spam, Invoice, Meeting). Parser validates response. High priority emails are sent to Slack. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for agent | — | Classify and Extract Entities (ai_languageModel) | ## 2. AI Classification & Entity Extraction<br>Agent looks through classifications in "System Message" (Inquiry, Support, Newsletter, Marketing, Personal, Urgent, Spam, Invoice, Meeting). Parser validates response. High priority emails are sent to Slack. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON output | — | Classify and Extract Entities (ai_outputParser) | ## 2. AI Classification & Entity Extraction<br>Agent looks through classifications in "System Message" (Inquiry, Support, Newsletter, Marketing, Personal, Urgent, Spam, Invoice, Meeting). Parser validates response. High priority emails are sent to Slack. |
| Classify and Extract Entities | @n8n/n8n-nodes-langchain.agent | Classify email + extract entities/action items | Extract and Clean Email Content | Parse and Validate AI Response | ## 2. AI Classification & Entity Extraction<br>Agent looks through classifications in "System Message" (Inquiry, Support, Newsletter, Marketing, Personal, Urgent, Spam, Invoice, Meeting). Parser validates response. High priority emails are sent to Slack. |
| Parse and Validate AI Response | code | Validate/coerce AI output into canonical fields | Classify and Extract Entities | Check Classification Success; Check if High Priority or Urgent | ## 2. AI Classification & Entity Extraction<br>Agent looks through classifications in "System Message" (Inquiry, Support, Newsletter, Marketing, Personal, Urgent, Spam, Invoice, Meeting). Parser validates response. High priority emails are sent to Slack. |
| Check if High Priority or Urgent | if | Decide if Slack escalation is needed | Parse and Validate AI Response | Send Urgent Email Alert | ## 2. AI Classification & Entity Extraction<br>Agent looks through classifications in "System Message" (Inquiry, Support, Newsletter, Marketing, Personal, Urgent, Spam, Invoice, Meeting). Parser validates response. High priority emails are sent to Slack. |
| Send Urgent Email Alert | slack | Send Slack alert for urgent/high emails | Check if High Priority or Urgent | — | ## 5. Slack Escalation<br>Slack alerts are generated for high-priority/urgent emails |
| Check Classification Success | if | Split success vs error logging | Parse and Validate AI Response | Route by Category; Log Classification Errors | ## 4. Error Handling & Logging<br>If classification fails:<br>- Email gets "Unclassified" category<br>- Logged to separate Error Log sheet<br>- Admin can manually review and reclassify<br>Google Sheets:<br>- Main log: All classified emails with full metadata<br>- Error log: Failed classifications for manual review |
| Route by Category | switch | Route to category-specific Gmail labeling branch | Check Classification Success | Label: Inquiry; Label: Support Request; Label: Newsletter; Label: Marketing; Label: Personal; Label: Urgent; Label: Spam; Label: Invoice/Receipt; Label: Meeting Request | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Label: Inquiry | gmail | Apply “Inquiry” label to Gmail message | Route by Category | Log All Emails | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Label: Support Request | gmail | Apply “Support Request” label | Route by Category | Log All Emails | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Label: Newsletter | gmail | Apply “Newsletter” label | Route by Category | Log All Emails | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Label: Marketing | gmail | Apply “Marketing” label | Route by Category | Log All Emails | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Label: Personal | gmail | Apply “Personal” label | Route by Category | Log All Emails | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Label: Urgent | gmail | Apply “Urgent” label | Route by Category | Log All Emails | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Label: Spam | gmail | Apply “Spam” label | Route by Category | Log All Emails | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Label: Invoice/Receipt | gmail | Apply “Invoice/Receipt” label | Route by Category | Log All Emails | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Label: Meeting Request | gmail | Apply “Meeting Request” label | Route by Category | Log All Emails | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Log All Emails | googleSheets | Append classified email record to main sheet | Label: Inquiry; Label: Support Request; Label: Newsletter; Label: Marketing; Label: Personal; Label: Urgent; Label: Spam; Label: Invoice/Receipt; Label: Meeting Request | — | ## 4. Error Handling & Logging<br>If classification fails:<br>- Email gets "Unclassified" category<br>- Logged to separate Error Log sheet<br>- Admin can manually review and reclassify<br>Google Sheets:<br>- Main log: All classified emails with full metadata<br>- Error log: Failed classifications for manual review |
| Log Classification Errors | googleSheets | Append failed classification details to error sheet | Check Classification Success (false) | — | ## 4. Error Handling & Logging<br>If classification fails:<br>- Email gets "Unclassified" category<br>- Logged to separate Error Log sheet<br>- Admin can manually review and reclassify<br>Google Sheets:<br>- Main log: All classified emails with full metadata<br>- Error log: Failed classifications for manual review |
| Email Ingestion | stickyNote | Comment | — | — | ## 1. Email Ingestion<br>Gmail inbox checked every 2 minutes, email is parsed for agent. |
| AI Classification | stickyNote | Comment | — | — | ## 2. AI Classification & Entity Extraction<br>Agent looks through classifications in "System Message" (Inquiry, Support, Newsletter, Marketing, Personal, Urgent, Spam, Invoice, Meeting). Parser validates response. High priority emails are sent to Slack. |
| Routing Logic | stickyNote | Comment | — | — | ## 3. Routing & Labeling<br>Switch node routes to 9 category-specific branches.<br>Each branch applies appropriate Gmail labels:<br>- Category label (e.g., 📧 Support)<br>- Priority label (🔴 High, 🟡 Medium, 🟢 Low)<br>- Sentiment label if negative/urgent<br><br>Urgent emails are starred automatically. |
| Integrations | stickyNote | Comment | — | — | ## 4. Error Handling & Logging<br>If classification fails:<br>- Email gets "Unclassified" category<br>- Logged to separate Error Log sheet<br>- Admin can manually review and reclassify<br>Google Sheets:<br>- Main log: All classified emails with full metadata<br>- Error log: Failed classifications for manual review |
| Error Handling | stickyNote | Comment | — | — | ## 5. Slack Escalation<br>Slack alerts are generated for high-priority/urgent emails |
| Email Ingestion1 | stickyNote | Comment | — | — | ## Classify Gmail with AI auto-labeler<br><br>This n8n workflow automates email classification in a Gmail inbox.<br><br>## How it works<br>1. **Gmail Intake:** A Gmail inbox is monitored for and scanned incoming emails.<br>2. **Classification and Extraction:** From the email the AI extracts category, priority, sentiment, etc. and parses the email. It also checks if it's high priority.<br>3. **Routing and Labeling:** Emails are labeled accordingly to user preferences.<br>4. **Error Handling and Logging:**  All emails are recorded in Google Sheets, and any emails that could not be labeled are flagged.  <br>5. **Slack Escalation:**  High priority emails are sent to a Slack channel<br><br>## Setup steps<br>- [ ] Connect **Gmail** account<br>- [ ] Connect **OpenAI Chat Model** and API key<br>- [ ] Connect **Slack** account<br>- [ ] Connect **Google Drive/Sheets** for logging<br><br>## Customization<br>1. Set up email classification categories in "System Message" of AI agent if needed<br>2. Change Code node to match any changes to the categories in the agent<br>2. Change labels on email categories on Gmail nodes to match |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger: Gmail Trigger**
   - Node: **Gmail Trigger**
   - Name: `Gmail Check Every 2 Min`
   - Credentials: connect Gmail OAuth2 in n8n (with permissions to read messages).
   - Filters: `Read status = Unread`
   - Polling: set to your desired interval (the JSON uses “every minute”; rename if desired).
   - Connect to next node.

2) **Add Code node: email normalization**
   - Node: **Code**
   - Name: `Extract and Clean Email Content`
   - Paste the logic that:
     - extracts sender name/email, subject
     - prefers plain text body and strips HTML if needed
     - truncates to 3000 chars
     - sets `messageId`, `threadId`, `receivedDate`, `labels`
   - Connect `Gmail Trigger → Extract and Clean Email Content`.

3) **Add OpenAI Chat Model (LangChain)**
   - Node: **OpenAI Chat Model** (LangChain)
   - Name: `OpenAI Chat Model`
   - Credentials: set OpenAI API key in n8n.
   - Model: `gpt-4.1-mini`
   - Options: temperature `0.2`, max tokens `1500`

4) **Add Structured Output Parser**
   - Node: **Structured Output Parser** (LangChain)
   - Name: `Structured Output Parser`
   - Configure the example JSON structure containing:
     - category, priority, sentiment
     - entities object with arrays (names, companies, locations, dates, amounts, phoneNumbers, urls, keywords)
     - actionItems array
     - summary string

5) **Add Agent node**
   - Node: **AI Agent** (LangChain Agent)
   - Name: `Classify and Extract Entities`
   - Text (prompt input):  
     - `Sender: {{$json.senderName}} ({{$json.senderEmail}})\nSubject: {{$json.subject}}\nBody: {{$json.bodyText}}`
   - System message: include:
     - exact allowed categories/priorities/sentiments
     - entity extraction instructions
     - “Return valid JSON only; no markdown”
   - Enable output parser usage (the node has `hasOutputParser`).
   - Connect:
     - `Extract and Clean Email Content → Classify and Extract Entities`
     - `OpenAI Chat Model` to the Agent via the **AI Language Model** connection
     - `Structured Output Parser` to the Agent via the **AI Output Parser** connection

6) **Add Code node: parse/validate AI response**
   - Node: **Code**
   - Name: `Parse and Validate AI Response`
   - Implement:
     - coercion/validation for category/priority/sentiment
     - guaranteed `entities` and `actionItems` arrays
     - `classificationValid` boolean and error fallback to `Unclassified`
     - include `entitiesJson` and `actionItemsJson`
   - Connect `Classify and Extract Entities → Parse and Validate AI Response`.

7) **Add IF node: Slack escalation gate**
   - Node: **IF**
   - Name: `Check if High Priority or Urgent`
   - Conditions (OR):
     - `{{$json.priority}} == "High"` OR `{{$json.category}} == "Urgent"` OR `{{$json.sentiment}} == "Urgent"`
   - Connect `Parse and Validate AI Response → Check if High Priority or Urgent`.

8) **Add Slack node: send alert**
   - Node: **Slack**
   - Name: `Send Urgent Email Alert`
   - Credentials: connect Slack in n8n (bot token or OAuth depending on node).
   - Operation: send message to channel
   - Choose Channel ID
   - Message body: include fields from the normalized JSON and Gmail thread link using `{{$json.threadId}}`
   - Set **Continue On Fail = true**
   - Connect `Check if High Priority or Urgent (true) → Send Urgent Email Alert`.

9) **Add IF node: success vs failure**
   - Node: **IF**
   - Name: `Check Classification Success`
   - Condition: `{{$json.classificationValid}} is true`
   - Connect `Parse and Validate AI Response → Check Classification Success`.

10) **Add Switch node: route by category**
   - Node: **Switch**
   - Name: `Route by Category`
   - Value to evaluate: `{{$json.category}}`
   - Add 9 rules for equals:
     - Inquiry, Support Request, Newsletter, Marketing, Personal, Urgent, Spam, Invoice/Receipt, Meeting Request
   - Connect `Check Classification Success (true) → Route by Category`.

11) **Add 9 Gmail nodes: apply labels**
   - For each category, add a **Gmail** node:
     - Name: `Label: <Category>`
     - Operation: **Add Labels**
     - Message ID: `={{ $('Extract and Clean Email Content').first().json.messageId }}`
     - Select the Gmail label(s) to apply (create them in Gmail first if needed).
   - Connect each switch output to its corresponding label node.

12) **Add Google Sheets node: main logging**
   - Node: **Google Sheets**
   - Name: `Log All Emails`
   - Credentials: Google Sheets OAuth2
   - Document: select your spreadsheet (set **Document ID**)
   - Sheet/tab: `Sheet1`
   - Operation: append (in this node: appendOrUpdate with append enabled)
   - Map columns to fields like:
     - Subject, Summary, Category, Entities (entitiesJson), Priority, Sentiment, Timestamp, Message ID, Sender Name, Sender Email, Action Items (actionItemsJson)
   - Connect all 9 label nodes → `Log All Emails`.

13) **Add Google Sheets node: error logging**
   - Node: **Google Sheets**
   - Name: `Log Classification Errors`
   - Same credentials and document, but sheet/tab: `Error Log`
   - Map columns: Subject, Timestamp, Message ID, Sender Email, Error Message, Email Body Preview
   - Connect `Check Classification Success (false) → Log Classification Errors`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Gmail polling interval in the node name (“Every 2 Min”) does not match the actual config (“every minute”). | Adjust poll time or rename node for accuracy. |
| Gmail “addLabels” nodes appear to lack explicit label selection in the provided JSON. | Ensure each Gmail node is configured with actual label(s) to apply, otherwise labeling may fail or do nothing. |
| Gmail link in Slack uses `mail/u/0`. | For multi-account environments, adjust `/u/<index>/` or use a more robust URL strategy. |
| Setup checklist (from sticky note): Connect Gmail, OpenAI model/API key, Slack, Google Drive/Sheets. | Embedded in sticky note “Classify Gmail with AI auto-labeler”. |
| Customization note: update categories in the Agent system message, then update validation code and Gmail label nodes accordingly. | Embedded in sticky note “Classify Gmail with AI auto-labeler”. |