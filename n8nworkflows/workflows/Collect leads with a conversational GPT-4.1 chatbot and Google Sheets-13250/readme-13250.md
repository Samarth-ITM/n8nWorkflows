Collect leads with a conversational GPT-4.1 chatbot and Google Sheets

https://n8nworkflows.xyz/workflows/collect-leads-with-a-conversational-gpt-4-1-chatbot-and-google-sheets-13250


# Collect leads with a conversational GPT-4.1 chatbot and Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow implements a conversational chatbot (GPT-4.1-mini via n8n’s LangChain Agent) that collects lead details **one question at a time** (Name, Phone, Email, Message) and, **only when complete**, stores the lead in **Google Sheets**, then returns the assistant’s reply to the user via webhook response.

**Primary use cases:**
- Conversational lead capture on websites/apps (contact forms, booking inquiries)
- Replacing rigid forms with natural chat while still producing structured lead rows
- Session-based multi-turn conversations where data is progressively collected

### 1.1 Input Reception
Receives a user message and any already-known lead fields via an n8n Webhook (query parameters).

### 1.2 AI Processing (Conversation + Memory + LLM)
Uses a LangChain Agent with:
- a system message enforcing one-question-at-a-time lead collection behavior,
- an OpenAI chat model (gpt-4.1-mini),
- session memory keyed by `timestamp` to maintain conversation continuity.

### 1.3 Lead Storage (Google Sheets)
Appends or updates a row in Google Sheets using `Timestamp` as the matching key.

### 1.4 Response Delivery
Responds to the webhook request with the agent output (so the calling site/app gets the chatbot’s response).

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Accepts inbound messages from an external client (website/app). The workflow expects lead fields to arrive as query parameters (including a timestamp used as a session key).

**Nodes involved:**
- Receive User Message via Webhook

#### Node: Receive User Message via Webhook
- **Type / role:** `Webhook` (trigger) — starts the workflow on HTTP requests.
- **Key configuration:**
  - **Path:** `ff8b6142-d902-41ca-a666-a9e734e8ef14` (also used as the webhook endpoint path)
  - Receives data in `query` (based on expressions used later).
- **Key variables / expected inputs (query params):**
  - `query.message` (user’s latest message)
  - `query.name`, `query.phone`, `query.email` (may be empty initially)
  - `query.timestamp` (critical: used as session memory key and Google Sheets upsert key)
- **Outputs / connections:**
  - Main output → **Conversational Lead Collection Agent**
- **Edge cases / failures:**
  - Missing `query.message` → agent prompt will be empty/invalid, producing poor replies or failing expectations.
  - Missing `query.timestamp` → memory key becomes empty; conversations will collide or reset; Sheets matching may overwrite/duplicate unexpectedly.
  - If the client sends JSON body instead of query params, the current expressions will not read it.

---

### Block 2 — AI Processing (Agent + Model + Memory)
**Overview:** The agent drives the conversation, asking only for missing details and using session memory to retain context across turns. It is instructed to write to Google Sheets only after all four details are collected.

**Nodes involved:**
- Conversational Lead Collection Agent
- OpenAI GPT-4.1 Mini Language Model
- Session Memory with Timestamp

#### Node: OpenAI GPT-4.1 Mini Language Model
- **Type / role:** `lmChatOpenAi` — chat LLM provider for the agent.
- **Key configuration:**
  - **Model:** `gpt-4.1-mini`
  - Uses OpenAI API credentials named **“testing”**
- **Connections:**
  - `ai_languageModel` → **Conversational Lead Collection Agent**
- **Edge cases / failures:**
  - Invalid/expired OpenAI credential → auth error.
  - Model not available in the account/region → model selection error.
  - Rate limits/timeouts → intermittent failures; consider retries/backoff in production.

#### Node: Session Memory with Timestamp
- **Type / role:** `memoryBufferWindow` — stores conversational history per session.
- **Key configuration choices:**
  - **Session ID type:** custom key
  - **Session key expression:**  
    `{{ $('Receive User Message via Webhook').item.json.query.timestamp }}`
  - This means conversation memory is keyed entirely by the inbound `timestamp`.
- **Connections:**
  - `ai_memory` → **Conversational Lead Collection Agent**
- **Edge cases / failures:**
  - If `timestamp` changes every message, you will *not* get multi-turn memory (each message becomes a new session).
  - If multiple users share the same timestamp (unlikely but possible with coarse timestamps), cross-talk could occur.
  - If `timestamp` is missing, memory will be keyed poorly (empty/undefined), causing collisions.

#### Node: Conversational Lead Collection Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates conversation, uses LLM + memory, and can call tools.
- **Key configuration:**
  - **User prompt (“text”):**  
    `## User Message\n{{ $json.query.message }}`
  - **System message:** defines strict lead-collection rules:
    - Collect exactly: Name, Phone number, Email address, User message/requirement
    - Ask **one** question at a time; do not repeat already collected info
    - Validate unclear/invalid input by re-asking politely
    - **Tool usage constraint:** only use Google Sheets tool after all 4 details are confidently collected; write exactly one row per conversation
    - After successful write: send a final confirmation message (provided)
    - Do not mention backend/tools/webhook/automation
