Triage Slack and Gmail requests with an AI-powered intake layer

https://n8nworkflows.xyz/workflows/triage-slack-and-gmail-requests-with-an-ai-powered-intake-layer-13180


# Triage Slack and Gmail requests with an AI-powered intake layer

## 1. Workflow Overview

**Purpose:**  
This workflow continuously collects inbound requests from **Slack** and **Gmail**, normalizes each item into a consistent JSON shape, **deduplicates** previously-seen items, then uses an **AI triage agent** to classify and prioritize the request. It creates a **Notion ticket** for tracking and optionally sends a **Slack alert** for very high-importance items.

**Primary use cases:**
- HR/Operations intake for mixed channels (Slack + email) with consistent triage outputs
- A “plug-and-play” intake layer where new sources can be added by connecting a trigger into the normalization block
- Automated ticket creation with AI-generated summary, priority signals, and response draft

### 1.1 Collect Items (Slack + Gmail)
Two entry points:
- A schedule poll that fetches recent Slack messages from a chosen channel
- A Gmail trigger that watches inbound email messages

### 1.2 Normalize + Dedupe
- Normalizes source-specific payloads into a shared schema using an LLM-backed extractor
- Uses an n8n **Data Table** to skip items already processed (dedupe)

### 1.3 AI Triage + Outputs
- AI agent produces structured triage JSON (category, urgency/importance/effort, due date, reply draft, etc.)
- Creates a Notion database page for tracking
- Sends Slack notification only when importance is above a threshold

---

## 2. Block-by-Block Analysis

### Block 1 — Collect Items (Slack + Gmail)

**Overview:**  
This block ingests new items from Slack and Gmail. Slack is polled on a schedule; Gmail is watched via trigger polling.

**Nodes involved:**
- **Run every minute**
- **Fetch recent Slack messages**
- **Watch inbound Gmail messages**

#### Node: Run every minute
- **Type / role:** Schedule Trigger (polling entry point)
- **Configuration (interpreted):**
  - Runs every minute.
- **Inputs / outputs:**
  - **Output →** Fetch recent Slack messages
- **Edge cases / failures:**
  - n8n instance sleep/paused workflows stop polling.
  - High frequency can increase API calls/cost; may hit Slack rate limits if scaled.

#### Node: Fetch recent Slack messages
- **Type / role:** Slack node (read messages from a channel)
- **Configuration (interpreted):**
  - Resource: **channel**
  - Operation: **get** (fetch messages)
  - **Channel ID is empty in the workflow JSON** → must be set before enabling.
  - Auth: **OAuth2**
- **Inputs / outputs:**
  - **Input ←** Run every minute
  - **Output →** Universal intake normalizer
- **Edge cases / failures:**
  - OAuth scope issues (missing `channels:history` / `groups:history` depending on channel type).
  - Rate limiting or pagination behavior (may not fetch everything expected).
  - If it returns multiple messages, downstream nodes will process each item individually.

#### Node: Watch inbound Gmail messages
- **Type / role:** Gmail Trigger (email intake entry point)
- **Configuration (interpreted):**
  - Trigger mode: polling every minute
  - Sender filter is empty → watches all senders unless configured.
  - “Simple” disabled, so it likely returns richer email fields.
- **Inputs / outputs:**
  - No input (trigger)
  - **Output →** Universal intake normalizer
- **Edge cases / failures:**
  - OAuth issues / revoked consent.
  - Gmail API quota limits.
  - Depending on Gmail trigger behavior, may re-deliver messages under some conditions; dedupe block is intended to protect against that.

---

### Block 2 — Normalize and Dedupe

**Overview:**  
All inbound items (Slack messages and Gmail emails) are transformed into a single consistent JSON schema (“universal intake”). A Data Table is then used to skip items that have already been processed.

**Nodes involved:**
- **LLM for message normalization**
- **Universal intake normalizer**
- **Skip already processed items**
- **Store processed IDs**

