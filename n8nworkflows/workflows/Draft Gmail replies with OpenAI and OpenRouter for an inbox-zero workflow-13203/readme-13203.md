Draft Gmail replies with OpenAI and OpenRouter for an inbox-zero workflow

https://n8nworkflows.xyz/workflows/draft-gmail-replies-with-openai-and-openrouter-for-an-inbox-zero-workflow-13203


# Draft Gmail replies with OpenAI and OpenRouter for an inbox-zero workflow

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically monitor a Gmail inbox, classify incoming emails to decide whether they require a human-style reply, and (only for relevant emails) generate a **Gmail draft reply** using an AI agent. For irrelevant/automated emails, it sends a Telegram notification instead. Optionally, it can use a Supabase-backed vector store as a knowledge base to ground replies.

**Target use cases:**
- “Inbox zero” triage: filter newsletters/receipts/system alerts vs. real client/prospect/support emails.
- Semi-automated customer support: AI drafts replies, user reviews and sends.
- Knowledge-base-assisted drafting (optional): include policy/FAQ snippets via vector retrieval.

### 1.1 Logical Blocks
1. **Email Input & Field Extraction**
   - Poll Gmail, extract body/thread/sender for downstream logic.
2. **AI Classification (OpenRouter)**
   - LLM classifies whether the email needs a response (`customerSupport: true/false`) and returns strict JSON.
3. **Routing + Notifications**
   - Switch routes: if support-worthy → drafting; if not → Telegram notification.
4. **AI Drafting Agent (OpenAI + Gmail Tool)**
   - Agent drafts a reply and invokes a Gmail “create draft” tool.
5. **Optional Knowledge Base (Supabase Vector Store)**
   - Embeddings + Supabase vector store exposed as a retrieval tool to the drafting agent.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Email Input & Field Extraction
**Overview:** Polls Gmail for new emails and normalizes key fields (body, thread ID, sender) used by classification and drafting.

**Nodes involved:**
- Gmail Trigger
- Set Content

#### Node: Gmail Trigger
- **Type / role:** `gmailTrigger` — polls Gmail for new messages.
- **Configuration (interpreted):**
  - Poll schedule: **every hour**.
  - “Simple” mode disabled (`simple: false`), implying richer output (headers, body fields like `text`, `threadId`, etc.).
- **Key outputs used later:**
  - `headers.subject`
  - `threadId`
  - `text`
  - `headers.from`
- **Connections:**
  - **Main →** Set Content
- **Potential failures / edge cases:**
  - OAuth token expiration/revocation for Gmail OAuth2.
  - Polling may miss/duplicate messages depending on Gmail history state and n8n trigger behavior.
  - Some emails may not have `text` populated (HTML-only or unusual MIME structures).

#### Node: Set Content
- **Type / role:** `set` — maps Gmail payload into a cleaner schema for prompting.
- **Configuration:**
  - Creates fields:
    - `emailBody` = `{{$json.text}}`
    - `threadID` = `{{$json.threadId}}`
    - `from` = `{{$json.headers.from}}`
- **Connections:**
  - **Main →** Client/Prospect Related?
- **Potential failures / edge cases:**
  - If `text` is missing/empty, downstream classification and drafting prompts may be low quality.
  - `headers.from` can include display name + email (e.g., `"Name <x@y>"`), which can affect “reply-to” handling later if not normalized.

---

### Block 2.2 — AI Classification (OpenRouter + Structured Parser)
**Overview:** Uses an OpenRouter chat model to classify whether an incoming email appears to be a genuine human inquiry requiring a response, returning structured JSON.

**Nodes involved:**
- OpenRouter Chat Model
- Structured Output Parser
- Client/Prospect Related?

#### Node: OpenRouter Chat Model
- **Type / role:** `lmChatOpenRouter` — LLM backend for classification.
- **Configuration:**
  - Default options (no explicit model parameters shown).
- **Connections:**
  - **AI Language Model →** Client/Prospect Related?
- **Potential failures / edge cases:**
  - OpenRouter credential issues, rate limits, or model/provider downtime.
  - If OpenRouter returns non-JSON or verbose text, parsing may fail (mitigated by output parser node).

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces a JSON schema-like output.
- **Configuration:**
  - Example schema provided:
    - `{ "customerSupport": "True" }`
  - Downstream logic expects boolean-like values; prompt explicitly requests:
    - `{"customerSupport": true}` or `{"customerSupport": false}`
- **Connections:**
  - **AI Output Parser →** Client/Prospect Related?
