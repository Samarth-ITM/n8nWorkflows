Optimize Webflow CMS SEO metadata using MCP and GPT‑4o-mini

https://n8nworkflows.xyz/workflows/optimize-webflow-cms-seo-metadata-using-mcp-and-gpt-4o-mini-13315


# Optimize Webflow CMS SEO metadata using MCP and GPT‑4o-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Optimize Webflow CMS SEO metadata using MCP and GPT‑4o-mini  
**Workflow name (in JSON):** Optimize SEO metadata for Webflow CMS using MCP and AI

This workflow bulk-optimizes SEO metadata for items in a Webflow CMS collection. It retrieves items via Webflow’s MCP endpoint, batches them, asks an AI agent (GPT‑4o-mini via OpenRouter) to propose SEO-safe improvements (title + meta description length targets plus content audits), updates only items that need changes, publishes them, then logs a run summary to Google Sheets.

### 1.1 Configuration & Trigger
- Manual start; set collection ID and SEO length constraints.

### 1.2 Webflow CMS Retrieval (MCP)
- Fetch up to 100 CMS items from the configured Webflow collection.

### 1.3 Batch Processing + AI SEO Optimization
- Process items in batches of 5.
- AI agent returns **strict JSON** describing per-item updates + audit flags.
- Structured parser validates/auto-fixes the agent output.

### 1.4 Update + Publish Back to Webflow (MCP)
- Format payload for Webflow.
- Update item fieldData (only non-null fields).
- Publish updated items.

### 1.5 Metrics & Logging
- Compute execution duration and summarize updated/published items.
- Append summary row to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input Reception & Global Settings
**Overview:** Starts the workflow manually and defines the collection ID plus SEO constraints used by the AI prompt and memory key.  
**Nodes involved:** `Manual Trigger`, `Set Fields`

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — human-initiated entry point.
- **Configuration:** No parameters.
- **Inputs/Outputs:** Output → `Set Fields`.
- **Edge cases:** None (only runs when manually executed).
- **Version notes:** TypeVersion 1.

#### Node: Set Fields
- **Type / role:** `n8n-nodes-base.set` — defines reusable constants for the run.
- **Key configuration choices:**
  - Sets:
    - `collectionId` (string): **placeholder** `xxxxxxxxxxxxxx`
    - `minTitleLength` (50), `maxTitleLength` (60)
    - `minSummaryLength` (120), `maxSummaryLength` (155)
- **Key expressions/variables:** None (static values).
- **Inputs/Outputs:** Input ← `Manual Trigger`; Output → `Fetch CMS Items`.
- **Edge cases / failures:**
  - Wrong `collectionId` leads to MCP 404 later.
  - If you adjust length limits, ensure the agent prompt and expectations remain consistent.
- **Version notes:** TypeVersion 3.4.

---

### Block 2.2 — Fetch Webflow CMS Items (MCP)
**Overview:** Calls Webflow MCP `data_cms_tool` to list items in a collection (up to 100) for SEO analysis.  
**Nodes involved:** `Fetch CMS Items`

#### Node: Fetch CMS Items
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClient` — executes an MCP tool call to Webflow.
- **Configuration choices (interpreted):**
  - **Endpoint URL:** `https://mcp.webflow.com/mcp`
  - **Tool:** `data_cms_tool`
  - **Action:** `list_collection_items`
  - **Request:** `{ limit: 100 }`
  - **Collection ID:** templated from `Set Fields`: `{{ $json.collectionId }}`
  - **Context:** “Retrieving collection items for SEO analysis”
  - Retries: `maxTries: 3`, `waitBetweenTries: 5000ms`, `retryOnFail: true`
- **Key expressions:**
  - `actions` is built as a string expression containing JSON:
    - `list_collection_items.collection_id = {{ $json.collectionId }}`
