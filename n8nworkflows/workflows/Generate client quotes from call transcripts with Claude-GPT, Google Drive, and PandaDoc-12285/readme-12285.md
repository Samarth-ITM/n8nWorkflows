Generate client quotes from call transcripts with Claude/GPT, Google Drive, and PandaDoc

https://n8nworkflows.xyz/workflows/generate-client-quotes-from-call-transcripts-with-claude-gpt--google-drive--and-pandadoc-12285


# Generate client quotes from call transcripts with Claude/GPT, Google Drive, and PandaDoc

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Generate client quotes from call transcripts with Claude/GPT, Google Drive, and PandaDoc  
Workflow name (in JSON): **AI-Powered Quote Generator from Call Transcripts using Multi-Agent Architecture**

---

## 1. Workflow Overview

This workflow automatically generates a client quote (PandaDoc document) from a newly uploaded **.VTT call transcript** in Google Drive, using a **multi-agent AI architecture** (main orchestrator + SOW agent + pricing agent). It then runs a **Slack approval flow**; if approved, it sends the PandaDoc document to the client and emails them a signing link. After signature, it updates a Notion CRM record, sends a welcome email, and posts a Slack confirmation.

### 1.1 Transcript Reception & Cleaning (Drive → text)
- Watches a Google Drive folder for new transcript files.
- Downloads the file, extracts text from binary, and removes timestamps/headers.

### 1.2 Meeting Context Enrichment (Calendar match)
- Pulls Google Calendar events for the last 30 days.
- Filters to the event whose title matches the transcript file name convention.

### 1.3 Main AI Orchestrator (multi-model agent)
- Receives meeting title, attendee email, current date, and cleaned transcript.
- Extracts client identity + proposal title.
- Creates the PandaDoc document via a “tool workflow call” pattern.
- Calls specialized agents in sequence (SOW then pricing).
- Outputs a structured summary (via structured output parser).

### 1.4 Specialized Sub-Agents (SOW + Pricing)
- **SOW Agent:** fills PandaDoc text tokens: problems, sub-problems, solutions, actions, durations.
- **Pricing Agent:** builds 1–3 pricing products (80%+ margin target), uses internal catalog + calculator + mandatory market search via Perplexity, then sends fields for pricing update.

### 1.5 PandaDoc Quote Creation, Polling & Editing
- Creates the PandaDoc document from a template.
- Waits/polls until document is in draft state.
- Updates tokens and pricing table via PATCH requests.

### 1.6 Review, Approval Flow & Client Delivery
- Gets PandaDoc “details” to retrieve review/shared link.
- Sends Slack “approve / reject” interactive message.
- If approved: sends PandaDoc document silently + emails the client the signing link.
- If rejected: collects feedback form in Slack and logs it to a Data Table.

### 1.7 Post-Signature Processing
- PandaDoc webhook receives “document signed” event.
- Fetches PandaDoc document details, finds the client in Notion CRM, updates status, sends welcome email, posts Slack notification.

---

## 2. Block-by-Block Analysis

### Block 1 — Transcript Cleaning & Meeting Extraction
**Overview:** Detects a new transcript file, converts it into clean plain text, then finds the matching calendar event to provide meeting metadata and attendee email for downstream AI processing.  
**Nodes involved:** Google Drive Trigger, Download file, Binary to text, Clean Transcript, Get many events, Match Meeting by Title, Sticky Note8

#### Node: Google Drive Trigger
- **Type / role:** `googleDriveTrigger` — entrypoint for new file events.
- **Configuration:** Triggers on **fileCreated** in a **specific folder** (“Sous-Titres .VTT”), polling every minute.
- **Key data:** Outputs file metadata including `id` and `name`.
- **Connections:** → Download file
- **Edge cases / failures:**
  - OAuth token expiry / insufficient Drive scope.
  - Duplicate triggers due to polling; consider deduping by file ID (not present here).
  - Non-.vtt files in the folder could break later assumptions.

#### Node: Download file
- **Type / role:** `googleDrive` — downloads the newly created file.
- **Configuration:** Operation **download** with `fileId = {{$json.id}}`.
- **Output:** Binary file content.
- **Connections:** → Binary to text
- **Edge cases:**
  - Drive file permission changes between trigger and download.
  - Very large files may hit n8n memory/time constraints.

#### Node: Binary to text
- **Type / role:** `extractFromFile` — extracts text from binary.
- **Configuration:** Operation **text**.
- **Output:** JSON containing extracted text under `data` (as used next).
- **Connections:** → Clean Transcript
- **Edge cases:**
  - If extraction produces a different field name (e.g., `text`), the next node expects `data`.
  - Encoding issues on unusual transcripts.

#### Node: Clean Transcript
- **Type / role:** `code` — removes WEBVTT/SRT artifacts.
- **Logic (interpreted):**
  - Reads `data.data` (string).
  - Splits into lines, removes empty lines, WEBVTT headers, NOTE, numeric line counters, and timestamp lines like `00:00:00.000 --> 00:00:00.000`.
  - Outputs `{ cleaned_text: "..." }`.
- **Connections:** → Get many events
- **Edge cases:**
  - If the extracted text is not in `data.data`, cleaning yields an empty transcript.
  - Timestamp regex is strict (requires milliseconds). Some VTT may omit them.

#### Node: Get many events
- **Type / role:** `googleCalendar` — fetches context events.
- **Configuration:** Operation **getAll**, time window: now minus 30 days → now.
- **Connections:** → Match Meeting by Title
- **Edge cases:**
  - Calendar timezone mismatches.
  - Multiple matching events with same title (filter later chooses all that match; may create multiple items).