- **Potential failures / edge cases:**
  - If the LLM returns invalid JSON, parser errors can occur.
  - Schema example uses `"True"` as string, but the prompt asks for booleans `true/false`; inconsistencies can lead to switch mismatches if the parser coerces unexpectedly.

#### Node: Client/Prospect Related?
- **Type / role:** `chainLlm` — runs a prompt through the connected LLM and applies the output parser.
- **Configuration:**
  - Prompt contains explicit rules for `customerSupport` TRUE/FALSE, plus many examples.
  - Requires output: **ONLY valid JSON** with boolean.
  - `hasOutputParser: true` and connected to Structured Output Parser.
- **Key variables:**
  - Uses `{{ $json.emailBody }}` from Set Content.
- **Connections:**
  - **Main →** Customer Support?
  - Receives **AI Language Model** from OpenRouter Chat Model.
  - Receives **AI Output Parser** from Structured Output Parser.
- **Potential failures / edge cases:**
  - If the email body contains long threads, token limits can be hit.
  - Misclassification risk: human-written marketing outreach might be marked as “support”; system emails without obvious markers might be marked “support”.
  - If output becomes `"customerSupport": "true"` (string), your switch rules must match that type/value.

---

### Block 2.3 — Routing + Notifications (Switch + Telegram)
**Overview:** Routes execution based on classification result. Sends Telegram alerts for non-support emails; forwards support-worthy items to drafting.

**Nodes involved:**
- Customer Support?
- Response Not Customer Support

#### Node: Customer Support?
- **Type / role:** `switch` — branching based on `customerSupport`.
- **Configuration:**
  - Two renamed outputs:
    - **Customer Support** when `{{$json.output.customerSupport}}` equals `"true"`
    - **Not Customer Support** when it equals `"false"`
  - `alwaysOutputData: true` (ensures output even if no match, but routes may still be empty).
- **Connections:**
  - **Customer Support (main index 0) →** Email Draft Agent
  - **Not Customer Support (main index 1) →** Response Not Customer Support
- **Potential failures / edge cases:**
  - Comparison uses **string** `"true"` / `"false"`. If the parser outputs boolean `true/false`, the condition may fail unless n8n coerces; safest is to compare to boolean or to `{{$json.output.customerSupport.toString()}}`.
  - If neither rule matches, nothing meaningful happens unless additional default routing is implemented.

#### Node: Response Not Customer Support
- **Type / role:** `telegram` — notify user about incoming non-support email.
- **Configuration:**
  - Message text:
    - `You received an email at {{ $now.format('hh:mm') }} saying:\n\n{{ $('Set Content').item.json.emailBody }}`
  - `chatId` is a placeholder: `YOUR_CHAT_ID`
  - Attribution disabled (`appendAttribution: false`).
- **Connections:**
  - Terminal (no downstream node).
- **Potential failures / edge cases:**
  - Telegram bot token invalid, chat ID wrong, bot not started by user.
  - Very long email bodies may exceed Telegram message limits.

---

### Block 2.4 — AI Drafting Agent (OpenAI + Gmail Draft Tool + Telegram)
**Overview:** For emails requiring a response, an AI agent drafts a reply using OpenAI and then creates a Gmail draft in the same thread. It sends a Telegram message with the agent’s output.

**Nodes involved:**
- OpenAI Chat Model
- Email Draft Agent
- createDraft
- Response

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — LLM backend for drafting.
- **Configuration:**
  - Default options; uses OpenAI API credential.
- **Connections:**
  - **AI Language Model →** Email Draft Agent
- **Potential failures / edge cases:**
  - OpenAI API quota/rate limits.
  - Drafting quality depends heavily on the system prompt and email body cleanliness.

#### Node: Email Draft Agent
- **Type / role:** `agent` — orchestrates tools (Gmail drafting tool, vector retrieval tool) with an LLM to accomplish the drafting task.
- **Configuration:**
  - User prompt: `Draft a response to this email: {{ $('Set Content').item.json.emailBody }}`
  - System message contains strict rules (reply to sender, use “I”, 2–3 paragraphs, end with sign-off, use vector store when needed).
  - Instructs agent to use **createDraft** tool.
- **Connections:**
  - **Main →** Response (Telegram)
  - Receives **AI Language Model** from OpenAI Chat Model.
  - Receives **AI Tool** from:
    - createDraft (Gmail tool)
    - Vector Store Tool (optional KB)
- **Potential failures / edge cases:**
  - If tools are unavailable/misconfigured, agent may fail to create the draft.
  - If the agent output is not what you expect, Telegram message may contain internal tool logs depending on agent behavior/version.
  - Prompt asks to always reply to sender; correctness depends on how `sendTo` is computed in createDraft.

