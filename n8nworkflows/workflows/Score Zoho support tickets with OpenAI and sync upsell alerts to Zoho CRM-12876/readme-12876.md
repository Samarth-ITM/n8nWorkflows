Score Zoho support tickets with OpenAI and sync upsell alerts to Zoho CRM

https://n8nworkflows.xyz/workflows/score-zoho-support-tickets-with-openai-and-sync-upsell-alerts-to-zoho-crm-12876


# Score Zoho support tickets with OpenAI and sync upsell alerts to Zoho CRM

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Zoho CRM - Help Desk to CRM Intelligence Bridge  
**Stated title:** Score Zoho support tickets with OpenAI and sync upsell alerts to Zoho CRM  
**Purpose:** Ingest support ticket events via webhook, ensure a corresponding Zoho CRM Contact exists (create or update), retrieve the customer’s CRM record as “history,” run an AI analysis to detect upsell potential, and email an account manager when the opportunity score meets a configured threshold.

### 1.1 Input Reception & Global Configuration
- Receives ticket payload (case details + customer email).
- Sets global parameters (notification email, upsell threshold) used later in branching and alerts.

### 1.2 CRM Contact Lookup, Create/Update, and History Retrieval
- Searches Zoho CRM Contacts.
- If found: updates a “Last Ticket Date” custom field.
- If not found: creates a new Contact from ticket data.
- Retrieves the CRM Contact record to provide customer “history” context for AI.

### 1.3 AI Analysis (LLM Agent + Structured Output)
- Sends current ticket + retrieved CRM data to an AI Agent (OpenAI chat model).
- Forces a structured JSON output (upsellScore, hasOpportunity, patterns, recommendations, reasoning).

### 1.4 Opportunity Filtering & Email Notification
- Checks the AI results against:
  - `hasOpportunity == true`
  - `upsellScore >= upsellThreshold`
- If both pass: emails the account manager with a detailed HTML summary.

---

## 2. Block-by-Block Analysis

### Block 1 — Webhook Trigger & Workflow Configuration
**Overview:** Accepts incoming support ticket events and defines reusable workflow-level parameters (email recipient and score threshold).  
**Nodes involved:** Webhook Trigger, Workflow Configuration

#### Node: Webhook Trigger
- **Type / role:** `Webhook` — entry point; receives POST requests from the support platform (Zoho Desk/Zoho CRM webhook or another help desk).
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `support-ticket` (final endpoint resembles `/webhook/support-ticket`)
  - **Response:** configured with empty response data (no custom payload returned).
- **Inputs/outputs:**
  - **No input** (trigger).
  - **Output →** Workflow Configuration.
- **Key data used downstream:**
  - Ticket fields referenced throughout as: `$('Webhook Trigger').item.json.body.<Field>`
  - Expected fields include: `Email, CaseId, CaseNumber, CaseOwner, Subject, Description, Priority, Status`
- **Edge cases / failures:**
  - Missing/renamed fields in `body` will break expressions later (e.g., `body.Email`).
  - If the sender uses `application/json` vs `x-www-form-urlencoded`, field locations can differ; confirm mapping.
  - Public webhook URL exposure: consider authentication or secret path.

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines global constants used later.
- **Configuration choices:**
  - Sets:
    - `accountManagerEmail` (string): `user@example.com`
    - `upsellThreshold` (number): `3`
  - “Include other fields” enabled, so it keeps incoming webhook payload alongside these config values.
- **Inputs/outputs:**
  - **Input ←** Webhook Trigger
  - **Output →** Get Customer Record
- **Edge cases / failures:**
  - If `includeOtherFields` is disabled, later nodes relying on webhook payload via item context may still work (because they reference the trigger node directly), but any downstream usage of merged fields could change.
  - Mis-typed threshold (string vs number) can cause IF numeric comparison issues.

**Sticky note coverage (applies to nodes in this block):**  
- “## Webhook Trigger on On Ticket Add/Edit & Set Configuration …”  
- “# How It Works …”

---

### Block 2 — CRM Data Synchronization and History Retrieval
**Overview:** Ensures a Zoho CRM Contact exists for the ticket email/owner context, updates a “Last Ticket Date” field for existing contacts, and fetches the contact record for analysis context.  
**Nodes involved:** Get Customer Record, Customer Exists?, Create Customer Record, Update Customer Record, Get Ticket History

#### Node: Get Customer Record
- **Type / role:** `Zoho CRM` — searches Contacts.
- **Configuration choices (interpreted):**
  - **Resource:** Contact
  - **Operation:** Get All
  - **Limit:** 1
  - **Credentials:** Zoho OAuth2 (“Zoho account 9”)
  - **Always Output Data:** true (prevents workflow from stopping when no records are returned).