#### Node: Match Meeting by Title
- **Type / role:** `filter` — selects the calendar event matching the transcript file.
- **Logic:** Compares `event.summary` to transcript filename prefix:
  - Right side: `{{ $('Google Drive Trigger').item.json.name.split(" -")[0] }}`
- **Connections:** → Analysis & Orchestrator Agent
- **Edge cases:**
  - Filename convention dependency: must include `" -"` delimiter.
  - If multiple events share the same summary, multiple quotes may be generated.

---

### Block 2 — Main AI Orchestrator
**Overview:** Central agent that merges transcript + meeting context, extracts client/proposal details, creates the PandaDoc quote, then coordinates SOW and pricing completion.  
**Nodes involved:** Analysis & Orchestrator Agent, Think, Sonnet 4.5, GPT4 Turbo, Structured Output Parser, quote_creation, Fetch Document Details, Sticky Note7

#### Node: Analysis & Orchestrator Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — multi-step AI orchestrator.
- **Input:** From Match Meeting by Title (calendar event), plus transcript from Clean Transcript referenced via expression.
- **Prompt payload (JSON string):**
  - meeting_title: `{{$json.summary}}`
  - client_email: `{{$json.attendees[0].email}}`
  - transcript: `{{ $('Clean Transcript').item.json.cleaned_text }}`
  - sender fields are placeholders and should be replaced.
- **System message highlights:**
  - Must create quote first, then call SOW agent, then Pricing agent.
  - Must pass created PandaDoc `document_id` to sub-agents.
  - Output only the document ID at the end.
- **AI tooling attached (via connections):**
  - Think tool
  - Fetch Document Details (PandaDoc template introspection)
  - quote_creation tool (subworkflow call back into same workflow route=main)
  - SOW Agent tool
  - Pricing Agent tool
- **Output parsing:** Has a structured output parser attached (Structured Output Parser).
- **Connections:** → Get_review_URL
- **Edge cases / failures:**
  - Attendee indexing: `attendees[0]` may be organizer/self; could pick wrong email.
  - Agent instruction conflict: “Output only document ID” vs structured parsing; ensure agent is actually configured to use parser safely.
  - Model/tool timeouts when transcript is long.

#### Node: Think
- **Type / role:** `toolThink` — allows the agent to do internal reasoning steps.
- **Connections:** Think is wired as an AI tool to the orchestrator.
- **Edge cases:** None operational; only used inside agent toolchain.

#### Node: Sonnet 4.5
- **Type / role:** `lmChatOpenRouter` — language model provider for orchestrator (primary).
- **Model:** `anthropic/claude-sonnet-4.5`
- **Credential:** OpenRouter.
- **Connections:** Supplies `ai_languageModel` to orchestrator.
- **Edge cases:** OpenRouter quota/model availability.