#### Node: createDraft
- **Type / role:** `gmailTool` — tool node exposed to the agent to create a Gmail **draft**.
- **Configuration:**
  - **Resource:** draft
  - **Subject:** from trigger header: `{{ $('Gmail Trigger').item.json.headers.subject }}`
  - **Thread:** `{{ $('Set Content').item.json.threadID }}`
  - **SendTo logic:**
    - `{{ $('Set Content').item.json.from === 'your-email@example.com' ? $('Set Content').item.json.to : $('Set Content').item.json.from }}`
    - Note: `Set Content` does not set `to`, so this conditional may not work as intended.
  - **Message body:** uses an AI tool parameter:
    - `{{$fromAI('Message', '', 'string')}}` (agent supplies the drafted message)
- **Connections:**
  - **AI Tool →** Email Draft Agent (meaning the agent can call it)
- **Potential failures / edge cases:**
  - **Critical:** `Set Content` does not define `to`; if email is from yourself, `sendTo` may evaluate to `undefined`.
  - `headers.from` often includes a name + email; Gmail may accept it, but best practice is to parse the email address.
  - If the email is part of a thread with multiple participants, “reply to sender” logic may need CC handling.
  - Gmail API permission scope must allow draft creation.

#### Node: Response
- **Type / role:** `telegram` — sends the agent’s final output to Telegram.
- **Configuration:**
  - Text: `{{ $json.output }}`
  - `chatId`: `YOUR_CHAT_ID`
  - Attribution disabled.
- **Connections:**
  - Terminal (no downstream node).
- **Potential failures / edge cases:**
  - Same Telegram constraints as above.
  - `$json.output` shape depends on agent node version; sometimes agent output may be under a different key.

---

### Block 2.5 — Knowledge Base (Optional) via Supabase Vector Store
**Overview:** Provides retrieval-augmented drafting by storing documents in Supabase and exposing a “documents” retrieval tool to the agent.

**Nodes involved:**
- Embeddings OpenAI
- Vector Storage (Supabase)
- OpenAI Chat Model1
- Vector Store Tool

#### Node: Embeddings OpenAI
- **Type / role:** `embeddingsOpenAi` — generates embeddings for vector storage/retrieval.
- **Configuration:**
  - Default options; OpenAI API credentials.
- **Connections:**
  - **AI Embedding →** Vector Storage
- **Potential failures / edge cases:**
  - Embedding model availability/quota.
  - Dimension mismatch if Supabase table expects a specific vector size (depends on your Supabase setup).

#### Node: Vector Storage
- **Type / role:** `vectorStoreSupabase` — Supabase-backed vector store.
- **Configuration:**
  - Table: `documents` (selected from list).
- **Connections:**
  - Receives **AI Embedding** from Embeddings OpenAI.
  - **AI Vector Store →** Vector Store Tool
- **Potential failures / edge cases:**
  - Supabase credentials/URL/key issues.
  - Table/schema not created or missing required columns/index for vector search.
  - If no documents are ingested, retrieval returns empty results (agent must handle gracefully).

#### Node: OpenAI Chat Model1
- **Type / role:** `lmChatOpenAi` — LLM used by the vector-store tool (often for query synthesis or tool reasoning, depending on node implementation).
- **Configuration:** Default options.
- **Connections:**
  - **AI Language Model →** Vector Store Tool
- **Potential failures / edge cases:**
  - Same OpenAI constraints; additionally, tool behavior can vary by node version.

#### Node: Vector Store Tool
- **Type / role:** `toolVectorStore` — exposes vector retrieval as a tool named `documents`.
- **Configuration:**
  - Tool name: `documents`
  - Description: “Retrieves information about our client support policies and FAQs”
- **Connections:**
  - Receives **AI Vector Store** from Vector Storage.
  - Receives **AI Language Model** from OpenAI Chat Model1.
  - **AI Tool →** Email Draft Agent (agent can call retrieval)
- **Potential failures / edge cases:**
  - If vector store connection is missing, agent won’t be able to retrieve context.
  - Poor retrieval quality if documents are not chunked/embedded appropriately (outside scope of this workflow but impacts results).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail inbox for new emails | — | Set Content | ### Email Input\nTriggers on new emails and extracts key fields |
