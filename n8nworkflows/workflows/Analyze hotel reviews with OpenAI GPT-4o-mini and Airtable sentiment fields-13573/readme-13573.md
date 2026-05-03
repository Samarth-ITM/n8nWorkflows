Analyze hotel reviews with OpenAI GPT-4o-mini and Airtable sentiment fields

https://n8nworkflows.xyz/workflows/analyze-hotel-reviews-with-openai-gpt-4o-mini-and-airtable-sentiment-fields-13573


# Analyze hotel reviews with OpenAI GPT-4o-mini and Airtable sentiment fields

## 1. Workflow Overview

**Workflow name:** *Hotel Review Sentiment Processor*  
**Purpose:** Automatically enrich newly created/updated hotel review records in Airtable with AI-derived sentiment fields (via OpenAI GPT-4o-mini), then write the structured results back to Airtable.  
**Primary use case:** Customer feedback processing for hotels (or any review dataset) where sentiment, topics, and structured insights are stored in Airtable fields.

### 1.1 Trigger & Pre-checks (Airtable → gating logic)
- Starts from an **Airtable Trigger** event.
- Prevents re-processing already processed reviews.
- Ensures a review payload exists before sending anything to the AI.

### 1.2 Payload normalization (clean input for AI)
- Normalizes/massages incoming Airtable trigger data into a consistent shape used by the LLM chain.

### 1.3 AI inference + structured parsing (LLM + schema parser)
- Uses a **Chat OpenAI** model node (GPT-4o-mini implied by the title) as the language model.
- A LangChain **chain** node performs the analysis.
- A **Structured Output Parser (JSON Schema)** enforces a JSON-shaped output.

### 1.4 Post-processing + validation (flatten, merge, verify)
- Flattens the structured AI result for easy mapping.
- Combines AI fields with original review data.
- Validates AI output before writing to Airtable; routes failures to a fallback handler.

### 1.5 Airtable update (write back results)
- Prepares final Airtable update payload.
- Updates the original review record.
- On AI failure, writes failure/empty/default fields (depending on the handler configuration).

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Pre-checks
**Overview:** Receives Airtable events, skips already-processed records, and stops if there is no review content to analyze.

**Nodes involved:**
- Airtable Trigger
- Check if Processed (IF)
- Processed Review: No Action Required (NoOp)
- Check if Review Exist (IF)
- No Review to Process (NoOp)

#### Node: **Airtable Trigger**
- **Type / role:** `n8n-nodes-base.airtableTrigger` — Entry point; emits items when Airtable records change.
- **Configuration choices (interpreted):** Not provided in JSON. Typically includes Base, Table, and trigger event type (create/update).
- **Inputs/Outputs:** No input; output → **Check if Processed**.
- **Potential failures / edge cases:**
  - Airtable credentials missing/expired.
  - Misconfigured base/table/view leading to no triggers.
  - Payload shape differs depending on Airtable Trigger mode (can impact downstream expressions).

#### Node: **Check if Processed**
- **Type / role:** `n8n-nodes-base.if` — Gate to prevent reprocessing.
- **Configuration choices:** Not provided; logically checks a boolean/flag field (e.g., `Processed = true` or presence of sentiment fields).
- **Connections:**
  - **True** (already processed) → **Processed Review: No Action Required**
  - **False** → **Check if Review Exist**
- **Potential failures / edge cases:**
  - If the field used for the condition is missing/null, the IF may route unexpectedly.
  - If Airtable returns different field names than expected, expressions can fail.

#### Node: **Processed Review: No Action Required**
- **Type / role:** `n8n-nodes-base.noOp` — Terminal placeholder for “do nothing”.
- **Connections:** No outputs.
- **Edge cases:** None (but note that “already processed” records are silently ignored).

#### Node: **Check if Review Exist**
- **Type / role:** `n8n-nodes-base.if` — Ensures review text exists before AI call.
- **Configuration choices:** Not provided; likely checks that a “Review”/“Comment” field is not empty.
- **Connections:**
  - **True** → **Normalize Review Payload**
  - **False** → **No Review to Process**
- **Potential failures / edge cases:**
  - Reviews with only whitespace may pass unless trimmed.
  - Multi-field reviews (title + body) may require concatenation; if not handled, AI gets partial context.

#### Node: **No Review to Process**
- **Type / role:** `n8n-nodes-base.noOp` — Terminal placeholder for missing review content.
- **Connections:** No outputs.

---

### Block 2 — Normalize Review Payload
**Overview:** Standardizes the inbound item into a consistent structure for the AI chain (e.g., extracting record ID, review text, metadata).

**Nodes involved:**
- Normalize Review Payload (Set)

