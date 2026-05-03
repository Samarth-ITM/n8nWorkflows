Generate photo-based construction cost estimates with GPT-4 Vision and DDC CWICR

https://n8nworkflows.xyz/workflows/generate-photo-based-construction-cost-estimates-with-gpt-4-vision-and-ddc-cwicr-12175


# Generate photo-based construction cost estimates with GPT-4 Vision and DDC CWICR

## 1. Workflow Overview

**Workflow name:** Photo Cost Estimate Pro v2.0  
**Purpose:** Accept a user-uploaded construction photo, analyze it with GPT-4 Vision to identify elements and approximate quantities, decompose the scene into granular construction work items, price each work item by retrieving matching rates from a **Qdrant vector database (DDC CWICR)**, validate results, and generate a **multi-language HTML cost estimate report** returned to the user via the n8n Form Trigger.

**Target use cases**
- Fast preliminary cost estimates from a single photo (bathroom/kitchen/rooms/exterior, etc.).
- Multi-region, multi-language estimation (9 locales) based on region-specific DDC CWICR collections.
- Producing a “client-ready” HTML estimate summary with work phases, pricing quality indicators, and charts.

### 1.1 Logical blocks
1. **Input Reception & Localization Setup**  
   Form intake → extract photo + dropdown selections → map language/region → select Qdrant collection → validate photo.
2. **Stage 1: Vision Photo Analysis (GPT-4 Vision)**  
   Prompt construction → vision inference → robust JSON parsing → structured “elements/fixtures/dimensions”.
3. **Stage 4: Work Decomposition (GPT-4)**  
   Prompt with mapping rules → LLM decomposes into work items → parse, normalize, and assign IDs/units/sequences.
4. **Stage 5: Pricing Loop via Qdrant Vector Search**  
   Prepare loop items → batch iteration → store/restore current item → embeddings + vector search → parse/score → compute costs → accumulate results.
5. **Stage 7.5: Aggregate & Validate**  
   Aggregate accumulated works → group into phases → compute totals and quality stats → run validation rules.
6. **Stage 9: HTML Report + Final Response**  
   Generate multilingual HTML report → return HTML file as binary to the form response.

---

## 2. Block-by-Block Analysis

### Block 1 — Photo Upload & Config

**Overview:** Receives the user submission (photo + dropdowns), extracts values, determines the language/region configuration, selects the correct Qdrant collection, and stops early if no photo is provided.

**Nodes involved:**
- Photo Upload Form
- Extract Input
- Configure Language
- Has Photo?
- Error No Photo

#### Node: **Photo Upload Form**
- **Type / role:** `Form Trigger` — public web form entry point.
- **Key configuration:**
  - Title: “📸 Photo Cost Estimate Pro v2”
  - **Fields:**
    - File upload (single image; accepts `.jpg,.jpeg,.png,.webp`; required)
    - Dropdown: “🌍 Region & Language” (9 options; required)
    - Dropdown: “🏗️ Work Type” (New Construction / Renovation / Repair / Auto-detect; required)
    - Textarea: optional description
  - **Response mode:** `lastNode` (the final node’s output becomes the HTTP response).
  - **Webhook ID:** `photo-estimate-pro-v3`
- **Input / output connections:**
  - Output → **Extract Input**
- **Edge cases / failures:**
  - Users submit unsupported file types or empty file (handled later by “Has Photo?”).
  - Form field label variations: later code node attempts fallback names.
- **Sticky note (applies):**
  - “# 📸 Photo Cost Estimate Pro v2 / Multi-stage AI decomposition pipeline”

#### Node: **Extract Input**
- **Type / role:** `Code` — normalizes form fields and extracts image as base64.
- **Key configuration choices:**
  - Detects region field and work type field using both emoji label and plain label fallbacks.
  - Maps 9 languages to a language code (`DE/EN/RU/ES/FR/PT/ZH/AR/HI`).
  - Converts selected “Work Type” to internal values:
    - `new_construction`, `renovation`, `repair`, or `auto`
  - Extracts first image binary and sets:
    - `photo_base64`, `photo_mime_type`, `has_photo`
- **Outputs:** A single JSON object with normalized keys:
  - `language_code`, `selected_region`, `work_type`, `user_description`, `photo_base64`, `has_photo`, etc.
- **Connections:**
  - Input ← Photo Upload Form
  - Output → Configure Language
- **Edge cases / failures:**
  - If binary key name differs, it iterates all binary keys and picks the first `image/*`.
  - `has_photo` is a heuristic (`base64.length > 100`); tiny images or unexpected encoding could fail this check.

#### Node: **Configure Language**
- **Type / role:** `Code` — selects city/currency/locale and **Qdrant collection name** by language code.
- **Key configuration choices:**
  - Hardcoded config per language:
    - City (pricing level)
    - Qdrant collection (e.g., `FR_PARIS_workitems_costs_resources_EMBEDDINGS_3072_DDC_CWICR`)
    - Currency and locale
    - `system_prompt_lang` (instruction to answer in language)
    - `search_lang` (language used for vector search queries)