#### Node: LLM for message normalization
- **Type / role:** LangChain Chat Model (OpenAI) used as the language model for extraction
- **Configuration (interpreted):**
  - Model: **gpt-4.1-mini**
  - No special options configured
- **Inputs / outputs:**
  - Connected as **ai_languageModel → Universal intake normalizer**
- **Version notes:**
  - Node type: `@n8n/n8n-nodes-langchain.lmChatOpenAi` (typeVersion 1.3)
  - Requires OpenAI credentials configured in n8n for LangChain/OpenAI chat model nodes.
- **Edge cases / failures:**
  - Model access not available in the connected OpenAI account.
  - Content policies / sensitive data concerns (workflow itself warns about sending content to LLMs).
  - Latency/timeouts if large Slack/email payloads are sent verbatim.

#### Node: Universal intake normalizer
- **Type / role:** LangChain Information Extractor (schema-based normalization)
- **Configuration (interpreted):**
  - Input text: `JSON.stringify($json)` (it passes the entire incoming item JSON as text to the extractor)
  - Schema mode: “fromJson” using the provided example schema:
    - `source`, `source_id`, `timestamp`, `author`, `context`, `text`, `links`, `attachments`
  - Uses **LLM for message normalization** as its language model input.
- **Inputs / outputs:**
  - **Inputs ←** Fetch recent Slack messages; Watch inbound Gmail messages
  - **AI model input ←** LLM for message normalization
  - **Main output →** Skip already processed items
- **Key expressions / variables:**
  - `{{ JSON.stringify($json) }}` (normalization prompt input)
- **Edge cases / failures:**
  - If Slack/Gmail payloads are large, costs and latency increase; may exceed model limits.
  - The extractor may produce missing/empty `source_id` or `source`, which will break dedupe logic (see next node).
  - If it outputs data under `output` (typical for these nodes), downstream expressions assume `item.json.output.*` exists.

#### Node: Skip already processed items
- **Type / role:** Data Table node (dedupe gate)
- **Configuration (interpreted):**
  - Operation: **rowNotExists**
  - Checks a column named `uniqueID` with value:
    - `{{ $json.output.source_id + $json.output.source }}`
  - Data Table ID is empty in JSON → must be selected/created.
- **Inputs / outputs:**
  - **Input ←** Universal intake normalizer
  - **Output (only if row does not exist) →** Store processed IDs
- **Key expressions / variables:**
  - Dedupe key concatenation: `source_id + source`
- **Edge cases / failures:**
  - If `source` or `source_id` is undefined/null, the dedupe key may be invalid (or become `"undefinedundefined"`), causing collisions or reprocessing.
  - If Data Table not created or column mismatch, node fails.
  - Concurrency: if multiple identical items arrive nearly simultaneously, both could pass “not exists” before insert occurs (race condition).

#### Node: Store processed IDs
- **Type / role:** Data Table node (write dedupe marker)
- **Configuration (interpreted):**
  - Inserts/updates a row with column `uniqueID` set to:
    - `{{ $json.output.source + $json.output.source_id }}`
  - Matching column: `uniqueID` (acts like a unique key)
  - Data Table ID is empty in JSON → must be selected/created.
- **Inputs / outputs:**
  - **Input ←** Skip already processed items
  - **Output →** Triage and prioritize request
- **Key expressions / variables:**
  - Note: the concatenation order here is `source + source_id`, while the lookup node uses `source_id + source`. For pure string concatenation, both are consistent only if treated as a set; **but they are different strings** and can break dedupe.
- **Important integration issue (dedupe bug):**
  - `Skip already processed items` uses: `source_id + source`
  - `Store processed IDs` writes: `source + source_id`
  - These should be identical to work reliably. As-is, the table will never match previously stored keys (unless by coincidence).
- **Edge cases / failures:**
  - Data Table misconfiguration (missing schema/column).
  - Same race condition risk as above.

---

