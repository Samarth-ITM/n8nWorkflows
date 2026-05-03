Summarize SE Ranking AI search visibility using OpenAI GPT-4.1-mini

https://n8nworkflows.xyz/workflows/summarize-se-ranking-ai-search-visibility-using-openai-gpt-4-1-mini-12291


# Summarize SE Ranking AI search visibility using OpenAI GPT-4.1-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Fetch AI search visibility time-series data from the SE Ranking API for a given target domain/region/engine, summarize the time-series with **OpenAI GPT-4.1-mini** into structured text fields, enrich the original dataset with those summaries, and export the final result as a JSON file to disk.

**Primary use cases:**
- AI search/SEO monitoring (AI Overview visibility trends)
- Generating executive summaries from time-series metrics
- Exporting structured insights for reporting pipelines

### 1.1 Entry & Parameterization
Manual start and definition of input fields (target site, engine, source).

### 1.2 SE Ranking Data Retrieval
HTTP request to SE Ranking’s AI Search Overview time-series endpoint.

### 1.3 LLM Summarization (Structured Extraction)
Use LangChain “Information Extractor” node with an OpenAI chat model to produce two summaries from the time-series JSON.

### 1.4 Data Enrichment & Export
Merge the raw time-series with the LLM output, convert JSON to binary, and write it to disk.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Parameterization
**Overview:** Starts the workflow manually and sets the parameters used to query SE Ranking (domain, engine type, country/market source).

**Nodes involved:**
- When clicking ‘Execute workflow’
- Set the Input Fields

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point for interactive runs.
- **Configuration choices:** No parameters; it simply emits a single empty item.
- **Connections:**
  - **Output →** Set the Input Fields
- **Edge cases / failures:** None (only user-driven execution).
- **Version notes:** Type version 1.

#### Node: Set the Input Fields
- **Type / role:** Set node (`n8n-nodes-base.set`) — defines query parameters for downstream nodes.
- **Configuration choices (interpreted):**
  - Sets three string fields:
    - `target_site`: `seranking.com`
    - `engine`: `ai-overview`
    - `source`: `us`
  - Keeps default options (no explicit “keep only set fields” shown; behaves as standard Set node assignments).
- **Key expressions/variables:** Static values in this workflow; referenced later via `$json.target_site`, `$json.engine`, `$json.source`.
- **Connections:**
  - **Input ←** Manual Trigger
  - **Output →** SE Ranking AI Request
- **Edge cases / failures:**
  - Invalid combinations (engine/source/target) won’t fail here but will cause downstream API errors or empty datasets.
- **Version notes:** Type version 3.4.

---

### Block 2 — SE Ranking Data Retrieval
**Overview:** Calls SE Ranking’s AI Search endpoint to retrieve AI visibility time-series for the configured target site and region.

**Nodes involved:**
- SE Ranking AI Request

#### Node: SE Ranking AI Request
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — fetches time-series data from SE Ranking.
- **Configuration choices (interpreted):**
  - **Method:** (not explicitly set in JSON; defaults to GET in n8n HTTP Request node for query-based calls)
  - **URL:** `https://api.seranking.com/v1/ai-search/overview/by-engine/time-series`
  - **Query parameters:**
    - `target` = `={{ $json.target_site }}`
    - `source` = `={{ $json.source }}`
    - `engine` = `={{ $json.engine }}`
  - **Auth:** Uses “generic credential type” with **HTTP Header Auth** selected (`genericAuthType: httpHeaderAuth`).
  - **Retry:** `retryOnFail: true`
- **Credentials used:**
  - `SE Ranking` (HTTP Header Auth) — actually used by this node selection.
  - Also present in the node’s credential list: `Thordata Webscraper API` (HTTP Bearer Auth). This appears configured but **not selected** as the active auth type for this request.
- **Inputs/Outputs:**
  - **Input ←** Set the Input Fields
  - **Output →** SE Ranking AI Summarizer
