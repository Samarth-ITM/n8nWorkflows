Generate AI search–driven FAQ insights for SEO with SE Ranking and OpenAI GPT-4.1-mini

https://n8nworkflows.xyz/workflows/generate-ai-search-driven-faq-insights-for-seo-with-se-ranking-and-openai-gpt-4-1-mini-12441


# Generate AI search–driven FAQ insights for SEO with SE Ranking and OpenAI GPT-4.1-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow pulls **real AI search prompts and answers** for a target domain from **SE Ranking’s AI Search API**, then uses **OpenAI (gpt-4.1-mini)** to **zero-shot classify** the extracted questions into SEO/content-intent categories. Finally, it aggregates everything into a single structured JSON and **writes it to disk**.

**Typical use cases:**
- Building FAQ sections based on real AI-search queries
- Content planning (intent mapping, topic clustering seeds)
- SERP/AI-mode prompt mining for competitive or domain research
- Automating recurring SEO insight exports

### 1.1 Input & Parameterization
Manual trigger + a Set node define the target domain, region/source, engine, pagination, sorting, and include/exclude keyword rules.

### 1.2 SE Ranking Data Retrieval
Calls `GET /v1/ai-search/prompts-by-target` to fetch AI prompts and their answers (including links).

### 1.3 Custom Extraction (Questions / QnA / Links / Raw Prompts)
Several Code nodes reshape the API response into separate datasets:
- questions-only list (for classification)
- question+answer pairs
- links list
- raw prompts array

### 1.4 AI Enrichment (Zero-shot Question Classification)
OpenAI model is attached to an Information Extractor node that outputs a schema-conformant classification with confidence.

### 1.5 Merging, Aggregation, and Export
Merges AI-enriched data with extracted datasets, aggregates to a single JSON object, converts to binary, and writes to a local file path.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Workflow intro / documentation (Sticky notes)
**Overview:** Visual documentation and setup notes embedded in the canvas. No runtime effect.  
**Nodes involved:** `Sticky Note4`, `Sticky Note5`

#### Node: Sticky Note4
- **Type / role:** Sticky Note (documentation)
- **Configuration (interpreted):**
  - Contains SE Ranking logo and note about OpenAI usage.
  - Image link (preserved):  
    `https://media.licdn.com/dms/image/v2/D4D0BAQHBbVpuDD3toA/company-logo_200_200/company-logo_200_200/0/1725976307233/se_ranking_logo?e=1768435200&v=beta&t=_HSGZks62sL6rTXwuo0U21QCKBCNzVT_8OkeIPUr4N8`
- **Connections:** none
- **Edge cases:** none

#### Node: Sticky Note5
- **Type / role:** Sticky Note (documentation)
- **Configuration (interpreted):**
  - Explains how the workflow works, credential requirements, customization ideas, and export considerations.
- **Connections:** none
- **Edge cases:** none

---

### Block 2.2 — Input reception & parameter setup
**Overview:** Defines all request parameters used to query SE Ranking’s API (domain, locale, engine, pagination, filters).  
**Nodes involved:** `When clicking ‘Execute workflow’`, `Set the Input Fields`

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (entry point)
- **Configuration:** Runs workflow manually from the editor.
- **Output:** A single empty item that triggers downstream nodes.
- **Connections:**
  - Output → `Set the Input Fields`
- **Edge cases:** none

#### Node: Set the Input Fields
- **Type / role:** Set node (creates structured inputs)
- **Configuration choices:**
  - Creates string fields used later as query parameters:
    - `target_site`: `seranking.com`
    - `engine`: `ai-mode`
    - `source`: `us`
    - `scope`: `domain`
    - `sort`: `volume`
    - `sort_order`: `desc`
    - `offset`: `0`
    - `limit`: `100`
    - `multi_keyword_included`: JSON string representing include rules  
    - `multi_keyword_excluded`: JSON string representing exclude rules
  - Note: both `multi_keyword_*` fields are **strings** that look like JSON arrays. The SE Ranking endpoint is expected to accept these as query parameter values.
