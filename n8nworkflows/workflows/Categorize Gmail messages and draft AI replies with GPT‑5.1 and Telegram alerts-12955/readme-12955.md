Categorize Gmail messages and draft AI replies with GPT‑5.1 and Telegram alerts

https://n8nworkflows.xyz/workflows/categorize-gmail-messages-and-draft-ai-replies-with-gpt-5-1-and-telegram-alerts-12955


# Categorize Gmail messages and draft AI replies with GPT‑5.1 and Telegram alerts

## 1. Workflow Overview

**Purpose:** Automatically process new Gmail emails for a support inbox using an AI agent (GPT‑5.1). The agent (1) categorizes each email into a small set of support categories, (2) applies the correct Gmail label, (3) drafts a reply when the knowledge base is sufficient, and (4) sends a Telegram alert containing a direct link to the created Gmail draft.

**Target use cases:**
- Lightweight customer support triage (order-status questions, quote requests)
- Semi-automated responses where a human reviews drafts before sending
- Consistent labeling and faster response preparation using a centralized knowledge base

### 1.1 Email Intake (Trigger)
Detects new incoming emails in Gmail on a schedule (polling).

### 1.2 Context Loading (Knowledge Base + Gmail Labels)
Loads a “Customer Support Knowledge Base” record from an n8n Data Table and fetches all Gmail labels, aggregating them into a format the AI agent can use.

### 1.3 AI Reasoning + Tool Actions (Label, Draft, Notify)
Uses a LangChain Agent with GPT‑5.1 plus three tools (Gmail label tool, Gmail draft tool, Telegram tool). The agent decides category/label, optionally creates a draft reply, then always sends a Telegram message with a draft link.

---

## 2. Block-by-Block Analysis

### Block 1 — Email Intake (Trigger)

**Overview:** Watches Gmail for new messages and starts the workflow once a message is detected.

**Nodes involved:**
- **New Email Trigger**

#### Node: New Email Trigger
- **Type / role:** `gmailTrigger` — polling trigger that emits an item per new email.
- **Key configuration:**
- Polling interval: **every minute**
- “simple”: `false` (so richer payload fields are typically returned)
- No explicit filters configured (all new emails eligible, depending on Gmail Trigger defaults).
- **Credentials:** Gmail OAuth2 (“Jim Halpert”).
- **Outputs:** Connects to **Get Knowledge Base Data**.
- **Key fields used later (from trigger output):**
- `id` (message ID)
- `threadId`
- `subject`
- `text`
- `from.value[0].address` (used as the draft recipient)
- **Failure/edge cases:**
- Gmail OAuth token expiry/consent issues.
- Duplicate processing can occur with polling triggers depending on Gmail/node behavior (e.g., if message state changes). Consider adding deduplication if needed.
- Payload shape may vary (e.g., missing `text` for HTML-only emails).

---

### Block 2 — Context Loading (Knowledge Base + Labels)

**Overview:** Loads support knowledge base content and the list of Gmail labels, then aggregates label data into a single field for AI consumption.

**Nodes involved:**
- **Get Knowledge Base Data**
- **Get Gmail Labels List**
- **Aggregate Labels Data**

#### Node: Get Knowledge Base Data
- **Type / role:** `dataTable` — fetches a record from an n8n Data Table containing support knowledge.
- **Key configuration:**
- Operation: **Get**
- Data Table: **Customer Support Knowledge Base**
- Limit: **1** (expects the first/only row to contain the knowledge base)
- **Inputs:** From **New Email Trigger**.
- **Outputs:** To **Get Gmail Labels List**.
- **Variables referenced later:**
- `$('Get Knowledge Base Data').item.json.knowledge_base`
- **Failure/edge cases:**
- Table not found / permission issues.
- No rows returned → `knowledge_base` becomes undefined and may degrade AI output.
- If the field name differs (not `knowledge_base`), the agent prompt will reference a missing field.