| Set Content | n8n-nodes-base.set | Extract/normalize email fields | Gmail Trigger | Client/Prospect Related? | ### Email Input\nTriggers on new emails and extracts key fields |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM for email classification | — | Client/Prospect Related? (AI) | ### Email Classification\nDetermines if the email needs your attention or is automated/system-generated |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON output | — | Client/Prospect Related? (AI) | ### Email Classification\nDetermines if the email needs your attention or is automated/system-generated |
| Client/Prospect Related? | @n8n/n8n-nodes-langchain.chainLlm | Runs classification prompt and parses JSON | Set Content | Customer Support? | ### Email Classification\nDetermines if the email needs your attention or is automated/system-generated |
| Customer Support? | n8n-nodes-base.switch | Route based on classification result | Client/Prospect Related? | Email Draft Agent; Response Not Customer Support | ### Notifications\nSends Telegram notifications for new drafts and non-support emails |
| Response Not Customer Support | n8n-nodes-base.telegram | Telegram alert for non-support emails | Customer Support? | — | ### Notifications\nSends Telegram notifications for new drafts and non-support emails |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for drafting replies | — | Email Draft Agent (AI) | ### Email Drafting Agent\nGenerates AI-powered draft responses. Customise the system prompt with your writing style and business context. |
| Email Draft Agent | @n8n/n8n-nodes-langchain.agent | Agent drafts reply, calls tools | Customer Support? | Response | ### Email Drafting Agent\nGenerates AI-powered draft responses. Customise the system prompt with your writing style and business context. |
| createDraft | n8n-nodes-base.gmailTool | Tool: create Gmail draft in thread | — (tool call) | Email Draft Agent (tool) | ### Email Drafting Agent\nGenerates AI-powered draft responses. Customise the system prompt with your writing style and business context. |
| Response | n8n-nodes-base.telegram | Telegram message with agent output | Email Draft Agent | — | ### Notifications\nSends Telegram notifications for new drafts and non-support emails |
| Embeddings OpenAI | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generate embeddings for vector store | — | Vector Storage (AI) | ### Knowledge Base (Optional)\nVector store for retrieving relevant docs when drafting responses |
| Vector Storage | @n8n/n8n-nodes-langchain.vectorStoreSupabase | Supabase vector store backend | Embeddings OpenAI (AI) | Vector Store Tool (AI) | ### Knowledge Base (Optional)\nVector store for retrieving relevant docs when drafting responses |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM supporting vector tool | — | Vector Store Tool (AI) | ### Knowledge Base (Optional)\nVector store for retrieving relevant docs when drafting responses |
| Vector Store Tool | @n8n/n8n-nodes-langchain.toolVectorStore | Tool: retrieve docs for agent | Vector Storage (AI), OpenAI Chat Model1 (AI) | Email Draft Agent (tool) | ### Knowledge Base (Optional)\nVector store for retrieving relevant docs when drafting responses |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — | (Comment node) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment | — | — | (Comment node) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment | — | — | (Comment node) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment | — | — | (Comment node) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment | — | — | (Comment node) |
| input-sticky | n8n-nodes-base.stickyNote | Comment | — | — | (Comment node) |
| notification-sticky | n8n-nodes-base.stickyNote | Comment | — | — | (Comment node) |
| main-sticky-note | n8n-nodes-base.stickyNote | Comment | — | — | (Comment node) |

> Note: Sticky notes are included as nodes in the workflow JSON; they don’t have runtime connections.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it e.g. “Auto-draft email replies with AI using Gmail and OpenAI”.
   - Set timezone (optional): `Australia/Brisbane`.

2. **Add Gmail Trigger**
   - Node: **Gmail Trigger**
   - Configure polling: **Every hour**.
   - Connect **Gmail OAuth2** credentials (Google project + consent screen + OAuth client, then authorize).
   - Ensure the trigger outputs message `headers`, `threadId`, and body `text` (disable “simple” mode if needed).

3. **Add Set node (“Set Content”)**
   - Node: **Set**
   - Add fields:
     - `emailBody` = expression `{{$json.text}}`
     - `threadID` = `{{$json.threadId}}`
     - `from` = `{{$json.headers.from}}`
   - Connect: **Gmail Trigger → Set Content**

4. **Add Classification chain**
   1) Node: **OpenRouter Chat Model**
      - Connect **OpenRouter API** credential.
   2) Node: **Structured Output Parser**
      - Provide JSON schema example like: `{"customerSupport": true}`
      - (Important) Prefer boolean in the example to match downstream routing.
   3) Node: **Chain LLM** (name: “Client/Prospect Related?”)
      - Prompt: paste your classification policy; include:
        - Clear TRUE/FALSE conditions
        - “Return ONLY valid JSON”
      - Reference the email body using expression: `{{ $json.emailBody }}`
      - Enable output parsing / connect the parser.
   - Connections:
     - **Set Content → Client/Prospect Related?**
     - **OpenRouter Chat Model (AI) → Client/Prospect Related?**
     - **Structured Output Parser (AI) → Client/Prospect Related?**