### Block 3 — AI Triage and Structured Output

**Overview:**  
This block uses an AI agent to classify each normalized request into a structured JSON output. A structured output parser enforces a schema and can auto-fix near-valid JSON.

**Nodes involved:**
- **LLM for triage agent**
- **Triage and prioritize request**
- **LLM for structured parsing**
- **Parse triage JSON**

#### Node: LLM for triage agent
- **Type / role:** LangChain Chat Model (OpenAI) for the triage agent
- **Configuration (interpreted):**
  - Model: **gpt-5.2**
- **Inputs / outputs:**
  - Connected as **ai_languageModel → Triage and prioritize request**
- **Edge cases / failures:**
  - Model availability/access depending on OpenAI account.
  - Higher cost vs smaller models; consider throttling or truncating payload.

#### Node: Triage and prioritize request
- **Type / role:** LangChain Agent node (prompted classifier + prioritizer)
- **Configuration (interpreted):**
  - Prompt includes the normalized item:
    - `Input item JSON: {{ JSON.stringify($('Universal intake normalizer').item.json.output) }}`
  - Asks for fields:
    - `summary`, `category`, `eisenhower_quadrant`, `urgency`, `importance`, `effort`, `risk`, `confidence`, `should_create_ticket`, `suggested_assignee_role`, `suggested_due_date`, `draft_reply`
  - Includes “today is {{ $now }}” to anchor date reasoning.
  - System message: HR operations triage assistant; return **structured JSON only**; don’t invent facts; if uncertain, lower confidence.
  - Has an output parser enabled.
- **Inputs / outputs:**
  - **Main input ←** Store processed IDs
  - **AI model input ←** LLM for triage agent
  - **AI output parser input ←** Parse triage JSON
  - **Main outputs →**
    - Only high importance items
    - Create Notion ticket
- **Key expressions / variables:**
  - `$('Universal intake normalizer').item.json.output` (cross-node reference)
  - `$now` (execution time)
- **Edge cases / failures:**
  - If the normalizer output is missing, the agent prompt may be empty or invalid.
  - If the agent returns non-JSON or partial JSON, parser must fix; otherwise execution fails.
  - Date formatting: due date must be ISO date (YYYY-MM-DD). Model may output natural language without strict enforcement; parser helps but not guaranteed.

#### Node: LLM for structured parsing
- **Type / role:** LangChain Chat Model (OpenAI) used by the structured parser for correction/auto-fix
- **Configuration (interpreted):**
  - Model: **gpt-5.2**
- **Inputs / outputs:**
  - Connected as **ai_languageModel → Parse triage JSON**
- **Edge cases / failures:**
  - Same model access/quota considerations as above.

#### Node: Parse triage JSON
- **Type / role:** Structured Output Parser (JSON schema validation + auto-fix)
- **Configuration (interpreted):**
  - Manual JSON schema with **additionalProperties: false**
  - Required fields include:
    - `ticket_title`, `summary`, `category`, `eisenhower_quadrant`, `urgency`, `importance`, `effort`, `risk`, `confidence`, `should_create_ticket`, `suggested_assignee_role`, `suggested_due_date`, `draft_reply`
  - `autoFix: true` to attempt correcting invalid JSON outputs using the connected parsing model.
- **Inputs / outputs:**
  - **AI language model input ←** LLM for structured parsing
  - Connected into the agent as its output parser (agent feeds its raw output into this parser)
- **Edge cases / failures:**
  - If the agent output is too malformed, auto-fix may fail and the workflow errors.
  - Strict enums can fail classification if the model outputs a value not in the allowed set (e.g., “IT Support” instead of “IT”).
  - `suggested_due_date` requires `format: date` (not datetime).

---

### Block 4 — Create Tracking Ticket and Notify

**Overview:**  
Every triaged request becomes a Notion ticket. Only requests with extremely high importance trigger an additional Slack alert.

