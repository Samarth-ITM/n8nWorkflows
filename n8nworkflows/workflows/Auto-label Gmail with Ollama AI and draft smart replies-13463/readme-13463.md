Auto-label Gmail with Ollama AI and draft smart replies

https://n8nworkflows.xyz/workflows/auto-label-gmail-with-ollama-ai-and-draft-smart-replies-13463


# Auto-label Gmail with Ollama AI and draft smart replies

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow monitors a Gmail inbox for **unread** emails, **extracts and cleans** the email content, sends it to a **local Ollama LLM** for classification and reply drafting, then **routes** emails by category to apply the appropriate **Gmail label** and (for important categories) **creates a Gmail reply draft**. Finally, it logs the result into **Google Sheets**.

**Target use cases:**
- Local/zero-paid-API email triage using Ollama.
- Inbox organization with consistent labels.
- Drafting quick, context-aware replies for messages that require attention.
- Tracking triage results over time for analytics (Sheets).

### Logical blocks
**1.1 Email Intake (Trigger + normalization)**  
Watches unread emails, extracts headers/body robustly across Gmail output formats, and normalizes fields for AI.

**1.2 AI Classification (Ollama + parsing)**  
Sends normalized email to an LLM agent (Ollama model), expects strict JSON output, then parses/validates classification.

**1.3 Route, Label & Draft (Switch + Gmail actions)**  
Routes by category to add the correct Gmail label; creates reply drafts for urgent and action-required.

**1.4 Logging (Prepare row + Google Sheets append)**  
Consolidates outputs back into a single format and appends a row to Google Sheets.

---

## 2. Block-by-Block Analysis

### 2.1 Email Intake (Trigger + normalization)

**Overview:**  
Polls Gmail every 30 minutes for unread emails. Then a Code node normalizes sender/subject/date/body, detects flags (attachments/reply/forward), and outputs clean fields for AI.

**Nodes involved:**
- 📥 Gmail Trigger
- 🔧 Extract Email Data

#### Node: 📥 Gmail Trigger
- **Type / role:** `n8n-nodes-base.gmailTrigger` — Polling trigger for new emails.
- **Configuration choices (interpreted):**
  - Poll interval: **every 30 minutes**
  - Filter: **readStatus = unread**
  - “Simple” mode disabled (`simple: false`) to provide richer payloads (varies by node version).
- **Key fields produced:** message metadata (id/threadId/headers/body/snippet) depending on Gmail node version and settings.
- **Connections:**
  - Output → **🔧 Extract Email Data**
- **Credentials:** Gmail OAuth2 (“Gmail account”)
  - Requires scopes sufficient for reading email; later nodes require modify/compose.
- **Version-specific notes:** `typeVersion 1.2`  
  Gmail trigger output structure can differ across versions; the next node explicitly handles multiple formats.
- **Edge cases / failures:**
  - OAuth token expiration / insufficient scopes.
  - Polling returns no items (workflow continues only when items exist).
  - Rate limits if interval is too low on large mailboxes.

#### Node: 🔧 Extract Email Data
- **Type / role:** `n8n-nodes-base.code` — JavaScript normalization and cleaning.
- **Configuration choices (interpreted):**
  - Iterates over all incoming items and builds `processedEmails[]`.
  - Robustly extracts headers from multiple possible Gmail payload shapes:
    - Flat fields (`email.from`, `email.subject`, etc.)
    - `email.payload.headers[]`
    - `email.headers` object
  - Extracts sender name/email/domain using regex and fallbacks.
  - Extracts body text using multiple strategies:
    - `text`, `textPlain`, `snippet`, `textHtml`
    - base64 decode of `payload.body.data`
    - multipart parts (`text/plain` then `text/html`)
  - Cleans body:
    - strips HTML tags, normalizes whitespace/newlines, decodes common entities
    - truncates to **3000 chars** for AI input control
  - Detects flags:
    - `hasAttachments` via payload parts/attachments arrays
    - `isReply` subject starts with `re:`
    - `isForward` subject starts with `fwd:` or `fw:`
  - Outputs standardized JSON fields like:
    - `gmailId`, `threadId`, `from`, `senderEmail`, `senderDomain`, `subject`, `date`, `bodyText`, `bodyPreview`, flags, `processedAt`
  - If zero emails: returns a single item `{ error: 'No emails to process' }`