#### Node: GPT4 Turbo
- **Type / role:** `lmChatOpenRouter` — secondary model option for orchestrator.
- **Model:** `openai/gpt-4-turbo`
- **Connections:** Also available to orchestrator.
- **Edge cases:** Same as above; plus vendor-specific prompt length limits.

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured`
- **Role:** Enforces/repairs structured JSON output (autoFix enabled) with a schema example including: `document_id`, `client`, `proposal`, `pricing`, `products[]`, `summary`.
- **Connections:** Connected as `ai_outputParser` to orchestrator.
- **Edge cases:**
  - If the agent is instructed to output only a document ID, it may contradict schema expectations.
  - Auto-fix can mask upstream extraction errors.

#### Node: quote_creation (tool)
- **Type / role:** `toolWorkflow` — tool the agent calls to create a PandaDoc document.
- **Workflow target:** This same workflow ID (`un7napkeeNqhsSTE`) using the internal “Called by Agent” entrypoint and routing.
- **Inputs sent:** `route="main"` plus client identity fields and proposal title.
- **Connections:** Used only as an agent tool (not in main line).
- **Edge cases:**
  - The agent must supply correct string fields; otherwise PandaDoc creation fails.
  - Uses template UUID fixed later in Create Quote.

#### Node: Fetch Document Details (tool)
- **Type / role:** `httpRequestTool` — AI tool to fetch template details from PandaDoc.
- **URL:** `/templates/3ktiP5fdeiwNyhiTXaz99e/details`
- **Purpose:** Lets agents inspect token structure if uncertain.
- **Edge cases:** PandaDoc auth; template UUID must match the one used in Create Quote.

---

### Block 3 — Specialized Sub-Agents
**Overview:** Two agent tools generate (1) textual scope-of-work tokens and (2) pricing products and deposit percentage, then send those fields back via workflow-tools that route into PandaDoc PATCH nodes.  
**Nodes involved:** SOW Agent, Pricing agent, Opus 4.5, GPT 4 TURBO, GPT4o TURBO, THINK, think, Simple Memory, Simple memory, tokens (tool), update_pricing (tool), project_duration_calculation (tool), DB_services_&_details (tool), Search Tools & pricing (tool), Calculator (tool), Grok, Sticky Note6

#### Node: SOW Agent
- **Type / role:** `agentTool` — specialized agent callable by orchestrator.
- **System message:** Fills only specific PandaDoc token fields: `probleme.general`, `sous.problème.*`, `titre.solution.*`, `desc.solution.*`, `action.*`, and `action.*.durée.jours`. Uses placeholders if unclear; concise.
- **Tools available to this agent:**
  - `project_duration_calculation` (date math tool)
  - `Fetch Document Details` (mentioned in prompt; connected to orchestrator toolset)
  - `tokens` tool to send token updates (must pass quote ID).
- **Models available:**
  - GPT 4 TURBO (as model index 0)
  - Opus 4.5 (as model index 1)
- **Memory:** “Simple Memory” with `sessionKey="SKU"` (buffer window).
- **Edge cases:**
  - Token names include dots and accents (`sous.problème...`), which must exactly match PandaDoc template token names.
  - Durations are strings in tool schema; risk of non-numeric text.

#### Node: Pricing agent
- **Type / role:** `agentTool` — generates pricing table fields for PandaDoc.
- **System message constraints:**
  - 1–3 products max; margin >80%.
  - Must use Perplexity search if prices are uncertain (“mandatory if uncertain”; prompt says invalid if no search when uncertain).
  - Deposit percentage mandatory (`acompte_percent` between 20 and 70).
  - Output must be valid JSON and directly sent to `update_pricing` tool.
- **Tools attached:**
  - DB_services_&_details (internal catalog)
  - Calculator (margin computation)
  - Search Tools & pricing (Perplexity)
  - Grok model available (also present) + GPT4o TURBO as a model input
  - think/toolThink node
- **Memory:** “Simple memory” with `sessionKey="previousprices"`.
- **Edge cases:**
  - Enforcing the “mandatory search” is not programmatic; agent may skip it.
  - The workflow later expects fields named `product_1_desc` etc., but the `update_pricing` tool schema includes accidental leading spaces in `" product_1_desc"` and `" product_3_price"` mappings—this can break payload building (see Block 5).

#### Node: tokens (tool)
- **Type / role:** `toolWorkflow` — sends token fields to workflow route `tokens`.
- **Workflow target:** same workflow ID.
- **Inputs:** `route="tokens"` plus extensive token fields + `id doc créé` (document id).
- **Edge cases:**
  - If any required PandaDoc token is missing, Update Tokens may still run but inject empty/invalid values.

#### Node: update_pricing (tool)
- **Type / role:** `toolWorkflow` — sends pricing fields to workflow route `pricing`.
- **Inputs:** `route="pricing"`, `id doc créé`, products 1–3, and `acompte_percent`.
- **Important issue:** Some input keys include leading spaces: `" product_1_desc"`, `" product_3_price"`. Downstream Build Pricing Payload expects `product_1_desc` / `product_3_price` (no leading space).
- **Edge cases:** High likelihood of missing descriptions/prices for product 1/3 unless corrected.

#### Node: project_duration_calculation
- **Type / role:** `dateTimeTool`
- **Operation:** `addToDate`, outputs field named `Livraison`.
- **Used by:** SOW Agent toolchain.
- **Edge cases:** Requires numeric `Duration`; “magnitude” naming suggests confusion (it is fed by `Date_to_Add_To`).

#### Node: DB_services_&_details
- **Type / role:** `dataTableTool` — reads internal services catalog.
- **Data table:** “Catalogue Prix | Agence IA”
- **Used by:** Pricing agent toolchain.
- **Edge cases:** Table ID must exist in the same n8n project; permissions required.

#### Node: Search Tools & pricing
- **Type / role:** `perplexityTool`
- **Used by:** Pricing agent toolchain for market research.
- **Inputs:** dynamic message + optional “simplify output”.
- **Edge cases:** Perplexity API quota; prompt injection risk from transcript content.

#### Node: Calculator
- **Type / role:** `toolCalculator`
- **Used by:** Pricing agent toolchain.
- **Edge cases:** None; purely internal.

#### Node: Grok / GPT4o TURBO / GPT 4 TURBO / Opus 4.5
- **Type / role:** OpenRouter chat model nodes supporting agents/parsers.
- **Edge cases:** Model availability; increased cost.

#### Node: THINK / think
- **Type / role:** `toolThink`
- **Used by:** SOW and Pricing agents respectively.

#### Node: Simple Memory / Simple memory
- **Type / role:** `memoryBufferWindow`
- **Role:** Gives short-term conversational memory across agent tool calls using `sessionIdType="customKey"`.
- **Edge cases:** If multiple executions share the same custom key, cross-run contamination is possible.

---

### Block 4 — PandaDoc Quote Creation, Polling & Editing (agent-called routes)
**Overview:** Implements a “multi-route” workflow pattern: agents call the same workflow via Execute Workflow Trigger and pass `route` to a Switch node. Depending on the route, the workflow creates the PandaDoc doc, updates tokens, or updates pricing.  
**Nodes involved:** Called by Agent, Route by Action Type, Create Quote, Wait for completion, Status ?, Document Ready?, Return Document Info, Update Tokens, Token Update Success, Token Update Error, Update Pricing Section, Build Pricing Payload, Pricing Update Success, Pricing Update Error, Sticky Note11

#### Node: Called by Agent
- **Type / role:** `executeWorkflowTrigger` — second entrypoint (subworkflow trigger).
- **Role:** Receives inputs from toolWorkflow calls (quote_creation / tokens / update_pricing).
- **Input schema:** Contains `route` plus many optional fields for tokens and pricing.
- **Connections:** → Route by Action Type
- **Edge cases:** If the toolWorkflow sends keys not in schema, they may be dropped.

#### Node: Route by Action Type
- **Type / role:** `switch`
- **Routing rules:** Based on `{{$json['route']}}`:
  - `main` → Create Quote
  - `tokens` → Update Tokens
  - `pricing` → Build Pricing Payload
- **Edge cases:** Any unexpected route value leads to no output path (no default configured).

#### Node: Create Quote
- **Type / role:** `httpRequest` to PandaDoc (create document from template).
- **Endpoint:** `POST https://api.pandadoc.com/public/v1/documents`
- **Body constructs:**
  - `name`: from `name (Proposition - proposition_title)`
  - `template_uuid`: `3ktiP5fdeiwNyhiTXaz99e`
  - recipient fields mapped from incoming JSON, role “Client”
  - sender tokens (company/name/signature) are hard-coded