- **Outputs:** Adds `language_config`, `qdrant_collection`, `city`, `currency_symbol`, etc.
- **Connections:**
  - Input ← Extract Input
  - Output → Has Photo?
- **Edge cases / failures:**
  - Unknown language code falls back to EN configuration.
  - If Qdrant collection does not exist, downstream vector search fails.

#### Node: **Has Photo?**
- **Type / role:** `IF` — gatekeeper.
- **Condition:** `$json.has_photo === true`
- **Connections:**
  - True → STAGE 1 Vision Prompt
  - False → Error No Photo
- **Edge cases / failures:**
  - False negatives if base64 extraction fails or `has_photo` heuristic is too strict.

#### Node: **Error No Photo**
- **Type / role:** `Code` — returns a minimal HTML error page payload in JSON.
- **Connections:** (terminal branch for “no photo” path)
- **Behavior:** Produces `success:false`, `error:true`, and `html_content` (HTML string).
- **Edge cases / failures:**
  - Because the Form Trigger responds from the last node on the *executed path*, this works if this branch ends execution and is the “last node”. If later nodes are executed in parallel paths (not the case here), response ambiguity could occur.

**Sticky notes for this block:**
- “## Block 1: Photo Upload & Config … Supported formats: JPG, PNG, WebP”

---

### Block 2 — STAGE 1: Vision Analysis

**Overview:** Builds a strict JSON-only vision prompt, runs GPT-4 Vision, then parses and normalizes the response into structured elements, fixtures, and dimension estimates.

**Nodes involved:**
- STAGE 1 Vision Prompt
- STAGE 1 Analyze Photo
- GPT-4 Vision
- Parse STAGE 1

#### Node: **STAGE 1 Vision Prompt**
- **Type / role:** `Set` — constructs `chatInput`.
- **Key configuration choices:**
  - Prompt instructs:
    - Identify room type first
    - List visible elements with materials/finishes
    - Estimate dimensions using reference objects
    - Output **ONLY valid JSON** with a fixed schema
  - Injects:
    - `$json.system_prompt_lang` (language constraint)
    - Optional `$json.user_description`
- **Connections:**
  - Input ← Has Photo? (true)
  - Output → STAGE 1 Analyze Photo
- **Edge cases / failures:**
  - If the model returns non-JSON, parser tries to salvage it later.

#### Node: **STAGE 1 Analyze Photo**
- **Type / role:** `LangChain Chain LLM` (`chainLlm`) — runs prompt as a message.
- **Key configuration choices:**
  - Message content: `{{$json.chatInput}}`
  - Uses a separate LLM node as its language model input.
- **Connections:**
  - Input ← STAGE 1 Vision Prompt
  - Output → Parse STAGE 1
  - AI model input ← GPT-4 Vision
- **Edge cases / failures:**
  - If the LLM node is misconfigured or credentials invalid, chain fails.
  - Vision input handling: this workflow extracts base64 but does not explicitly show attaching image content to the chain in node params; depending on n8n LangChain node behavior/version, you may need to pass image data explicitly (see reproduction section).

#### Node: **GPT-4 Vision**
- **Type / role:** `OpenAI Chat Model` (`lmChatOpenAi`) — provides the model for Stage 1.
- **Key configuration choices:**
  - Model: `chatgpt-4o-latest`
  - `maxTokens: 4000`, `temperature: 0.2`
  - Credential: “OpenAi account WS”
- **Connections:**
  - Output (as AI language model) → STAGE 1 Analyze Photo
- **Edge cases / failures:**
  - OpenAI auth/quota errors
  - Model availability changes (`chatgpt-4o-latest` alias may shift)
  - Token limit issues if prompts grow (large element lists)

#### Node: **Parse STAGE 1**
- **Type / role:** `Code` — robust JSON extraction and normalization.
- **Key configuration choices:**
  - Extracts content from `text/content/response`
  - Attempts to parse:
    - ```json fenced blocks
    - generic ``` blocks
    - first `{ ... }` substring
  - If parsing fails, uses defaults.
  - Merges config from **Configure Language** node into output.
  - Sets:
    - `work_type_detected` (from vision output or selected work type or default)
- **Connections:**
  - Input ← STAGE 1 Analyze Photo
  - Output → STAGE 4 Decompose Prompt
- **Edge cases / failures:**
  - Partial JSON / trailing commas → parse failure; falls back silently (logs to console only).
  - Wrong schema from model → downstream decomposition may be low quality.

**Sticky notes for this block:**
- “## Block 2: STAGE 1 - Vision Analysis … Output: Structured JSON with elements”
- “## 🧠 AI Models Used … GPT-4 Vision / GPT-4 … Can be replaced with Claude/Gemini/OpenRouter”

