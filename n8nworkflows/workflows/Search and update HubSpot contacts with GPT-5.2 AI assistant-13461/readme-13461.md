Search and update HubSpot contacts with GPT-5.2 AI assistant

https://n8nworkflows.xyz/workflows/search-and-update-hubspot-contacts-with-gpt-5-2-ai-assistant-13461


# Search and update HubSpot contacts with GPT-5.2 AI assistant

## 1. Workflow Overview

**Purpose:**  
This workflow implements a conversational AI assistant that can **search, create, and update HubSpot contacts** via natural-language chat. The assistant uses **GPT-5.2** (OpenAI Chat Model) plus a set of **HubSpot tool nodes** exposed to the agent, allowing it to decide which HubSpot action to run based on user intent.

**Primary use cases:**
- Search contacts by **email**
- Search contacts by **company name**
- **Create or update** a contact (lead/customer) with basic fields

### 1.1 Input Reception (Chat Entry Point)
Receives user chat messages and hands them to the AI agent.

### 1.2 AI Reasoning + Memory
GPT-5.2 interprets the message, uses conversation memory to keep context, and decides which HubSpot tool(s) to call.

### 1.3 HubSpot Contact Operations (Tools)
Three HubSpot “tool” nodes are available to the agent:
- Search by email
- Search by company name
- Create/update contact

### 1.4 Documentation / Operator Notes
Two sticky notes provide configuration steps and a video link.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Chat Entry Point)

**Overview:**  
Accepts incoming chat messages and triggers the workflow, passing the message to the agent.

**Nodes involved:**
- When Chat Message Received from User

**Node details**

1) **When Chat Message Received from User**  
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — chat-based trigger node (webhook-backed).
- **Key configuration (interpreted):**
  - Acts as the workflow entry point for chat messages.
  - Uses an internal `webhookId` to receive chat events.
  - No special options configured.
- **Inputs:** None (trigger).
- **Outputs / connections:**
  - **Main output →** `HubSpot Contact Management Agent`
- **Version-specific notes:** TypeVersion **1.4** (chat trigger behavior can vary slightly across n8n releases; ensure your n8n supports this node/version).
- **Potential failures / edge cases:**
  - If chat is not properly set up in the n8n UI/environment, no events arrive.
  - Webhook URL rotation or environment changes can break external chat integrations if used.

---

### Block 2 — AI Reasoning + Memory (Agent Orchestration)

**Overview:**  
The agent receives the user message, uses GPT-5.2 to interpret intent, optionally uses memory for context continuity, and calls HubSpot tools as needed.

**Nodes involved:**
- HubSpot Contact Management Agent
- OpenAI Chat Model
- Chat Conversation Memory

**Node details**

1) **HubSpot Contact Management Agent**  
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates LLM + tools to fulfill tasks.
- **Key configuration (interpreted):**
  - No explicit agent options set (defaults apply).
  - Receives:
    - A chat model (GPT-5.2) via `ai_languageModel`
    - A memory buffer via `ai_memory`
    - HubSpot tools via `ai_tool`
- **Inputs:**
  - **Main input** from `When Chat Message Received from User`
- **Outputs:**
  - Returns the final conversational response to the chat context (exact output handling depends on n8n’s chat UI/runtime).
- **Connections:**
  - **Language model:** `OpenAI Chat Model` → Agent (`ai_languageModel`)
  - **Memory:** `Chat Conversation Memory` → Agent (`ai_memory`)
  - **Tools:** all three HubSpot tool nodes → Agent (`ai_tool`)
- **Version-specific notes:** TypeVersion **3**. Ensure your n8n includes the LangChain agent node and supports tool connections.
- **Potential failures / edge cases:**
  - If the model is misconfigured/unavailable, the agent cannot respond.
  - Tool calls can fail due to HubSpot auth/scopes/rate limits; agent may need retries or clearer prompts (not configured here).
  - If user input lacks required identifiers (no email/company), the agent may guess or call tools with empty values—see tool node edge cases below.

2) **OpenAI Chat Model**  
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat LLM to the agent.
- **Key configuration (interpreted):**
  - **Model:** `gpt-5.2` selected from the model list.
  - No extra model options set (defaults like temperature, max tokens etc. are not explicitly overridden).
  - **Credentials:** “OpenAi account” (OpenAI API key-based credential in n8n).