- **Connections:**
  - Main input ← **Receive User Message via Webhook**
  - `ai_languageModel` ← **OpenAI GPT-4.1 Mini Language Model**
  - `ai_memory` ← **Session Memory with Timestamp**
  - Main output → **Save Lead to Google Sheets**
- **Important integration caveat (design risk):**
  - The system message instructs the agent to only use the Google Sheets tool when complete, but in this workflow the **Google Sheets node is not wired as an agent tool**; it is simply the next node in the chain.
  - As implemented, **Save Lead to Google Sheets will run on every execution**, regardless of whether all details are collected—unless the agent outputs no item or fields are empty and you rely on Sheets behavior.
- **Edge cases / failures:**
  - If the calling client does not persist state and resend collected fields, the workflow currently references `query.name/email/phone` from the webhook—not from the agent’s extracted memory—so the sheet row may remain empty even after a “successful” conversation.
  - If the agent replies but the workflow also writes incomplete fields, you may store partial leads (contrary to the system rules).

---

### Block 3 — Lead Storage (Google Sheets)
**Overview:** Writes the lead data into a Google Sheet, using `Timestamp` as a unique matching key (append or update).

**Nodes involved:**
- Save Lead to Google Sheets

#### Node: Save Lead to Google Sheets
- **Type / role:** `Google Sheets` — persistence layer for lead capture.
- **Operation:** `appendOrUpdate`
- **Document:** “Sparkwritersretreat.com - Leads Tracking”  
  Document ID: `1L709LroC9hYWF1EI4snFgTEzlSVT7aFwTb5mdGNO8NQ`
- **Sheet:** “Session Leads” (gid `768094711`)
- **Matching column:** `Timestamp`  
  This means if the same timestamp arrives again, the row can be updated rather than appended.
- **Column mapping (expressions):** Values are taken from the webhook query:
  - `Name` ← `{{ $('Receive User Message via Webhook').item.json.query.name }}`
  - `Phone No.` ← `{{ $('Receive User Message via Webhook').item.json.query.phone }}`
  - `Email` ← `{{ $('Receive User Message via Webhook').item.json.query.email }}`
  - `Message` ← `{{ $('Receive User Message via Webhook').item.json.query.message }}`
  - `Timestamp` ← `{{ $('Receive User Message via Webhook').item.json.query.timestamp }}`
- **Credentials:** Google Sheets OAuth2 named **“Google Sheets - Krishna”**
- **Connections:**
  - Main input ← **Conversational Lead Collection Agent**
  - Main output → **Send AI Reply to User**
- **Edge cases / failures:**
  - OAuth token revoked/expired → auth failure.
  - Sheet column names mismatch (“Phone No.” vs “Phone No_” mentioned in system text) → mapping confusion. The node uses **“Phone No.”** (with a dot), while the system message says `Phone_No_` (underscore). If the agent were tool-writing, this mismatch would matter; here it’s a documentation/consistency risk.
  - `appendOrUpdate` with missing `Timestamp` can create duplicates or update unintended rows (depending on node behavior).
  - Writes may occur even when lead fields are missing (because there is no explicit IF gate).

---

### Block 4 — Response Delivery
**Overview:** Returns the workflow output to the caller, enabling a synchronous chatbot experience.

**Nodes involved:**
- Send AI Reply to User

#### Node: Send AI Reply to User
- **Type / role:** `Respond to Webhook` — sends HTTP response back to the original webhook call.
- **Configuration:**
  - **Respond with:** `allIncomingItems`  
    This typically returns the last node’s output item(s) as JSON to the caller.
- **Connections:**
  - Main input ← **Save Lead to Google Sheets**
- **Edge cases / failures:**
  - If upstream nodes return multiple items, caller may get an array unexpectedly.
  - If the agent’s reply text is not clearly surfaced in the returned JSON, the client may not find the message without additional formatting.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive User Message via Webhook | Webhook | Entry point; receives user message and fields via query params | — | Conversational Lead Collection Agent | ## Message Receive & AI Processing; Receives user messages via webhook, processes them through the conversational AI agent that intelligently asks for missing lead information. |