---

### Block 3 — STAGE 4: Work Decomposition

**Overview:** Converts detected elements into a detailed list of construction work items, ensuring preparation/finishing/MEP phases and renovation demolition logic, and creates vector-search-friendly queries.

**Nodes involved:**
- STAGE 4 Decompose Prompt
- STAGE 4 Decompose LLM
- GPT-4 Decompose
- Parse STAGE 4

#### Node: **STAGE 4 Decompose Prompt**
- **Type / role:** `Set` — constructs decomposition prompt (`chatInput`).
- **Key configuration choices:**
  - Embeds Stage 1 results (room type, description, dimensions, elements, fixtures).
  - Includes explicit mapping rules by room/category (bathroom/kitchen/living room, etc.).
  - Critical rules:
    - Never empty; minimum 3 works per element
    - Include preparation + finishing
    - Renovation requires demolition first
    - Units correctness (m²/m/pcs)
    - `search_query` must be in `search_lang`
  - Requires JSON-only output:
    - `work_items[]` with `work_name`, `search_query`, quantities, units, flags.
- **Connections:**
  - Input ← Parse STAGE 1
  - Output → STAGE 4 Decompose LLM
- **Edge cases / failures:**
  - Very large `elements` arrays can inflate prompt and token usage.
  - LLM may return duplicates or unrealistic quantities—handled only lightly later.

#### Node: **STAGE 4 Decompose LLM**
- **Type / role:** `LangChain Chain LLM` — runs decomposition prompt.
- **Connections:**
  - Input ← STAGE 4 Decompose Prompt
  - Output → Parse STAGE 4
  - AI model input ← GPT-4 Decompose

#### Node: **GPT-4 Decompose**
- **Type / role:** `OpenAI Chat Model`
- **Key configuration:**
  - Model: `chatgpt-4o-latest`
  - `maxTokens: 8000`, `temperature: 0.3`
- **Edge cases / failures:**
  - Higher max tokens increases cost; may still truncate if the model or account imposes limits.

#### Node: **Parse STAGE 4**
- **Type / role:** `Code` — parses and normalizes work items.
- **Key configuration choices:**
  - JSON extraction similar to Stage 1 parser.
  - Enriches each work item with:
    - `work_id` = `W001`, `W002`, …
    - Ensures `search_query` is non-trivial
    - `project_quantity` numeric, default 1
    - Default unit `m²`
  - Sorts by `work_sequence`
  - Sets `stage4_success: workItems.length >= 3`
- **Connections:**
  - Input ← STAGE 4 Decompose LLM
  - Output → Prepare Works
- **Edge cases / failures:**
  - If parsing fails, `work_items` may be empty → later block returns “No work items generated”.

**Sticky notes for this block:**
- “## Block 3: STAGE 4 - Work Decomposition … Rules: Minimum 3 works per element … renovation demolition first”

---

### Block 4 — STAGE 5: Pricing Loop (Vector Search + Scoring + Costing)

**Overview:** Iterates through each work item, queries Qdrant using embeddings, parses candidate rate documents, scores match quality, computes total costs and labor hours, and accumulates results in workflow static data.

**Nodes involved:**
- Prepare Works
- Loop Works
- Store Work Data
- Wait
- Restore Work Data
- Embeddings
- Vector Search
- STAGE 5 Parse & Score
- Accumulate

#### Node: **Prepare Works**
- **Type / role:** `Code` — initializes loop and stores configuration in static data.
- **Key configuration choices:**
  - Stores in `$getWorkflowStaticData('global')`:
    - `photo_config` (language/currency/city/room_type/dimensions/elements/fixtures)
    - resets `work_results = []`
  - If no works: returns a single item `{ _no_works: true }`
  - Otherwise maps each work item to an item stream for batching, including:
    - `qdrant_collection`, `currency_symbol`, `expected_unit`, etc.
- **Connections:**
  - Input ← Parse STAGE 4
  - Output → Loop Works
- **Edge cases / failures:**
  - Static data is global: concurrent executions can overwrite each other unless n8n is configured to isolate executions (this is a significant concurrency risk in multi-user scenarios).

#### Node: **Loop Works**
- **Type / role:** `Split In Batches` — iterates work items.
- **Key configuration:**
  - `reset: false` (continues batches until exhausted)
- **Connections:**
  - Input ← Prepare Works
  - Output 1 → Store Work Data (batch item path)
  - Output 0 → STAGE 7.5 Aggregate & Validate (when no more items)
- **Edge cases / failures:**
  - If loop never returns to `Loop Works` (it does via Accumulate), it would process only one item.

#### Node: **Store Work Data**
- **Type / role:** `Code` — saves the current work item into static data.
- **Key configuration:** `staticData.currentWork = work`
- **Connections:**
  - Input ← Loop Works
  - Output → Wait
- **Why it exists:** The workflow uses a Wait node; static storage ensures the current item survives the wait/resume boundary.