- **Auth:** generic header auth credential “Pandadoc”
- **Connections:** → Wait for completion
- **Edge cases:**
  - PandaDoc token names must exist in template.
  - Recipient role must match PandaDoc template role.

#### Node: Wait for completion
- **Type / role:** `wait`
- **Configuration:** waits `amount=1` (default unit: seconds in n8n Wait node unless configured otherwise).
- **Connections:** → Status ?
- **Edge cases:** Document may not be ready after 1 second; relies on looping with Status?/Document Ready?.

#### Node: Status ?
- **Type / role:** `httpRequest` — fetch document status.
- **Endpoint:** `GET https://api.pandadoc.com/public/v1/documents/{{ $json.id }}`
- **Auth header:** Hard-coded API key in header parameter (security risk; should be credential-based).
- **Connections:** → Document Ready?
- **Edge cases:**
  - If Create Quote returns a different field than `id`, request fails.
  - Hard-coded API key may be invalid in other environments.

#### Node: Document Ready?
- **Type / role:** `if`
- **Condition:** `{{$json.status}} == "document.draft"`
- **True branch:** → Return Document Info
- **False branch:** → Status ? (loop)
- **Edge cases:** Infinite loop if status never becomes `document.draft` or API errors occur.

#### Node: Return Document Info
- **Type / role:** `set`
- **Outputs:** `etat="Document créé avec succès !"`, `id={{$json.id}}`
- **Used by:** Returns doc info back to the calling agent tool invocation.

#### Node: Update Tokens
- **Type / role:** `httpRequest` (PATCH PandaDoc document tokens)
- **Endpoint:** `PATCH /documents/{{ $json['id doc créé'] }}`
- **Body:** `tokens: [{name, value}, ...]` for all SOW fields.
- **Error handling:** `onError="continueErrorOutput"` then success/error sets.
- **Connections:** → Token Update Success and Token Update Error (two outputs due to continueErrorOutput)
- **Important bug:** `desc.solution.3` is set to `{{ $json['titre.solution.3'] }}` instead of `{{ $json['desc.solution.3'] }}`.
- **Edge cases:** Any token name mismatch causes PandaDoc to ignore or error.

#### Node: Token Update Success / Token Update Error
- **Type / role:** `set`
- **Purpose:** Returns a simple message for tool caller.

#### Node: Build Pricing Payload
- **Type / role:** `code`
- **Purpose:** Converts `product_1..3_*` fields into PandaDoc pricing table payload format (v2).
- **Logic highlights:**
  - Builds rows only if name exists and price is numeric.
  - Applies VAT 20% to each item.
  - Adds pricing table discount object used as “Acompte” (but semantically it’s discount; PandaDoc “discount” used to represent deposit percent).
  - Sets token `acompte.pourcent` to deposit percent.
- **Connections:** → Update Pricing Section
- **Edge cases:**
  - If `product_i_desc` fields are missing (due to upstream mapping bug), descriptions become blank.
  - If `acompte_percent` is not numeric, defaults to 30.
  - Uses “discount percent” to store deposit; verify template logic matches.

#### Node: Update Pricing Section
- **Type / role:** `httpRequest` (PATCH PandaDoc document)
- **Endpoint:** `PATCH /documents/{{ $('Called by Agent').item.json["id doc créé"] }}`
- **Body:** Entire JSON from Build Pricing Payload.
- **Error handling:** `continueErrorOutput`
- **Critical security issue:** Authorization header is hard-coded: `API-Key ...`
- **Connections:** → Pricing Update Success and Pricing Update Error
- **Edge cases:** PandaDoc expects correct pricing table schema; if rows is empty, pricing table may be removed/invalid.

#### Node: Pricing Update Success / Pricing Update Error
- **Type / role:** `set`
- **Purpose:** Returns a simple message for tool caller.

---

### Block 5 — Review, Approval Flow & Client Email Delivery
**Overview:** Retrieves PandaDoc review details, asks internal approval in Slack, then either sends the document and emails the client (approved) or gathers rejection feedback and logs it.  
**Nodes involved:** Get_review_URL, Send for Review, Is Approved?, Send_mode, Send a message, confirmation, feedback_collection, Log Rejection Feedback, Sticky Note13

#### Node: Get_review_URL
- **Type / role:** `httpRequest`
- **Endpoint:** `GET /documents/{{ $json.output.document_id }}/details`
- **Input dependency:** Expects upstream JSON at `output.document_id` from orchestrator.
- **Connections:** → Send for Review
- **Edge cases:** If orchestrator outputs only an ID string (not `{output: {document_id}}`), this breaks.