#### Node: **Normalize Review Payload**
- **Type / role:** `n8n-nodes-base.set` — Creates/renames fields for downstream use.
- **Configuration choices:** Not provided; typical actions:
  - Map Airtable record ID into a stable field (e.g., `recordId`).
  - Extract `reviewText`, `rating`, `hotelName`, `language`, etc.
- **Connections:** Input from **Check if Review Exist** → output to **AI: Analyze Review Sentiment**.
- **Potential failures / edge cases:**
  - If you “Keep Only Set” without including needed fields, later Airtable update may lack `recordId`.
  - Field name mismatches between Airtable and Set node mappings.

---

### Block 3 — AI inference + structured parsing
**Overview:** Runs GPT analysis and parses the response into a schema-conformant JSON object using LangChain nodes.

**Nodes involved:**
- Analyze Review Sentiment (Chat OpenAI model)
- Parse AI Output (JSON Schema) (Structured Output Parser)
- AI: Analyze Review Sentiment (Chain)

#### Node: **Analyze Review Sentiment**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Provides the chat model (LLM) to the chain.
- **Configuration choices (interpreted):**
  - Parameters are empty in JSON, so key settings (model name, temperature, API credential) are configured in UI or defaults.
  - **maxTries=2** and **retryOnFail=true**: will attempt up to 2 retries on failure.
  - **alwaysOutputData=true**: node outputs an item even if something fails (important for failure routing).
- **Connections:**
  - Output (special port) `ai_languageModel` → **AI: Analyze Review Sentiment**
- **Version-specific notes:** Type version **1.2** (LangChain OpenAI chat wrapper; UI fields vary slightly by n8n version).
- **Potential failures / edge cases:**
  - OpenAI credential missing/invalid.
  - Model not available in your OpenAI account/region.
  - Rate limits/timeouts; retries may still fail.
  - If prompt expects certain input fields not present, the chain may produce irrelevant output.

#### Node: **Parse AI Output (JSON Schema)**
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Forces/validates JSON output matching a schema.
- **Configuration choices:** Empty in JSON; in UI, typically defines a JSON Schema with sentiment fields (e.g., polarity, confidence, pros/cons, categories).
- **Connections:**
  - Output (special port) `ai_outputParser` → **AI: Analyze Review Sentiment**
- **Version-specific notes:** Type version **1.3**.
- **Edge cases / failures:**
  - If the model output cannot be parsed to schema, parser may error or return empty/partial depending on configuration.
  - Schema too strict causes frequent failures; schema too loose reduces reliability.

#### Node: **AI: Analyze Review Sentiment**
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — Orchestrates prompt + model + parser to generate structured output.
- **Configuration choices:** Empty in JSON; in UI it typically includes:
  - Prompt template referencing fields from **Normalize Review Payload** (e.g., `{{$json.reviewText}}`).
  - Uses **Analyze Review Sentiment** as its model and **Parse AI Output** as its structured parser (via special connections).
- **Connections:**
  - Main input from **Normalize Review Payload**
  - `ai_languageModel` input from **Analyze Review Sentiment**
  - `ai_outputParser` input from **Parse AI Output (JSON Schema)**
  - Main output → **Flatten AI Output**
- **Version-specific notes:** Type version **1.7**.
- **Potential failures / edge cases:**
  - Prompt variables missing → empty prompt sections → bad output.
  - Non-English reviews: if not handled in prompt, sentiment may be inconsistent.
  - Long reviews may exceed context limits depending on selected model/settings.

---

### Block 4 — Post-processing + validation
**Overview:** Flattens and merges AI data with original review data, validates AI output presence/shape, and branches into success vs. failure handling.

**Nodes involved:**
- Flatten AI Output (Set)
- Combine AI Output with Review Data (Set)
- Check AI Output Valid (IF)
- AI Failure Handler (Set)

#### Node: **Flatten AI Output**
- **Type / role:** `n8n-nodes-base.set` — Converts nested AI JSON into flat fields for Airtable mapping.
- **Configuration choices:** Not shown; typical patterns:
  - `sentiment.label`, `sentiment.score` → `sentimentLabel`, `sentimentScore`
  - `topics[]` → string join or multi-select mapping
- **Connections:** Input from **AI: Analyze Review Sentiment** → output to **Combine AI Output with Review Data**.
- **Edge cases:**
  - Arrays/objects may not map cleanly into Airtable field types.
  - If AI output is missing keys, expressions may resolve to `undefined`.

#### Node: **Combine AI Output with Review Data**
- **Type / role:** `n8n-nodes-base.set` — Ensures the outgoing item includes both record identifiers and AI results.
- **Configuration choices:** Not shown; usually:
  - Keep `recordId` and original Airtable metadata.
  - Add flattened AI fields.
- **Connections:** → **Check AI Output Valid**
- **Edge cases:**
  - If this node overwrites `id`/`recordId`, Airtable update will fail (record not found).