| Conversational Lead Collection Agent | LangChain Agent | Conversational logic; asks for missing lead data; coordinates LLM + memory | Receive User Message via Webhook; OpenAI GPT-4.1 Mini Language Model (AI); Session Memory with Timestamp (AI) | Save Lead to Google Sheets | ## Message Receive & AI Processing; Receives user messages via webhook, processes them through the conversational AI agent that intelligently asks for missing lead information. |
| OpenAI GPT-4.1 Mini Language Model | OpenAI Chat Model (LangChain) | Provides GPT-4.1-mini responses to the agent | — | Conversational Lead Collection Agent | ## Message Receive & AI Processing; Receives user messages via webhook, processes them through the conversational AI agent that intelligently asks for missing lead information. |
| Session Memory with Timestamp | Buffer Window Memory (LangChain) | Persists conversation history per session key | — | Conversational Lead Collection Agent | ## Message Receive & AI Processing; Receives user messages via webhook, processes them through the conversational AI agent that intelligently asks for missing lead information. |
| Save Lead to Google Sheets | Google Sheets | Stores lead row (append/update by Timestamp) | Conversational Lead Collection Agent | Send AI Reply to User | ##  Lead Storage; Once all 4 details are collected (Name, Phone, Email, Message), the AI saves the complete lead data to Google Sheets using timestamp. |
| Send AI Reply to User | Respond to Webhook | Returns response payload to the webhook caller | Save Lead to Google Sheets | — | ## Response Delivery; Sends the AI's conversational reply back to the user via webhook, continuing the natural conversation flow. |
| Sticky Note | Sticky Note | Documentation panel | — | — | ## AI Booking Chatbot with Lead Collection; This workflow creates a conversational AI chatbot that naturally collects lead information (Name, Phone, Email, Message) one question at a time—without feeling like a form. The AI remembers the conversation context using session-based memory, asks only for missing details, and saves complete leads to Google Sheets. Perfect for booking systems, contact forms, or any lead capture that needs a human-like conversation experience. |
| Sticky Note1 | Sticky Note | Documentation panel | — | — | ## Message Receive & AI Processing; Receives user messages via webhook, processes them through the conversational AI agent that intelligently asks for missing lead information. |
| Sticky Note2 | Sticky Note | Documentation panel | — | — | ##  Lead Storage; Once all 4 details are collected (Name, Phone, Email, Message), the AI saves the complete lead data to Google Sheets using timestamp. |
| Sticky Note3 | Sticky Note | Documentation panel | — | — | ## Response Delivery; Sends the AI's conversational reply back to the user via webhook, continuing the natural conversation flow. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add node: Webhook**
   - Name: **Receive User Message via Webhook**
   - Method: (default for webhook node; configure as needed)
   - Path: set to a unique value (e.g. the provided UUID-like path)
   - Ensure your client will send lead fields as **query parameters**:
     - `message`, `name`, `phone`, `email`, `timestamp`

3. **Add node: OpenAI Chat Model (LangChain)**
   - Name: **OpenAI GPT-4.1 Mini Language Model**
   - Model: **gpt-4.1-mini**
   - Credentials: create/select **OpenAI API** credentials (API key)

4. **Add node: Memory Buffer Window (LangChain)**
   - Name: **Session Memory with Timestamp**
   - Session ID type: **Custom Key**
   - Session Key expression: reference the webhook timestamp, e.g.  
     `{{ $('Receive User Message via Webhook').item.json.query.timestamp }}`  
   - Important: ensure the **client keeps the same timestamp for the entire conversation** if you want multi-turn memory.

5. **Add node: Agent (LangChain)**
   - Name: **Conversational Lead Collection Agent**
   - Prompt type: **Define**
   - Text (user message prompt):  
     `## User Message` followed by expression to `query.message`
   - System message: paste the provided instruction block (one-question-at-a-time, collect 4 fields, don’t mention backend, etc.).
   - Connect:
     - Webhook **main** → Agent **main**
     - OpenAI Model → Agent via **AI Language Model** connection
     - Memory → Agent via **AI Memory** connection

6. **Add node: Google Sheets**
   - Name: **Save Lead to Google Sheets**
   - Credentials: connect **Google Sheets OAuth2**
   - Operation: **Append or Update**
   - Document: select your spreadsheet
   - Sheet: select your tab (e.g., “Session Leads”)
   - Matching column: **Timestamp**
   - Map columns using expressions (from webhook query):
     - Timestamp, Name, Phone No., Email, Message

7. **Add node: Respond to Webhook**
   - Name: **Send AI Reply to User**
   - Respond with: **All Incoming Items**
   - Connect:
     - Google Sheets → Respond to Webhook

8. **Activate and test**
   - Call the webhook URL with query params, e.g.:
     - `?message=Hi&timestamp=<stable-session-id>`
   - Continue sending messages using the **same timestamp** to preserve memory.

**Recommended hardening (to match the system rules):**
- Add an **IF** node (or a dedicated “lead completeness” check) before Google Sheets to prevent writing rows unless all 4 fields are present and valid.
- Alternatively, restructure so the agent truly “tool-calls” Sheets (rather than always running the Sheets node).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Booking Chatbot with Lead Collection… Perfect for booking systems, contact forms, or any lead capture that needs a human-like conversation experience.” | Workflow description (Sticky Note) |
| Setup steps included: connect Google Sheets OAuth, add OpenAI key, update Sheet URL/name, integrate webhook URL, test with query parameters. | Workflow description (Sticky Note) |
| Disclaimer (provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | Compliance/context statement from request |