- **Key expressions/variables:**
  - Uses `Buffer.from(..., 'base64').toString('utf-8')` for Gmail payload decoding.
- **Connections:**
  - Input ← **📥 Gmail Trigger**
  - Output → **🤖 AI Email Classifier**
- **Version-specific notes:** `typeVersion 2` (Code node v2)
- **Edge cases / failures:**
  - Some Gmail bodies are only HTML; stripping tags may remove structure and reduce classification quality.
  - Very large emails are truncated, which can omit crucial info (mitigation: adjust 3000 limit).
  - Attachments detection may miss attachments if Gmail payload structure differs.
  - If trigger returns minimal fields (depending on settings), extraction may rely on `snippet`.

---

### 2.2 AI Classification (Ollama + parsing)

**Overview:**  
A LangChain Agent sends the normalized email to a local Ollama chat model with a strict system prompt requiring raw JSON output. A subsequent Code node attempts to robustly extract/validate that JSON and applies defaults on failure.

**Nodes involved:**
- Ollama Chat Model
- 🤖 AI Email Classifier
- 🎯 Extract Classification

#### Node: Ollama Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOllama` — LLM connector to a local Ollama instance.
- **Configuration choices (interpreted):**
  - Model: `mistral` (default)
  - No special options set.
- **Connections:**
  - Output (AI language model) → **🤖 AI Email Classifier** (as its language model)
- **Credentials:** Ollama API (“Ollama account”)
  - Typically points to local Ollama base URL (e.g., `http://localhost:11434`) depending on your credential setup.
- **Version-specific notes:** `typeVersion 1`
- **Edge cases / failures:**
  - Ollama not running / wrong host / firewall.
  - Model not pulled (`ollama pull mistral`) causing model-not-found.
  - Slow inference/timeouts on large emails or small hardware.

#### Node: 🤖 AI Email Classifier
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Agent that prompts the model to classify and optionally draft replies.
- **Configuration choices (interpreted):**
  - Prompt includes structured email data fields (from/to/cc/subject/date/flags/domain) + body text.
  - System message defines:
    - 6 strict categories: `urgent`, `action_required`, `follow_up`, `newsletter`, `automated`, `spam_promo`
    - Reply draft rules: only for urgent/action_required/follow_up
    - Output constraints: **RAW JSON only**, no markdown fences; schema includes category, priority, confidence, reason, summary, keyAction, replyDraft, replySubject, suggestedLabel, shouldAutoArchive, sentimentTone.
  - `hasOutputParser: true` (node may attempt structured parsing, but downstream still validates).
- **Key expressions/variables used:**
  - Uses n8n expressions in prompt: `{{ $json.from }}`, `{{ $json.subject }}`, `{{ $json.bodyText }}`, etc.
- **Connections:**
  - Input ← **🔧 Extract Email Data**
  - Output → **🎯 Extract Classification**
  - AI Language Model input ← **Ollama Chat Model**
- **Version-specific notes:** `typeVersion 2.1`
- **Edge cases / failures:**
  - Model may return non-JSON despite instructions (handled downstream).
  - Hallucinated fields or invalid category strings (handled downstream).
  - Prompt injection risk: email body could try to override instructions; system message helps, but consider additional sanitization or stricter parsing.