5. **Add Switch node (“Customer Support?”)**
   - Node: **Switch**
   - Add two rules on value `{{$json.output.customerSupport}}`:
     - Route A: equals `true`
     - Route B: equals `false`
   - (Recommended) Match boolean type, not `"true"` as string.
   - Connect: **Client/Prospect Related? → Customer Support?**

6. **Add Telegram notification for non-support**
   - Node: **Telegram**
   - Configure credentials: create a Telegram bot with BotFather, paste token in n8n credential.
   - Set `chatId` to your chat ID.
   - Message text example:
     - `You received an email at {{ $now.format('HH:mm') }} saying:\n\n{{ $('Set Content').item.json.emailBody }}`
   - Connect: **Customer Support? (false route) → Telegram node**

7. **Add Drafting Agent (LangChain Agent)**
   - Node: **AI Agent** (LangChain Agent in n8n)
   - User prompt:
     - `Draft a response to this email:\n\n{{ $('Set Content').item.json.emailBody }}`
   - System message:
     - Add your rules (professional, short, “I” voice, end with sign-off, etc.)
     - Instruct it to use a draft-creation tool.
   - Connect: **Customer Support? (true route) → Email Draft Agent**

8. **Add OpenAI Chat Model for drafting**
   - Node: **OpenAI Chat Model**
   - Connect OpenAI API credential.
   - Connect **AI language model output → Email Draft Agent**.

9. **Add Gmail Tool node to create drafts (“createDraft”)**
   - Node: **Gmail Tool**
   - Resource: **Draft**
   - Subject: `{{ $('Gmail Trigger').item.json.headers.subject }}`
   - Thread ID: `{{ $('Set Content').item.json.threadID }}`
   - “SendTo”:
     - Recommended robust approach:
       - Use the actual sender email parsed from headers, or `reply-to` if present.
   - Message/body:
     - Use the node’s AI-input mechanism so the agent provides the message (as in `$fromAI('Message', ...)`).
   - Connect **createDraft as an AI Tool to the agent** (tool connection, not main).

10. **Add Telegram node for drafting result (“Response”)**
   - Node: **Telegram**
   - Text: `{{ $json.output }}` (adjust if your agent returns a different field)
   - Connect: **Email Draft Agent → Response**

11. **(Optional) Add Knowledge Base via Supabase Vector Store**
   1) Node: **Embeddings OpenAI**
      - OpenAI credential (may be same as drafting).
   2) Node: **Supabase Vector Store**
      - Supabase credential (URL + service key or anon key depending on setup).
      - Table: `documents` (must exist).
      - Connect embeddings: **Embeddings OpenAI (AI embedding) → Vector Storage**
   3) Node: **OpenAI Chat Model** (second model, optional)
      - Connect to **Vector Store Tool** as its language model.
   4) Node: **Vector Store Tool**
      - Name: `documents`
      - Description: “Retrieves information about our client support policies and FAQs”
      - Connect:
        - **Vector Storage (AI vector store) → Vector Store Tool**
        - **OpenAI Chat Model1 (AI language model) → Vector Store Tool**
        - **Vector Store Tool (AI tool) → Email Draft Agent**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Auto-draft email replies with AI using Gmail and OpenAI — workflow notes, setup steps, and optional Supabase KB guidance | From the workflow’s main sticky note content |
| “Email Classification” note: determines if email needs attention or is automated/system-generated | Sticky note near classification nodes |
| Customisation required: adjust classification prompt and output variable if `customerSupport` isn’t relevant | Sticky note near classification section |
| “Email Drafting Agent” note: customise system prompt with writing style and business context | Sticky note near agent |
| Customisation required: add business details + 5–10 example emails | Sticky note near agent |
| “Knowledge Base (Optional)” note: vector store for retrieving docs when drafting responses | Sticky note near Supabase/vector nodes |
| “Notifications” note: Telegram notifications for drafts and non-support emails | Sticky note near Telegram routing |

### Key implementation warnings (important)
- **Switch rule type mismatch risk:** your switch compares to `"true"/"false"` strings, but your prompt requests booleans. Align to booleans to avoid silent misrouting.
- **`sendTo` expression references `to` that is not set** in “Set Content”. Either add `to` from Gmail headers, or simplify to always reply to `from` (or parse `reply-to`).
- **From header parsing:** `headers.from` often contains a display name; consider extracting the email address for reliable Gmail draft creation.