#### Node: Send for Review
- **Type / role:** `slack` (sendAndWait with approvals)
- **Message content:** Includes client name/company tokens, PandaDoc review link, total, deposit %, and a list of products referencing `$('Analysis & Orchestrator Agent').item.json.products[...]` (indexes 0..4).
- **Approval options:** double approval with 👍 approve and ❌ disapprove.
- **Connections:** → Is Approved?
- **Edge cases:**
  - References 5 products, but pricing agent limits to 1–3; indexing may fail.
  - If PandaDoc details payload shape differs (pricing/tokens paths), message template errors.

#### Node: Is Approved?
- **Type / role:** `if`
- **Condition:** `{{$json.data.approved}} == true`
- **True:** → Send_mode
- **False:** → feedback_collection

#### Node: Send_mode
- **Type / role:** `httpRequest` (PandaDoc send)
- **Endpoint:** `POST /documents/{{ $('Get_review_URL').item.json.id }}/send`
- **Body:** `{ "silent": true }`
- **Auth:** header auth credential “Pandadoc” (but header parameters list is empty; relies on credential)
- **Connections:** → Send a message
- **Edge cases:** PandaDoc “silent send” still requires recipient; may fail if doc not in sendable state.

#### Node: Send a message
- **Type / role:** `gmail` — sends client email with PandaDoc shared link.
- **To:** `{{$json.recipients[0].email}}`
- **Uses:** `{{$json.recipients[0].shared_link}}`
- **Connections:** → confirmation (Slack)
- **Edge cases:** Gmail OAuth; shared_link presence depends on PandaDoc details.

#### Node: confirmation
- **Type / role:** `slack`
- **Message:** Confirms proposal was sent.
- **Edge cases:** None.

#### Node: feedback_collection
- **Type / role:** `slack` sendAndWait with a custom form (dropdown, radio, textarea).
- **Connections:** → Log Rejection Feedback
- **Edge cases:** If user doesn’t submit, workflow waits indefinitely.

#### Node: Log Rejection Feedback
- **Type / role:** `dataTable` — stores rejection reasons.
- **Writes columns:** id_doc, clarity, comment, main reason.
- **Data table:** “Data Rating Agent Créateur de Devis”
- **Edge cases:** Table schema mismatch or missing permissions.

---

### Block 6 — Post-Signature Processing
**Overview:** When PandaDoc notifies that the document has been signed, the workflow looks up the client in Notion CRM, updates status, sends a welcome email, and posts a Slack alert.  
**Nodes involved:** when_doc_is_signed, details, Search_CRM, Find Client in CRM, Update status, welcome_mail, confirmation_message, Sticky Note14, Sticky Note17

#### Node: when_doc_is_signed
- **Type / role:** `webhook`
- **Method/path:** POST at `/7ca56f48-be92-4e3d-8adf-fffc2508beca`
- **Purpose:** PandaDoc webhook target (configure in PandaDoc).
- **Connections:** → details
- **Edge cases:** Signature event payload validation not enforced; anyone could post unless protected (no auth configured).

#### Node: details
- **Type / role:** `httpRequest`
- **Endpoint:** `GET /documents/AKdYLJHnbzJfo6rro8VptS/details`
- **Critical issue:** Document ID is hard-coded (`AKdYLJH...`) instead of taken from webhook payload. This prevents correct per-document processing.
- **Connections:** → Search_CRM
- **Edge cases:** Will always fetch the same doc; breaks real signature automation.

#### Node: Search_CRM
- **Type / role:** `notion` getAll database pages
- **Database:** “CRM”
- **Connections:** → Find Client in CRM
- **Edge cases:** Large CRM databases; pagination; rate limits.

#### Node: Find Client in CRM
- **Type / role:** `filter`
- **Condition:** `{{$json.name}} == "{{ $('details').item.json.tokens[1].value }} {{ $('details').item.json.tokens[2].value }}"`
- **Connections:** → Update status
- **Edge cases:** Exact string matching on name is fragile (spacing/case/diacritics).

#### Node: Update status
- **Type / role:** `notion` update page
- **Property updated:** `Client Status` select → `Client (signé)`
- **Connections:** → welcome_mail
- **Edge cases:** Notion select option must exist exactly.

#### Node: welcome_mail
- **Type / role:** `gmail`
- **To:** `{{ $('Find Client in CRM').item.json.property_contact_email }}`
- **Message includes:** Kickoff link `https://cal.com/hugo-monetizia/kick-off-projet`
- **Connections:** → confirmation_message
- **Edge cases:** Email field name `property_contact_email` must exist in Notion item JSON.