- **Inputs/Outputs:** Input ← `Set Fields`; Output → `Loop Over Items`.
- **Edge cases / failures:**
  - **Auth errors** if MCP OAuth2 credentials aren’t configured/authorized for Webflow.
  - **404 Resource not found** if `collectionId` is wrong.
  - **400 Invalid input for 'actions'** if the action JSON is malformed (string interpolation issues).
  - Pagination not handled (only first 100 items). Large collections will require offset batching.
- **Version notes:** TypeVersion 1.

---

### Block 2.3 — Batching, AI Agent Analysis, and Output Parsing
**Overview:** Splits items into batches, submits each batch to the AI agent to propose SEO updates and audits, then validates the response into structured JSON.  
**Nodes involved:** `Loop Over Items`, `SEO Analysis Agent`, `Chat Model`, `Simple Memory`, `Structured Output Parser`, `Parser Model`

#### Node: Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — batch iterator.
- **Configuration choices:**
  - `batchSize = 5` (expression but constant)
  - `reset: false` (continues batches in the same run)
- **Inputs/Outputs:**
  - Input ← `Fetch CMS Items`
  - Output (batch stream) → `SEO Analysis Agent` (connected on the node’s batch output path)
- **Edge cases / failures:**
  - Too large batch size can cause AI token limits or MCP update timeouts later.
  - If input from MCP is not a flat list of items, batching may not behave as intended.
- **Version notes:** TypeVersion 3.

#### Node: SEO Analysis Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent that generates per-item optimization decisions.
- **Configuration choices:**
  - **Prompt input text:** `JSON.stringify($input.all().map(item => item.json))` (the whole batch as JSON array)
  - **System message:** SEO specialist instructions including:
    - Enforced target ranges:
      - title (`name`) length: from `Set Fields`
      - summary (`project-summary`) length: from `Set Fields`
    - Required checks:
      - services-rendered H1/H2 structure
      - missing alt text
      - word count in services-rendered
    - Output must be **JSON array only**, no prose/markdown
    - Must never modify: `id`, `slug`, `_archived`, `_draft`, `featured-project`, `color`
  - **Model binding:** uses `Chat Model` connection (OpenRouter GPT‑4o-mini).
  - **Output parser:** uses `Structured Output Parser` (autoFix enabled).
  - Resilience:
    - `retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000ms`
    - `continueOnFail: true` (workflow continues even if a batch fails)
- **Inputs/Outputs:**
  - Input ← `Loop Over Items`
  - Output → `Format for Update`
  - AI connections:
    - Language model ← `Chat Model`
    - Memory ← `Simple Memory`
    - Output parser ← `Structured Output Parser`
- **Edge cases / failures:**
  - LLM may return non-JSON; parser attempts auto-fix but can still fail or produce partial results.
  - If your Webflow field names differ, updates won’t apply (case-sensitive and hyphen-sensitive).
  - If items contain large rich text fields, token limits can be hit; consider reducing fields passed to agent.
- **Version notes:** TypeVersion 1.7.

#### Node: Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — LLM provider for the agent.
- **Configuration choices:**
  - Model: `openai/gpt-4o-mini`
  - Credentials: OpenRouter API (`OpenRouter account (Dummy)`)
- **Inputs/Outputs:** Provides `ai_languageModel` to `SEO Analysis Agent`.
- **Edge cases / failures:**
  - Invalid/expired OpenRouter key, model unavailable, rate limits.
- **Version notes:** TypeVersion 1.

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — conversational/batch memory.
- **Configuration choices:**
  - `sessionIdType: customKey`
  - `sessionKey = {{ $('Set Fields').item.json.collectionId }}` (ties memory to collection)
- **Inputs/Outputs:** Provides `ai_memory` to `SEO Analysis Agent`.
- **Edge cases / failures:**
  - If you reuse the same collectionId across different runs, memory may influence outputs unexpectedly; clear/disable if you need stateless behavior.