- **Key variables/expressions:** none (static assignments)
- **Connections:**
  - Input ← `When clicking ‘Execute workflow’`
  - Output → `SE Ranking Prompts by Target`
- **Potential failure/edge cases:**
  - Misformatted filter strings can cause API validation errors (400) or silently yield unexpected results.
  - Pagination (`offset`, `limit`) are strings; if the API expects integers, it may still parse them, but strict APIs could reject.

---

### Block 2.3 — SE Ranking prompt retrieval
**Overview:** Calls SE Ranking API to fetch AI search prompts, answers, and links for the domain and filters.  
**Nodes involved:** `Sticky Note2`, `SE Ranking Prompts by Target`

#### Node: Sticky Note2
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Describes the SE Ranking “Prompts by Target” purpose.
- **Connections:** none
- **Edge cases:** none

#### Node: SE Ranking Prompts by Target
- **Type / role:** HTTP Request (API integration)
- **Configuration choices (interpreted):**
  - **Method:** (not explicitly shown; typically GET for query parameters; node uses `sendQuery: true`)
  - **URL:** `https://api.seranking.com/v1/ai-search/prompts-by-target`
  - **Authentication:** Generic credential type → **HTTP Header Auth** (intended for SE Ranking)
  - **Query parameters (expressions):**
    - `target={{ $json.target_site }}`
    - `scope={{ $json.scope }}`
    - `source={{ $json.source }}`
    - `engine={{ $json.engine }}`
    - `sort={{ $json.sort }}`
    - `sort_order={{ $json.sort_order }}`
    - `offset={{ $json.offset }}`
    - `limit={{ $json.limit }}`
    - `filter[multi_keyword_included]={{ $json.multi_keyword_included }}`
    - `filter[multi_keyword_excluded]={{ $json.multi_keyword_excluded }}`
  - **Reliability:** `retryOnFail: true`
- **Credentials:**
  - Uses `httpHeaderAuth` credential named **“SE Ranking”**
  - Also lists a `httpBearerAuth` credential named **“Thordata Webscraper API”**, but the node’s selected auth mode is **httpHeaderAuth**; the bearer credential likely isn’t used here.
- **Connections:**
  - Input ← `Set the Input Fields`
  - Output → `Extract All Links`, `Extract QnA`, `Extract Questions Only`, `Extract Prompts` (fan-out)
- **Potential failure/edge cases:**
  - 401/403 if SE Ranking header auth is missing/invalid.
  - 429 rate limiting; retries may help but can still fail without backoff tuning.
  - Response shape changes (e.g., `prompts` not present) will break downstream Code nodes unless guarded (these Code nodes do guard with `|| []`).

---

### Block 2.4 — Custom data extraction & structuring
**Overview:** Transforms SE Ranking’s raw response into separate structured datasets for enrichment and reporting.  
**Nodes involved:** `Sticky Note`, `Extract Questions Only`, `Extract All Links`, `Extract QnA`, `Extract Prompts`

#### Node: Sticky Note
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Describes extraction of questions/answers/links into usable structures.
- **Connections:** none
- **Edge cases:** none

#### Node: Extract Questions Only
- **Type / role:** Code node (JavaScript transformation)
- **Configuration (interpreted):**
  - Reads: `$input.first().json.prompts || []`
  - Filters to prompts where `p.prompt` exists and `p.answer.text` exists.
  - Outputs: `{ questions: [ { question: <prompt> }, ... ] }`
- **Key code behavior:**
  - Guards missing `prompts` with `|| []`.
  - Uses optional chaining `p.answer?.text`.
- **Connections:**
  - Input ← `SE Ranking Prompts by Target`
  - Output → `AI QnA Zeroshot Classifier` and `Perform Custom Data Merge` (same output item is sent to both)
- **Potential failure/edge cases:**
  - If response is not an object or is empty, output becomes `{questions: []}` (safe).
  - If prompts exist but none have `answer.text`, you classify nothing; later merges may still run but with empty classifications.