#### Node: Get Gmail Labels List
- **Type / role:** `gmail` — retrieves Gmail label metadata.
- **Key configuration:**
- Resource: **label**
- Return all: **true**
- **Credentials:** Gmail OAuth2 (“Jim Halpert”).
- **Inputs:** From **Get Knowledge Base Data**.
- **Outputs:** To **Aggregate Labels Data**.
- **Failure/edge cases:**
- Gmail auth/permissions.
- Large label lists could increase prompt size downstream.

#### Node: Aggregate Labels Data
- **Type / role:** `aggregate` — converts multiple label items into a single JSON array field.
- **Key configuration:**
- Aggregate mode: **aggregateAllItemData**
- Include: **specifiedFields** → `id,name`
- Destination field: `labels`
- **Inputs:** From **Get Gmail Labels List** (many label items).
- **Outputs:** To **Email Categorization AI Agent** (single aggregated item containing `labels`).
- **Failure/edge cases:**
- If Gmail labels return unexpected structure, aggregation may omit needed fields.
- Prompt bloat if many labels exist; consider filtering labels if necessary.

---

### Block 3 — AI Reasoning + Tool Actions (Label, Draft, Notify)

**Overview:** A LangChain Agent (powered by GPT‑5.1) receives the email content, label list, and knowledge base. It decides which label to apply, whether a draft reply should be created, and always sends a Telegram message (typically containing a Gmail draft link).

**Nodes involved:**
- **Email Categorization AI Agent**
- **OpenAI Chat Model**
- **Add label to message in Gmail** (tool)
- **Create a draft in Gmail** (tool)
- **Send a text message in Telegram** (tool)

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — the chat model provider used by the agent.
- **Key configuration:**
- Model: **gpt-5.1**
- Options: default
- **Credentials:** OpenAI API (“OpenAi account”).
- **Connections:**
- Feeds the agent through the `ai_languageModel` connection.
- **Failure/edge cases:**
- API key invalid, quota exceeded, model not available in account/region.
- Timeouts or large prompts (knowledge base + labels + email) exceeding limits.

#### Node: Email Categorization AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates reasoning and tool use.
- **Key configuration (interpreted):**
- Prompt instructs the agent to:
  - Categorize into: `existing_order`, `quote_request`, or `none`
  - Draft a response **only** if answerable from KB and not complaint/billing dispute/manager request, and enough info exists; else set `draft_response` to `null`
  - Use label IDs from the provided label list; **only add one label** and provide only the label **id** in tool calls
  - After draft creation, construct a link:  
    `https://mail.google.com/mail/u/0/#drafts?compose={draft_message_id}`
  - **Always send** a Telegram notification with that link (and/or relevant info)
- Injected context via expressions:
- `{{ JSON.stringify($json.labels) }}` from **Aggregate Labels Data**
- `{{ $('Get Knowledge Base Data').item.json.knowledge_base }}`
- Email fields from **New Email Trigger**: `id`, `threadId`, `subject`, `text`
- **Inputs:**
- Main input from **Aggregate Labels Data** (contains `labels`)
- Model input from **OpenAI Chat Model**
- Tool availability from:
  - **Add label to message in Gmail**
  - **Create a draft in Gmail**
  - **Send a text message in Telegram**
- **Outputs:** Not explicitly connected further; the agent primarily acts by invoking tools.
- **Failure/edge cases:**
- If `from.value[0].address` or `text` is missing, drafting may fail or be low quality.
- Agent may choose a label name rather than an ID unless prompt/tool schema strongly enforces it; here the Gmail label tool field is named `Label_Names_or_IDs`, so the agent might pass a name—prompt insists “ONLY write the id”.
- If no draft is created but the prompt says “ALWAYS SEND IT!”, the agent still must send Telegram; ensure your intended Telegram message content handles “no draft created” scenarios gracefully (otherwise the agent may invent a draft id unless constrained).