#### Node: 🎯 Extract Classification
- **Type / role:** `n8n-nodes-base.code` — Parses/normalizes the agent output into a stable schema.
- **Configuration choices (interpreted):**
  - Reads original email data from **🔧 Extract Email Data** via: `$('🔧 Extract Email Data').item.json`
  - Attempts multiple extraction strategies:
    - If `$json.category` exists, uses it directly
    - Recursively searches common properties (`output`, `text`, `message`, `content`, `result`, etc.)
    - Tries JSON.parse after stripping ``` fences
    - Regex fallback to find a JSON object containing `"category"`
  - Validates `category` against allowed set:
    - If invalid/unparseable → defaults to:
      - `category: 'follow_up'`, priority 2, confidence 0.5, reason indicates parsing failure
  - Outputs a consolidated object containing:
    - Email metadata + classification fields + `processedAt`
- **Connections:**
  - Input ← **🤖 AI Email Classifier**
  - Output → **🔀 Route by Category**
- **Version-specific notes:** `typeVersion 2`
- **Edge cases / failures:**
  - If upstream node names are changed, `$('🔧 Extract Email Data')` lookup breaks.
  - Multi-item execution: using `.item` references a single item; if multiple emails arrive at once, this can mismatch if not aligned. (Current code assumes item alignment; consider using `$input.item` or merging within same item path.)
  - If AI output is deeply nested in an unexpected structure, extraction may still fail (defaults help).

---

### 2.3 Route, Label & Draft (Switch + Gmail actions)

**Overview:**  
Routes by AI category. Each route applies a Gmail label. For `urgent` and `action_required`, it also creates a Gmail draft reply in the same thread.

**Nodes involved:**
- 🔀 Route by Category
- 🏷️ Label: Urgent
- 📝 Draft Reply (Urgent)
- 🏷️ Label: Action Required
- 📝 Draft Reply (Action)
- 🏷️ Label: Follow-up
- 🏷️ Label: Newsletter
- 🏷️ Label: Automated
- 🏷️ Label: Spam-Promo

#### Node: 🔀 Route by Category
- **Type / role:** `n8n-nodes-base.switch` — Conditional routing into 6 category paths.
- **Configuration choices (interpreted):**
  - 6 rules, each checks: `{{ $json.category }}` equals one of:
    - urgent, action_required, follow_up, newsletter, automated, spam_promo
  - `allMatchingOutputs: false` so only the first matching route executes.
  - Each rule has “renameOutput” enabled (primarily UI; logic is standard).
- **Connections:**
  - Input ← **🎯 Extract Classification**
  - Outputs:
    - urgent → **🏷️ Label: Urgent**
    - action_required → **🏷️ Label: Action Required**
    - follow_up → **🏷️ Label: Follow-up**
    - newsletter → **🏷️ Label: Newsletter**
    - automated → **🏷️ Label: Automated**
    - spam_promo → **🏷️ Label: Spam-Promo**
- **Version-specific notes:** `typeVersion 3.2`
- **Edge cases / failures:**
  - If category contains whitespace/case differences, it won’t match (classifier enforces exact strings; the extractor also validates).

#### Node: 🏷️ Label: Urgent
- **Type / role:** `n8n-nodes-base.gmail` — Adds a Gmail label to the message.
- **Configuration choices (interpreted):**
  - Operation: `addLabels`
  - Message ID: `{{ $json.gmailId }}`
  - Label IDs: `YOUR_LABEL_ID_URGENT` (placeholder; must be replaced with real Gmail label ID)
- **Connections:**
  - Input ← **🔀 Route by Category** (urgent output)
  - Output → **📝 Draft Reply (Urgent)**
- **Credentials:** Gmail OAuth2
- **Version-specific notes:** `typeVersion 2.2`
- **Edge cases / failures:**
  - Label ID invalid/not found.
  - Missing `gmailId` if trigger payload is unexpected.
  - Insufficient Gmail scope for modifying labels.

#### Node: 📝 Draft Reply (Urgent)
- **Type / role:** `n8n-nodes-base.gmail` — Creates a Gmail draft reply.
- **Configuration choices (interpreted):**
  - Resource: `draft`
  - Subject: `{{ $json.replySubject }}`
  - Message body: `{{ $json.replyDraft }}`
  - Options: `threadId: {{ $json.threadId }}` to keep the reply in-thread.
- **Connections:**
  - Input ← **🏷️ Label: Urgent**
  - Output → **📋 Prepare Log Entry**
- **Credentials:** Gmail OAuth2 (requires compose/draft permissions)
- **Version-specific notes:** `typeVersion 2.2`
- **Edge cases / failures:**
  - Empty `replyDraft` (AI rules say it should exist for urgent; but parsing/defaults may produce empty).
  - Missing/invalid `threadId` may create a new draft not threaded (or fail depending on API behavior).
  - Draft creation rate limits.

#### Node: 🏷️ Label: Action Required
- **Type / role:** Gmail addLabels.
- **Configuration:**
  - `labelIds: ["YOUR_LABEL_ID_ACTION_REQUIRED"]`
  - `messageId: {{ $json.gmailId }}`
- **Connections:**
  - Input ← switch (action_required)
  - Output → **📝 Draft Reply (Action)**
- **Credentials / version / edge cases:** same pattern as urgent label node.

#### Node: 📝 Draft Reply (Action)
- **Type / role:** Gmail draft create.
- **Configuration:**
  - `message: {{ $json.replyDraft }}`
  - `subject: {{ $json.replySubject }}`
  - `options.threadId: {{ $json.threadId }}`
- **Connections:**
  - Input ← **🏷️ Label: Action Required**
  - Output → **📋 Prepare Log Entry**
- **Edge cases:** same as urgent draft.

#### Node: 🏷️ Label: Follow-up
- **Type / role:** Gmail addLabels.
- **Configuration:** `labelIds: ["YOUR_LABEL_ID_FOLLOW_UP"]`, `messageId: {{ $json.gmailId }}`
- **Connections:** Input ← switch (follow_up) → Output → **📋 Prepare Log Entry**
- **Note:** No draft is created here, even though the AI prompt allows drafts for follow_up. The workflow currently only drafts for urgent/action_required.

#### Node: 🏷️ Label: Newsletter
- **Type / role:** Gmail addLabels.
- **Configuration:** `labelIds: ["YOUR_LABEL_ID_NEWSLETTER"]`
- **Connections:** → **📋 Prepare Log Entry**

#### Node: 🏷️ Label: Automated
- **Type / role:** Gmail addLabels.
- **Configuration:** `labelIds: ["YOUR_LABEL_ID_AUTOMATED"]`
- **Connections:** → **📋 Prepare Log Entry**

#### Node: 🏷️ Label: Spam-Promo
- **Type / role:** Gmail addLabels.
- **Configuration:** `labelIds: ["YOUR_LABEL_ID_SPAM_PROMO"]`
- **Connections:** → **📋 Prepare Log Entry**

**Common edge cases for labeling nodes:**
- Gmail labels must exist and the node must use **label IDs**, not label names.
- Message already has label (usually harmless).
- If Gmail trigger doesn’t include message `id`, downstream will fail.

---

### 2.4 Logging (Prepare row + Google Sheets append)

**Overview:**  
All category paths converge into a code node that prepares a single row structure for Sheets, then appends it to a configured Google Sheet tab.

**Nodes involved:**
- 📋 Prepare Log Entry
- 📊 Log to Sheet

#### Node: 📋 Prepare Log Entry
- **Type / role:** `n8n-nodes-base.code` — Normalizes data into a Google Sheets row.
- **Configuration choices (interpreted):**
  - Takes the first incoming item: `$input.all()[0].json`
  - Attempts to fetch canonical classification from `$('🎯 Extract Classification').item.json`; fallback to current item if lookup fails.
  - Produces row fields:
    - Date, From, Subject, Category, Priority, Confidence, Reason, Summary, Key Action,
      Has Reply Draft, Sentiment, Auto Archive, Gmail ID, Processed At
- **Connections:**
  - Inputs ← all labeling/draft nodes (each route merges here)
  - Output → **📊 Log to Sheet**
- **Edge cases / failures:**
  - Multi-item batches: `$input.all()[0]` logs only the first item if multiple arrive in a single execution path.
  - Cross-item mismatch risk again due to `$('🎯 Extract Classification').item.json` if multiple items are processed simultaneously.
  - If classification fields are missing, logging still proceeds with blanks.

#### Node: 📊 Log to Sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — Appends a row to Google Sheets.
- **Configuration choices (interpreted):**
  - Operation: `append`
  - Spreadsheet: placeholder `YOUR_GOOGLE_SHEET_ID`
  - Sheet/tab: “log entry” (gid `772918340` in cached URL)
  - Mapping mode: auto-map input fields to columns defined in node schema.
- **Connections:**
  - Input ← **📋 Prepare Log Entry**
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account”)
- **Version-specific notes:** `typeVersion 4.6`
- **Edge cases / failures:**
  - Spreadsheet ID/tab not found or permission denied.
  - Column mismatch if sheet headers differ from expected names.
  - Google API quotas.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Main Overview | stickyNote | Documentation / overview panel |  |  | ## Classify Gmail emails with AI and auto-label with smart reply drafts\n\nAutomatically classify every incoming email using a local AI model (Ollama), apply Gmail labels, and generate smart reply drafts for important messages — zero paid APIs required.\n\n### How it works\n1. **Gmail Trigger** watches your inbox for new unread emails every 30 minutes.\n2. **Extract Email Data** parses the sender, subject, body, and metadata into clean fields.\n3. **AI Classifier** (Ollama, local) analyzes the email and classifies it into one of 6 categories: 🔴 Urgent, 📋 Action Required, 💬 Follow-up, 📰 Newsletter, 🤖 Automated, or 🚫 Spam/Promotional.\n4. **Smart Router** sends each email down its dedicated path based on classification.\n5. **Gmail Labels** are automatically applied so your inbox is organized without lifting a finger.\n6. **Reply Drafts** are generated by AI for Urgent and Action Required emails — ready for you to review, edit, and send.\n7. **Summary Log** tracks every processed email in Google Sheets for analytics and review.\n\n### Setup steps\n1. **Gmail OAuth** — Connect your Google account with read, modify, and compose permissions.\n2. **Create Gmail Labels** — In Gmail, manually create these labels:\n   - `🔴 Urgent`, `📋 Action Required`, `💬 Follow-up`, `📰 Newsletter`, `🤖 Automated`, `🚫 Spam-Promo`\n3. **Get Label IDs** — Use Gmail API or n8n to find each label's ID. Update the `labelIds` in each Gmail Label node.\n4. **Ollama** — Ensure Ollama is running locally with your preferred model pulled (default: `mistral`). Change the model name in the Ollama Chat Model node if needed.\n5. **Google Sheets** (optional) — Connect your credential and set a spreadsheet ID for the logging node.\n6. **Test** — Send yourself test emails (urgent, newsletter, promotional) and run manually to verify accuracy.\n\n### Customization\n- Swap `mistral` for `llama3`, `gemma2`, or any Ollama model.\n- Add more categories by editing the AI prompt and adding Switch outputs.\n- Change the reply draft tone (formal, casual, friendly) in the AI prompt.\n- Add Slack/Telegram notifications for urgent emails.\n- Add auto-archive for newsletters and spam.\n- Decrease the polling interval for near-real-time processing.\n- Add sender whitelist/blacklist logic before the AI node. |
| Sticky Note - Email Intake | stickyNote | Documentation: Email intake block |  |  | ## 📥 Email Intake\nGmail trigger watches inbox for unread emails, then extracts and cleans email data |
| Sticky Note - AI Classification | stickyNote | Documentation: AI classification block |  |  | ## 🤖 AI Classification\nOllama analyzes email content and returns category, priority, and reply draft |
| Sticky Note - Route and Label | stickyNote | Documentation: Routing/labeling block |  |  | ## 🔀 Route, Label & Draft\nSwitch by category, apply Gmail labels, create reply drafts for important emails |
| Sticky Note - Logging | stickyNote | Documentation: Logging block |  |  | ## 📊 Logging\nEvery processed email is logged to Google Sheets for analytics |
| Sticky Note - Warning Labels | stickyNote | Documentation: Label prerequisite warning |  |  | ## ⚠️ Create Gmail Labels First\nYou must manually create these labels in Gmail before activating:\n`🔴 Urgent`, `📋 Action Required`, `💬 Follow-up`, `📰 Newsletter`, `🤖 Automated`, `🚫 Spam-Promo`\nThen update the Label IDs in each labeling node. |
| Sticky Note - Warning Ollama | stickyNote | Documentation: Ollama prerequisite warning |  |  | ## ⚠️ Check Ollama Model\nDefault model: `mistral`. Make sure it's pulled:\n`ollama pull mistral`\nOr change to `llama3`, `gemma2`, etc. |
| 📥 Gmail Trigger | gmailTrigger | Poll unread emails | (Entry) | 🔧 Extract Email Data | ## 📥 Email Intake\nGmail trigger watches inbox for unread emails, then extracts and cleans email data |
| 🔧 Extract Email Data | code | Normalize headers/body/flags for AI | 📥 Gmail Trigger | 🤖 AI Email Classifier | ## 📥 Email Intake\nGmail trigger watches inbox for unread emails, then extracts and cleans email data |
| Ollama Chat Model | lmChatOllama | Local LLM provider for agent | (none; model provider) | 🤖 AI Email Classifier (ai_languageModel) | ## 🤖 AI Classification\nOllama analyzes email content and returns category, priority, and reply draft |
| 🤖 AI Email Classifier | langchain.agent | Classify and (optionally) draft reply text | 🔧 Extract Email Data | 🎯 Extract Classification | ## 🤖 AI Classification\nOllama analyzes email content and returns category, priority, and reply draft |
| 🎯 Extract Classification | code | Parse/validate AI JSON; apply defaults | 🤖 AI Email Classifier | 🔀 Route by Category | ## 🤖 AI Classification\nOllama analyzes email content and returns category, priority, and reply draft |
| 🔀 Route by Category | switch | Route to labeling/drafting path | 🎯 Extract Classification | 🏷️ Label: Urgent; 🏷️ Label: Action Required; 🏷️ Label: Follow-up; 🏷️ Label: Newsletter; 🏷️ Label: Automated; 🏷️ Label: Spam-Promo | ## 🔀 Route, Label & Draft\nSwitch by category, apply Gmail labels, create reply drafts for important emails |
| 🏷️ Label: Urgent | gmail | Add “Urgent” label | 🔀 Route by Category | 📝 Draft Reply (Urgent) | ## 🔀 Route, Label & Draft\nSwitch by category, apply Gmail labels, create reply drafts for important emails |
| 📝 Draft Reply (Urgent) | gmail | Create draft reply in thread | 🏷️ Label: Urgent | 📋 Prepare Log Entry | ## 🔀 Route, Label & Draft\nSwitch by category, apply Gmail labels, create reply drafts for important emails |
| 🏷️ Label: Action Required | gmail | Add “Action Required” label | 🔀 Route by Category | 📝 Draft Reply (Action) | ## 🔀 Route, Label & Draft\nSwitch by category, apply Gmail labels, create reply drafts for important emails |
| 📝 Draft Reply (Action) | gmail | Create draft reply in thread | 🏷️ Label: Action Required | 📋 Prepare Log Entry | ## 🔀 Route, Label & Draft\nSwitch by category, apply Gmail labels, create reply drafts for important emails |
| 🏷️ Label: Follow-up | gmail | Add “Follow-up” label | 🔀 Route by Category | 📋 Prepare Log Entry | ## 🔀 Route, Label & Draft\nSwitch by category, apply Gmail labels, create reply drafts for important emails |
| 🏷️ Label: Newsletter | gmail | Add “Newsletter” label | 🔀 Route by Category | 📋 Prepare Log Entry | ## 🔀 Route, Label & Draft\nSwitch by category, apply Gmail labels, create reply drafts for important emails |
| 🏷️ Label: Automated | gmail | Add “Automated” label | 🔀 Route by Category | 📋 Prepare Log Entry | ## 🔀 Route, Label & Draft\nSwitch by category, apply Gmail labels, create reply drafts for important emails |
| 🏷️ Label: Spam-Promo | gmail | Add “Spam-Promo” label | 🔀 Route by Category | 📋 Prepare Log Entry | ## 🔀 Route, Label & Draft\nSwitch by category, apply Gmail labels, create reply drafts for important emails |
| 📋 Prepare Log Entry | code | Create Sheets row object from classification | 📝 Draft Reply (Urgent); 📝 Draft Reply (Action); 🏷️ Label: Follow-up; 🏷️ Label: Newsletter; 🏷️ Label: Automated; 🏷️ Label: Spam-Promo | 📊 Log to Sheet | ## 📊 Logging\nEvery processed email is logged to Google Sheets for analytics |
| 📊 Log to Sheet | googleSheets | Append triage record row | 📋 Prepare Log Entry | (end) | ## 📊 Logging\nEvery processed email is logged to Google Sheets for analytics |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: *Classify Gmail emails with AI and auto-label with smart reply drafts using Ollama*
- Keep workflow **inactive** until labels/IDs and credentials are configured.

2) **Add Gmail Trigger**
- Node: **Gmail Trigger**
- Filter: **Unread** emails
- Polling: **Every 30 minutes**
- Credential: **Gmail OAuth2**
  - Ensure permissions include at least: read; and later nodes need **modify + compose/drafts**.

3) **Add Code node: “Extract Email Data”**
- Node: **Code**
- Paste logic that:
  - Extracts `from/subject/date/to/cc/replyTo`
  - Extracts/decodes body text, cleans it, truncates (e.g., 3000 chars)
  - Computes flags: `hasAttachments`, `isReply`, `isForward`
  - Outputs: `gmailId`, `threadId`, `senderEmail`, `senderDomain`, `bodyText`, `bodyPreview`, `processedAt`, etc.
- Connect: **Gmail Trigger → Extract Email Data**

4) **Add Ollama model connector**
- Node: **Ollama Chat Model** (LangChain)
- Model name: `mistral` (or change to your installed model)
- Credential: **Ollama API**
  - Configure base URL for your Ollama instance (commonly `http://localhost:11434`).
  - On the host running Ollama, ensure the model is pulled, e.g.: `ollama pull mistral`