#### Node: Extract All Links
- **Type / role:** Code node (link extraction)
- **Configuration (interpreted):**
  - Collects `p.answer.links` across prompts (`flatMap`).
  - Cleans:
    - removes falsy
    - trims
    - keeps only strings starting with `"http"`
  - Outputs: `{ urls: [ ... ] }`
- **Connections:**
  - Input ← `SE Ranking Prompts by Target`
  - Output → `Perform Custom Data Merge` (input index 1)
- **Potential failure/edge cases:**
  - If `links` contains non-strings, `.trim()` can throw. Current code assumes each link is a string; if API sometimes returns objects, add a type check.
  - Duplicates are not removed; may want `new Set()` if deduplication is needed.

#### Node: Extract QnA
- **Type / role:** Code node (Q&A structuring)
- **Configuration (interpreted):**
  - Builds array: `{ question: p.prompt, answer: p.answer.text }`
  - Outputs: `{ qna: [ ... ] }`
- **Connections:**
  - Input ← `SE Ranking Prompts by Target`
  - Output → `Perform Custom Data Merge` (input index 2)
- **Potential failure/edge cases:**
  - Same as above: if `answer.text` missing, item is excluded.

#### Node: Extract Prompts
- **Type / role:** Code node (raw data passthrough)
- **Configuration (interpreted):**
  - Outputs raw prompts array: `{ prompts: prompts }`
- **Connections:**
  - Input ← `SE Ranking Prompts by Target`
  - Output → `Perform Custom Data Merge` (input index 3)
- **Potential failure/edge cases:** Minimal; guarded with `|| []`.

---

### Block 2.5 — AI enrichment (zero-shot classification)
**Overview:** Uses OpenAI gpt-4.1-mini to classify each question into a single intent category with a confidence score, validated against a strict JSON Schema.  
**Nodes involved:** `Sticky Note1`, `OpenAI Chat Model for Zeroshot Classifier`, `AI QnA Zeroshot Classifier`

#### Node: Sticky Note1
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Explains that classification is applied and results are aggregated for downstream usage.
- **Connections:** none
- **Edge cases:** none

#### Node: OpenAI Chat Model for Zeroshot Classifier
- **Type / role:** LangChain Chat Model (OpenAI)
- **Configuration choices (interpreted):**
  - Model: `gpt-4.1-mini`
  - Acts as the LLM provider for the extractor node via an **AI language model connection** (not the normal “main” connection).
- **Credentials:** OpenAI API credential named **“OpenAi account”**
- **Connections:**
  - AI language model output → `AI QnA Zeroshot Classifier` (ai_languageModel)
- **Potential failure/edge cases:**
  - 401 if OpenAI key invalid.
  - Model availability/renames in the OpenAI account/region.
  - Cost/rate limits (429), timeouts for large payloads.

#### Node: AI QnA Zeroshot Classifier
- **Type / role:** LangChain Information Extractor (structured extraction)
- **Configuration choices (interpreted):**
  - **Prompt:** Asks the model to classify each question into exactly one category among:
    `HOW_TO, DEFINITION, TOOLS, PRICING, COMPARISON, ALTERNATIVES, LEGAL, DECISION, TROUBLESHOOTING, GENERAL`
  - Injects questions via expression:  
    `{{ $json.questions.toJsonString() }}`
  - **Schema enforcement:** Manual JSON Schema expecting an **array** of objects:
    - required: `id` (integer >= 1), `question` (string), `category` (enum), `confidence` (0..1)
    - `additionalProperties: false`
  - `retryOnFail: true`
- **Connections:**
  - Main input ← `Extract Questions Only`
  - Main output → `Merge Enriched Data` (input 0)
  - AI language model input ← `OpenAI Chat Model for Zeroshot Classifier`
- **Potential failure/edge cases:**
  - **Schema mismatch:** If the model returns invalid JSON or violates schema (extra fields, missing id, confidence out of range), extraction can fail.
  - **Expression dependency:** If `$json.questions` is missing, `toJsonString()` may fail; current upstream always outputs `questions`, but ensure no other branch calls it.
  - **Token limits:** Many questions (limit=100) can be large; consider batching if you increase the limit.