#### Node: **Wait**
- **Type / role:** `Wait` — small delay/throttling.
- **Key configuration:** `amount: 0.3` (seconds)
- **Webhook ID:** `wait-photo-v3`
- **Connections:**
  - Input ← Store Work Data
  - Output → Restore Work Data
- **Edge cases / failures:**
  - In high throughput, this slows overall execution.
  - Wait/resume requires n8n to be able to persist executions correctly (DB-backed execution storage).

#### Node: **Restore Work Data**
- **Type / role:** `Code` — restores current work from static data.
- **Connections:** Output → Vector Search
- **Edge cases / failures:**
  - If static data was overwritten by another execution, wrong work item may be priced (again, concurrency risk).

#### Node: **Embeddings**
- **Type / role:** `OpenAI Embeddings` (`embeddingsOpenAi`) — generates embeddings for the vector query.
- **Key configuration:**
  - Model: `text-embedding-3-large`
  - Dimensions: `3072` (must match Qdrant collection embedding dimensionality)
- **Connections:**
  - Output (AI embedding) → Vector Search
- **Edge cases / failures:**
  - If Qdrant collection embeddings were built with a different model/dimension, search quality collapses or errors.

#### Node: **Vector Search**
- **Type / role:** `Qdrant Vector Store` (`vectorStoreQdrant`) — retrieves top-K rate documents.
- **Key configuration choices:**
  - Mode: `load`
  - `topK: 5`
  - Prompt/query: `{{$json.search_query || $json.work_name}}`
  - Qdrant collection dynamically selected: `{{$json.qdrant_collection}}`
  - Payload key: `contentPayloadKey: content` (expects content in that payload key)
  - Credential: “QdrantApi account 2”
- **Connections:**
  - Inputs: main from Restore Work Data; embeddings from Embeddings node
  - Output → STAGE 5 Parse & Score
- **Edge cases / failures:**
  - Qdrant auth/URL errors
  - Collection missing
  - Payload schema mismatch (document stored differently than parser expects)

#### Node: **STAGE 5 Parse & Score**
- **Type / role:** `Code` — parses returned docs, extracts costs/resources, scores quality, and computes totals.
- **Key configuration choices:**
  - **Critical fix:** explicitly extracts text from nested `document.pageContent` / `document.content`.
  - Extracts:
    - `total_cost_position` from patterns like “Total cost: 3127.16 EUR”
    - `rate_code`, `rate_name`, `rate_unit` from metadata or content
    - “RESOURCES:” section parsing into resource lines and classifies resource type:
      - labor vs machine vs material using multilingual keyword patterns
  - Quality scoring (0–100) based on:
    - price presence, resource count, unit match, keyword overlap, vector score
  - Cost calculation:
    - Handles rate units like `100 m²` by using `unit_divisor`
    - `total_cost = (project_quantity / unit_divisor) * unit_cost`
    - Labor hours scaled similarly
  - Resource cost fallback:
    - If resource costs are zero but total cost > 0, splits into 35/55/10 labor/material/machine
- **Connections:**
  - Input ← Vector Search (receives multiple items via `$input.all()`)
  - Output → Accumulate
- **Edge cases / failures:**
  - If DDC CWICR document formatting changes, regex extraction may fail → unit_cost becomes 0.
  - Multi-language parsing is heuristic; some locales may produce lower match.
  - If Qdrant returns fewer/empty results, it returns NOT_FOUND object with zero costs.

#### Node: **Accumulate**
- **Type / role:** `Code` — pushes priced work item into `staticData.work_results`.
- **Connections:**
  - Input ← STAGE 5 Parse & Score
  - Output → Loop Works (to request next batch item)
- **Edge cases / failures:**
  - Same concurrency problem: global static array shared across executions.

**Sticky notes for this block:**
- “## Block 4: STAGE 5 - Pricing Loop … Database: DDC CWICR 700,000+ construction rates”
- “### 🔍 Vector Search … 3072-dim embeddings … Top 5 matches”
- “### ⚡ STAGE 5.2 Parse & Score … FIXED: Correct document extraction … Extract costs & resources”
- “### 📥 Vector Database Setup … Install Qdrant … Upload DDC CWICR dataset … [GitHub](https://github.com/datadrivenconstruction)”

---

### Block 5 — STAGE 7.5 Aggregate & Validate

**Overview:** After the loop finishes, aggregates all priced works, groups them into phases, calculates totals and quality stats, and runs validation checks before report generation.

**Nodes involved:**
- STAGE 7.5 Aggregate & Validate