- **Inputs:** None (it’s attached via AI connection).
- **Outputs / connections:**
  - **AI language model output →** `HubSpot Contact Management Agent`
- **Version-specific notes:** TypeVersion **1.3**.
- **Potential failures / edge cases:**
  - Invalid/expired API key, missing billing, model not available for the account/region.
  - Rate limiting / timeouts under heavy usage.
  - Model name mismatch if your n8n/OpenAI integration list differs (ensure `gpt-5.2` exists in your environment).

3) **Chat Conversation Memory**  
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — keeps a rolling window of conversation context.
- **Key configuration (interpreted):**
  - Uses default buffer/window settings (none specified).
- **Inputs:** None (provided to agent via AI connection).
- **Outputs / connections:**
  - **AI memory output →** `HubSpot Contact Management Agent`
- **Version-specific notes:** TypeVersion **1.3**.
- **Potential failures / edge cases:**
  - Memory window defaults may be too small/large depending on usage; could cause loss of context or higher token usage.
  - In multi-user environments, ensure chat sessions are separated correctly by n8n runtime (otherwise memory leakage across sessions is possible depending on deployment/session handling).

---

### Block 3 — HubSpot Contact Operations (Agent Tools)

**Overview:**  
These nodes are configured as “tools” the agent can call. The tool parameters are populated using `$fromAI(...)` expressions, meaning the agent supplies structured inputs (email, first name, etc.) at runtime.

**Nodes involved:**
- Search contacts in HubSpot
- Find Contact by Company Name
- Create or update a contact in HubSpot

**Node details**

1) **Search contacts in HubSpot**  
- **Type / role:** `n8n-nodes-base.hubspotTool` — HubSpot tool node, configured to **search** contacts.
- **Key configuration (interpreted):**
  - **Operation:** `search`
  - **Return all:** enabled (returns all matches, not just the first page)
  - **Authentication:** App Token
  - **Filter:** property `email` (HubSpot property referenced as `email|string`)
  - **Operator:** `CONTAINS_TOKEN`
  - **Filter value:** set by AI via expression:  
    - `{{$fromAI('filterGroupsValues0_filterValues0_Value', '', 'string')}}`
- **Credentials:** HubSpot App Token credential “HubSpot account”.
- **Inputs:** None via main; invoked by agent as an AI tool.
- **Outputs / connections:**
  - Tool output returns to `HubSpot Contact Management Agent` (`ai_tool`)
- **Version-specific notes:** TypeVersion **2.2** of HubSpot tool node.
- **Potential failures / edge cases:**
  - **Empty/invalid filter value** from the agent could yield broad/empty results.
  - `CONTAINS_TOKEN` on email can match partial tokens; may return multiple contacts (ambiguity).
  - HubSpot API rate limits or permission errors if scopes are missing.
  - If HubSpot property naming differs or email is unset, results may be inconsistent.

2) **Find Contact by Company Name**  
- **Type / role:** `n8n-nodes-base.hubspotTool` — HubSpot tool node, configured to **search** contacts by company.
- **Key configuration (interpreted):**
  - **Operation:** `search`
  - **Return all:** enabled
  - **Authentication:** App Token
  - **Filter:** property `company` (referenced as `company|string`)
  - **Operator:** `CONTAINS_TOKEN`
  - **Filter value:** AI-provided:  
    - `{{$fromAI('filterGroupsValues0_filterValues0_Value', '', 'string')}}`
- **Credentials:** HubSpot App Token “HubSpot account”.
- **Inputs:** Tool invocation only.
- **Outputs / connections:**
  - Tool output returns to `HubSpot Contact Management Agent` (`ai_tool`)
- **Version-specific notes:** TypeVersion **2.2**.
- **Potential failures / edge cases:**
  - Many contacts can share similar company tokens → large result sets and ambiguity.
  - Company field may not be populated in HubSpot for many contacts.
  - Same auth/scope/rate-limit concerns as above.