---

### Block 2.6 — Merging & aggregation into final dataset
**Overview:** Combines extracted datasets (links, qna, prompts, questions) and AI classifications, then aggregates into a single JSON object.  
**Nodes involved:** `Perform Custom Data Merge`, `Custom Data Aggregation`, `Merge Enriched Data`, `Final Data Aggregation`

#### Node: Perform Custom Data Merge
- **Type / role:** Merge node (multi-input combine)
- **Configuration choices (interpreted):**
  - `numberInputs: 4`
  - Receives four parallel branches from extraction nodes.
  - Default merge behavior in n8n Merge can vary by mode; with `numberInputs: 4` it’s intended to combine multiple input streams into one stream for aggregation.
- **Inputs (by connection index):**
  - Index 0: from `Extract Questions Only` (questions)
  - Index 1: from `Extract All Links` (urls)
  - Index 2: from `Extract QnA` (qna)
  - Index 3: from `Extract Prompts` (prompts)
- **Output:**
  - Combined items forwarded to aggregation.
- **Connections:**
  - Output → `Custom Data Aggregation`
- **Potential failure/edge cases:**
  - If one branch produces zero items while others produce one, merge semantics can drop/duplicate items depending on merge mode. If you see missing fields, switch to a merge strategy that explicitly combines by position or always passes-through (then aggregate).

#### Node: Custom Data Aggregation
- **Type / role:** Aggregate node (collects items into one object field)
- **Configuration choices (interpreted):**
  - `aggregateAllItemData` with `destinationFieldName: custom_aggregate`
  - This typically collapses incoming items into a single item, placing aggregated content under `custom_aggregate`.
- **Connections:**
  - Input ← `Perform Custom Data Merge`
  - Output → `Merge Enriched Data` (input 1)
- **Potential failure/edge cases:**
  - If upstream merge produces unexpected item shapes, aggregated output may be nested in a way you didn’t intend.

#### Node: Merge Enriched Data
- **Type / role:** Merge node (combine AI classification + extracted aggregates)
- **Configuration choices (interpreted):**
  - Default merge settings (not specified in JSON).
  - Combines:
    - AI classifier output (structured classifications array)
    - Custom aggregated extraction output (`custom_aggregate`)
- **Connections:**
  - Input 0 ← `AI QnA Zeroshot Classifier`
  - Input 1 ← `Custom Data Aggregation`
  - Output → `Final Data Aggregation`
- **Potential failure/edge cases:**
  - If either side has multiple items, merge behavior may not align; ensure both sides output single items or configure merge by position.

#### Node: Final Data Aggregation
- **Type / role:** Aggregate node (final “one JSON blob” consolidation)
- **Configuration choices (interpreted):**
  - `aggregateAllItemData` to produce a single final item for export.
- **Connections:**
  - Input ← `Merge Enriched Data`
  - Output → `Create a Binary Data`
- **Potential failure/edge cases:**
  - If previous steps already produce a single item, this may add extra nesting; validate the final JSON structure you want.

---

### Block 2.7 — Export handling (JSON → binary → file)
**Overview:** Converts the final JSON into binary data and writes it to a local disk path.  
**Nodes involved:** `Sticky Note3`, `Create a Binary Data`, `Write File to Disk`

#### Node: Sticky Note3
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Explains conversion and storage for downstream automation.
- **Connections:** none
- **Edge cases:** none

#### Node: Create a Binary Data
- **Type / role:** Function node (creates binary file content)
- **Configuration (interpreted):**
  - Writes `items[0].binary.data.data` as base64-encoded JSON string:
    - `JSON.stringify(items[0].json, null, 2)`
  - Ensures output includes a `binary` property named `data`.
- **Connections:**
  - Input ← `Final Data Aggregation`
  - Output → `Write File to Disk`
- **Potential failure/edge cases:**
  - If there is no `items[0]` (empty run), this will throw.
  - Uses `new Buffer(...)` (deprecated in modern Node.js). Prefer `Buffer.from(...)` in updated environments.