#### Node: **STAGE 7.5 Aggregate & Validate**
- **Type / role:** `Code`
- **Key configuration choices:**
  - Reads `staticData.photo_config` and `staticData.work_results`
  - Computes:
    - Grand totals (cost, labor hours, breakdown totals)
    - Quality distribution (high/medium/low/not_found)
    - `found_percent`
  - Groups works by `work_category` into ordered phases: `PREPARATION`, `MAIN`, `FINISHING`, `MEP`
  - Produces `by_phase` structure expected by HTML generator.
  - Validation checks:
    1. At least 3 work items
    2. Found rate % >= 50%
    3. Zero-cost items <= 30%
    4. Renovation must include demolition-like work (keyword-based)
  - Clears static data at the end:
    - `staticData.work_results = []`
    - `staticData.photo_config = null`
- **Connections:**
  - Input ← Loop Works (when batches exhausted)
  - Output → STAGE 9 HTML Report
- **Edge cases / failures:**
  - If loop did not accumulate correctly, works may be empty.
  - Clearing static data helps, but does not fully solve concurrency issues if overlapping executions occur mid-loop.

**Sticky note for this block:**
- “## ✅ STAGE 7.5: Validation … Checks performed … Groups works by category … PREPARATION/MAIN/FINISHING/MEP”

---

### Block 6 — STAGE 9 Report Generation & Form Response

**Overview:** Builds a professional multilingual HTML report, then returns it to the user as an HTML file via the form’s last-node response.

**Nodes involved:**
- STAGE 9 HTML Report
- Final HTML Output

#### Node: **STAGE 9 HTML Report**
- **Type / role:** `Code` — generates a full HTML document.
- **Key configuration choices:**
  - Supports 9 languages via a translation dictionary keyed by `language_code`.
  - Formats currency via `Intl.NumberFormat(locale, { style:'currency', currency })`.
  - Report contents:
    - Header with project name, pricing level/city, timestamp
    - Photo analysis summary (room type, size, work type, elements/fixtures)
    - Validation bar (passed vs issues list)
    - KPI cards (total, hours, days)
    - Cost structure bars + phase-by-phase charts + timeline + treemap
    - Expand/collapse table with phases → types → works → resources
    - Quality dots for each work item
    - Rate code links use:
      - `https://openconstructionestimate.com/all-estimates/?utm=OCE`
  - Includes branding links:
    - `https://github.com/datadrivenconstruction/OpenConstructionEstimate-DDC-CWICR`
    - `https://datadrivenconstruction.io/`
- **Connections:**
  - Input ← STAGE 7.5 Aggregate & Validate
  - Output → Final HTML Output
- **Edge cases / failures:**
  - Large result sets produce very large HTML (may exceed browser/memory limits).
  - If `Intl` fails for some locale/currency combination, falls back to manual formatting.

#### Node: **Final HTML Output**
- **Type / role:** `Code` — returns HTML as binary “file download” and JSON summary.
- **Key configuration choices:**
  - `binary.data`:
    - base64 HTML
    - mimeType: `text/html`
    - fileName: `PhotoEstimate_v2.html`
  - JSON: `success`, `message`, `summary`, `validation`
  - Works with Form Trigger `responseMode: lastNode`.
- **Connections:**
  - Input ← STAGE 9 HTML Report
- **Edge cases / failures:**
  - If `html_content` missing, returns a minimal error HTML.
  - Depending on how the form client handles binary responses, user may see a download rather than rendered HTML.

**Sticky notes for this block:**
- “## Block 6: Report Generation … Professional design … 9 language support … Output: HTML + XLS files”  
  (Note: the implementation returns HTML; no XLS generation node exists in this workflow.)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Photo Upload Form | n8n-nodes-base.formTrigger | Entry point web form (photo + settings) | — | Extract Input | # 📸 Photo Cost Estimate Pro v2 / Multi-stage AI decomposition pipeline |