- **Version notes:** TypeVersion 1.3.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — validates and coerces agent output to a required schema.
- **Configuration choices:**
  - `schemaType: manual`
  - `autoFix: true`
  - Schema: array of objects containing `id`, `needsUpdate`, `status`, `updates`, `audit`, `issues`
- **Inputs/Outputs:**
  - Receives a parser model via `Parser Model`
  - Exposes `ai_outputParser` to `SEO Analysis Agent`
- **Edge cases / failures:**
  - If the agent output is too malformed, autoFix may still fail.
  - Schema uses `"project-summary"` key; must match your collection field API name.
- **Version notes:** TypeVersion 1.3.

#### Node: Parser Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — model used by the structured parser to repair/normalize output.
- **Configuration choices:** Model `openai/gpt-4o-mini`, OpenRouter credentials.
- **Inputs/Outputs:** `ai_languageModel` → `Structured Output Parser`.
- **Edge cases / failures:** Same as `Chat Model` (auth/rate limits/model availability).
- **Version notes:** TypeVersion 1.

---

### Block 2.4 — Prepare Updates, Update Items (MCP), Publish Items (MCP)
**Overview:** Filters only items marked `needsUpdate`, builds the Webflow update payload without null fields, updates CMS items, then publishes them live.  
**Nodes involved:** `Format for Update`, `Update Items`, `Publish Items`

#### Node: Format for Update
- **Type / role:** `n8n-nodes-base.code` — transforms agent results into Webflow MCP update format.
- **Configuration choices (interpreted):**
  - Reads agent output from `$input.first().json`, then uses `data.output || data` to handle agent output shape variations.
  - Filters: `output.filter(i => i.needsUpdate)`
  - Builds `items: [{ id, fieldData }]` and **removes null values**:
    - sets `fieldData.name` only if not null
    - sets `fieldData['project-summary']` only if not null
  - If no updates needed, returns `[]` (halts downstream update/publish/log for that batch).
  - **Hardcoded collectionId inside this node:** `collectionId: "xxxxxxxxxx"` (must match the real collection)
  - Adds: `itemIds`, `count`, `timestamp`
- **Inputs/Outputs:** Input ← `SEO Analysis Agent`; Output → `Update Items`.
- **Edge cases / failures:**
  - **Critical mismatch risk:** this node’s `collectionId` is independent of `Set Fields.collectionId`. If they differ, updates/publish may target the wrong collection or fail.
  - If the agent returns `needsUpdate: true` but `updates` are null, you may send empty `fieldData` objects (Webflow may reject).
  - If field names don’t exist in the collection, Webflow will reject or ignore.
- **Version notes:** TypeVersion 2.

#### Node: Update Items
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClient` — sends `update_collection_items` to Webflow.
- **Configuration choices:**
  - Endpoint: `https://mcp.webflow.com/mcp`
  - Tool: `data_cms_tool`
  - **Input mode:** JSON
  - JSON payload builds:
    - `collection_id: {{ $json.collectionId }}`
    - `request.items: {{ JSON.stringify($json.items) }}`
  - Timeout: `60000ms`
  - Resilience: retries (3 tries, 5s wait), `continueOnFail: true`
- **Inputs/Outputs:** Input ← `Format for Update`; Output → `Publish Items` (plus an unused second output).
- **Edge cases / failures:**
  - 400 Bad Request if request wrapper is wrong, or if `fieldData` includes invalid/null fields.
  - Timeout for large batches or slow Webflow responses.
  - Auth/permission errors if the OAuth scope/workspace access is missing.
- **Version notes:** TypeVersion 1.