5) **Add AI Agent: “AI Email Classifier”**
- Node: **AI Agent** (LangChain Agent)
- Prompt: include email fields and body from the extracted JSON using expressions like:
  - `{{ $json.from }}`, `{{ $json.subject }}`, `{{ $json.bodyText }}`, etc.
- System message: define:
  - Allowed categories (6 values)
  - Reply-draft rules and JSON-only response schema
- Connect:
  - **Extract Email Data → AI Email Classifier**
  - **Ollama Chat Model → AI Email Classifier** (as the agent’s language model input)

6) **Add Code node: “Extract Classification”**
- Node: **Code**
- Implement robust JSON extraction/validation:
  - Try direct object
  - Try parsing strings (strip code fences)
  - Validate category against the known list
  - Default safely to `follow_up` on failure
- Ensure it merges original email metadata with classification fields (category/priority/confidence/replyDraft/replySubject/etc.).
- Connect: **AI Email Classifier → Extract Classification**

7) **Add Switch node: “Route by Category”**
- Node: **Switch**
- Add 6 rules:
  - `{{$json.category}} == "urgent"`
  - `... == "action_required"`
  - `... == "follow_up"`
  - `... == "newsletter"`
  - `... == "automated"`
  - `... == "spam_promo"`
- Connect: **Extract Classification → Route by Category**