#### Node: **Check AI Output Valid**
- **Type / role:** `n8n-nodes-base.if` — Validates AI output before writing.
- **Configuration choices:** Not shown; likely checks required fields exist (e.g., `sentimentLabel` not empty).
- **Connections:**
  - **True** → **Prepare Airtable Update**
  - **False** → **AI Failure Handler**
- **Edge cases:**
  - Overly strict validation can route many legitimate outputs to failure.
  - Overly loose validation can write garbage to Airtable.

#### Node: **AI Failure Handler**
- **Type / role:** `n8n-nodes-base.set` — Creates a safe fallback update payload (e.g., sets error status fields).
- **Configuration choices:** Not shown; commonly sets:
  - `processingStatus = "AI_FAILED"` or similar
  - stores `errorMessage` or raw model output (if captured)
- **Connections:** → **Update Airtable Review Record**
- **Edge cases:**
  - If it doesn’t include `recordId`, the update cannot target the correct record.

---

### Block 5 — Prepare & Update Airtable
**Overview:** Maps fields into Airtable’s expected update format and updates the originating record.

**Nodes involved:**
- Prepare Airtable Update (Set)
- Update Airtable Review Record (Airtable)

#### Node: **Prepare Airtable Update**
- **Type / role:** `n8n-nodes-base.set` — Final field mapping and formatting for Airtable.
- **Configuration choices:** Not shown; typically:
  - Builds `fields` object with Airtable column names.
  - Ensures correct types: numbers, single select strings, multi-select arrays, long text.
- **Connections:** → **Update Airtable Review Record**
- **Edge cases:**
  - Airtable field type mismatch (e.g., sending string to number field).
  - Multi-select requires array of strings; sending comma-separated string may fail.

#### Node: **Update Airtable Review Record**
- **Type / role:** `n8n-nodes-base.airtable` — Writes back sentiment fields to the review record.
- **Configuration choices:** Not present; typically “Update” operation with:
  - Base + Table
  - Record ID from earlier node (expression)
  - Fields mapping
- **Connections:** Terminal (no downstream nodes).
- **Version-specific notes:** Type version **2.1**.
- **Potential failures / edge cases:**
  - Record ID invalid or missing.
  - Airtable permission denied / base not shared with token.
  - Rate limits on Airtable API.
  - Partial updates: fields not found if column names differ.

---

### Block 6 — Sticky Notes (documentation placeholders)
**Overview:** The workflow contains multiple sticky notes, but all have empty content in the provided JSON.

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note5
- Sticky Note6

#### Node: **Sticky Note / Sticky Note1 / Sticky Note2 / Sticky Note3 / Sticky Note5 / Sticky Note6**
- **Type / role:** `n8n-nodes-base.stickyNote` — Canvas annotations.
- **Configuration choices:** `content` is empty for all; no operational effect.
- **Connections:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Airtable Trigger | airtableTrigger | Entry trigger from Airtable changes | — | Check if Processed |  |
| Check if Processed | if | Skip already-processed reviews | Airtable Trigger | Processed Review: No Action Required; Check if Review Exist |  |
| Processed Review: No Action Required | noOp | Stop path for already processed items | Check if Processed | — |  |
| Check if Review Exist | if | Ensure review content exists | Check if Processed | Normalize Review Payload; No Review to Process |  |
| No Review to Process | noOp | Stop path when review text missing | Check if Review Exist | — |  |
| Normalize Review Payload | set | Normalize Airtable payload for AI | Check if Review Exist | AI: Analyze Review Sentiment |  |
| Analyze Review Sentiment | lmChatOpenAi | OpenAI chat model provider for chain | — (model connection) | AI: Analyze Review Sentiment (ai_languageModel) |  |
| Parse AI Output (JSON Schema) | outputParserStructured | Enforce schema for AI output | — (parser connection) | AI: Analyze Review Sentiment (ai_outputParser) |  |
| AI: Analyze Review Sentiment | chainLlm | Run prompt+LLM+parser to produce structured result | Normalize Review Payload; (model+parser ports) | Flatten AI Output |  |
| Flatten AI Output | set | Flatten structured JSON into fields | AI: Analyze Review Sentiment | Combine AI Output with Review Data |  |
| Combine AI Output with Review Data | set | Merge AI results with record identifiers | Flatten AI Output | Check AI Output Valid |  |
| Check AI Output Valid | if | Validate AI output before update | Combine AI Output with Review Data | Prepare Airtable Update; AI Failure Handler |  |
| Prepare Airtable Update | set | Map fields to Airtable update payload | Check AI Output Valid | Update Airtable Review Record |  |
| AI Failure Handler | set | Build fallback update on AI failure | Check AI Output Valid | Update Airtable Review Record |  |
| Update Airtable Review Record | airtable | Update Airtable record with sentiment fields | Prepare Airtable Update; AI Failure Handler | — |  |
| Sticky Note | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note1 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note2 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note3 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note5 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note6 | stickyNote | Canvas annotation (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name: **Hotel Review Sentiment Processor**
   - Set timezone (optional): **Asia/Manila** (to match existing settings)

2) **Add Airtable Trigger**
   - Node: **Airtable Trigger**
   - Configure credentials: Airtable Personal Access Token (PAT)
   - Select **Base** and **Table** that contains hotel reviews
   - Choose trigger event(s): typically “When record is created/updated”
   - Ensure the trigger includes the fields needed downstream (record ID + review text + processed flag)