- **Edge cases / failure modes:**
  - **401/403** if the SE Ranking API key/header is missing/invalid.
  - **429** rate limiting; retries may still fail without backoff tuning.
  - **Non-2xx/timeout** network failures.
  - **Unexpected response shape:** the downstream summarizer expects a `time_series` field; missing/renamed fields will break later expressions.
- **Version notes:** Type version 4.3.

---

### Block 3 — LLM Summarization (Structured Extraction)
**Overview:** Feeds the retrieved `time_series` JSON into an LLM prompt and extracts two structured summary fields (`comprehensive_summary`, `abstract_summary`) using a schema-driven extractor.

**Nodes involved:**
- OpenAI Chat Model
- SE Ranking AI Summarizer

#### Node: OpenAI Chat Model
- **Type / role:** LangChain OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the LLM backend to other LangChain nodes via an **AI connection**.
- **Configuration choices (interpreted):**
  - **Model:** `gpt-4.1-mini`
  - No extra options or built-in tools configured.
- **Credentials:**
  - OpenAI API credential: `OpenAi account`
- **Connections:**
  - **AI languageModel output →** SE Ranking AI Summarizer (as its model provider)
- **Edge cases / failure modes:**
  - Invalid OpenAI key / missing org permissions.
  - Model availability changes, quota limits, or policy/permission constraints.
  - Latency/timeouts on large inputs.
- **Version notes:** Type version 1.3.

#### Node: SE Ranking AI Summarizer
- **Type / role:** Information Extractor (`@n8n/n8n-nodes-langchain.informationExtractor`) — prompts the LLM and parses output into a structured object per the provided JSON schema.
- **Configuration choices (interpreted):**
  - **Text/prompt input:**
    - Prefix instruction: “Use the following JSON to come up with an overview. Provide a human friendly descrptive and comprehensive summary”
    - Injected content: `{{ $json.time_series.toJsonString() }}`
  - **Schema mode:** Manual
  - **Expected output schema:**
    - `comprehensive_summary` (string)
    - `abstract_summary` (string)
  - **Retry:** `retryOnFail: true`
- **Key expressions/variables:**
  - `{{$json.time_series.toJsonString()}}` — requires `time_series` to exist and be serializable.
- **Connections:**
  - **Main input ←** SE Ranking AI Request (contains `time_series`)
  - **AI model input ←** OpenAI Chat Model
  - **Main output →** Enrich Data
- **Edge cases / failure modes:**
  - If `time_series` is null/undefined, `toJsonString()` can fail (expression error).
  - Large `time_series` may exceed model context limits, yielding truncation or extraction failure.
  - The extractor may return malformed output if the model deviates; schema enforcement usually mitigates but can still fail.
- **Version notes:** Type version 1.2.

---

### Block 4 — Data Enrichment & Export
**Overview:** Combines the raw SE Ranking time-series with the extracted summaries, converts it into binary JSON content, and writes it to a fixed file path.

**Nodes involved:**
- Enrich Data
- Create a Binary Data
- Write File to Disk

#### Node: Enrich Data
- **Type / role:** Code node (`n8n-nodes-base.code`) — merges outputs from different nodes into one consolidated JSON object.
- **Configuration choices (interpreted):**
  - JavaScript constructs a single-item array with:
    - `time_series`: taken from `SE Ranking AI Request` first item `json.time_series`
    - `summary`: taken from the current input (the summarizer) as `$input.first().json.output`
- **Key expressions/variables:**
  - `$('SE Ranking AI Request').first().json.time_series` — cross-node access by name.
  - `$input.first().json.output` — depends on extractor’s output field naming (`output`).
- **Connections:**
  - **Input ←** SE Ranking AI Summarizer
  - **Output →** Create a Binary Data
- **Edge cases / failure modes:**
  - If the extractor output is not under `.json.output`, `summary` becomes undefined.
  - If the HTTP node returns no items, `.first()` may throw or return undefined depending on runtime behavior.