8) **Create Gmail labels in Gmail UI (required)**
- Create exactly these labels:
  - `🔴 Urgent`
  - `📋 Action Required`
  - `💬 Follow-up`
  - `📰 Newsletter`
  - `🤖 Automated`
  - `🚫 Spam-Promo`

9) **Find Gmail Label IDs**
- Use Gmail API / an n8n Gmail “Get Many Labels” operation, or a one-off script to list label IDs.
- You must use **label IDs**, not label names, in the next nodes.

10) **Add Gmail nodes for labeling**
For each category route, add a **Gmail** node (operation **Add Labels**) configured as:
- Message ID: `{{ $json.gmailId }}`
- Label IDs: the correct label ID for that category

Create these nodes:
- “Label: Urgent”
- “Label: Action Required”
- “Label: Follow-up”
- “Label: Newsletter”
- “Label: Automated”
- “Label: Spam-Promo”

Connect each Switch output to its corresponding label node.

11) **Add Gmail draft nodes for urgent + action_required**
- Node: **Gmail** with resource **Draft** (create)
- Subject: `{{ $json.replySubject }}`
- Message body: `{{ $json.replyDraft }}`
- Option threadId: `{{ $json.threadId }}`
Create:
- “Draft Reply (Urgent)” connected after “Label: Urgent”
- “Draft Reply (Action)” connected after “Label: Action Required”

