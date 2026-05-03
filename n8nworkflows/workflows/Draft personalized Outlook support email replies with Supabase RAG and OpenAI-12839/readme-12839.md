Draft personalized Outlook support email replies with Supabase RAG and OpenAI

https://n8nworkflows.xyz/workflows/draft-personalized-outlook-support-email-replies-with-supabase-rag-and-openai-12839


# Draft personalized Outlook support email replies with Supabase RAG and OpenAI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Draft personalized Outlook support email replies with Supabase RAG and OpenAI

**Purpose:**  
Automatically draft **personalized Outlook email replies** for inbound support messages. The workflow enriches the email with **CRM contact context from Supabase**, retrieves relevant documentation via a **Supabase Vector Store (RAG)**, then uses **OpenAI GPT‑4o** (via the n8n LangChain Agent node) to produce a **short HTML reply**. Finally, it creates an Outlook **draft reply** and logs key metadata to **Google Sheets**.

**Primary use cases:**
- Customer support teams who want draft replies (human-in-the-loop) rather than auto-send.
- Organizations with product manuals/specs/how-to guides indexed in a vector store.
- Personalization based on CRM data (department/role/hospital/company).

### 1.1 Email Intake & Workflow Initialization
Triggered by new emails in Microsoft Outlook, then passes data onward.

### 1.2 CRM Context Lookup (Supabase)
Looks up sender by email in a Supabase “contacts” table to personalize greeting and context.

### 1.3 Prompt/Context Construction
Builds a consolidated “chatInput” including sender details, email subject/body, and business rules.

### 1.4 AI Reasoning + RAG Retrieval (LangChain Agent)
Agent uses GPT‑4o and a Supabase Vector Store retrieval tool (with OpenAI embeddings) to draft a structured JSON response containing `reply_html` and classification fields.

### 1.5 Draft Creation in Outlook + Logging
Creates an Outlook **draft reply** and appends/updates a row in Google Sheets for tracking.

---

## 2. Block-by-Block Analysis

### Block 1 — Email Intake & Basic Pipeline Start
**Overview:** Watches for incoming Outlook emails and starts the workflow execution chain.  
**Nodes involved:** `On Email Received`, `Config Variables`

#### Node: On Email Received
- **Type / role:** Microsoft Outlook Trigger (`microsoftOutlookTrigger`) — entry point polling for new emails.
- **Configuration (interpreted):**
  - Polling mode set to **every minute**.
  - No explicit filters configured (processes all incoming emails visible to the connected mailbox scope).
- **Key data used later:** `json.from`, `json.subject`, `json.body` / `json.bodyPreview`.
- **Outputs:** Main output → `Config Variables`.
- **Edge cases / failures:**
  - Outlook credential expiry / OAuth consent issues.
  - High email volume may cause processing backlog.
  - If body is not returned (permissions or message format), downstream prompt may rely on `bodyPreview`.

#### Node: Config Variables
- **Type / role:** Set (`set`) — placeholder for workflow-level variables/config.
- **Configuration (interpreted):**
  - `includeOtherFields: true` so it passes through all incoming email fields unchanged.
  - No assignments currently defined (acts as a pass-through).
- **Input:** `On Email Received`.
- **Output:** `Fetch Contact Info`.
- **Edge cases / failures:**
  - None functionally; but commonly used to store constants—currently empty, so any expected config must be placed elsewhere.

---

### Block 2 — Context Building (CRM Lookup in Supabase)
**Overview:** Enriches the email with sender profile fields from Supabase to improve personalization and routing.  
**Nodes involved:** `Fetch Contact Info`

#### Node: Fetch Contact Info
- **Type / role:** Supabase (`supabase`) — query contact record by sender email.
- **Configuration (interpreted):**
  - Operation: **Get** from a table defined by placeholder: `<__PLACEHOLDER_VALUE__SUPABASE_CONTACTS_TABLE__>`.
  - Filter condition:
    - Column `email` equals `$('On Email Received').first().json.from`