- **Inputs/outputs:**
  - **Input ←** Workflow Configuration
  - **Output →** Customer Exists?
- **Important caveat (logic gap):**
  - The node is not configured with any filter criteria (e.g., by email). As written, it will likely return the first contact in the CRM (or an arbitrary one), not the matching customer.
- **Edge cases / failures:**
  - OAuth token expiration / revoked consent.
  - API rate limits.
  - Empty result set: with Always Output Data, downstream IF must correctly handle missing `id`.

#### Node: Customer Exists?
- **Type / role:** `If` — branches based on whether a contact id exists.
- **Condition:**
  - Checks `{{$json.id}}` **is not empty**.
  - This assumes the incoming item from “Get Customer Record” has a top-level `id`.
- **Outputs:**
  - **True →** Update Customer Record
  - **False →** Create Customer Record
- **Edge cases:**
  - If Zoho node returns data under a different structure (e.g., `data[0].id`), this condition will always fail.
  - If “Get Customer Record” returns multiple items, only the current item is tested; limit=1 helps, but filtering remains crucial.

#### Node: Create Customer Record
- **Type / role:** `Zoho CRM` — creates a Contact when none exists.
- **Key configuration:**
  - **Resource:** Contact
  - **Operation:** Create (implied by presence of create fields)
  - **Last Name:** from ticket owner: `$('Webhook Trigger').item.json.body.CaseOwner`
  - **Email:** `$('Webhook Trigger').item.json.body.Email`
- **Inputs/outputs:**
  - **Input ←** Customer Exists? (false branch)
  - **Output →** Get Ticket History
- **Edge cases:**
  - Zoho CRM may require `LastName` and enforce uniqueness rules on email; duplicates can fail.
  - If CaseOwner is empty, record creation may fail due to required LastName.
  - Email format invalid → API validation error.

#### Node: Update Customer Record
- **Type / role:** `Zoho CRM` — updates a custom field on an existing contact.
- **Key configuration:**
  - **Resource:** Contact
  - **Operation:** Update
  - **Contact ID:** `{{$json.id}}` (from Get Customer Record item)
  - **Custom Field update:** sets “Last Ticket Date Field ID” to `{{$now.toISO()}}`
  - The “Last Ticket Date Field ID” is a placeholder and must be replaced with a real Zoho CRM field id.
- **Inputs/outputs:**
  - **Input ←** Customer Exists? (true branch)
  - **Output →** Get Ticket History
- **Edge cases / failures:**
  - Wrong field id → Zoho API error.
  - Insufficient permission to edit that field/module.
  - `$now.toISO()` produces an ISO timestamp; the target field type must accept date-time (or Zoho may reject/convert).

#### Node: Get Ticket History
- **Type / role:** `Zoho CRM` — fetches the contact record to provide historical context.
- **Key configuration:**
  - **Resource:** Contact
  - **Operation:** Get
  - **Contact ID:** `{{ $('Get Customer Record').item.json.id }}`
- **Inputs/outputs:**
  - **Input ←** Create Customer Record OR Update Customer Record
  - **Output →** Analyze Ticket Patterns
- **Important caveat (logic gap):**
  - On the “create” path, the contact id should come from “Create Customer Record”, but the node always references **Get Customer Record**’s id. If no contact existed, this id may be empty or unrelated.
- **Edge cases / failures:**
  - If the referenced ID is missing/invalid → Zoho “record not found”.
  - If multiple parallel items exist, fixed `$('Get Customer Record').item` references can mismatch the current branch item.

**Sticky note coverage (applies to nodes in this block):**  
- “## CRM Data Synchronization and History Retrieval …”  
- “# How It Works …”

---

### Block 3 — AI-Driven Ticket Analysis and Pattern Recognition
**Overview:** Uses an OpenAI-powered Agent to analyze the current ticket and the customer’s CRM record, returning a strictly-typed JSON object that downstream logic can reliably evaluate.  
**Nodes involved:** OpenAI Chat Model, Analyze Ticket Patterns, Structured Output Parser

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — provides the LLM used by the agent.
- **Key configuration:**
  - **Model:** `gpt-4.1-mini`
  - **Credentials:** OpenAI API (“OpenAi account 32”)
- **Connections:**
  - **Output (ai_languageModel) →** Analyze Ticket Patterns (as its language model provider)