12) **Merge into logging preparation**
- Add a **Code** node: “Prepare Log Entry”
- Map fields into a flat object matching your Google Sheet headers:
  - Date, From, Subject, Category, Priority, Confidence, Reason, Summary, Key Action, Has Reply Draft, Sentiment, Auto Archive, Gmail ID, Processed At
- Connect:
  - From “Draft Reply (Urgent)” → Prepare Log Entry
  - From “Draft Reply (Action)” → Prepare Log Entry
  - From each other label node (follow_up/newsletter/automated/spam_promo) → Prepare Log Entry

13) **Add Google Sheets node: “Log to Sheet”**
- Node: **Google Sheets**
- Operation: **Append**
- Credential: **Google Sheets OAuth2**
- Document ID: your spreadsheet ID
- Sheet name/tab: your chosen tab (ensure headers match the fields you output)
- Mapping: Auto-map input data to columns (or define manual mapping).
- Connect: **Prepare Log Entry → Log to Sheet**

14) **Test**
- Manually execute with a few test emails.
- Confirm:
  - labels are applied correctly (IDs correct)
  - drafts are created for urgent/action_required
  - Sheets logging appends a row
  - Ollama returns valid JSON consistently

15) **Activate workflow**
- Once credentials, label IDs, and sheet IDs are correct, activate the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| You must manually create these Gmail labels before activating: `🔴 Urgent`, `📋 Action Required`, `💬 Follow-up`, `📰 Newsletter`, `🤖 Automated`, `🚫 Spam-Promo`. Then update Label IDs in each labeling node. | Sticky note “⚠️ Create Gmail Labels First” |
| Default Ollama model is `mistral`. Ensure it is available: `ollama pull mistral` (or switch to `llama3`, `gemma2`, etc.). | Sticky note “⚠️ Check Ollama Model” |
| The workflow is designed to avoid paid APIs by using a local LLM via Ollama. | Sticky note “Main Overview” |
| Suggested customization ideas include adding notifications, auto-archiving, changing the model, or adding whitelist/blacklist logic. | Sticky note “Main Overview” |