#### Node: Write File to Disk
- **Type / role:** Read/Write File (write operation)
- **Configuration choices (interpreted):**
  - Operation: `write`
  - File name/path: `C:\\SERanking_PromptFAQ.json`
  - Data property name: `data` (binary property produced by prior node)
- **Connections:**
  - Input ← `Create a Binary Data`
- **Potential failure/edge cases:**
  - Path is Windows-specific; will fail on Linux containers (common for n8n).
  - Needs filesystem write permissions; in Docker/cloud this often fails unless a volume is mounted.
  - If file exists, behavior depends on node defaults (typically overwrite).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation (branding/context) |  |  | ![Logo](https://media.licdn.com/dms/image/v2/D4D0BAQHBbVpuDD3toA/company-logo_200_200/company-logo_200_200/0/1725976307233/se_ranking_logo?e=1768435200&v=beta&t=_HSGZks62sL6rTXwuo0U21QCKBCNzVT_8OkeIPUr4N8)  \nOpenAI GPT-4o-mini for the Structured Data Extraction and Data Mining Purposes |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation (how it works/setup/customize) |  |  | ## **How It Works** … (setup instructions + customization + scheduling notes) |
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Manual entry point |  | Set the Input Fields | ## **How It Works** … (setup instructions + customization + scheduling notes) |
| Set the Input Fields | n8n-nodes-base.set | Defines SE Ranking query parameters and filters | When clicking ‘Execute workflow’ | SE Ranking Prompts by Target | ## **How It Works** … (setup instructions + customization + scheduling notes) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documents SE Ranking prompt fetch block |  |  | ## SE Ranking Prompts by Target … |
| SE Ranking Prompts by Target | n8n-nodes-base.httpRequest | Calls SE Ranking API to fetch prompts/answers | Set the Input Fields | Extract All Links; Extract QnA; Extract Questions Only; Extract Prompts | ## SE Ranking Prompts by Target … |
| Sticky Note | n8n-nodes-base.stickyNote | Documents custom extraction block |  |  | ## Custom Data Extraction … |
| Extract Questions Only | n8n-nodes-base.code | Builds `{questions:[{question}]}` for AI classification | SE Ranking Prompts by Target | AI QnA Zeroshot Classifier; Perform Custom Data Merge | ## Custom Data Extraction … |
| Extract All Links | n8n-nodes-base.code | Extracts/cleans answer links into `{urls:[]}` | SE Ranking Prompts by Target | Perform Custom Data Merge | ## Custom Data Extraction … |
| Extract QnA | n8n-nodes-base.code | Extracts `{qna:[{question,answer}]}` | SE Ranking Prompts by Target | Perform Custom Data Merge | ## Custom Data Extraction … |
| Extract Prompts | n8n-nodes-base.code | Passes raw prompts array | SE Ranking Prompts by Target | Perform Custom Data Merge | ## Custom Data Extraction … |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documents AI enrichment and formatting |  |  | ## Data Enrichment … |
| OpenAI Chat Model for Zeroshot Classifier | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM (gpt-4.1-mini) to extractor |  | AI QnA Zeroshot Classifier (ai_languageModel) | ## Data Enrichment … |
| AI QnA Zeroshot Classifier | @n8n/n8n-nodes-langchain.informationExtractor | Zero-shot classifies questions to schema output | Extract Questions Only; OpenAI Chat Model for Zeroshot Classifier (ai) | Merge Enriched Data | ## Data Enrichment … |
| Perform Custom Data Merge | n8n-nodes-base.merge | Combines 4 extracted datasets | Extract Questions Only; Extract All Links; Extract QnA; Extract Prompts | Custom Data Aggregation | ## Data Enrichment … |
| Custom Data Aggregation | n8n-nodes-base.aggregate | Aggregates merged extraction into `custom_aggregate` | Perform Custom Data Merge | Merge Enriched Data | ## Data Enrichment … |
| Merge Enriched Data | n8n-nodes-base.merge | Merges AI classification + custom aggregate | AI QnA Zeroshot Classifier; Custom Data Aggregation | Final Data Aggregation | ## Data Enrichment … |
| Final Data Aggregation | n8n-nodes-base.aggregate | Produces final single JSON object | Merge Enriched Data | Create a Binary Data | ## Export Data Handling … |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documents export handling |  |  | ## Export Data Handling … |
| Create a Binary Data | n8n-nodes-base.function | Converts JSON to base64 binary payload | Final Data Aggregation | Write File to Disk | ## Export Data Handling … |
| Write File to Disk | n8n-nodes-base.readWriteFile | Writes binary JSON to disk | Create a Binary Data |  | ## Export Data Handling … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name it: `When clicking ‘Execute workflow’`

3. **Add Set node for inputs**
   - Node: **Set**
   - Name: `Set the Input Fields`
   - Add fields (all as **String**):
     - `target_site` = `seranking.com`
     - `engine` = `ai-mode`
     - `source` = `us`
     - `scope` = `domain`
     - `sort` = `volume`
     - `sort_order` = `desc`
     - `offset` = `0`
     - `limit` = `100`
     - `multi_keyword_included` = `[  [  {  "type": "contains",  "value": "seo"  }  ] ]`
     - `multi_keyword_excluded` = `[   [     { "type": "contains", "value": "seo"   },     { "type": "contains", "value": "tools" }   ],   [     { "type": "contains", "value": "backlinks" }   ] ]`
   - Connect: **Manual Trigger → Set**

4. **Add HTTP Request node (SE Ranking)**
   - Node: **HTTP Request**
   - Name: `SE Ranking Prompts by Target`
   - URL: `https://api.seranking.com/v1/ai-search/prompts-by-target`
   - Enable **Send Query Parameters**
   - Add query parameters using expressions:
     - `target` = `={{ $json.target_site }}`
     - `scope` = `={{ $json.scope }}`
     - `source` = `={{ $json.source }}`
     - `engine` = `={{ $json.engine }}`
     - `sort` = `={{ $json.sort }}`
     - `sort_order` = `={{ $json.sort_order }}`
     - `offset` = `={{ $json.offset }}`
     - `limit` = `={{ $json.limit }}`
     - `filter[multi_keyword_included]` = `={{ $json.multi_keyword_included }}`
     - `filter[multi_keyword_excluded]` = `={{ $json.multi_keyword_excluded }}`
   - Turn on **Retry on Fail**
   - **Credentials (required):**
     - Create credential: **HTTP Header Auth**
     - Name: `SE Ranking`
     - Put SE Ranking API token/header as required by SE Ranking (for example: `Authorization: Bearer <token>` or their specified header).
   - Connect: **Set → HTTP Request**

5. **Add Code node: Extract Questions**
   - Node: **Code**
   - Name: `Extract Questions Only`
   - Paste code:
     ```js
     const prompts = $input.first().json.prompts || [];

     const questions = prompts
       .filter(p => p.prompt && p.answer?.text)
       .map(p => ({
         question: p.prompt
       }));

     return [{ json: { questions } }];
     ```
   - Connect: **SE Ranking Prompts by Target → Extract Questions Only**

6. **Add Code node: Extract Links**
   - Node: **Code**
   - Name: `Extract All Links`
   - Paste code:
     ```js
     const prompts = $input.first().json.prompts || [];

     const urls = prompts.flatMap(p => p.answer?.links || []);

     const cleaned = urls
       .filter(Boolean)
       .map(u => u.trim())
       .filter(u => u.startsWith("http"));

     return [{ json: { urls: cleaned } }];
     ```
   - Connect: **SE Ranking Prompts by Target → Extract All Links**

7. **Add Code node: Extract QnA**
   - Node: **Code**
   - Name: `Extract QnA`
   - Paste code:
     ```js
     const prompts = $input.first().json.prompts || [];

     const qna = prompts
       .filter(p => p.prompt && p.answer?.text)
       .map(p => ({
         question: p.prompt,
         answer: p.answer.text
       }));

     return [{ json: { qna } }];
     ```
   - Connect: **SE Ranking Prompts by Target → Extract QnA**

8. **Add Code node: Extract Prompts**
   - Node: **Code**
   - Name: `Extract Prompts`
   - Paste code:
     ```js
     const prompts = $input.first().json.prompts || [];

     return [{ json: { prompts } }];
     ```
   - Connect: **SE Ranking Prompts by Target → Extract Prompts**

9. **Add OpenAI Chat Model node**
   - Node: **OpenAI Chat Model** (LangChain)
   - Name: `OpenAI Chat Model for Zeroshot Classifier`
   - Model: `gpt-4.1-mini`
   - **Credentials (required):**
     - Create/attach **OpenAI API** credential (`OpenAi account`) with a valid API key.

10. **Add Information Extractor node (classification)**
    - Node: **Information Extractor** (LangChain)
    - Name: `AI QnA Zeroshot Classifier`
    - Text/prompt:
      - Include your category rules and inject questions with:
        `{{ $json.questions.toJsonString() }}`
    - Schema type: **Manual**
    - Paste JSON Schema (same as in workflow) requiring:
      - array of `{id, question, category, confidence}`
      - `category` enum with the 10 categories
      - confidence 0..1
      - `additionalProperties: false`
    - Enable **Retry on Fail**
    - Connect:
      - **Extract Questions Only → AI QnA Zeroshot Classifier** (main)
      - **OpenAI Chat Model → AI QnA Zeroshot Classifier** (ai_languageModel connection)

11. **Add Merge node for extracted data (4 inputs)**
    - Node: **Merge**
    - Name: `Perform Custom Data Merge`
    - Set **Number of Inputs** = `4`
    - Connect:
      - `Extract Questions Only` → Merge input 0
      - `Extract All Links` → Merge input 1
      - `Extract QnA` → Merge input 2
      - `Extract Prompts` → Merge input 3

12. **Add Aggregate node for custom extraction**
    - Node: **Aggregate**
    - Name: `Custom Data Aggregation`
    - Mode: **Aggregate All Item Data**
    - Destination field name: `custom_aggregate`
    - Connect: **Perform Custom Data Merge → Custom Data Aggregation**

13. **Add Merge node to combine AI + custom aggregate**
    - Node: **Merge**
    - Name: `Merge Enriched Data`
    - Connect:
      - `AI QnA Zeroshot Classifier` → Merge input 0
      - `Custom Data Aggregation` → Merge input 1

14. **Add final Aggregate node**
    - Node: **Aggregate**
    - Name: `Final Data Aggregation`
    - Mode: **Aggregate All Item Data**
    - Connect: **Merge Enriched Data → Final Data Aggregation**

15. **Add Function node to create binary**
    - Node: **Function**
    - Name: `Create a Binary Data`
    - Code:
      ```js
      items[0].binary = {
        data: {
          data: new Buffer(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```
      (Optional modernization: replace `new Buffer` with `Buffer.from`.)
    - Connect: **Final Data Aggregation → Create a Binary Data**

16. **Add Read/Write File node**
    - Node: **Read/Write File**
    - Name: `Write File to Disk`
    - Operation: **Write**
    - File name: `C:\\SERanking_PromptFAQ.json` (change for your OS/container)
    - Data property name: `data`
    - Connect: **Create a Binary Data → Write File to Disk**

17. (Optional) **Add sticky notes** for documentation, mirroring the provided content.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SE Ranking logo used in canvas note | https://media.licdn.com/dms/image/v2/D4D0BAQHBbVpuDD3toA/company-logo_200_200/company-logo_200_200/0/1725976307233/se_ranking_logo?e=1768435200&v=beta&t=_HSGZks62sL6rTXwuo0U21QCKBCNzVT_8OkeIPUr4N8 |
| Setup reminders: configure SE Ranking (HTTP Header Auth) + OpenAI (gpt-4.1-mini), update input filters, ensure output path is valid, then execute | Included in the workflow’s “How It Works / Setup Instructions / Customize” sticky note |
| Deployment consideration: writing to `C:\` is Windows-specific and often fails in Docker/cloud n8n unless volumes/permissions are configured | Relevant to `Write File to Disk` node configuration |