| Extract Input | n8n-nodes-base.code | Normalize form fields; extract image base64; map language/work type | Photo Upload Form | Configure Language | ## Block 1: Photo Upload & Config … Supported formats: JPG, PNG, WebP |
| Configure Language | n8n-nodes-base.code | Select locale/currency/city and Qdrant collection | Extract Input | Has Photo? | ## Block 1: Photo Upload & Config … Supported formats: JPG, PNG, WebP |
| Has Photo? | n8n-nodes-base.if | Validate photo presence | Configure Language | STAGE 1 Vision Prompt; Error No Photo | ## Block 1: Photo Upload & Config … Supported formats: JPG, PNG, WebP |
| Error No Photo | n8n-nodes-base.code | Early exit with HTML error | Has Photo? (false) | — | ## Block 1: Photo Upload & Config … Supported formats: JPG, PNG, WebP |
| STAGE 1 Vision Prompt | n8n-nodes-base.set | Build strict JSON-only vision prompt | Has Photo? (true) | STAGE 1 Analyze Photo | ## Block 2: STAGE 1 - Vision Analysis … Output: Structured JSON with elements |
| STAGE 1 Analyze Photo | @n8n/n8n-nodes-langchain.chainLlm | Run Stage 1 via LangChain chain | STAGE 1 Vision Prompt | Parse STAGE 1 | ## Block 2: STAGE 1 - Vision Analysis … Output: Structured JSON with elements |
| GPT-4 Vision | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for vision analysis | — | STAGE 1 Analyze Photo (ai_languageModel) | ## 🧠 AI Models Used … GPT-4 Vision / GPT-4 … Can be replaced with Claude/Gemini/OpenRouter |
| Parse STAGE 1 | n8n-nodes-base.code | Parse/sanitize Stage 1 JSON; normalize outputs | STAGE 1 Analyze Photo | STAGE 4 Decompose Prompt | ## Block 2: STAGE 1 - Vision Analysis … Output: Structured JSON with elements |
| STAGE 4 Decompose Prompt | n8n-nodes-base.set | Build JSON-only decomposition prompt + mapping rules | Parse STAGE 1 | STAGE 4 Decompose LLM | ## Block 3: STAGE 4 - Work Decomposition … renovation demolition first |
| STAGE 4 Decompose LLM | @n8n/n8n-nodes-langchain.chainLlm | Run decomposition chain | STAGE 4 Decompose Prompt | Parse STAGE 4 | ## Block 3: STAGE 4 - Work Decomposition … renovation demolition first |
| GPT-4 Decompose | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for decomposition | — | STAGE 4 Decompose LLM (ai_languageModel) | ## 🧠 AI Models Used … GPT-4 Vision / GPT-4 … Can be replaced with Claude/Gemini/OpenRouter |
| Parse STAGE 4 | n8n-nodes-base.code | Parse and normalize work items; add IDs; ensure queries | STAGE 4 Decompose LLM | Prepare Works | ## Block 3: STAGE 4 - Work Decomposition … renovation demolition first |
| Prepare Works | n8n-nodes-base.code | Store config in staticData; expand works into items | Parse STAGE 4 | Loop Works | ## Block 4: STAGE 5 - Pricing Loop … Database: DDC CWICR 700,000+ construction rates |
| Loop Works | n8n-nodes-base.splitInBatches | Iterate through work items; route to aggregate when done | Prepare Works; Accumulate | Store Work Data; STAGE 7.5 Aggregate & Validate | ## Block 4: STAGE 5 - Pricing Loop … Database: DDC CWICR 700,000+ construction rates |
| Store Work Data | n8n-nodes-base.code | Save current work item into staticData | Loop Works | Wait | ## Block 4: STAGE 5 - Pricing Loop … Database: DDC CWICR 700,000+ construction rates |
| Wait | n8n-nodes-base.wait | Throttle loop execution (0.3s) | Store Work Data | Restore Work Data | ## Block 4: STAGE 5 - Pricing Loop … Database: DDC CWICR 700,000+ construction rates |
| Restore Work Data | n8n-nodes-base.code | Restore current work item from staticData | Wait | Vector Search | ## Block 4: STAGE 5 - Pricing Loop … Database: DDC CWICR 700,000+ construction rates |
| Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Create 3072-d embeddings for Qdrant query | — | Vector Search (ai_embedding) | ### 🔍 Vector Search … 3072-dim embeddings … Top 5 matches per query |
| Vector Search | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Retrieve topK matching DDC CWICR rate docs | Restore Work Data; Embeddings | STAGE 5 Parse & Score | ### 🔍 Vector Search … 3072-dim embeddings … Top 5 matches per query |
| STAGE 5 Parse & Score | n8n-nodes-base.code | Parse docs; extract costs/resources; score; compute totals | Vector Search | Accumulate | ### ⚡ STAGE 5.2 Parse & Score … FIXED: Correct document extraction … Extract costs & resources |
| Accumulate | n8n-nodes-base.code | Append priced work to staticData.work_results | STAGE 5 Parse & Score | Loop Works | ### ⚡ STAGE 5.2 Parse & Score … FIXED: Correct document extraction … Extract costs & resources |
| STAGE 7.5 Aggregate & Validate | n8n-nodes-base.code | Aggregate all works; group by phases; validate | Loop Works (done) | STAGE 9 HTML Report | ## ✅ STAGE 7.5: Validation … Checks performed … PREPARATION/MAIN/FINISHING/MEP |
| STAGE 9 HTML Report | n8n-nodes-base.code | Generate multilingual HTML report | STAGE 7.5 Aggregate & Validate | Final HTML Output | ## Block 6: Report Generation … Professional design … 9 language support … Output: HTML + XLS files |
| Final HTML Output | n8n-nodes-base.code | Return HTML as binary file + summary JSON | STAGE 9 HTML Report | — | ⭐ If you find our tools helpful… star our repository: https://github.com/datadrivenconstruction/OpenConstructionEstimate-DDC-CWICR |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: `Photo Cost Estimate Pro v2.0` (or your preferred name)
   - Execution order: `v1` (matches the provided workflow setting)