- **Edge cases / failures:**
  - Invalid API key / quota exceeded.
  - Model name not available in the account/region.
  - Latency/timeouts on large prompts.

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces a JSON schema for the agent’s output.
- **Schema (manual):**
  - `upsellScore` (number 0–10)
  - `hasOpportunity` (boolean)
  - `patterns` (string)
  - `recommendations` (string)
  - `reasoning` (string)
- **Connections:**
  - **Output (ai_outputParser) →** Analyze Ticket Patterns (as its output parser)
- **Edge cases / failures:**
  - If the model returns non-JSON or violates schema → parsing error.
  - “upsellScore” returned as string could fail strict typing in downstream IF.

#### Node: Analyze Ticket Patterns
- **Type / role:** `LangChain Agent` — orchestrates prompt + model + parser to produce structured analysis.
- **Prompt construction:**
  - **Text input** includes:
    - “Current Ticket” section built from webhook body fields (CaseId, CaseNumber, Subject, Description, Priority, Status, Email)
    - “Customer History:” includes `{{$json}}` (whatever came from Get Ticket History)
  - **System message** instructs the agent to:
    - identify patterns, sentiment, engagement
    - determine upsell opportunities
    - compute `upsellScore` 0–10
    - provide product recommendations
    - return structured JSON
- **Connections:**
  - **Main input ←** Get Ticket History
  - **Main output →** Upsell Opportunity?
  - **AI language model input ←** OpenAI Chat Model
  - **AI output parser input ←** Structured Output Parser
- **Output shape expectation:**
  - Downstream node expects `{{$json.output.hasOpportunity}}` and `{{$json.output.upsellScore}}` (i.e., agent output wrapped under `output`).
- **Edge cases / failures:**
  - If Get Ticket History returns very large objects, prompt can grow and increase costs / token limit risk.
  - If webhook fields contain null/undefined, prompt will include blanks.
  - Schema mismatch can halt execution.

**Sticky note coverage (applies to nodes in this block):**  
- “## AI-Driven Ticket Analysis and Pattern Recognition …”  
- “# How It Works …”

---

### Block 4 — Opportunity Filtering and Stakeholder Notification
**Overview:** Applies business rules to the AI output and sends a detailed alert email when criteria are met.  
**Nodes involved:** Upsell Opportunity?, Alert Account Manager

#### Node: Upsell Opportunity?
- **Type / role:** `If` — decision gate for notifications.
- **Conditions (AND):**
  1. `{{$json.output.hasOpportunity}}` is **true**
  2. `{{$json.output.upsellScore}}` **>=** `{{ $('Workflow Configuration').item.json.upsellThreshold }}`
- **Connections:**
  - **Input ←** Analyze Ticket Patterns
  - **True output →** Alert Account Manager
  - **False output →** (no node connected; workflow ends silently)
- **Edge cases / failures:**
  - If agent output is not under `output`, conditions will fail.
  - If `upsellScore` is missing or not numeric, strict validation can cause false negatives.

#### Node: Alert Account Manager
- **Type / role:** `Gmail` — sends HTML email to account manager.
- **Key configuration:**
  - **To:** `{{ $('Workflow Configuration').item.json.accountManagerEmail }}`
  - **Subject:** `Upsell Opportunity: <CaseOwner> - Case #<CaseNumber>`
  - **HTML message:** includes customer/ticket details and AI fields:
    - Uses `{{ $json.upsellScore }}`, `{{ $json.patterns }}`, etc.
- **Important caveat (data mismatch risk):**
  - Previous IF checks `output.*`, but this email node references `$json.upsellScore`, `$json.patterns`, etc.  
  - If the agent result is actually under `$json.output`, the email will render blanks unless data is mapped/flattened.
- **Credentials:** Gmail OAuth2 (“Gmail account 8”)
- **Edge cases / failures:**
  - Gmail OAuth token expired / missing scopes.
  - Sending limits / blocked by Google policies.
  - HTML content may need sanitization depending on downstream mail clients.