#### Node: Publish Items
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClient` — publishes updated items to the live site.
- **Configuration choices:**
  - Action: `publish_collection_items`
  - `collection_id` and `itemIds` are taken specifically from the **Format for Update** node:
    - `collection_id: {{ $('Format for Update').item.json.collectionId }}`
    - `itemIds: {{ JSON.stringify($('Format for Update').item.json.itemIds) }}`
  - Resilience: retries, `continueOnFail: true`
- **Inputs/Outputs:** Input ← `Update Items`; Output → `Track Metrics`.
- **Edge cases / failures:**
  - **Parameter name is case-sensitive:** must be `itemIds` (not `itemIDs`).
  - If `Update Items` partially fails but workflow continues, publish may run with IDs that weren’t updated.
  - Collection ID mismatch between `Format for Update` and earlier retrieval can publish wrong items if misconfigured.
- **Version notes:** TypeVersion 1.

---

### Block 2.5 — Metrics + Google Sheets Logging
**Overview:** Computes run duration and builds a summary record, then appends it to a Google Sheet for tracking.  
**Nodes involved:** `Track Metrics`, `Save Summary`

#### Node: Track Metrics
- **Type / role:** `n8n-nodes-base.code` — computes execution duration and prepares logging fields.
- **Configuration choices (interpreted):**
  - Reads `formatData` from `$('Format for Update').item.json`
  - Calculates duration: `(now - timestamp)/1000`
  - Outputs:
    - `timestamp` in America/New_York locale string
    - `collectionId`
    - `itemsUpdated`, `itemsPublished` (both = `formatData.count`)
    - `executionDuration` as string
    - `postNames` list from `formatData.items[].fieldData.name`
    - `itemIds` list
- **Inputs/Outputs:** Input ← `Publish Items`; Output → `Save Summary`.
- **Edge cases / failures:**
  - If `Format for Update` returned nothing (no updates), downstream nodes won’t run anyway.
  - If some items had only summary updates (no name), `postNames` may contain blanks/undefined.
- **Version notes:** TypeVersion 2.

#### Node: Save Summary
- **Type / role:** `n8n-nodes-base.googleSheets` — appends a row to a Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - DocumentId: `xxxxxxx` (placeholder)
  - Sheet: by ID `2109721884`
  - Column mapping:
    - `Timestamp`, `Collection ID`, `Items Updated`, `itemsPublished`, `postNames`, `itemIds`
- **Credentials:** Google Sheets OAuth2 (`Google Sheets (Dummy Account)`).
- **Inputs/Outputs:** Input ← `Track Metrics`; no downstream nodes.
- **Edge cases / failures:**
  - OAuth expired / missing permissions.
  - Sheet/tab ID mismatch, document not found, or schema mismatch (column names).
- **Version notes:** TypeVersion 4.7.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual entry point | — | Set Fields | # 🚀 Webflow CMS SEO Bulk Optimizer; This workflow optimizes SEO metadata for Webflow CMS collections using AI. *(See full note content in Sticky Note1)* |
| Set Fields | n8n-nodes-base.set | Set collection + SEO constraints | Manual Trigger | Fetch CMS Items | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### Start Here: Configure Your Collection *(Sticky Note)* |
| Fetch CMS Items | @n8n/n8n-nodes-langchain.mcpClient | List CMS items via Webflow MCP | Set Fields | Loop Over Items | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### Fetch CMS Items Retrieves up to 100 items… Endpoint URL https://mcp.webflow.com/mcp *(Sticky Note5)* |
| Loop Over Items | n8n-nodes-base.splitInBatches | Batch items for processing | Fetch CMS Items | SEO Analysis Agent | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### Start Here: Configure Your Collection *(Sticky Note)* |
| SEO Analysis Agent | @n8n/n8n-nodes-langchain.agent | AI SEO optimization & audits | Loop Over Items | Format for Update | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### AI Agent: Customize for Your Fields *(Sticky Note2)* |
| Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM for agent | — (AI connection) | SEO Analysis Agent (AI) | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### AI Agent: Customize for Your Fields *(Sticky Note2)* |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversation memory keyed by collection | — (AI connection) | SEO Analysis Agent (AI) | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### AI Agent: Customize for Your Fields *(Sticky Note2)* |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce/repair strict JSON schema | — (AI connection) | SEO Analysis Agent (AI) | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### AI Agent: Customize for Your Fields *(Sticky Note2)* |
| Parser Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM used to auto-fix parsed output | — (AI connection) | Structured Output Parser (AI) | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### AI Agent: Customize for Your Fields *(Sticky Note2)* |
| Format for Update | n8n-nodes-base.code | Filter + build Webflow update payload | SEO Analysis Agent | Update Items | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### Update Collection ID Line 23 in this code node MUST match your collection ID. *(Sticky Note3)* |
| Update Items | @n8n/n8n-nodes-langchain.mcpClient | Update CMS items via MCP | Format for Update | Publish Items | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### Update Items Updates fieldData… Input Mode JSON… Timeout 60000ms *(Sticky Note6)* |
| Publish Items | @n8n/n8n-nodes-langchain.mcpClient | Publish updated items via MCP | Update Items | Track Metrics | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### Publish Items Publishes updated items… Parameter name itemIds *(Sticky Note7)* |
| Track Metrics | n8n-nodes-base.code | Compute duration + log fields | Publish Items | Save Summary | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)* |
| Save Summary | n8n-nodes-base.googleSheets | Append summary to Google Sheets | Track Metrics | — | # 🚀 Webflow CMS SEO Bulk Optimizer *(Sticky Note1)*; #### Summary All optimizations are logged to Google Sheets… optional *(Sticky Note4)* |
| Sticky Note1 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | (This node is itself the note content shown in the workflow canvas) |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation | — | — | (This node is itself the note content shown in the workflow canvas) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | (This node is itself the note content shown in the workflow canvas) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | (This node is itself the note content shown in the workflow canvas) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | (This node is itself the note content shown in the workflow canvas) |
| Sticky Note5 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | (This node is itself the note content shown in the workflow canvas) |
| Sticky Note6 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | (This node is itself the note content shown in the workflow canvas) |
| Sticky Note7 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | (This node is itself the note content shown in the workflow canvas) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Manual Trigger**
   - Type: *Manual Trigger*
   - Leave defaults.
3. **Add node: Set Fields**
   - Type: *Set*
   - Add fields:
     - `collectionId` (String): your Webflow collection ID
     - `minTitleLength` (Number): 50
     - `maxTitleLength` (Number): 60
     - `minSummaryLength` (Number): 120
     - `maxSummaryLength` (Number): 155
   - Connect: **Manual Trigger → Set Fields**
4. **Add node: Fetch CMS Items (MCP Client)**
   - Type: *MCP Client* (LangChain MCP)
   - **Endpoint URL:** `https://mcp.webflow.com/mcp`
   - **Credentials:** create/select **MCP OAuth2 API** credentials
     - Server URL: `https://mcp.webflow.com/mcp`
     - Save and authorize access to Webflow workspaces/sites
   - **Tool:** `data_cms_tool`
   - Configure parameters to perform `list_collection_items`:
     - `collection_id`: use expression `{{ $json.collectionId }}`
     - `request.limit`: `100`
     - `context`: “Retrieving collection items for SEO analysis”
   - Connect: **Set Fields → Fetch CMS Items**