2. **Add “Photo Upload Form” (Form Trigger)**
   - Node type: **Form Trigger**
   - Form title: `📸 Photo Cost Estimate Pro v2`
   - Description: include the provided multi-stage note (optional)
   - Add fields:
     1) File: label `📷 Upload Photo`, required, single file, accept `.jpg,.jpeg,.png,.webp`  
     2) Dropdown: label `🌍 Region & Language`, required, 9 options (Berlin/Toronto/.../Mumbai)  
     3) Dropdown: label `🏗️ Work Type`, required, options (New Construction / Renovation / Repair / Auto-detect)  
     4) Textarea: label `📝 Description (optional)`  
   - Response mode: **Last node**

3. **Add “Extract Input” (Code)**
   - Node type: **Code**
   - Implement:
     - Read `$input.first()` form payload
     - Map the chosen “Region & Language” string to a language code
     - Convert “Work Type” to `new_construction/renovation/repair/auto`
     - Extract the first `image/*` from `input.binary` and set `photo_base64` and `photo_mime_type`
     - Compute `has_photo`
   - Connect: **Photo Upload Form → Extract Input**

4. **Add “Configure Language” (Code)**
   - Node type: **Code**
   - Create a dictionary keyed by `language_code` with:
     - `city`, `currency`, `currencySymbol`, `locale`
     - `systemPromptLang` (language instruction)
     - `searchLang`
     - `vectorDb` (the Qdrant collection name)
   - Output fields must include at least:
     - `qdrant_collection`, `city`, `currency`, `currency_symbol`, `locale`, `system_prompt_lang`, `search_lang`
   - Connect: **Extract Input → Configure Language**

5. **Add “Has Photo?” (IF)**
   - Node type: **IF**
   - Condition: boolean equals `{{$json.has_photo}}` → `true`
   - Connect: **Configure Language → Has Photo?**

6. **Add “Error No Photo” (Code)**
   - Node type: **Code**
   - Return:
     - `success:false`, `error:true`, `message:'❌ No photo provided'`
     - `html_content` with a minimal HTML error page
   - Connect: **Has Photo? (false) → Error No Photo**

7. **Add “STAGE 1 Vision Prompt” (Set)**
   - Node type: **Set**
   - Create a string field named `chatInput`:
     - Include `{{$json.system_prompt_lang}}`
     - Instruct GPT to analyze the photo and output JSON only with the fixed schema
     - Append user note if present
   - Connect: **Has Photo? (true) → STAGE 1 Vision Prompt**

8. **Add GPT model node “GPT-4 Vision”**
   - Node type: **OpenAI Chat Model** (n8n LangChain)
   - Credential: configure OpenAI API credential
   - Model: `chatgpt-4o-latest` (or another vision-capable model)
   - Options: `maxTokens: 4000`, `temperature: 0.2`

9. **Add “STAGE 1 Analyze Photo” (LangChain Chain LLM)**
   - Node type: **Chain LLM**
   - Messages: one user message = `{{$json.chatInput}}`
   - Set its **AI language model** input to the “GPT-4 Vision” node.
   - **Important (vision input):** ensure the node actually receives the uploaded image. Depending on your n8n/langchain node version, you may need to:
     - Attach binary image to the model message (image URL/base64), or
     - Use a dedicated “vision message” format if supported.
     If you don’t do this, the model will only see text and will hallucinate.
   - Connect: **STAGE 1 Vision Prompt → STAGE 1 Analyze Photo**