- **Input:** `Config Variables` (email data passed through).
- **Output:** `Build AI Prompt` (contact record becomes `$json` for the next node).
- **Edge cases / failures:**
  - Table name placeholder not replaced → runtime failure.
  - No matching contact:
    - Depending on Supabase node behavior, may output empty result or error. Downstream code assumes `$json` exists; if empty, personalization may degrade or fail.
  - Supabase auth errors / row-level security (RLS) denying reads.
- **Version-specific notes:** Supabase node `typeVersion: 1` (older vs newer variants); ensure your n8n has this node available and configured similarly.

**Sticky note context:**  
“### 1. Context Building … look up the sender in Supabase CRM to get their Name, Role, and Department.”

---

### Block 3 — Prompt & Business Rules Construction
**Overview:** Combines the inbound email and CRM lookup into a single `chatInput` string with explicit business rules to control tone, safety, and brevity.  
**Nodes involved:** `Build AI Prompt`

#### Node: Build AI Prompt
- **Type / role:** Code (`code`) — transforms email + contact into an LLM-ready prompt payload.
- **Configuration (interpreted):**
  - Reads:
    - `email` from `$('On Email Received').first().json`
    - `contact` from the current node input (`$json`) coming from Supabase
  - Extracts contact fields (must match Supabase schema):
    - `full_name`, `hospital_name`, `department_name`, `role`
  - Builds `businessRules` string with constraints:
    - short reply (2–3 paragraphs, optional bullets)
    - no clinical/emergency advice
    - accessory suggestions only if relevant
    - consultative selling, no hard sell
    - if unsure, promise to confirm instead of guessing
  - Builds `chatInput` with sender + email subject/body + rules
  - Returns:
    - `json.chatInput`
    - `json.originalEmail`
- **Input:** `Fetch Contact Info` (contact record).
- **Output:** `AI Support Agent` (agent receives `chatInput`).
- **Key expressions/variables:**
  - `$('On Email Received').first().json` to reference the trigger data.
  - `email.body || email.bodyPreview || ''` to handle missing body.
- **Edge cases / failures:**
  - If Supabase returns no item and n8n provides no `$json`, `contact.full_name` etc. may throw depending on runtime. If it returns `{}` this is fine.
  - Email body can be HTML; passing raw HTML into prompt may reduce quality—consider stripping tags if needed.
- **Version-specific notes:** Code node `typeVersion: 2`.

---

### Block 4 — AI Reasoning + RAG (LangChain Agent with Tools)
**Overview:** Uses a LangChain Agent configured with GPT‑4o and a Supabase Vector Store retrieval tool (RAG) to (1) classify intent/urgency/department/products and (2) draft an HTML reply, returning a single JSON object.  
**Nodes involved:** `AI Support Agent`, `GPT-4o Model`, `Vector Store (Supabase)`, `OpenAI Embeddings`

#### Node: AI Support Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates reasoning and tool usage, produces structured JSON response.
- **Configuration (interpreted):**
  - Input text: `={{ $json.chatInput }}`
  - Prompt type: “define” (custom system message)
  - System message defines:
    - Persona placeholders: `[YOUR_AGENT_NAME]`, `[YOUR_JOB_TITLE]`, `[YOUR_COMPANY_NAME]`, `[YOUR_PRODUCT_TYPE]`
    - Required decisions: `intent`, `primary_product_codes[]`, `urgency`, `department`
    - Tool-use instruction: use vector store tool for relevant department/products
    - Output format constraint: **Return a single JSON object ONLY** with keys:
      - `reply_html`, `intent`, `primary_product_codes`, `urgency`, `department`
- **Inputs:**
  - Main input from `Build AI Prompt`
  - AI language model input from `GPT-4o Model`
  - AI tool input from `Vector Store (Supabase)`
- **Output:** Main output → `Draft Outlook Reply`
- **Edge cases / failures:**
  - Model may return non-JSON or invalid JSON (common LLM failure mode) → downstream nodes referencing `$json.reply_html` can fail.
  - If vector store tool misconfigured, retrieval fails and answer quality decreases; agent may still respond but with less grounding.
  - Placeholder persona values not replaced → replies include literal `[YOUR_AGENT_NAME]` etc.