- **Operational behavior:**
  - `alwaysOutputData: true` helps downstream nodes still run even if code returns empty (but the code itself can still error if references fail).
- **Version notes:** Type version 2.

#### Node: Create a Binary Data
- **Type / role:** Function node (`n8n-nodes-base.function`) — converts the enriched JSON into a binary field for file writing.
- **Configuration choices (interpreted):**
  - Creates `items[0].binary.data.data` containing base64-encoded JSON string.
  - Uses Node.js `Buffer` to encode JSON (`JSON.stringify(items[0].json, null, 2)`).
- **Connections:**
  - **Input ←** Enrich Data
  - **Output →** Write File to Disk
- **Edge cases / failure modes:**
  - If there are no items (`items[0]` missing), it will throw.
  - Very large JSON could impact memory.
- **Version notes:** Type version 1.

#### Node: Write File to Disk
- **Type / role:** Read/Write File (`n8n-nodes-base.readWriteFile`) — writes the binary data to a local path.
- **Configuration choices (interpreted):**
  - **Operation:** write
  - **File path:** `C:\SERanking_Search.json`
  - **Data property name:** `=data` (expects binary property called `data`)
- **Connections:**
  - **Input ←** Create a Binary Data
  - No downstream nodes.
- **Edge cases / failure modes:**
  - Path is **Windows-specific**; will fail on Linux/Docker unless changed.
  - n8n host may not have permission to write to `C:\`.
  - If running in container, local filesystem may be ephemeral unless mounted.
- **Version notes:** Type version 1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Set the Input Fields | ## **How It Works**  This workflow automates AI-powered search insights by combining SE Ranking AI Search data with OpenAI summarization. It starts with a manual trigger and fetches the time-series AI visibility data via the SE Ranking API. The response is summarized using OpenAI to produce both detailed and concise insights.  ## **Setup Instructions**  1. Import the workflow JSON into your n8n instance. 2. Configure credentials: * **SE Ranking API** via HTTP Header Authentication. * **OpenAI API** for the summarization node. 3. Update the input fields (target site, engine, source) in the “Set the Input Fields” node as needed. 4. Verify the file path in the “Write File to Disk” node matches your environment. 5. Click **Execute Workflow** to run the pipeline.  ## **Customize**  * Change the `target_site`, `engine`, or `source` to analyze different domains, AI engines, or regions. * Adjust the OpenAI prompt or schema to generate different summary formats or additional insights. * Replace the file output node with database, cloud storage, or webhook nodes for integration with dashboards or BI tools. * Schedule the workflow to run periodically for automated SEO and AI search monitoring. |
| Set the Input Fields | Set | Define target/site/engine/source parameters | When clicking ‘Execute workflow’ | SE Ranking AI Request | ## AI Search with SE Ranking  Fetches AI search visibility and time-series data from SE Ranking based on target site and region. Acts as the primary data source for AI-powered search insights. |
| SE Ranking AI Request | HTTP Request | Retrieve SE Ranking AI overview time-series | Set the Input Fields | SE Ranking AI Summarizer | ## AI Search with SE Ranking  Fetches AI search visibility and time-series data from SE Ranking based on target site and region. Acts as the primary data source for AI-powered search insights. |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | Provide GPT-4.1-mini model to extractor | — (model provider) | SE Ranking AI Summarizer (AI connection) | ![Logo](https://media.licdn.com/dms/image/v2/D4D0BAQHBbVpuDD3toA/company-logo_200_200/company-logo_200_200/0/1725976307233/se_ranking_logo?e=1768435200&v=beta&t=_HSGZks62sL6rTXwuo0U21QCKBCNzVT_8OkeIPUr4N8)  OpenAI GPT-4o-mini for the Structured Data Extraction and Data Mining Purposes |
| SE Ranking AI Summarizer | Information Extractor (LangChain) | Summarize time_series into structured fields | SE Ranking AI Request + OpenAI Chat Model | Enrich Data | ## Data Enrichment  Combines raw SE Ranking metrics with OpenAI-generated summaries. Transforms analytical data into human-readable insights. |
| Enrich Data | Code | Merge raw time-series + extracted summaries | SE Ranking AI Summarizer | Create a Binary Data | ## Data Enrichment  Combines raw SE Ranking metrics with OpenAI-generated summaries. Transforms analytical data into human-readable insights. |
| Create a Binary Data | Function | Convert JSON to base64 binary for file write | Enrich Data | Write File to Disk | ## Export Data Handling  Converts enriched results into structured JSON output. Stores the final data for reporting and downstream automation. |
| Write File to Disk | Read/Write File | Persist final JSON to local disk | Create a Binary Data | — | ## Export Data Handling  Converts enriched results into structured JSON output. Stores the final data for reporting and downstream automation. |
| Sticky Note | Sticky Note | Comment block | — | — | ## Data Enrichment  Combines raw SE Ranking metrics with OpenAI-generated summaries. Transforms analytical data into human-readable insights. |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Export Data Handling  Converts enriched results into structured JSON output. Stores the final data for reporting and downstream automation. |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## AI Search with SE Ranking  Fetches AI search visibility and time-series data from SE Ranking based on target site and region. Acts as the primary data source for AI-powered search insights. |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## **How It Works**  This workflow automates AI-powered search insights by combining SE Ranking AI Search data with OpenAI summarization. It starts with a manual trigger and fetches the time-series AI visibility data via the SE Ranking API. The response is summarized using OpenAI to produce both detailed and concise insights.  ## **Setup Instructions**  1. Import the workflow JSON into your n8n instance. 2. Configure credentials: * **SE Ranking API** via HTTP Header Authentication. * **OpenAI API** for the summarization node. 3. Update the input fields (target site, engine, source) in the “Set the Input Fields” node as needed. 4. Verify the file path in the “Write File to Disk” node matches your environment. 5. Click **Execute Workflow** to run the pipeline.  ## **Customize**  * Change the `target_site`, `engine`, or `source` to analyze different domains, AI engines, or regions. * Adjust the OpenAI prompt or schema to generate different summary formats or additional insights. * Replace the file output node with database, cloud storage, or webhook nodes for integration with dashboards or BI tools. * Schedule the workflow to run periodically for automated SEO and AI search monitoring. |
| Sticky Note4 | Sticky Note | Branding/comment block | — | — | ![Logo](https://media.licdn.com/dms/image/v2/D4D0BAQHBbVpuDD3toA/company-logo_200_200/company-logo_200_200/0/1725976307233/se_ranking_logo?e=1768435200&v=beta&t=_HSGZks62sL6rTXwuo0U21QCKBCNzVT_8OkeIPUr4N8)  OpenAI GPT-4o-mini for the Structured Data Extraction and Data Mining Purposes |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **SE Ranking AI Search Overview with Summarizer using OpenAI GPT 4.1-mini** (or your preferred name).

2. **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - Name: **When clicking ‘Execute workflow’**
   - No configuration needed.

3. **Add node: Set**
   - Node type: **Set**
   - Name: **Set the Input Fields**
   - Add fields (String):
     - `target_site` = `seranking.com`
     - `engine` = `ai-overview`
     - `source` = `us`

4. **Connect:** Manual Trigger → Set the Input Fields

5. **Add node: HTTP Request**
   - Node type: **HTTP Request**
   - Name: **SE Ranking AI Request**
   - Configure:
     - URL: `https://api.seranking.com/v1/ai-search/overview/by-engine/time-series`
     - Method: **GET**
     - Enable **Send Query Parameters**
     - Query parameters:
       - `target` = `{{$json.target_site}}`
       - `source` = `{{$json.source}}`
       - `engine` = `{{$json.engine}}`
     - Authentication: **Header Auth** (HTTP Header Authentication)
     - Turn on **Retry on Fail** (optional but matches workflow)