**Nodes involved:**
- **Create Notion ticket**
- **Only high importance items**
- **Notify in Slack**

#### Node: Create Notion ticket
- **Type / role:** Notion node (create database page)
- **Configuration (interpreted):**
  - Resource: databasePage
  - Database ID: placeholder `REPLACE_WITH_NOTION_DATABASE_ID`
  - Maps triage output into Notion properties (must match your database property names exactly), including:
    - Name (title) ← `ticket_title`
    - Description ← `summary`
    - Assignee Role ← `suggested_assignee_role`
    - confidence_score, urgency, importance, effort, risk ← numeric fields
    - Priority (select) ← `eisenhower_quadrant`
    - Due Date (date) ← `suggested_due_date`
    - Helpful other things ← `draft_reply`
    - Status (status) hard-coded to `Backlog`
- **Inputs / outputs:**
  - **Input ←** Triage and prioritize request
  - Output not connected further
- **Edge cases / failures:**
  - Property name mismatches are the most common failure (Notion returns validation errors).
  - `Priority|select` expects the select option to already exist (e.g., “DoNow”, “Plan”, “Delegate”, “Defer”).
  - `Due Date|date` expects a valid date string; invalid values cause failure.

#### Node: Only high importance items
- **Type / role:** Filter node (conditional routing)
- **Configuration (interpreted):**
  - شرط: `{{ $json.output.importance }} > 9`
  - Strict type validation enabled.
- **Inputs / outputs:**
  - **Input ←** Triage and prioritize request
  - **If matched →** Notify in Slack
- **Edge cases / failures:**
  - If `importance` is missing or not numeric, strict validation can cause the condition to fail or error (depending on n8n behavior/version).
  - Threshold is very high (only 10 passes if integer 0–10).

#### Node: Notify in Slack
- **Type / role:** Slack node (send a direct message to a user)
- **Configuration (interpreted):**
  - Sends message text: `{{ $('Triage and prioritize request').item.json.output.summary }}`
  - Target: a specific Slack **user** (ID placeholder: `REPLACE_WITH_SLACK_USER_ID`)
  - Auth: OAuth2
- **Inputs / outputs:**
  - **Input ←** Only high importance items
  - Output not connected further
- **Edge cases / failures:**
  - Missing Slack OAuth scopes (e.g., `chat:write`, `im:write` depending on how n8n sends to users).
  - Wrong user ID or app not allowed to DM the user.
  - If you want channel alerts instead, change to channel message operation.

---

### Block 5 — Sticky Notes (Documentation in-canvas)

**Overview:**  
These are non-executing notes that document setup, safety, and how to extend the workflow.

**Nodes involved:**
- **Main sticky note**
- **Group sticky note: Collect items**
- **Group sticky note: Extract content**
- **Group sticky note: Analyze and send**