3) **Create or update a contact in HubSpot**  
- **Type / role:** `n8n-nodes-base.hubspotTool` — HubSpot tool node to **create or update** a contact record.
- **Key configuration (interpreted):**
  - **Authentication:** App Token
  - **Primary identifier:** `email` is provided by AI:  
    - `{{$fromAI('Email', '', 'string')}}`
  - **Additional fields (AI provided):**
    - `firstName`: `{{$fromAI('First_Name', '', 'string')}}`
    - `lastName`: `{{$fromAI('Last_Name', '', 'string')}}`
    - `companyName`: `{{$fromAI('Company_Name', '', 'string')}}`
- **Credentials:** HubSpot App Token “HubSpot account”.
- **Inputs:** Tool invocation only.
- **Outputs / connections:**
  - Tool output returns to `HubSpot Contact Management Agent` (`ai_tool`)
- **Version-specific notes:** TypeVersion **2.2**.
- **Potential failures / edge cases:**
  - **Email missing or malformed**: HubSpot may reject create/update or create duplicates depending on API behavior and node implementation.
  - If the agent passes empty strings for names/company, you may overwrite existing values with blanks (depends on node behavior; consider adding conditional field setting if needed).
  - Permission/scopes required:
    - read/write contacts (see sticky note)
  - API rate limits, validation errors, or conflict errors.

**About `$fromAI(...)`:**  
These expressions indicate fields are *structured inputs supplied by the agent at runtime* (rather than coming from upstream node JSON). If the agent cannot infer a field, it may pass an empty string, which can reduce search quality or cause update issues.

---

### Block 4 — Documentation / Operator Notes (Sticky Notes)

**Overview:**  
Contains setup requirements for HubSpot and OpenAI plus a video link.

**Nodes involved:**
- Workflow Description (sticky note)
- Video Walkthrough (sticky note)

**Node details**

1) **Workflow Description** (Sticky Note)  
- **Type / role:** `n8n-nodes-base.stickyNote` — internal documentation.
- **Content highlights:**
  - HubSpot private app setup at `developers.hubspot.com`
  - Required scopes:
    - `crm.objects.contacts.read`
    - `crm.objects.contacts.write`
  - Create HubSpot credential in n8n using **APP Token**
  - OpenAI credentials required
  - Tools listed (email search, company search, create/update)
  - Model is GPT-5.2 but can be changed

2) **Video Walkthrough** (Sticky Note)  
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Link:** https://youtu.be/GBKXYh2j74o  
- **Thumbnail:** https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_reczH8mtRj3GIIRiU.jpg

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When Chat Message Received from User | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry trigger | — | HubSpot Contact Management Agent |  |
| HubSpot Contact Management Agent | @n8n/n8n-nodes-langchain.agent | AI agent orchestrator (LLM + tools + memory) | When Chat Message Received from User | (Chat response; tool calls) |  |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider (GPT-5.2) | — | HubSpot Contact Management Agent |  |
| Chat Conversation Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversation context memory | — | HubSpot Contact Management Agent |  |
| Search contacts in HubSpot | n8n-nodes-base.hubspotTool | HubSpot tool: search contacts by email | — (agent tool call) | HubSpot Contact Management Agent |  |
| Find Contact by Company Name | n8n-nodes-base.hubspotTool | HubSpot tool: search contacts by company | — (agent tool call) | HubSpot Contact Management Agent |  |
| Create or update a contact in HubSpot | n8n-nodes-base.hubspotTool | HubSpot tool: upsert contact by email + fields | — (agent tool call) | HubSpot Contact Management Agent |  |
| Workflow Description | n8n-nodes-base.stickyNote | Embedded configuration notes | — | — | ## Workflow Overview; HubSpot private app setup at `developers.hubspot.com`; scopes `crm.objects.contacts.read`, `crm.objects.contacts.write`; create HubSpot APP Token credential; OpenAI credentials; tools list; GPT-5.2 configurable |
| Video Walkthrough | n8n-nodes-base.stickyNote | Embedded video link | — | — | # Video Walkthrough; https://youtu.be/GBKXYh2j74o; thumbnail https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_reczH8mtRj3GIIRiU.jpg |

---

## 4. Reproducing the Workflow from Scratch