6. **Create credentials: SE Ranking (HTTP Header Auth)**
   - In n8n Credentials, create **HTTP Header Auth** (or the equivalent “Generic” header auth supported by your n8n version).
   - Configure it to send the SE Ranking API key/header required by SE Ranking (commonly an `Authorization: Bearer …` or vendor-specific header, depending on your SE Ranking account/API docs).
   - Select this credential in **SE Ranking AI Request**.

7. **Connect:** Set the Input Fields → SE Ranking AI Request

8. **Add node: OpenAI Chat Model (LangChain)**
   - Node type: **OpenAI Chat Model** (the LangChain LLM node)
   - Name: **OpenAI Chat Model**
   - Model: **gpt-4.1-mini**

9. **Create credentials: OpenAI**
   - Add an **OpenAI API** credential with a valid API key.
   - Select it in **OpenAI Chat Model**.

10. **Add node: Information Extractor (LangChain)**
    - Node type: **Information Extractor**
    - Name: **SE Ranking AI Summarizer**
    - Text field (prompt) set to:
      - `Use the following JSON to come up with an overview. Provide a human friendly descrptive and comprehensive summary`
      - followed by `{{$json.time_series.toJsonString()}}`
    - Schema type: **Manual**
    - Input schema (two string fields):
      - `comprehensive_summary` (string)
      - `abstract_summary` (string)
    - Enable **Retry on Fail** (optional but matches workflow).