(Details captured in the Summary Table’s “Sticky Note” column.)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run every minute | Schedule Trigger | Polling entry point for Slack intake | — | Fetch recent Slack messages | ### Collect items \* Schedule trigger polls a Slack channel. \* Gmail trigger watches for new messages. \* Configure the Slack channel and Gmail sender filter before enabling. |
| Fetch recent Slack messages | Slack | Fetch messages from a Slack channel | Run every minute | Universal intake normalizer | ### Collect items \* Schedule trigger polls a Slack channel. \* Gmail trigger watches for new messages. \* Configure the Slack channel and Gmail sender filter before enabling. |
| Watch inbound Gmail messages | Gmail Trigger | Poll Gmail for inbound messages | — | Universal intake normalizer | ### Collect items \* Schedule trigger polls a Slack channel. \* Gmail trigger watches for new messages. \* Configure the Slack channel and Gmail sender filter before enabling. |
| LLM for message normalization | LangChain OpenAI Chat Model | LLM backing the normalization extractor | — | Universal intake normalizer (AI model input) | ### Normalize and dedupe \* Normalizes sources into one schema. \* Uses a data table to skip repeats. \* Ensure the data table exists and has a uniqueID column. \* Makes it possible to add a new source as plug-and-play |
| Universal intake normalizer | LangChain Information Extractor | Normalize Slack/Gmail payloads into a universal schema | Fetch recent Slack messages; Watch inbound Gmail messages | Skip already processed items | ### Normalize and dedupe \* Normalizes sources into one schema. \* Uses a data table to skip repeats. \* Ensure the data table exists and has a uniqueID column. \* Makes it possible to add a new source as plug-and-play |
| Skip already processed items | Data Table | Dedupe gate (only allow unseen items) | Universal intake normalizer | Store processed IDs | ### Normalize and dedupe \* Normalizes sources into one schema. \* Uses a data table to skip repeats. \* Ensure the data table exists and has a uniqueID column. \* Makes it possible to add a new source as plug-and-play |
| Store processed IDs | Data Table | Persist dedupe marker (uniqueID) | Skip already processed items | Triage and prioritize request | ### Normalize and dedupe \* Normalizes sources into one schema. \* Uses a data table to skip repeats. \* Ensure the data table exists and has a uniqueID column. \* Makes it possible to add a new source as plug-and-play |
| LLM for triage agent | LangChain OpenAI Chat Model | LLM backing the triage agent | — | Triage and prioritize request (AI model input) | ### Analyze and send \* AI agent outputs structured fields. \* Notion ticket is created for tracking. \* Slack notification is sent only when the filter matches. \* Adjust the importance threshold as needed. |
| Triage and prioritize request | LangChain Agent | Classify, score, and draft reply; outputs structured JSON | Store processed IDs | Only high importance items; Create Notion ticket | ### Analyze and send \* AI agent outputs structured fields. \* Notion ticket is created for tracking. \* Slack notification is sent only when the filter matches. \* Adjust the importance threshold as needed. |
| LLM for structured parsing | LangChain OpenAI Chat Model | LLM used to auto-fix/validate structured JSON | — | Parse triage JSON (AI model input) | ### Analyze and send \* AI agent outputs structured fields. \* Notion ticket is created for tracking. \* Slack notification is sent only when the filter matches. \* Adjust the importance threshold as needed. |
| Parse triage JSON | Structured Output Parser | Enforce JSON schema; auto-fix malformed model output | (from agent output parser connection) | (feeds back into agent output parsing) | ### Analyze and send \* AI agent outputs structured fields. \* Notion ticket is created for tracking. \* Slack notification is sent only when the filter matches. \* Adjust the importance threshold as needed. |
| Only high importance items | Filter | Route only items with importance > 9 | Triage and prioritize request | Notify in Slack | ### Analyze and send \* AI agent outputs structured fields. \* Notion ticket is created for tracking. \* Slack notification is sent only when the filter matches. \* Adjust the importance threshold as needed. |
| Notify in Slack | Slack | Send alert DM to a Slack user | Only high importance items | — | ### Analyze and send \* AI agent outputs structured fields. \* Notion ticket is created for tracking. \* Slack notification is sent only when the filter matches. \* Adjust the importance threshold as needed. |
| Create Notion ticket | Notion | Create a Notion database page for the request | Triage and prioritize request | — | ### Analyze and send \* AI agent outputs structured fields. \* Notion ticket is created for tracking. \* Slack notification is sent only when the filter matches. \* Adjust the importance threshold as needed. |
| Main sticky note | Sticky Note | Canvas documentation (non-executing) | — | — | ## How it works \* This workflow collects inbound requests from Slack and Gmail. \* Each item is normalized into a consistent JSON structure. \* A data table is used to dedupe items so the same request is not processed twice. \* An AI agent classifies the request, scores urgency and importance, and suggests a reply. \* A Notion ticket is created for tracking, and very important items can trigger a Slack alert. / Safety note: This workflow sends content to LLMs. Keep third party terms in mind and avoid processing restricted or sensitive data. / ## Setup steps \* Connect Slack, Gmail, Notion, and OpenAI credentials in n8n. \* Select a Slack channel to read from and a Slack user to notify. \* Point the Notion node to your own database and confirm the property names match. \* Create or select a data table and map the uniqueID column. \* Tune the filter threshold and consider truncating message text to control cost. / ## Extending \* Just connect a trigger from any source at the beginning and connect to normalize node |
| Group sticky note: Collect items | Sticky Note | Canvas documentation for intake block (non-executing) | — | — | ### Collect items \* Schedule trigger polls a Slack channel. \* Gmail trigger watches for new messages. \* Configure the Slack channel and Gmail sender filter before enabling. |
| Group sticky note: Extract content | Sticky Note | Canvas documentation for normalization/dedupe (non-executing) | — | — | ### Normalize and dedupe \* Normalizes sources into one schema. \* Uses a data table to skip repeats. \* Ensure the data table exists and has a uniqueID column. \* Makes it possible to add a new source as plug-and-play |
| Group sticky note: Analyze and send | Sticky Note | Canvas documentation for triage/output (non-executing) | — | — | ### Analyze and send \* AI agent outputs structured fields. \* Notion ticket is created for tracking. \* Slack notification is sent only when the filter matches. \* Adjust the importance threshold as needed. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials in n8n**
   1) **Slack OAuth2** credential (ensure scopes allow reading channel history and sending messages/DMs).  
   2) **Gmail OAuth2** credential for the trigger.  
   3) **Notion** credential with access to your target database.  
   4) **OpenAI** credential usable by n8n LangChain OpenAI Chat Model nodes.