1) **Create HubSpot credentials (App Token)**
   1. In HubSpot: create a **private app** (Legacy Apps) at `https://developers.hubspot.com`.
   2. Add scopes:
      - `crm.objects.contacts.read`
      - `crm.objects.contacts.write`
   3. Copy the **Access token**.
   4. In n8n: **Credentials → New → HubSpot** (App Token / Private App token method).
   5. Paste the token and save (name e.g., “HubSpot account”).

2) **Create OpenAI credentials**
   1. In n8n: **Credentials → New → OpenAI API**.
   2. Add your API key (name e.g., “OpenAi account”).
   3. Save.

3) **Add trigger node**
   1. Create node: **Chat Trigger** (`When Chat Message Received from User`).
   2. Keep default options (no special configuration required).

4) **Add the agent node**
   1. Create node: **AI Agent** (LangChain Agent) named `HubSpot Contact Management Agent`.
   2. Keep default agent options (as in the workflow).

5) **Connect trigger to agent**
   - Connect: `When Chat Message Received from User` (main) → `HubSpot Contact Management Agent` (main).

6) **Add the OpenAI chat model node**
   1. Create node: **OpenAI Chat Model**.
   2. Select **model:** `gpt-5.2` (or a compatible alternative available in your n8n).
   3. Assign **OpenAI credentials:** “OpenAi account”.
   4. Connect via AI port: `OpenAI Chat Model` → `HubSpot Contact Management Agent` using **ai_languageModel** connection.

7) **Add conversation memory**
   1. Create node: **Chat Conversation Memory** (`memoryBufferWindow`).
   2. Leave defaults.
   3. Connect via AI port: `Chat Conversation Memory` → `HubSpot Contact Management Agent` using **ai_memory** connection.

8) **Add HubSpot tool: search by email**
   1. Create node: **HubSpot Tool**.
   2. Set **Authentication:** App Token; choose “HubSpot account”.
   3. Set **Operation:** Search.
   4. Enable **Return All**.
   5. Add a filter group with:
      - **Property:** `email` (string)
      - **Operator:** `CONTAINS_TOKEN`
      - **Value expression:** `{{$fromAI('filterGroupsValues0_filterValues0_Value', '', 'string')}}`
   6. Connect via AI port: this HubSpot tool → Agent using **ai_tool**.

9) **Add HubSpot tool: search by company name**
   1. Create another **HubSpot Tool** node named `Find Contact by Company Name`.
   2. Set **Operation:** Search, **Return All** = true, **Authentication:** App Token (same credential).
   3. Filter:
      - **Property:** `company` (string)
      - **Operator:** `CONTAINS_TOKEN`
      - **Value expression:** `{{$fromAI('filterGroupsValues0_filterValues0_Value', '', 'string')}}`
   4. Connect to agent via **ai_tool**.

10) **Add HubSpot tool: create/update contact**
   1. Create **HubSpot Tool** node named `Create or update a contact in HubSpot`.
   2. Set **Authentication:** App Token (same credential).
   3. Select operation **Create or Update** contact (wording may vary by node UI).
   4. Set **Email** field to expression: `{{$fromAI('Email', '', 'string')}}`
   5. In **Additional Fields**, map:
      - First name: `{{$fromAI('First_Name', '', 'string')}}`
      - Last name: `{{$fromAI('Last_Name', '', 'string')}}`
      - Company name: `{{$fromAI('Company_Name', '', 'string')}}`
   6. Connect to agent via **ai_tool**.

11) **(Optional) Add sticky notes**
   - Add two Sticky Notes containing:
     - The setup/scopes notes and the OpenAI note
     - Video link: https://youtu.be/GBKXYh2j74o

12) **Test**
   - Start a chat and try prompts like:
     - “Find the HubSpot contact with email john@example.com”
     - “Search contacts at Acme Inc”
     - “Create a contact: Jane Doe, jane@acme.com, company Acme”

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| HubSpot developer/private app setup; scopes required: `crm.objects.contacts.read`, `crm.objects.contacts.write`; create n8n HubSpot credential using APP Token | `https://developers.hubspot.com` |
| OpenAI credentials must be configured in n8n; model used is GPT-5.2 but can be changed | n8n Credentials (OpenAI) |
| Video Walkthrough | https://youtu.be/GBKXYh2j74o |
| Video thumbnail image | https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_reczH8mtRj3GIIRiU.jpg |