#### Node: Add label to message in Gmail
- **Type / role:** `gmailTool` — AI tool to apply labels to the triggering email.
- **Key configuration:**
- Operation: **addLabels**
- `messageId`: expression → `$('New Email Trigger').item.json.id`
- `labelIds`: filled by AI via `$fromAI('Label_Names_or_IDs', ...)`
- **Credentials:** Gmail OAuth2 (“Jim Halpert”).
- **Connections:** Exposed to **Email Categorization AI Agent** as an `ai_tool`.
- **Failure/edge cases:**
- If the agent provides a label **name** instead of **id**, Gmail may reject it.
- If label id is invalid/not found, Gmail returns an error.
- Permission issues if labeling is restricted by account policy.

#### Node: Create a draft in Gmail
- **Type / role:** `gmailTool` — AI tool to create a Gmail draft reply in the same thread.
- **Key configuration:**
- Resource: **draft**
- `subject`: filled by AI via `$fromAI('Subject', ...)`
- `message`: filled by AI via `$fromAI('Message', ...)`
- Options:
  - `sendTo`: expression → `$('New Email Trigger').item.json.from.value[0].address`
  - `threadId`: expression → `$('New Email Trigger').item.json.threadId`
- **Credentials:** Gmail OAuth2 (“Jim Halpert”).
- **Connections:** Exposed to **Email Categorization AI Agent** as an `ai_tool`.
- **Draft link expectation:** The agent is instructed to use the returned draft message id to build:  
  `https://mail.google.com/mail/u/0/#drafts?compose={draft_message_id}`
- **Failure/edge cases:**
- Missing/invalid recipient address in `from.value[0].address`.
- Thread ID missing → draft may not be associated correctly.
- Agent may draft when it shouldn’t unless the prompt is strictly followed; consider additional deterministic checks if critical.