**Sticky note coverage (applies to nodes in this block):**  
- “## Opportunity Filtering and Stakeholder Notification …”  
- “# How It Works …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | n8n-nodes-base.webhook | Entry point: receive support ticket POST payload | — | Workflow Configuration | ## Webhook Trigger on On Ticket Add/Edit & Set Configuration … |
| Webhook Trigger | n8n-nodes-base.webhook | Entry point: receive support ticket POST payload | — | Workflow Configuration | # How It Works … |
| Workflow Configuration | n8n-nodes-base.set | Define global params (alert email, threshold) | Webhook Trigger | Get Customer Record | ## Webhook Trigger on On Ticket Add/Edit & Set Configuration … |
| Workflow Configuration | n8n-nodes-base.set | Define global params (alert email, threshold) | Webhook Trigger | Get Customer Record | # How It Works … |
| Get Customer Record | n8n-nodes-base.zohoCrm | Fetch a contact (currently unfiltered) | Workflow Configuration | Customer Exists? | ## CRM Data Synchronization and History Retrieval … |
| Get Customer Record | n8n-nodes-base.zohoCrm | Fetch a contact (currently unfiltered) | Workflow Configuration | Customer Exists? | # How It Works … |
| Customer Exists? | n8n-nodes-base.if | Branch: contact found vs not found | Get Customer Record | Update Customer Record; Create Customer Record | ## CRM Data Synchronization and History Retrieval … |
| Customer Exists? | n8n-nodes-base.if | Branch: contact found vs not found | Get Customer Record | Update Customer Record; Create Customer Record | # How It Works … |
| Create Customer Record | n8n-nodes-base.zohoCrm | Create new Zoho CRM Contact | Customer Exists? (false) | Get Ticket History | ## CRM Data Synchronization and History Retrieval … |
| Create Customer Record | n8n-nodes-base.zohoCrm | Create new Zoho CRM Contact | Customer Exists? (false) | Get Ticket History | # How It Works … |
| Update Customer Record | n8n-nodes-base.zohoCrm | Update “Last Ticket Date” field on contact | Customer Exists? (true) | Get Ticket History | ## CRM Data Synchronization and History Retrieval … |
| Update Customer Record | n8n-nodes-base.zohoCrm | Update “Last Ticket Date” field on contact | Customer Exists? (true) | Get Ticket History | # How It Works … |
| Get Ticket History | n8n-nodes-base.zohoCrm | Retrieve contact record for analysis context | Create Customer Record; Update Customer Record | Analyze Ticket Patterns | ## CRM Data Synchronization and History Retrieval … |
| Get Ticket History | n8n-nodes-base.zohoCrm | Retrieve contact record for analysis context | Create Customer Record; Update Customer Record | Analyze Ticket Patterns | # How It Works … |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for agent | — | Analyze Ticket Patterns (AI model input) | ## AI-Driven Ticket Analysis and Pattern Recognition … |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for agent | — | Analyze Ticket Patterns (AI model input) | # How It Works … |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for agent output | — | Analyze Ticket Patterns (AI parser input) | ## AI-Driven Ticket Analysis and Pattern Recognition … |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for agent output | — | Analyze Ticket Patterns (AI parser input) | # How It Works … |
| Analyze Ticket Patterns | @n8n/n8n-nodes-langchain.agent | Analyze ticket + history; produce structured insights | Get Ticket History | Upsell Opportunity? | ## AI-Driven Ticket Analysis and Pattern Recognition … |
| Analyze Ticket Patterns | @n8n/n8n-nodes-langchain.agent | Analyze ticket + history; produce structured insights | Get Ticket History | Upsell Opportunity? | # How It Works … |
| Upsell Opportunity? | n8n-nodes-base.if | Gate notifications based on AI output + threshold | Analyze Ticket Patterns | Alert Account Manager (true) | ## Opportunity Filtering and Stakeholder Notification … |
| Upsell Opportunity? | n8n-nodes-base.if | Gate notifications based on AI output + threshold | Analyze Ticket Patterns | Alert Account Manager (true) | # How It Works … |
| Alert Account Manager | n8n-nodes-base.gmail | Send detailed upsell alert email | Upsell Opportunity? (true) | — | ## Opportunity Filtering and Stakeholder Notification … |
| Alert Account Manager | n8n-nodes-base.gmail | Send detailed upsell alert email | Upsell Opportunity? (true) | — | # How It Works … |
| Sticky Note10 | n8n-nodes-base.stickyNote | Documentation / on-canvas notes | — | — | # How It Works … |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation / on-canvas notes | — | — | ## Webhook Trigger on On Ticket Add/Edit & Set Configuration … |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / on-canvas notes | — | — | ## CRM Data Synchronization and History Retrieval … |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / on-canvas notes | — | — | ## AI-Driven Ticket Analysis and Pattern Recognition … |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / on-canvas notes | — | — | ## Opportunity Filtering and Stakeholder Notification … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Webhook Trigger (Webhook)**
   - Method: **POST**
   - Path: **support-ticket**
   - Use the generated Production URL in your help desk/Zoho webhook configuration.