- **Sticky note context:**  
“### 2. AI Reasoning (RAG) … uses GPT‑4o … Vector Store Tool …”

#### Node: GPT-4o Model
- **Type / role:** LangChain Chat Model (`lmChatOpenAi`) — provides GPT‑4o as the agent’s LLM.
- **Configuration (interpreted):**
  - Model: `gpt-4o`
  - No special options set.
- **Connection:** AI language model output → `AI Support Agent` (as `ai_languageModel`).
- **Edge cases / failures:**
  - OpenAI credential missing/invalid.
  - Rate limits / timeouts.
  - Model availability or name mismatch in older n8n versions.

#### Node: Vector Store (Supabase)
- **Type / role:** LangChain Vector Store (`vectorStoreSupabase`) — retrieval tool used by the agent for RAG.
- **Configuration (interpreted):**
  - Mode: **retrieve-as-tool** (exposed to the agent as a callable tool)
  - Supabase table: `<__PLACEHOLDER_VALUE__SUPABASE_VECTOR_TABLE__>`
  - Query function name: `<__PLACEHOLDER_VALUE__SUPABASE_FUNCTION_NAME__>` (commonly a Postgres function for similarity search)
  - Tool description: retrieval purpose description (used by agent to decide to call it)
- **Connections:**
  - Receives embeddings provider from `OpenAI Embeddings` (as `ai_embedding`)
  - Exposes `ai_tool` output to `AI Support Agent`
- **Edge cases / failures:**
  - Placeholders not replaced → runtime failure.
  - Supabase RPC/function not deployed or wrong signature.
  - Vector table schema mismatch (expected columns like embedding/vector, content, metadata).
  - RLS denies access.
  - Large retrieval payloads can increase token usage.

#### Node: OpenAI Embeddings
- **Type / role:** LangChain Embeddings (`embeddingsOpenAi`) — generates embeddings for the retrieval queries.
- **Configuration (interpreted):**
  - Uses default embeddings model/options (not explicitly set here).
- **Connection:** `ai_embedding` → `Vector Store (Supabase)`.
- **Edge cases / failures:**
  - OpenAI credential/rate limits.
  - If default embeddings model changes, retrieval quality may shift.

---

### Block 5 — Outlook Draft + Google Sheets Logging
**Overview:** Creates an Outlook draft reply using the AI-generated HTML body, then logs metadata (status, urgency, department, product) into Google Sheets.  
**Nodes involved:** `Draft Outlook Reply`, `Log to Google Sheets`

#### Node: Draft Outlook Reply
- **Type / role:** Microsoft Outlook (`microsoftOutlook`) — creates a **draft** email.
- **Configuration (interpreted):**
  - Resource: `draft`
  - Subject: `RE: {{ $('On Email Received').first().json.subject }}`
  - To recipients: `={{ $('On Email Received').first().json.from }}`
  - Body:
    - `bodyContent`: `={{ $json.reply_html }}`
    - `bodyContentType`: `html`
- **Input:** `AI Support Agent` output (must include `reply_html`).
- **Output:** `Log to Google Sheets`
- **Edge cases / failures:**
  - If `reply_html` missing/invalid → draft may be empty or node may error.
  - Outlook permissions/scopes may prevent drafting.
  - `from` field format differences (could be name+email vs email) may require normalization.

#### Node: Log to Google Sheets
- **Type / role:** Google Sheets (`googleSheets`) — append or update a row for monitoring.
- **Configuration (interpreted):**
  - Operation: `appendOrUpdate`
  - Matching column: `Email Subject` (used to decide update vs append)
  - Sheet:
    - Name placeholder: `<__PLACEHOLDER_VALUE__SHEET_NAME__>`
    - Document ID placeholder: `<__PLACEHOLDER_VALUE__GOOGLE_SHEET_ID__>`
  - Columns mapped:
    - From: Outlook `from`
    - Status: `"Success"`
    - Product: `$('AI Support Agent').first().json.primary_product_codes[0] || ''`
    - Urgency: `$('AI Support Agent').first().json.urgency`
    - Timestamp: `$now.toISO()`
    - Department: `$('AI Support Agent').first().json.department`
    - Email Subject: Outlook subject