10. **Add “Parse STAGE 1” (Code)**
   - Node type: **Code**
   - Implement robust JSON extraction:
     - Prefer fenced ```json blocks, otherwise first `{...}` match
     - `JSON.parse`
     - Defaults on failure
   - Merge in data from “Configure Language”
   - Connect: **STAGE 1 Analyze Photo → Parse STAGE 1**

11. **Add “STAGE 4 Decompose Prompt” (Set)**
   - Node type: **Set**
   - Field: `chatInput`
   - Include:
     - Room info, dimensions
     - `elements` and `fixtures` JSON
     - Category-to-work mapping rules
     - “CRITICAL RULES” including minimum works and renovation demolition logic
     - Require JSON-only output with `work_items[]`
   - Connect: **Parse STAGE 1 → STAGE 4 Decompose Prompt**

12. **Add GPT model node “GPT-4 Decompose”**
   - Node type: **OpenAI Chat Model**
   - Same OpenAI credential
   - Model: `chatgpt-4o-latest`
   - Options: `maxTokens: 8000`, `temperature: 0.3`

13. **Add “STAGE 4 Decompose LLM” (Chain LLM)**
   - Messages: one user message = `{{$json.chatInput}}`
   - AI language model input from “GPT-4 Decompose”
   - Connect: **STAGE 4 Decompose Prompt → STAGE 4 Decompose LLM**

14. **Add “Parse STAGE 4” (Code)**
   - Node type: **Code**
   - Parse JSON and map `work_items` to normalized objects:
     - add `work_id`, enforce `search_query`, numeric quantities, default units
     - sort by `work_sequence`
   - Connect: **STAGE 4 Decompose LLM → Parse STAGE 4**

15. **Add “Prepare Works” (Code)**
   - Node type: **Code**
   - Store `photo_config` into `$getWorkflowStaticData('global')`
   - Reset `staticData.work_results = []`
   - Output: one item per work (for batching), including `qdrant_collection`, `currency_symbol`, `expected_unit`, etc.
   - Connect: **Parse STAGE 4 → Prepare Works**

16. **Add “Loop Works” (Split In Batches)**
   - Node type: **Split In Batches**
   - `reset: false`
   - Connect: **Prepare Works → Loop Works**
   - Later you will connect:
     - Loop output (items) → pricing path
     - Loop “done” output → aggregation

17. **Add “Store Work Data” (Code)**
   - Node type: **Code**
   - Set `staticData.currentWork = $input.first().json`
   - Connect: **Loop Works (items output) → Store Work Data**

18. **Add “Wait”**
   - Node type: **Wait**
   - Amount: `0.3` seconds (or adjust)
   - Connect: **Store Work Data → Wait**

19. **Add “Restore Work Data” (Code)**
   - Node type: **Code**
   - Return `staticData.currentWork`
   - Connect: **Wait → Restore Work Data**

20. **Add “Embeddings” (OpenAI Embeddings)**
   - Node type: **OpenAI Embeddings**
   - Credential: same OpenAI API credential
   - Model: `text-embedding-3-large`
   - Dimensions: `3072`

21. **Add “Vector Search” (Qdrant Vector Store)**
   - Node type: **Qdrant Vector Store**
   - Credential: configure Qdrant API credential (URL + API key as applicable)
   - Collection: expression `{{$json.qdrant_collection}}`
   - Query/prompt: `{{$json.search_query || $json.work_name}}`
   - `topK: 5`
   - Connect:
     - **Restore Work Data → Vector Search** (main)
     - **Embeddings → Vector Search** (AI embedding connection)

22. **Add “STAGE 5 Parse & Score” (Code)**
   - Node type: **Code**
   - Must use `$input.all()` to read vector results
   - Must read `staticData.currentWork` for the current priced work
   - Implement:
     - parse `document.pageContent` (or equivalent)
     - extract total cost, unit, code, resources
     - score candidates; choose best
     - compute `total_cost`, `estimated_labor_hours`, `quality_level`
   - Connect: **Vector Search → STAGE 5 Parse & Score**

23. **Add “Accumulate” (Code)**
   - Node type: **Code**
   - Append `$input.first().json` to `staticData.work_results`
   - Connect: **STAGE 5 Parse & Score → Accumulate**

24. **Close the loop**
   - Connect: **Accumulate → Loop Works**  
     (This causes the next batch item to be processed.)

25. **Add “STAGE 7.5 Aggregate & Validate” (Code)**
   - Node type: **Code**
   - Read:
     - `staticData.photo_config`
     - `staticData.work_results`
   - Compute totals, group into phases, create `by_phase`, create validation issues list, create quality stats.
   - Clear static data at end (recommended).
   - Connect: **Loop Works (done output) → STAGE 7.5 Aggregate & Validate**

26. **Add “STAGE 9 HTML Report” (Code)**
   - Node type: **Code**
   - Build multilingual HTML:
     - translations dictionary
     - summary + charts + collapsible table
     - include validation bar
     - include links:
       - `https://openconstructionestimate.com/all-estimates/?utm=OCE`
       - `https://github.com/datadrivenconstruction/OpenConstructionEstimate-DDC-CWICR`
       - `https://datadrivenconstruction.io/`
   - Connect: **STAGE 7.5 Aggregate & Validate → STAGE 9 HTML Report**

27. **Add “Final HTML Output” (Code)**
   - Node type: **Code**
   - Convert HTML to base64 and return as binary `text/html` with filename `PhotoEstimate_v2.html`
   - Return JSON summary as well
   - Connect: **STAGE 9 HTML Report → Final HTML Output**
   - Because Form Trigger uses **responseMode: lastNode**, the form response will return from this node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “⭐ Star our repository …” | https://github.com/datadrivenconstruction/OpenConstructionEstimate-DDC-CWICR |
| DDC organization | https://github.com/datadrivenconstruction |
| Rate link used in report | https://openconstructionestimate.com/all-estimates/?utm=OCE |
| DataDrivenConstruction site | https://datadrivenconstruction.io/ |
| Pipeline overview (Stages 1 / 4 / 5 / 7.5 / 9) | Included in sticky note “📊 Pipeline Overview” |
| Qdrant setup note: install Qdrant, load dataset, choose correct collection (3072-d) | Included in sticky note “📥 Vector Database Setup” |

Disclaimer (provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… (as supplied by the user).