2. **Add intake triggers**
   1) Add **Schedule Trigger** named **“Run every minute”**  
      - Interval: every **1 minute**
   2) Add **Slack** node named **“Fetch recent Slack messages”**  
      - Resource: **Channel**  
      - Operation: **Get** (messages)  
      - Authentication: **OAuth2**  
      - Set **Channel ID** to the channel you want to poll  
      - Connect: **Run every minute → Fetch recent Slack messages**
   3) Add **Gmail Trigger** named **“Watch inbound Gmail messages”**  
      - Poll: **every minute**  
      - Optional: set **Sender filter** if you only want specific senders  
      - This node is a second entry point (no incoming connection).

3. **Add the normalization LLM**
   1) Add **LangChain OpenAI Chat Model** named **“LLM for message normalization”**  
      - Model: **gpt-4.1-mini**

4. **Add the universal normalization node**
   1) Add **LangChain Information Extractor** named **“Universal intake normalizer”**  
      - Text input: `{{ JSON.stringify($json) }}`  
      - Schema: use a JSON example defining:
        - `source`, `source_id`, `timestamp`, `author{name,handle,email}`, `context{channel,thread_ts,email_subject}`, `text`, `links[]`, `attachments[]`
   2) Connect:
      - **Fetch recent Slack messages → Universal intake normalizer**
      - **Watch inbound Gmail messages → Universal intake normalizer**
      - **LLM for message normalization (AI) → Universal intake normalizer**

5. **Create a Data Table for dedupe**
   1) In n8n, create/select a **Data Table** containing a column:
      - `uniqueID` (type: string)

6. **Add dedupe gate**
   1) Add **Data Table** node named **“Skip already processed items”**  
      - Operation: **Row does not exist**  
      - Data Table: select your dedupe table  
      - Filter condition:
        - Column: `uniqueID`
        - Value: choose **one** consistent expression, for example:  
          `{{ $json.output.source + ':' + $json.output.source_id }}`
   2) Add **Data Table** node named **“Store processed IDs”**  
      - Operation: insert/upsert row (as supported by the Data Table node UI)  
      - Data Table: same table  
      - Map `uniqueID` to the **exact same expression** you used above.
   3) Connect:
      - **Universal intake normalizer → Skip already processed items**
      - **Skip already processed items → Store processed IDs**