- **Input:** `Draft Outlook Reply`
- **Output:** None (end of workflow)
- **Edge cases / failures:**
  - Placeholders not replaced → runtime failure.
  - Matching on “Email Subject” can cause collisions (different emails with same subject overwrite).
  - If agent output missing fields (`urgency`, `department`) expressions may evaluate to `undefined` (usually allowed but can break strict schemas).
  - Google API quota/auth issues.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Template Header | Sticky Note | Documentation / workflow instructions | — | — | ## 🤖 AI Email Support Agent with RAG  \n**Overview**  \nThis workflow automates email responses by combining CRM data (Supabase) with technical knowledge (Vector Store) to draft accurate, personalized replies in Outlook.  \n**Requirements**  \n- **Microsoft Outlook** (Business Account)  \n- **Supabase** (Postgres DB & Vector Store)  \n- **OpenAI** (API Key for GPT-4o & Embeddings)  \n- **Google Sheets** (For logging)  \n**Setup Instructions**  \n1. **Credentials**: Connect your Outlook, OpenAI, Supabase, and Google accounts.  \n2. **Supabase**: Ensure you have a 'contacts' table and a Vector Store set up for your product manuals.  \n3. **Placeholders**: Update the **Green** nodes with your specific IDs (Sheet ID, Table Name, etc.).  \n4. **Code Node**: Update the 'Business Rules' and Persona details in the 'Build AI Prompt' node. |
| Context Sticky Note | Sticky Note | Documentation / block header | — | — | ### 1. Context Building  \nWe trigger on new emails, then immediately look up the sender in our Supabase CRM to get their Name, Role, and Department. This allows the AI to personalize the greeting. |
| AI Sticky Note | Sticky Note | Documentation / block header | — | — | ### 2. AI Reasoning (RAG)  \nThe Agent uses **GPT-4o** to analyze the email intent. It utilizes the **Vector Store Tool** to look up technical answers from your documentation before formulating a reply. |
| On Email Received | Microsoft Outlook Trigger | Entry point: poll mailbox for new emails | — | Config Variables | |
| Config Variables | Set | Pass-through / place for constants | On Email Received | Fetch Contact Info | |
| Fetch Contact Info | Supabase | Lookup sender in CRM by email | Config Variables | Build AI Prompt | |
| Build AI Prompt | Code | Build prompt context + business rules | Fetch Contact Info | AI Support Agent | |
| AI Support Agent | LangChain Agent | Classify + draft reply using model and vector tool | Build AI Prompt; GPT-4o Model; Vector Store (Supabase) | Draft Outlook Reply | |
| GPT-4o Model | OpenAI Chat Model (LangChain) | Provides GPT‑4o to agent | — | AI Support Agent | |
| Vector Store (Supabase) | Supabase Vector Store (LangChain) | Retrieval tool for manuals/docs (RAG) | OpenAI Embeddings | AI Support Agent | |
| OpenAI Embeddings | OpenAI Embeddings (LangChain) | Embeddings provider for vector queries | — | Vector Store (Supabase) | |
| Draft Outlook Reply | Microsoft Outlook | Create draft reply email (HTML body) | AI Support Agent | Log to Google Sheets | |
| Log to Google Sheets | Google Sheets | Append/update tracking row | Draft Outlook Reply | — | |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Microsoft Outlook Trigger** node named **“On Email Received”**  
   - Set polling to **Every Minute**.  
   - Connect Microsoft Outlook credentials (OAuth2) with mailbox access.
3. **Add a Set node** named **“Config Variables”**  
   - Turn on **Include Other Fields**.  
   - Leave assignments empty (or add constants if you want later).  
   - Connect: `On Email Received → Config Variables`.
4. **Add a Supabase node** named **“Fetch Contact Info”**  
   - Operation: **Get**  
   - Table: set to your contacts table (replace `<__PLACEHOLDER_VALUE__SUPABASE_CONTACTS_TABLE__>`).  
   - Filter: column `email` equals expression `{{$('On Email Received').first().json.from}}`  
   - Configure Supabase credentials (URL + service key or anon key depending on RLS and your security model).  
   - Connect: `Config Variables → Fetch Contact Info`.