3) **Add IF: “Check if Processed”**
   - Condition: check a field such as `Processed` is **true** (or sentiment fields already filled)
   - **True** output → add **NoOp** node named **Processed Review: No Action Required**
   - **False** output → continue

4) **Add IF: “Check if Review Exist”**
   - Condition: review text field exists and is not empty (e.g., `Review` / `Comment`)
   - **False** output → add **NoOp** node named **No Review to Process**
   - **True** output → continue

5) **Add Set: “Normalize Review Payload”**
   - Extract and standardize fields, at minimum:
     - `recordId` = Airtable record ID
     - `reviewText` = the review body (optionally concatenate title + body)
     - (Optional) `rating`, `hotelName`, `source`, `language`
   - Prefer keeping original fields unless you are sure they are not needed later.

6) **Add LangChain nodes for structured sentiment**
   - Add **Chat OpenAI** node named **Analyze Review Sentiment**
     - Credentials: OpenAI API key
     - Model: choose **gpt-4o-mini** (or the closest equivalent available)
     - Set temperature (commonly 0–0.3 for consistent classification)
     - Keep retries enabled (this workflow uses retries; max tries 2)
   - Add **Structured Output Parser** named **Parse AI Output (JSON Schema)**
     - Define a JSON Schema for fields you want in Airtable (example categories):
       - `sentimentLabel` (e.g., Positive/Neutral/Negative)
       - `sentimentScore` (0–1 or -1–1)
       - `summary`
       - `pros` (array)
       - `cons` (array)
       - `topics` (array)
   - Add **LLM Chain** node named **AI: Analyze Review Sentiment**
     - Prompt: instruct model to analyze `{{$json.reviewText}}` and output strictly according to schema
     - Connect:
       - **Normalize Review Payload (main)** → **AI: Analyze Review Sentiment (main)**
       - **Analyze Review Sentiment (ai_languageModel)** → **AI: Analyze Review Sentiment**
       - **Parse AI Output (ai_outputParser)** → **AI: Analyze Review Sentiment**

7) **Add Set: “Flatten AI Output”**
   - Map parser result into flat fields for Airtable:
     - Convert nested objects into top-level keys.
     - Convert arrays into Airtable-friendly formats (either arrays for multi-select, or joined text for long text).

8) **Add Set: “Combine AI Output with Review Data”**
   - Ensure the outgoing item includes:
     - `recordId`
     - all Airtable update fields from AI result
     - optional audit fields like `processedAt` timestamp

9) **Add IF: “Check AI Output Valid”**
   - Condition examples:
     - `sentimentLabel` exists
     - `sentimentScore` is a number
   - **True** → success path
   - **False** → failure path

10) **Add Set: “Prepare Airtable Update”** (success path)
   - Build fields exactly matching Airtable column names/types.
   - Example mappings:
     - `Sentiment` (single select) ← `sentimentLabel`
     - `Sentiment Score` (number) ← `sentimentScore`
     - `AI Summary` (long text) ← `summary`
     - `Topics` (multi-select) ← array `topics`
     - `Processed` (checkbox) ← true

11) **Add Set: “AI Failure Handler”** (failure path)
   - Set safe defaults and status fields, e.g.:
     - `AI Status` = "failed"
     - `Processed` = false (or true with “failed” state, depending on your policy)
     - Optionally store a short error note if available

12) **Add Airtable node: “Update Airtable Review Record”**
   - Operation: **Update**
   - Record ID: expression pointing to `{{$json.recordId}}`
   - Fields: map from either **Prepare Airtable Update** or **AI Failure Handler**
   - Connect both paths to this node:
     - **Prepare Airtable Update** → **Update Airtable Review Record**
     - **AI Failure Handler** → **Update Airtable Review Record**

13) **(Optional) Add an Error Workflow**
   - The workflow references an **errorWorkflow** ID in settings; to replicate behavior, configure **Workflow Settings → Error Workflow** to your own error handler workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow settings include a configured Error Workflow reference (ID present in JSON). To reproduce fully, create/select your own Error Workflow in n8n settings. | Workflow-level reliability / incident handling |
| All sticky notes in the provided workflow are empty; they don’t document configuration or assumptions. | Canvas annotations (none provided) |