#### Node: confirmation_message
- **Type / role:** `slack`
- **Message:** “{First Last} de {Company} a signé ton devis !” using tokens from PandaDoc details.
- **Edge cases:** Token indices `[0] [1] [2]` assume fixed ordering.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Drive Trigger | googleDriveTrigger | Watches transcript folder for new files | — | Download file | ## 1. Transcript Cleaning & Meeting Extraction |
| Download file | googleDrive | Downloads new transcript file | Google Drive Trigger | Binary to text | ## 1. Transcript Cleaning & Meeting Extraction |
| Binary to text | extractFromFile | Extracts plain text from file binary | Download file | Clean Transcript | ## 1. Transcript Cleaning & Meeting Extraction |
| Clean Transcript | code | Removes timestamps/headers from VTT | Binary to text | Get many events | ## 1. Transcript Cleaning & Meeting Extraction |
| Get many events | googleCalendar | Fetches recent calendar events | Clean Transcript | Match Meeting by Title | ## 1. Transcript Cleaning & Meeting Extraction |
| Match Meeting by Title | filter | Selects matching event based on transcript filename | Get many events | Analysis & Orchestrator Agent | ## 2. Main AI Orchestrator |
| Analysis & Orchestrator Agent | langchain.agent | Main orchestrator: extract info, create quote, call sub-agents | Match Meeting by Title | Get_review_URL | ## 2. Main AI Orchestrator |
| Think | toolThink | Reasoning tool for orchestrator | — (AI tool) | — | ## 2. Main AI Orchestrator |
| Sonnet 4.5 | lmChatOpenRouter | Primary LLM for orchestrator | — | — (AI model) | ## 2. Main AI Orchestrator |
| GPT4 Turbo | lmChatOpenRouter | Secondary LLM for orchestrator | — | — (AI model) | ## 2. Main AI Orchestrator |
| Structured Output Parser | outputParserStructured | Enforces JSON structure for orchestrator | — (AI parser) | — | ## 2. Main AI Orchestrator |
| quote_creation | toolWorkflow | Agent tool: route=main to create PandaDoc doc | — (AI tool) | — | ## 2. Main AI Orchestrator |
| Fetch Document Details | httpRequestTool | Agent tool: get template details | — (AI tool) | — | ## 2. Main AI Orchestrator |
| SOW Agent | agentTool | Generates problems/solutions/actions tokens | — (AI tool) | — | ## 3. Specialized Sub-Agents |
| Pricing agent | agentTool | Generates 1–3 priced products + deposit | — (AI tool) | — | ## 3. Specialized Sub-Agents |
| tokens | toolWorkflow | Agent tool: route=tokens (send token fields) | — (AI tool) | — | ## 3. Specialized Sub-Agents |
| update_pricing | toolWorkflow | Agent tool: route=pricing (send product fields) | — (AI tool) | — | ## 3. Specialized Sub-Agents |
| project_duration_calculation | dateTimeTool | Tool: compute milestone dates | — (AI tool) | — | ## 3. Specialized Sub-Agents |
| DB_services_&_details | dataTableTool | Tool: read internal price catalog | — (AI tool) | — | ## 3. Specialized Sub-Agents |
| Search Tools & pricing | perplexityTool | Tool: market research | — (AI tool) | — | ## 3. Specialized Sub-Agents |
| Calculator | toolCalculator | Tool: compute totals/margins | — (AI tool) | — | ## 3. Specialized Sub-Agents |
| Grok | lmChatOpenRouter | LLM option for pricing | — | — (AI model) | ## 3. Specialized Sub-Agents |
| GPT 4 TURBO | lmChatOpenRouter | LLM for SOW agent + parser | — | — (AI model) | ## 3. Specialized Sub-Agents |
| Opus 4.5 | lmChatOpenRouter | LLM option for SOW agent | — | — (AI model) | ## 3. Specialized Sub-Agents |
| GPT4o TURBO | lmChatOpenRouter | LLM option for pricing agent | — | — (AI model) | ## 3. Specialized Sub-Agents |
| THINK | toolThink | Reasoning tool for SOW agent | — (AI tool) | — | ## 3. Specialized Sub-Agents |
| think | toolThink | Reasoning tool for Pricing agent | — (AI tool) | — | ## 3. Specialized Sub-Agents |
| Simple Memory | memoryBufferWindow | Memory for SOW agent | — | — (AI memory) | ## 3. Specialized Sub-Agents |
| Simple memory | memoryBufferWindow | Memory for Pricing agent | — | — (AI memory) | ## 3. Specialized Sub-Agents |
| Called by Agent | executeWorkflowTrigger | Entry for agent toolWorkflow calls | — | Route by Action Type | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Route by Action Type | switch | Routes: main/tokens/pricing | Called by Agent | Create Quote / Update Tokens / Build Pricing Payload | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Create Quote | httpRequest | POST PandaDoc document from template | Route by Action Type (main) | Wait for completion | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Wait for completion | wait | Short delay before polling status | Create Quote | Status ? | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Status ? | httpRequest | GET PandaDoc document status | Wait for completion / Document Ready? (loop) | Document Ready? | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Document Ready? | if | Checks status==document.draft | Status ? | Return Document Info / Status ? | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Return Document Info | set | Returns success + id to caller | Document Ready? (true) | — | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Update Tokens | httpRequest | PATCH PandaDoc tokens (SOW text fields) | Route by Action Type (tokens) | Token Update Success / Token Update Error | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Token Update Success | set | Tool response success message | Update Tokens | — | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Token Update Error | set | Tool response error message | Update Tokens | — | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Build Pricing Payload | code | Builds PandaDoc pricing table payload | Route by Action Type (pricing) | Update Pricing Section | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Update Pricing Section | httpRequest | PATCH PandaDoc pricing table | Build Pricing Payload | Pricing Update Success / Pricing Update Error | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Pricing Update Success | set | Tool response success message | Update Pricing Section | — | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Pricing Update Error | set | Tool response error message | Update Pricing Section | — | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Get_review_URL | httpRequest | Gets PandaDoc document details for review | Analysis & Orchestrator Agent | Send for Review | ## 5. Review, Approval Flow &  Client Email Delivery |
| Send for Review | slack (sendAndWait) | Slack approval with 👍/❌ | Get_review_URL | Is Approved? | ## 5. Review, Approval Flow &  Client Email Delivery |
| Is Approved? | if | Routes approved vs rejected | Send for Review | Send_mode / feedback_collection | ## 5. Review, Approval Flow &  Client Email Delivery |
| Send_mode | httpRequest | Sends PandaDoc doc (silent) | Is Approved? (true) | Send a message | ## 5. Review, Approval Flow &  Client Email Delivery |
| Send a message | gmail | Emails client PandaDoc link | Send_mode | confirmation | ## 5. Review, Approval Flow &  Client Email Delivery |
| confirmation | slack | Slack confirmation “proposal sent” | Send a message | — | ## 5. Review, Approval Flow &  Client Email Delivery |
| feedback_collection | slack (sendAndWait) | Collects rejection form | Is Approved? (false) | Log Rejection Feedback | ## 5. Review, Approval Flow &  Client Email Delivery |
| Log Rejection Feedback | dataTable | Stores rejection feedback | feedback_collection | — | ## 5. Review, Approval Flow &  Client Email Delivery |
| when_doc_is_signed | webhook | Entry for PandaDoc signature events | — | details | ## 6.  Post-Signature Processing |
| details | httpRequest | Fetches PandaDoc document details (currently hard-coded ID) | when_doc_is_signed | Search_CRM | ## 6.  Post-Signature Processing |
| Search_CRM | notion | Loads Notion CRM pages | details | Find Client in CRM | ## 6.  Post-Signature Processing |
| Find Client in CRM | filter | Matches CRM entry to signed doc client name | Search_CRM | Update status | ## 6.  Post-Signature Processing |
| Update status | notion | Sets CRM status to “Client (signé)” | Find Client in CRM | welcome_mail | ## 6.  Post-Signature Processing |
| welcome_mail | gmail | Sends onboarding + kickoff booking email | Update status | confirmation_message | ## 6.  Post-Signature Processing |
| confirmation_message | slack | Notifies internal team of signature | welcome_mail | — | ## 6.  Post-Signature Processing |
| Sticky Note8 | stickyNote | Section header | — | — | ## 1. Transcript Cleaning & Meeting Extraction |
| Sticky Note7 | stickyNote | Section header | — | — | ## 2. Main AI Orchestrator |
| Sticky Note6 | stickyNote | Section header | — | — | ## 3. Specialized Sub-Agents |
| Sticky Note11 | stickyNote | Section header | — | — | ## 4. PandaDoc Quote Creation, Polling & Editing |
| Sticky Note13 | stickyNote | Section header | — | — | ## 5. Review, Approval Flow &  Client Email Delivery |
| Sticky Note14 | stickyNote | Section header | — | — | ## 6.  Post-Signature Processing |
| Sticky Note15 | stickyNote | Setup reminder | — | — | ⚠️**Setup Required**\n\n```markdown\nAdd your API keys:\n- PandaDoc API Key\n- OpenRouter API Key  \n- Perplexity API Key\n``` |
| Sticky Note17 | stickyNote | Setup reminder | — | — | ⚠️ **PandaDoc Webhook**\n\n```markdown\nConfigure webhook\nin PandaDoc to \ntrigger on\ndocument signature\n``` |
| Sticky Note18 | stickyNote | Global description & setup checklist | — | — | ## Generate quotes from call transcripts with multi-agent AI\n\n### How it works\n\n1. **Google Drive trigger:** monitors folder for new .vtt transcript files\n2. **Transcript cleaning:** removes timestamps and formats raw text\n3. **Calendar matching:** finds meeting context from Google Calendar\n4. **Main AI agent (Claude/GPT):** orchestrates quote creation, extracts client info\n5. **SOW Agent:** generates scope of work content (problems, solutions, actions)\n6. **Pricing Agent:** creates pricing table with 80%+ margins using market research\n7. **PandaDoc:** creates document from template, fills tokens and pricing\n8. **Slack review:** sends for approval with approve/reject buttons\n9. **Gmail delivery:** sends signed document link to client\n10. **CRM update:** updates Notion database with client status\n\n### Setup\n\n☐ Connect **Google Drive** OAuth and set trigger folder\n☐ Add **OpenRouter** API key for AI models\n☐ Add **PandaDoc** API key and template UUID\n☐ Connect **Slack** OAuth for approval workflow\n☐ Connect **Gmail** for client emails\n☐ Connect **Notion** database for CRM\n☐ Configure **PandaDoc webhook** for signature tracking\n\n### Customize\n\n- **AI Models:** swap Claude/GPT models in agent nodes\n- **Pricing:** adjust margin rules in Pricing Agent prompt\n- **Template:** modify PandaDoc template tokens as needed |
| Sticky Note | stickyNote | Video link | — | — | ### Video tutorial\n\n[youtube](https://www.youtube.com/watch?v=h6ssuN2ceOg) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Google Drive OAuth2 credential (access to the transcript folder).
   2. Google Calendar OAuth2 credential (read calendar events).
   3. OpenRouter API credential (for Claude/GPT/Grok models).
   4. Perplexity API credential (for market search tool).
   5. PandaDoc API credential (Header Auth / API key).
   6. Slack OAuth2 credential (for sendAndWait approval + messages).
   7. Gmail OAuth2 credential (send emails).
   8. Notion API credential (CRM database access).