5. **Add a Code node** named **“Build AI Prompt”**  
   - Paste logic that:
     - Reads the trigger email via `$('On Email Received').first().json`
     - Reads the Supabase contact record from `$json`
     - Builds `chatInput` and returns `{ json: { chatInput, originalEmail } }`
   - Ensure the contact fields match your schema (e.g., `full_name`, `department_name`, etc.).  
   - Connect: `Fetch Contact Info → Build AI Prompt`.
6. **Add an OpenAI Chat Model (LangChain) node** named **“GPT-4o Model”**  
   - Select model **gpt-4o**.  
   - Attach OpenAI API credentials.
7. **Add an OpenAI Embeddings (LangChain) node** named **“OpenAI Embeddings”**  
   - Keep defaults or explicitly set your preferred embeddings model.  
   - Attach OpenAI API credentials (can be same).
8. **Add a Supabase Vector Store (LangChain) node** named **“Vector Store (Supabase)”**  
   - Mode: **Retrieve as tool**  
   - Table name: your vector table (replace `<__PLACEHOLDER_VALUE__SUPABASE_VECTOR_TABLE__>`).  
   - Query function name: your similarity-search RPC/function (replace `<__PLACEHOLDER_VALUE__SUPABASE_FUNCTION_NAME__>`).  
   - Set tool description to match your documentation domain.  
   - Configure Supabase credentials (must have access to vector table/function).  
   - Connect: `OpenAI Embeddings (ai_embedding) → Vector Store (Supabase)`.
9. **Add a LangChain Agent node** named **“AI Support Agent”**  
   - Input text: `{{$json.chatInput}}`  
   - Provide a **System Message** that:
     - Defines persona ([YOUR_AGENT_NAME], etc. — replace with real values)
     - Requires JSON-only output with `reply_html`, `intent`, `primary_product_codes`, `urgency`, `department`
   - Connect:
     - `Build AI Prompt → AI Support Agent` (main)
     - `GPT-4o Model → AI Support Agent` (ai_languageModel)
     - `Vector Store (Supabase) → AI Support Agent` (ai_tool)
10. **Add Microsoft Outlook node** named **“Draft Outlook Reply”**  
   - Resource: **Draft**  
   - Subject: `RE: {{$('On Email Received').first().json.subject}}`  
   - To: `{{$('On Email Received').first().json.from}}`  
   - Body type: **HTML**  
   - Body content: `{{$json.reply_html}}`  
   - Connect: `AI Support Agent → Draft Outlook Reply`.
11. **Add Google Sheets node** named **“Log to Google Sheets”**  
   - Operation: **Append or Update**  
   - Document ID: your spreadsheet ID (replace placeholder).  
   - Sheet name: your target tab (replace placeholder).  
   - Matching column: **Email Subject** (or choose a safer unique key).  
   - Map columns using expressions:
     - From: `{{$('On Email Received').first().json.from}}`
     - Status: `Success`
     - Product: `{{$('AI Support Agent').first().json.primary_product_codes[0] || ''}}`
     - Urgency: `{{$('AI Support Agent').first().json.urgency}}`
     - Department: `{{$('AI Support Agent').first().json.department}}`
     - Timestamp: `{{$now.toISO()}}`
     - Email Subject: `{{$('On Email Received').first().json.subject}}`
   - Connect: `Draft Outlook Reply → Log to Google Sheets`.
12. **(Optional) Add sticky notes** to document blocks and placeholders for future maintainers.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Ensure you have a `contacts` table in Supabase and a Vector Store set up for product manuals/docs. | From “Template Header” sticky note |
| Update placeholder values (Sheet ID, sheet name, Supabase table/function names, persona fields). | From “Template Header” sticky note |
| Update “Business Rules” and persona details inside the “Build AI Prompt” / agent system message to match your organization’s policy. | From “Template Header” sticky note |
| Context building: sender lookup enables personalization (name/role/department). | From “Context Sticky Note” |
| AI Reasoning: GPT‑4o + Vector Store tool for documentation-grounded replies. | From “AI Sticky Note” |