5. **Add node: Loop Over Items**
   - Type: *Split In Batches*
   - Batch size: `5` (adjust to 5–20 depending on safety vs speed)
   - Connect: **Fetch CMS Items → Loop Over Items**
6. **Add node: Chat Model**
   - Type: *OpenRouter Chat Model*
   - Model: `openai/gpt-4o-mini`
   - Credentials: configure OpenRouter API key.
7. **Add node: Parser Model**
   - Type: *OpenRouter Chat Model*
   - Model: `openai/gpt-4o-mini`
   - Credentials: same OpenRouter credential.
8. **Add node: Structured Output Parser**
   - Type: *Structured Output Parser*
   - Enable **Auto-fix**
   - Provide a manual schema matching:
     - Array of objects with `id`, `needsUpdate`, `status`, `updates.name`, `updates.project-summary`, `audit`, `issues`
   - Connect AI model: **Parser Model → Structured Output Parser** (AI language model connection).
9. **Add node: Simple Memory**
   - Type: *Memory Buffer Window*
   - Session mode: custom key
   - Session key expression: `{{ $('Set Fields').item.json.collectionId }}`
10. **Add node: SEO Analysis Agent**
    - Type: *AI Agent*
    - Input text expression: `{{ JSON.stringify($input.all().map(item => item.json)) }}`
    - System message: include the constraints and required **JSON-only** output; reference limits using:
      - `{{ $('Set Fields').first().json.minTitleLength }}` etc.
    - Connect AI components:
      - **Chat Model → SEO Analysis Agent** (AI language model)
      - **Simple Memory → SEO Analysis Agent** (AI memory)
      - **Structured Output Parser → SEO Analysis Agent** (AI output parser)
    - Connect main flow: **Loop Over Items → SEO Analysis Agent**