2) **Create Google Drive Trigger**
   - Node: **Google Drive Trigger**
   - Trigger: *File Created*
   - Watch: *Specific folder* → select the folder containing `.vtt` transcripts.
   - Poll interval: Every minute.

3) **Download and extract transcript**
   - Add **Google Drive** node “Download file” → Operation: Download → File ID: `{{$json.id}}`.
   - Add **Extract From File** node “Binary to text” → Operation: Text.

4) **Clean transcript**
   - Add **Code** node “Clean Transcript” implementing:
     - Remove WEBVTT headers/NOTE, timestamps, numeric counters, empty lines.
     - Output `cleaned_text`.

5) **Fetch calendar events and match meeting**
   - Add **Google Calendar** node “Get many events” → Operation: Get All
     - `timeMin: {{$now.minus({ days: 30 })}}`
     - `timeMax: {{$now}}`
   - Add **Filter** node “Match Meeting by Title”
     - Left: `{{$json.summary}}`
     - Right: `{{ $('Google Drive Trigger').item.json.name.split(" -")[0] }}`

6) **Add AI model nodes (OpenRouter)**
   - Add OpenRouter chat model nodes:
     - “Sonnet 4.5” → model `anthropic/claude-sonnet-4.5`
     - “GPT4 Turbo” → model `openai/gpt-4-turbo`
     - “Opus 4.5” → model `anthropic/claude-opus-4.5`
     - “GPT4o TURBO” → model `openai/gpt-4-turbo` (as in JSON; rename if needed)
     - “Grok” → model `x-ai/grok-4`
   - Connect them to the appropriate agent nodes as `ai_languageModel`.