3. **Add node: Workflow Configuration (Set)**
   - Add fields:
     - `accountManagerEmail` (String) = the recipient email for alerts
     - `upsellThreshold` (Number) = minimum score (e.g., 3)
   - Enable **Include Other Fields**.
   - Connect: **Webhook Trigger → Workflow Configuration**
4. **Add node: Get Customer Record (Zoho CRM)**
   - Credentials: connect **Zoho OAuth2** (Zoho CRM)
   - Resource: **Contact**
   - Operation: **Get All**
   - Limit: **1**
   - (Recommended when rebuilding) Add a filter by Email matching the webhook email; otherwise the logic is unreliable.
   - Connect: **Workflow Configuration → Get Customer Record**
5. **Add node: Customer Exists? (IF)**
   - Condition: String → `{{$json.id}}` **is not empty**
   - Connect: **Get Customer Record → Customer Exists?**
6. **Add node: Update Customer Record (Zoho CRM)**
   - Resource: Contact; Operation: **Update**
   - Contact ID: `{{$json.id}}`
   - Update a custom field “Last Ticket Date” using:
     - Value: `{{$now.toISO()}}`
     - Replace placeholder **“Last Ticket Date Field ID”** with your real Zoho field id.
   - Connect: **Customer Exists? (true) → Update Customer Record**
7. **Add node: Create Customer Record (Zoho CRM)**
   - Resource: Contact; Operation: **Create**
   - Last Name: `{{ $('Webhook Trigger').item.json.body.CaseOwner }}`
   - Email: `{{ $('Webhook Trigger').item.json.body.Email }}`
   - Connect: **Customer Exists? (false) → Create Customer Record**
8. **Add node: Get Ticket History (Zoho CRM)**
   - Resource: Contact; Operation: **Get**
   - Contact ID: (as implemented) `{{ $('Get Customer Record').item.json.id }}`
   - Connect: **Update Customer Record → Get Ticket History**
   - Connect: **Create Customer Record → Get Ticket History**
   - (Recommended when rebuilding) Use the ID from whichever branch executed (updated contact id or created contact id) to avoid mismatches.
9. **Add node: OpenAI Chat Model (LangChain Chat Model OpenAI)**
   - Credentials: OpenAI API key
   - Model: `gpt-4.1-mini`
10. **Add node: Structured Output Parser (Structured Output Parser)**
   - Schema type: **Manual**
   - Define properties: `upsellScore` (number), `hasOpportunity` (boolean), `patterns` (string), `recommendations` (string), `reasoning` (string).
11. **Add node: Analyze Ticket Patterns (LangChain Agent)**
   - Prompt type: **Define**
   - System message: sales intelligence analyst instructions (as in workflow).
   - Text prompt: include:
     - ticket details from `$('Webhook Trigger').item.json.body.*`
     - “Customer History” from `{{$json}}` (input from Get Ticket History)
   - Attach integrations:
     - Connect **OpenAI Chat Model → Analyze Ticket Patterns** via **AI Language Model** connection.
     - Connect **Structured Output Parser → Analyze Ticket Patterns** via **AI Output Parser** connection.
   - Connect: **Get Ticket History → Analyze Ticket Patterns**
12. **Add node: Upsell Opportunity? (IF)**
   - Condition 1 (boolean): `{{$json.output.hasOpportunity}}` is true
   - Condition 2 (number): `{{$json.output.upsellScore}}` >= `{{ $('Workflow Configuration').item.json.upsellThreshold }}`
   - Connect: **Analyze Ticket Patterns → Upsell Opportunity?**
13. **Add node: Alert Account Manager (Gmail)**
   - Credentials: Gmail OAuth2
   - To: `{{ $('Workflow Configuration').item.json.accountManagerEmail }}`
   - Subject: `Upsell Opportunity: {{ CaseOwner }} - Case #{{ CaseNumber }}`
   - Message: HTML body including ticket details and AI analysis fields.
   - Connect: **Upsell Opportunity? (true) → Alert Account Manager**
14. **Activate** the workflow after validating webhook payload structure and credential permissions.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Point your support platform’s POST requests to the n8n Webhook URL.” | Sticky note “How It Works” (Trigger) |
| “Link credentials for Zoho CRM (data), OpenAI (AI) and Gmail (alerts).” | Sticky note “How It Works” (Connections) |
| “Set your alert email and upsell score threshold and replace the Zoho CRM placeholder with your specific ‘Last Ticket’ Field ID.” | Sticky note “How It Works” (Configuration) |
| “The system filters results and sends a detailed email only when the AI detects an opportunity meeting your criteria.” | Sticky note “How It Works” (Notifications) |