#### Node: Send a text message in Telegram
- **Type / role:** `telegramTool` — AI tool to send a notification.
- **Key configuration:**
- `chatId`: **123456789** (must be replaced with your actual chat/user/group id)
- `text`: filled by AI via `$fromAI('Text', ...)`
- **Credentials:** Telegram API (“MilanBot”).
- **Connections:** Exposed to **Email Categorization AI Agent** as an `ai_tool`.
- **Failure/edge cases:**
- Wrong chat id → message not delivered.
- Bot not allowed in group / not started by user.
- Telegram rate limits if email volume is high.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Email Trigger | n8n-nodes-base.gmailTrigger | Entry point; polls Gmail for new emails | — | Get Knowledge Base Data | ## Workflow Overview  This workflow creates an AI-powered email assistant that automatically processes incoming Gmail messages. When a new email arrives, the AI agent categorizes it (such as "existing_order" or "quote_request"), adds the appropriate Gmail label, and drafts intelligent responses based on your company's knowledge base. You'll receive a Telegram notification with a direct link to review the draft before sending.  ### First Setup  **Required Accounts & Credentials:** - Gmail account (OAuth2 connection) - OpenAI API account for the chat model - Telegram bot for notifications  **Initial Configuration:** 1. Connect your Gmail account to both the Gmail Trigger and Gmail tool nodes 2. Create a Data Table called "Customer Support Knowledge Base" with your business information, pricing, policies, and common responses 3. Set up your Telegram bot credentials 4. Configure your OpenAI API credentials  ### Configuration  **Knowledge Base:** Customize the Data Table with your company's information, response style guide, product details, pricing, and FAQs. This is what the AI uses to draft accurate responses.  **Telegram Notifications:** Update the chat ID in the Telegram node to your own.  **AI Prompt:** The prompt in the AI Agent node defines categorization rules and when to draft responses. Adjust categories and response criteria to match your needs.  **Polling Frequency:** The Gmail Trigger checks every minute by default. Modify in the trigger settings if needed. |
| Get Knowledge Base Data | n8n-nodes-base.dataTable | Fetches KB content for AI grounding | New Email Trigger | Get Gmail Labels List | (same as above) |
| Get Gmail Labels List | n8n-nodes-base.gmail | Retrieves all Gmail labels | Get Knowledge Base Data | Aggregate Labels Data | (same as above) |
| Aggregate Labels Data | n8n-nodes-base.aggregate | Aggregates label id/name pairs into `labels` array | Get Gmail Labels List | Email Categorization AI Agent | (same as above) |
| Email Categorization AI Agent | @n8n/n8n-nodes-langchain.agent | Core AI orchestration: categorize, label, draft, notify | Aggregate Labels Data; OpenAI Chat Model (ai_languageModel) | — (invokes tools) | (same as above) |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT‑5.1 to the agent | — | Email Categorization AI Agent (ai_languageModel) | (same as above) |
| Add label to message in Gmail | n8n-nodes-base.gmailTool | Tool: apply one Gmail label to the triggering message | — (called by agent) | Email Categorization AI Agent (ai_tool) | (same as above) |
| Create a draft in Gmail | n8n-nodes-base.gmailTool | Tool: create a reply draft in the same thread | — (called by agent) | Email Categorization AI Agent (ai_tool) | (same as above) |
| Send a text message in Telegram | n8n-nodes-base.telegramTool | Tool: send Telegram notification (draft link, etc.) | — (called by agent) | Email Categorization AI Agent (ai_tool) | (same as above) |
| Workflow Description | n8n-nodes-base.stickyNote | Documentation / usage notes | — | — | ## Workflow Overview  This workflow creates an AI-powered email assistant that automatically processes incoming Gmail messages. When a new email arrives, the AI agent categorizes it (such as "existing_order" or "quote_request"), adds the appropriate Gmail label, and drafts intelligent responses based on your company's knowledge base. You'll receive a Telegram notification with a direct link to review the draft before sending.  ### First Setup  **Required Accounts & Credentials:** - Gmail account (OAuth2 connection) - OpenAI API account for the chat model - Telegram bot for notifications  **Initial Configuration:** 1. Connect your Gmail account to both the Gmail Trigger and Gmail tool nodes 2. Create a Data Table called "Customer Support Knowledge Base" with your business information, pricing, policies, and common responses 3. Set up your Telegram bot credentials 4. Configure your OpenAI API credentials  ### Configuration  **Knowledge Base:** Customize the Data Table with your company's information, response style guide, product details, pricing, and FAQs. This is what the AI uses to draft accurate responses.  **Telegram Notifications:** Update the chat ID in the Telegram node to your own.  **AI Prompt:** The prompt in the AI Agent node defines categorization rules and when to draft responses. Adjust categories and response criteria to match your needs.  **Polling Frequency:** The Gmail Trigger checks every minute by default. Modify in the trigger settings if needed. |
| Video Walkthrough | n8n-nodes-base.stickyNote | External reference link | — | — | # Video Walkthrough  [![image.png](https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_rec5lxlsWWeV1B327.jpg)](https://www.youtube.com/watch?v=UEcE0cXlQ5g) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Gmail OAuth2 credentials**
   - In n8n, add **Gmail OAuth2** credentials.
   - Ensure scopes allow reading emails, managing labels, and creating drafts.
   - Name example: “Jim Halpert”.

2. **Create OpenAI credentials**
   - Add **OpenAI API** credentials (API key).
   - Name example: “OpenAi account”.

3. **Create Telegram bot credentials**
   - Create a bot via BotFather, obtain token.
   - Add **Telegram API** credentials in n8n.
   - Name example: “MilanBot”.
   - Determine your target `chatId` (user or group) and note it.

4. **Create the Data Table (Knowledge Base)**
   - In n8n, create a Data Table named **Customer Support Knowledge Base**.
   - Add a column/field named **`knowledge_base`** (text/long text).
   - Insert **one row** containing your full KB text (policies, product/pricing, FAQs, response style).

5. **Add node: “New Email Trigger”**
   - Node type: **Gmail Trigger**
   - Polling schedule: **Every minute**
   - Connect Gmail OAuth2 credential: “Jim Halpert”
   - Leave filters empty (or add label/from constraints if desired).