7) **Create Main Orchestrator Agent**
   - Node: **AI Agent** “Analysis & Orchestrator Agent”
   - Input text: JSON including meeting title (`$json.summary`), attendee email, date, and `$('Clean Transcript').item.json.cleaned_text`.
   - System message: enforce sequence (create quote → SOW → pricing).
   - Attach tools:
     - **Think tool**
     - **HTTP Request Tool** “Fetch Document Details” (template details endpoint)
     - **Tool Workflow** “quote_creation”
     - **AgentTool** “SOW Agent”
     - **AgentTool** “Pricing agent”
   - Attach **Structured Output Parser** (optional, but then align the agent output format accordingly).

8) **Create Specialized Agents**
   - Node: **Agent Tool** “SOW Agent”
     - Provide system message listing the exact PandaDoc token fields to produce.
     - Attach memory “Simple Memory” (optional).
     - Attach tools:
       - DateTime tool “project_duration_calculation”
       - Tool Workflow “tokens”
   - Node: **Agent Tool** “Pricing agent”
     - Provide pricing constraints (1–3 products, deposit, margin).
     - Attach tools:
       - Data Table Tool “DB_services_&_details”
       - Calculator tool
       - Perplexity tool
       - Tool Workflow “update_pricing”
     - Attach memory “Simple memory” (optional).

9) **Implement the “agent-called routes” entrypoint**
   - Add **Execute Workflow Trigger** node “Called by Agent”.
   - Define workflow inputs including `route`, `id doc créé`, token fields, and pricing fields.
   - Add **Switch** node “Route by Action Type”:
     - route == `main`
     - route == `tokens`
     - route == `pricing`

10) **Route=main: Create PandaDoc document and poll**
   - Add **HTTP Request** “Create Quote” (POST PandaDoc /documents)
     - Use template UUID (your PandaDoc template).
     - Include recipient role matching the template.
   - Add **Wait** “Wait for completion” (small delay).
   - Add **HTTP Request** “Status ?” (GET PandaDoc /documents/{id}).
   - Add **IF** “Document Ready?” checking status `document.draft`.
   - Add **Set** “Return Document Info” returning `id`.

11) **Route=tokens: PATCH PandaDoc tokens**
   - Add **HTTP Request** “Update Tokens” (PATCH /documents/{id doc créé})
     - Build `tokens[]` list mapping to your PandaDoc template token names.
   - Add **Set** nodes for success/error messages.
   - Fix the mapping bug: ensure `desc.solution.3` uses the correct field.

12) **Route=pricing: Build pricing payload and PATCH**
   - Add **Code** “Build Pricing Payload” that:
     - Builds `pricing_tables` with `rows` for products 1–3.
     - Adds token `acompte.pourcent`.
   - Add **HTTP Request** “Update Pricing Section” (PATCH /documents/{id})
   - Add **Set** nodes for success/error.

13) **Review and approval**
   - Add **HTTP Request** “Get_review_URL” (GET /documents/{document_id}/details).
   - Add **Slack** “Send for Review” (sendAndWait) with 👍/❌.
   - Add **IF** “Is Approved?”
     - True → send PandaDoc + Gmail
     - False → Slack form + Data Table logging

14) **Client delivery**
   - Add **HTTP Request** “Send_mode” POST /send with `{silent:true}`.
   - Add **Gmail** “Send a message” using `recipients[0].shared_link`.
   - Add **Slack** “confirmation”.

15) **Post-signature webhook**
   - Add **Webhook** “when_doc_is_signed” (POST).
   - In PandaDoc, configure webhook to call this URL on “document completed/signed”.
   - Add **HTTP Request** “details” to fetch the document details **using the document ID from webhook payload** (do not hard-code).
   - Add **Notion** getAll “Search_CRM”, then **Filter** to match client, then **Notion Update** “Update status”.
   - Add **Gmail** “welcome_mail” and **Slack** “confirmation_message”.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Add your API keys: PandaDoc, OpenRouter, Perplexity | Sticky note “Setup Required” |
| Configure PandaDoc webhook to trigger on document signature | Sticky note “PandaDoc Webhook” |
| Video link | https://www.youtube.com/watch?v=h6ssuN2ceOg |
| Kickoff booking link used in welcome email | https://cal.com/hugo-monetizia/kick-off-projet |

---