7. **Add triage agent LLM**
   1) Add **LangChain OpenAI Chat Model** named **“LLM for triage agent”**  
      - Model: **gpt-5.2**

8. **Add structured output parser and its model**
   1) Add **LangChain OpenAI Chat Model** named **“LLM for structured parsing”**  
      - Model: **gpt-5.2**
   2) Add **Structured Output Parser** named **“Parse triage JSON”**  
      - Enable **Auto-fix**
      - Schema: create a strict schema with required fields:
        - `ticket_title` (string, max 90)
        - `summary` (string, max 140)
        - `category` enum: Payroll, Time off, Contract, Hiring, IT, Facilities, Finance, Other
        - `eisenhower_quadrant` enum: DoNow, Plan, Delegate, Defer
        - `urgency`, `importance`, `effort`, `risk` integers 0–10
        - `confidence` number 0–1
        - `should_create_ticket` boolean
        - `suggested_assignee_role` enum: HR Ops, Recruiter, People Lead, IT, Finance
        - `suggested_due_date` string format **date**
        - `draft_reply` string max 600
   3) Connect:
      - **LLM for structured parsing (AI) → Parse triage JSON**

9. **Add the triage agent**
   1) Add **LangChain Agent** named **“Triage and prioritize request”**
      - Prompt type: define
      - System message: HR operations triage assistant; JSON-only; don’t invent facts; lower confidence if unsure
      - User prompt includes:
        - `Input item JSON: {{ JSON.stringify($('Universal intake normalizer').item.json.output) }}`
        - Field requirements list
        - `today is {{ $now }}`
      - Enable output parser:
        - Connect the agent’s **output parser** to **Parse triage JSON**
      - Connect the agent’s **language model** to **LLM for triage agent**
   2) Connect:
      - **Store processed IDs → Triage and prioritize request**

10. **Create Notion ticket**
   1) Add **Notion** node named **“Create Notion ticket”**
      - Resource: **Database Page**
      - Database ID: choose your database
      - Map properties to match your database’s property names:
        - Title (Name) ← `{{ $json.output.ticket_title }}`
        - Description ← `{{ $json.output.summary }}`
        - Priority/select ← `{{ $json.output.eisenhower_quadrant }}`
        - Status ← “Backlog”
        - Due Date ← `{{ $json.output.suggested_due_date }}`
        - Numeric scores from `urgency/importance/effort/risk/confidence`
        - Draft reply ← `{{ $json.output.draft_reply }}`
   2) Connect:
      - **Triage and prioritize request → Create Notion ticket**

11. **Add high-importance Slack alert**
   1) Add **Filter** node named **“Only high importance items”**
      - Condition: number `{{ $json.output.importance }}` **> 9**
   2) Add **Slack** node named **“Notify in Slack”**
      - Send message to **User**
      - User: set your Slack user ID
      - Text: `{{ $('Triage and prioritize request').item.json.output.summary }}`
   3) Connect:
      - **Triage and prioritize request → Only high importance items → Notify in Slack**

12. **(Optional but recommended) Fix the dedupe key mismatch**
   - Ensure both Data Table nodes use the **same** `uniqueID` expression, ideally with a delimiter (e.g., `source + ':' + source_id`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow sends content to LLMs. Keep third party terms in mind and avoid processing restricted or sensitive data. | From canvas note (“Safety note”) |
| Connect Slack, Gmail, Notion, and OpenAI credentials in n8n. Configure Slack channel, Slack user, Notion database properties, and the Data Table `uniqueID` column before enabling. | From canvas note (“Setup steps”) |
| Extending: connect a trigger from any source at the beginning and connect to the normalize node. | From canvas note (“Extending”) |
| Dedupe warning: `Skip already processed items` and `Store processed IDs` must use the same `uniqueID` construction, otherwise dedupe will not work. | Observed from node expressions (integration issue) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.