6. **Add node: “Get Knowledge Base Data”**
   - Node type: **Data Table**
   - Operation: **Get**
   - Data table: **Customer Support Knowledge Base**
   - Limit: **1**
   - Connect: **New Email Trigger → Get Knowledge Base Data**

7. **Add node: “Get Gmail Labels List”**
   - Node type: **Gmail**
   - Resource: **Label**
   - Return All: **true**
   - Credentials: “Jim Halpert”
   - Connect: **Get Knowledge Base Data → Get Gmail Labels List**

8. **Add node: “Aggregate Labels Data”**
   - Node type: **Aggregate**
   - Aggregate: **Aggregate All Item Data**
   - Include: **Specified Fields** → `id`, `name`
   - Destination Field Name: **`labels`**
   - Connect: **Get Gmail Labels List → Aggregate Labels Data**

9. **Add node: “OpenAI Chat Model”**
   - Node type: **OpenAI Chat Model** (LangChain)
   - Model: **gpt-5.1**
   - Credentials: “OpenAi account”

10. **Add node: “Add label to message in Gmail” (Tool)**
    - Node type: **Gmail Tool**
    - Operation: **addLabels**
    - `messageId`: expression → `{{ $('New Email Trigger').item.json.id }}`
    - `labelIds`: set to **AI-filled** value (use `$fromAI(...)` field as configured by n8n when adding as a tool)
    - Credentials: “Jim Halpert”

11. **Add node: “Create a draft in Gmail” (Tool)**
    - Node type: **Gmail Tool**
    - Resource: **Draft**
    - `subject`: AI-filled (`$fromAI('Subject', ...)`)
    - `message`: AI-filled (`$fromAI('Message', ...)`)
    - Options:
      - Send To: `{{ $('New Email Trigger').item.json.from.value[0].address }}`
      - Thread ID: `{{ $('New Email Trigger').item.json.threadId }}`
    - Credentials: “Jim Halpert”

12. **Add node: “Send a text message in Telegram” (Tool)**
    - Node type: **Telegram Tool**
    - Operation: send message (text)
    - `chatId`: set to your real chat id (replace `123456789`)
    - `text`: AI-filled (`$fromAI('Text', ...)`)
    - Credentials: “MilanBot”

13. **Add node: “Email Categorization AI Agent”**
    - Node type: **AI Agent (LangChain)**
    - Prompt type: **Define**
    - Paste/configure a prompt equivalent to the workflow’s intent, including:
      - Categories: `existing_order`, `quote_request`, `none`
      - Drafting rules (KB-only; avoid complaints/billing/manager; null if unsure)
      - Provide label list: `{{ JSON.stringify($json.labels) }}`
      - Provide KB text: `{{ $('Get Knowledge Base Data').item.json.knowledge_base }}`
      - Provide email fields: id, threadId, subject, text from trigger
      - Draft link template: `https://mail.google.com/mail/u/0/#drafts?compose={draft_message_id}`
      - Instruction to always notify via Telegram
    - Connect main input: **Aggregate Labels Data → Email Categorization AI Agent**

14. **Wire the AI model and tools into the agent**
    - Connect **OpenAI Chat Model → Email Categorization AI Agent** using the **AI Language Model** connection type.
    - Connect each tool node to the agent using the **AI Tool** connection type:
      - Add label tool → agent
      - Create draft tool → agent
      - Telegram tool → agent

15. **Test**
    - Send a test email into the Gmail inbox.
    - Verify:
      - A label is added (exactly one).
      - A draft is created when appropriate.
      - Telegram message arrives and includes the correct draft URL.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Video Walkthrough | https://www.youtube.com/watch?v=UEcE0cXlQ5g |
| Draft link pattern used by the AI | `https://mail.google.com/mail/u/0/#drafts?compose={draft_message_id}` |
| Sticky note guidance (accounts, KB table, chatId, polling) | Included in “Workflow Description” sticky note inside the canvas |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.