11. **Connect (main):** SE Ranking AI Request → SE Ranking AI Summarizer

12. **Connect (AI model):** OpenAI Chat Model → SE Ranking AI Summarizer
    - Use the **AI / languageModel** connection type (not the normal “main” connection).

13. **Add node: Code**
    - Node type: **Code**
    - Name: **Enrich Data**
    - JavaScript:
      - Build one object containing:
        - `time_series` from the HTTP node output
        - `summary` from the extractor output
      - Equivalent logic to:
        - `time_series = $('SE Ranking AI Request').first().json.time_series`
        - `summary = $input.first().json.output`
    - Ensure “Always Output Data” is enabled if you want downstream execution even with minimal results.

14. **Connect:** SE Ranking AI Summarizer → Enrich Data

15. **Add node: Function**
    - Node type: **Function**
    - Name: **Create a Binary Data**
    - Function logic:
      - Base64 encode `items[0].json` and store it under `items[0].binary.data.data`

16. **Connect:** Enrich Data → Create a Binary Data

17. **Add node: Read/Write File**
    - Node type: **Read/Write File**
    - Name: **Write File to Disk**
    - Operation: **Write**
    - File name/path: `C:\SERanking_Search.json` (adjust for your OS/environment)
    - Data property name: `data` (the binary property)

18. **Connect:** Create a Binary Data → Write File to Disk

19. **(Optional) Add sticky notes**
    - Add notes describing blocks (“AI Search with SE Ranking”, “Data Enrichment”, “Export Data Handling”, “How It Works”, branding/logo link) to match documentation and maintainability.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ![Logo](https://media.licdn.com/dms/image/v2/D4D0BAQHBbVpuDD3toA/company-logo_200_200/company-logo_200_200/0/1725976307233/se_ranking_logo?e=1768435200&v=beta&t=_HSGZks62sL6rTXwuo0U21QCKBCNzVT_8OkeIPUr4N8) | Branding image used in a sticky note |
| “OpenAI GPT-4o-mini for the Structured Data Extraction and Data Mining Purposes” | Sticky note text (note: workflow actually uses **gpt-4.1-mini**) |
| File output path is Windows-specific (`C:\SERanking_Search.json`) | Adjust for Linux/Docker (e.g., `/data/SERanking_Search.json`) and ensure write permissions/mounts |
| Extractor prompt includes full `time_series` JSON | If the time-series is large, consider truncating, summarizing upstream, or selecting a shorter date range to avoid context limits |
| HTTP node has an extra credential listed (Thordata Webscraper API) | Present but not active given `httpHeaderAuth` is selected; remove if not needed to reduce confusion |