11. **Add node: Format for Update**
    - Type: *Code*
    - Implement logic to:
      - read agent output
      - keep only `needsUpdate === true`
      - build `{ id, fieldData }` and **omit null fields**
      - output `collectionId`, `items`, `itemIds`, `count`, `timestamp`
    - Important: set `collectionId` here to the **same value as Set Fields** (or better: reference it via expression to avoid drift).
    - Connect: **SEO Analysis Agent → Format for Update**
12. **Add node: Update Items (MCP Client)**
    - Type: *MCP Client*
    - Endpoint: `https://mcp.webflow.com/mcp`
    - Tool: `data_cms_tool`
    - **Input mode:** JSON
    - Action: `update_collection_items` with:
      - `collection_id: {{ $json.collectionId }}`
      - `request.items: {{ JSON.stringify($json.items) }}`
    - Timeout: 60000 ms
    - Connect: **Format for Update → Update Items**
13. **Add node: Publish Items (MCP Client)**
    - Type: *MCP Client*
    - Endpoint: `https://mcp.webflow.com/mcp`
    - Tool: `data_cms_tool`
    - **Input mode:** JSON
    - Action: `publish_collection_items` with:
      - `collection_id`: reference the same collection id you used for update
      - `request.itemIds`: the array of updated IDs (case-sensitive: `itemIds`)
    - Connect: **Update Items → Publish Items**
14. **Add node: Track Metrics**
    - Type: *Code*
    - Compute:
      - duration based on the timestamp created in Format for Update
      - `postNames`, `itemIds`, counts
    - Connect: **Publish Items → Track Metrics**
15. **Add node: Save Summary (Google Sheets)**
    - Type: *Google Sheets*
    - Credentials: Google Sheets OAuth2
    - Operation: Append
    - Select Spreadsheet (Document) and Sheet tab
    - Map columns: Timestamp, Collection ID, Items Updated, itemsPublished, postNames, itemIds
    - Connect: **Track Metrics → Save Summary**
16. **(Optional) Add Sticky Notes** to document configuration and common errors (recommended for maintainability).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Webflow MCP Endpoint URL: `https://mcp.webflow.com/mcp` | Used by all MCP Client nodes |
| MCP credential setup: use **MCP OAuth2 API** with Server URL `https://mcp.webflow.com/mcp`, then authorize Webflow workspaces/sites | Mentioned in Sticky Note1 |
| Common MCP errors: 404 wrong collection ID; 400 malformed `actions`/missing `request`; field name mismatches; `itemIds` case sensitivity | Mentioned in Sticky Note1 |
| If field names differ, update in three places: agent system prompt, structured parser schema, and Format for Update code | Mentioned in Sticky Note1 and Sticky Note2 |
| Google Sheets logging is optional; you can remove the Save Summary node | Mentioned in Sticky Note4 |