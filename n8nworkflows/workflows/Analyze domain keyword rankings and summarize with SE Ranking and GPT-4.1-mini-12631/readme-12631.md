Analyze domain keyword rankings and summarize with SE Ranking and GPT-4.1-mini

https://n8nworkflows.xyz/workflows/analyze-domain-keyword-rankings-and-summarize-with-se-ranking-and-gpt-4-1-mini-12631


# Analyze domain keyword rankings and summarize with SE Ranking and GPT-4.1-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Fetch a domain’s keyword ranking dataset from **SE Ranking** (with filters/columns), then use **OpenAI GPT-4.1-mini** (via LangChain nodes) to generate structured summaries. The workflow then aggregates/enriches the output and persists it to multiple destinations (DataTable + Google Sheets) and also prepares a file export.

**Target use cases:** SEO performance monitoring, client/executive reporting, keyword opportunity discovery, competitive analysis, scalable SEO automation.

### 1.1 Input & Parameterization
Defines the target domain, dataset scope (organic/paid), region, limits, filters, and output columns.

### 1.2 SE Ranking Data Retrieval
Calls SE Ranking’s Domain Keywords endpoint with the configured query parameters and authentication.

### 1.3 AI Summarization (LLM + Structured Extraction)
Feeds the SE Ranking JSON into an “information extractor” prompt to produce a structured output with two summary fields.

### 1.4 Enrichment & Consolidation
Extracts both the AI summary and the raw API response, merges them, and aggregates into a single consolidated object for downstream storage.

### 1.5 Persistence & Export
Writes the consolidated result into:
- n8n **DataTable**
- **Google Sheets**
- File pipeline (JSON → binary → file object) for export-ready handling

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Parameterization
**Overview:** Manual start, then sets all query parameters used by the SE Ranking API request (domain, region, filters, selected columns).

**Nodes involved:**
- When clicking ‘Execute workflow’
- Set the Input Fields

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger — entry point for interactive runs.
- **Configuration:** No parameters.
- **Connections:**
  - Output → **Set the Input Fields**
- **Failure modes / edge cases:** None (only user-initiated execution).
- **Version notes:** Node v1.

#### Node: Set the Input Fields
- **Type / role:** Set — constructs a consistent input JSON object used downstream.
- **Key fields set (static defaults):**
  - `target_site`: `seranking.com`
  - `type`: `organic` (switchable to paid if supported by API)
  - `limit`: `100` (string; API likely accepts numeric—see edge cases)
  - `filter`: `sitelinks` (used as `filter[serp_features]`)
  - `source`: `us`
  - `columns`: `keyword,cpc,volume,snippet_title,competition,difficulty,url,total_sites,traffic,traffic_percent,price,serp_features,intents`
- **Expressions/variables:** None (hard-coded values).
- **Connections:**
  - Output → **SE Ranking Request**
- **Failure modes / edge cases:**
  - `limit` is stored as a **string**, which can be accepted by many APIs but may cause validation issues if strict typing is enforced. Consider setting it as number.
  - `columns` must match SE Ranking `cols` allowed values; invalid columns can return API errors.
- **Version notes:** Set node v3.4.

---

### Block 2 — SE Ranking Data Retrieval
**Overview:** Calls SE Ranking Domain Keywords API using the parameters from the Set node.

**Nodes involved:**
- SE Ranking Request

#### Node: SE Ranking Request
- **Type / role:** HTTP Request — fetches domain keyword ranking data from SE Ranking.
- **Endpoint:** `https://api.seranking.com/v1/domain/keywords`
- **Authentication:**
  - Configured as **genericCredentialType → httpHeaderAuth** (named “SE Ranking”).
  - A bearer credential (“Thordata Webscraper API”) is also present in credentials list, but the node is set to header auth; validate which one is actually used in your n8n version/UI.
- **Query parameters (all via expressions):**
  - `source` = `{{$json.source}}`
  - `url` = `{{$json.target_site}}` (SE Ranking uses `url` for domain or exact URL depending on API behavior)
  - `type` = `{{$json.type}}`
  - `limit` = `{{$json.limit}}`
  - `filter[serp_features]` = `{{$json.filter}}`
  - `cols` = `{{$json.columns}}`
- **Runtime options:** `retryOnFail: true`
- **Connections:**
  - Output → **SE Ranking AI Summarizer**
- **Failure modes / edge cases:**
  - Auth failures (401/403) if the header token is missing/expired or header name is wrong.
  - 400 validation errors for invalid `cols`, `type`, `source`, or filter syntax.
  - Large payload/timeouts if `limit` is high or the response includes many columns.
  - Domain vs URL ambiguity: if SE Ranking expects a domain format or a full URL, using `seranking.com` vs `https://seranking.com/` may change results; standardize input format.
- **Version notes:** HTTP Request node v4.3.

---

### Block 3 — AI Summarization (LLM + Structured Extraction)
**Overview:** Uses GPT-4.1-mini to transform raw SE Ranking JSON into a structured pair of summaries (`comprehensive_summary`, `abstract_summary`).

**Nodes involved:**
- OpenAI Chat Model
- SE Ranking AI Summarizer

#### Node: OpenAI Chat Model
- **Type / role:** LangChain Chat Model (OpenAI) — provides the LLM backend.
- **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI API credential (“OpenAi account”).
- **Connections:**
  - AI Language Model output → **SE Ranking AI Summarizer** (as its model provider)
- **Failure modes / edge cases:**
  - OpenAI auth / billing / quota issues.
  - Model availability changes (if `gpt-4.1-mini` name changes in your region/account).
- **Version notes:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` v1.3.

#### Node: SE Ranking AI Summarizer
- **Type / role:** LangChain Information Extractor — prompt + schema-driven extraction.
- **Input text (prompt):**
  - Begins with instruction: “Use the following JSON to come up with an overview…”
  - Injects the entire inbound JSON via: `{{ $json.toJsonString() }}`
- **Extraction schema (manual):**
  - Object with:
    - `comprehensive_summary` (string)
    - `abstract_summary` (string)
- **Operational flags:**
  - `retryOnFail: true`
  - `executeOnce: false`
- **Connections:**
  - Main output → **Extract Summary**
  - Main output → **Extract Raw Response**
- **Failure modes / edge cases:**
  - Token/context overflow if the SE Ranking response is very large (limit/columns too high). Mitigation: reduce `limit`, limit columns, or pre-summarize/trim before LLM.
  - Schema mismatch: if the LLM fails to produce valid structured output, downstream nodes may not find expected properties.
  - Hallucinated or overly generic summaries if prompt doesn’t specify what KPIs/actions to highlight; consider adding guidance (top keywords, traffic share, intents distribution, etc.).
- **Version notes:** `@n8n/n8n-nodes-langchain.informationExtractor` v1.2.

---

### Block 4 — Enrichment & Consolidation
**Overview:** Separates AI output and raw SE Ranking response into distinct fields, merges them, then aggregates into a single object for downstream persistence.

**Nodes involved:**
- Extract Summary
- Extract Raw Response
- Merge
- Aggregate

#### Node: Extract Summary
- **Type / role:** Code node — normalizes the summarizer output into a `summary` field.
- **Logic:** Returns:
  - `summary = $input.first().json.output`
- **Assumption:** The Information Extractor outputs extracted fields under `json.output`.
- **Connections:**
  - Output → **Merge** (input index 0)
- **Failure modes / edge cases:**
  - If the extractor output format changes (e.g., uses `extracted` instead of `output`), `summary` will be undefined.
  - If no input items exist, `$input.first()` can be null; in practice, upstream always sends one item.
- **Version notes:** Code node v2; `alwaysOutputData: true`.

#### Node: Extract Raw Response
- **Type / role:** Code node — captures the raw SE Ranking response for merging.
- **Logic:** Returns:
  - `domain_keywords_response = $('SE Ranking Request').first().json`
- **Key detail:** Uses **node reference** to “SE Ranking Request”, not `$input`.
- **Connections:**
  - Output → **Merge** (input index 1)
- **Failure modes / edge cases:**
  - If “SE Ranking Request” errors or returns no items, `.first()` may fail or return empty.
  - Referencing by node name means renaming nodes can break the expression.
- **Version notes:** Code node v2; `alwaysOutputData: true`.

#### Node: Merge
- **Type / role:** Merge node — combines summary + raw response into one item stream.
- **Mode:** Not explicitly specified; default merge behavior in n8n Merge v3.2 typically merges by position when both inputs have items.
- **Connections:**
  - Output → **Aggregate**
- **Failure modes / edge cases:**
  - If one side returns 0 items, merge output may be empty depending on mode; ensure both branches always emit.
  - If both branches produce multiple items, results can pair incorrectly; here both branches return single-item arrays, so it’s stable.
- **Version notes:** Merge node v3.2.

#### Node: Aggregate
- **Type / role:** Aggregate node — consolidates all item data into a single aggregated structure.
- **Operation:** “aggregateAllItemData” (aggregates all items into one)
- **Connections (fan-out):**
  - Output → **Create a Binary Data**
  - Output → **Persist on DataTable**
  - Output → **Append or update row in sheet**
- **Failure modes / edge cases:**
  - Large merged object can exceed memory limits if upstream payload is huge.
  - If merge produces multiple items unexpectedly, aggregation will wrap them; downstream may store nested arrays rather than a single object.
- **Version notes:** Aggregate node v1.

---

### Block 5 — Persistence & Export
**Overview:** Stores the aggregated result to a DataTable and Google Sheets, and prepares a file export by converting JSON → binary → file.

**Nodes involved:**
- Persist on DataTable
- Append or update row in sheet
- Create a Binary Data
- Convert to File

#### Node: Persist on DataTable
- **Type / role:** DataTable — persists JSON payload as a string in an n8n DataTable.
- **Mapping:**
  - Column `data` = `={{ $json.data.toJsonString() }}`
  - Matching column: `data` (used to decide update vs insert behavior depending on DataTable semantics)
- **Target DataTable:** “DomainKeywordRanking” (ID cached in node)
- **Connections:** None further (terminal branch).
- **Failure modes / edge cases:**
  - DataTable not found / permissions issues in the project.
  - Row matching on a full JSON string is fragile (small ordering/format changes create new rows); consider adding stable keys (domain, run timestamp).
- **Version notes:** DataTable node v1.1.

#### Node: Append or update row in sheet
- **Type / role:** Google Sheets — stores the aggregated JSON as a string in a sheet.
- **Operation:** Append or Update
- **Document:** “Domain Keyword Rankings”
- **Sheet tab:** “Sheet1” (gid=0)
- **Column mapping:**
  - `json_data` = `={{ $json.data.toJsonString() }}`
  - Matching column: `json_data` (again: fragile for matching)
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account”).
- **Connections:** None further (terminal branch).
- **Failure modes / edge cases:**
  - OAuth token expiration/revocation.
  - Sheet structure mismatch (missing `json_data` column header).
  - Matching on full JSON means updates are unlikely; tends to append repeatedly.
- **Version notes:** Google Sheets node v4.7.

#### Node: Create a Binary Data
- **Type / role:** Function node — creates a binary property from the JSON so it can be exported as a file.
- **Logic:** Writes `items[0].binary.data.data` as base64-encoded JSON string of `items[0].json`.
- **Connections:**
  - Output → **Convert to File**
- **Failure modes / edge cases:**
  - Uses legacy `new Buffer(...)` (Node.js deprecations): prefer `Buffer.from(...)` for future compatibility.
  - Assumes at least one item exists (`items[0]`).
- **Version notes:** Function node v1.

#### Node: Convert to File
- **Type / role:** Convert to File — turns binary data into a file object (for email, disk write, S3, etc.).
- **Configuration:** Defaults (no filename/MIME explicitly set).
- **Connections:** None (terminal; no downstream “write file” node is present despite sticky note instructions).
- **Failure modes / edge cases:**
  - Without explicit filename/MIME type, downstream systems (if added later) may not recognize it as JSON.
- **Version notes:** Convert to File node v1.1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Entry point (manual run) | — | Set the Input Fields |  |
| Set the Input Fields | Set | Define SE Ranking query parameters | When clicking ‘Execute workflow’ | SE Ranking Request | ##  SE Ranking Domain Keyword Ranking  \nRetrieves keyword ranking data for a specified domain or exact URL from a regional database. Supports organic and paid results and provides advanced filtering across keyword properties, SERP features, search intents, and traffic metrics. |
| SE Ranking Request | HTTP Request | Call SE Ranking Domain Keywords API | Set the Input Fields | SE Ranking AI Summarizer | ##  SE Ranking Domain Keyword Ranking  \nRetrieves keyword ranking data for a specified domain or exact URL from a regional database. Supports organic and paid results and provides advanced filtering across keyword properties, SERP features, search intents, and traffic metrics. |
| OpenAI Chat Model | LangChain OpenAI Chat Model | Provide GPT-4.1-mini model for extraction | — | SE Ranking AI Summarizer (AI model connection) | ## Data Enrichment  \nCombines raw SE Ranking metrics with OpenAI-generated summaries. Transforms analytical data into human-readable insights. Merge the response from the Summarization + SE Ranking Domain Analysis. Also perform data aggregation for downstream consumption. |
| SE Ranking AI Summarizer | LangChain Information Extractor | Summarize SE Ranking JSON into structured fields | SE Ranking Request + OpenAI Chat Model | Extract Summary; Extract Raw Response | ## Data Enrichment  \nCombines raw SE Ranking metrics with OpenAI-generated summaries. Transforms analytical data into human-readable insights. Merge the response from the Summarization + SE Ranking Domain Analysis. Also perform data aggregation for downstream consumption. |
| Extract Summary | Code | Normalize LLM output into `summary` | SE Ranking AI Summarizer | Merge | ## Data Enrichment  \nCombines raw SE Ranking metrics with OpenAI-generated summaries. Transforms analytical data into human-readable insights. Merge the response from the Summarization + SE Ranking Domain Analysis. Also perform data aggregation for downstream consumption. |
| Extract Raw Response | Code | Capture raw SE Ranking response into `domain_keywords_response` | SE Ranking AI Summarizer (but references SE Ranking Request) | Merge | ## Data Enrichment  \nCombines raw SE Ranking metrics with OpenAI-generated summaries. Transforms analytical data into human-readable insights. Merge the response from the Summarization + SE Ranking Domain Analysis. Also perform data aggregation for downstream consumption. |
| Merge | Merge | Combine summary + raw response | Extract Summary; Extract Raw Response | Aggregate | ## Data Enrichment  \nCombines raw SE Ranking metrics with OpenAI-generated summaries. Transforms analytical data into human-readable insights. Merge the response from the Summarization + SE Ranking Domain Analysis. Also perform data aggregation for downstream consumption. |
| Aggregate | Aggregate | Consolidate into one payload for storage/export | Merge | Create a Binary Data; Persist on DataTable; Append or update row in sheet | ## Data Enrichment  \nCombines raw SE Ranking metrics with OpenAI-generated summaries. Transforms analytical data into human-readable insights. Merge the response from the Summarization + SE Ranking Domain Analysis. Also perform data aggregation for downstream consumption. |
| Create a Binary Data | Function | Convert JSON to base64 binary | Aggregate | Convert to File | ## Export Data Handling  \nConverts enriched results into structured JSON output. Stores the final data for reporting and downstream automation. |
| Convert to File | Convert to File | Produce file object from binary data | Create a Binary Data | — | ## Export Data Handling  \nConverts enriched results into structured JSON output. Stores the final data for reporting and downstream automation. |
| Persist on DataTable | DataTable | Persist aggregated JSON to n8n DataTable | Aggregate | — | ## Export Data Handling  \nConverts enriched results into structured JSON output. Stores the final data for reporting and downstream automation. |
| Append or update row in sheet | Google Sheets | Store aggregated JSON in Google Sheet | Aggregate | — |  |
| Sticky Note | Sticky Note | Documentation / grouping | — | — | ## Data Enrichment  \nCombines raw SE Ranking metrics with OpenAI-generated summaries. Transforms analytical data into human-readable insights. Merge the response from the Summarization + SE Ranking Domain Analysis. Also perform data aggregation for downstream consumption. |
| Sticky Note1 | Sticky Note | Documentation / grouping | — | — | ## Export Data Handling  \nConverts enriched results into structured JSON output. Stores the final data for reporting and downstream automation. |
| Sticky Note2 | Sticky Note | Documentation / grouping | — | — | ##  SE Ranking Domain Keyword Ranking  \nRetrieves keyword ranking data for a specified domain or exact URL from a regional database. Supports organic and paid results and provides advanced filtering across keyword properties, SERP features, search intents, and traffic metrics. |
| Sticky Note3 | Sticky Note | Documentation / how it works | — | — | ## **How It Works**  \nThis workflow analyzes keyword ranking performance for a domain using SE Ranking and enriches the results with AI-generated summaries. It begins with a manual trigger where the target domain, search type (organic or paid), region, result limits, and keyword filters are defined.  \n\nThese parameters are sent to the **SE Ranking Domain Keywords API** to retrieve detailed keyword ranking data, including positions, search volume, CPC, competition, SERP features, intents, and estimated traffic.\n\n## **Setup Instructions**\n1. Configure credentials:\n   * **SE Ranking API** using HTTP Header Authentication\n   * **OpenAI API** for GPT-4.1-mini\n2. Update the **Set the Input Fields** node with your domain, region, filters, and limits.\n3. Verify the output file path in the **Write File to Disk** node.\n4. Click **Execute Workflow** to generate keyword ranking insights.\n\n## **Customize**\n* Switch between organic and paid keyword analysis.\n* Modify columns, filters, or limits to focus on specific SEO metrics.\n* Adjust the OpenAI prompt to generate competitive insights or action items.\n* Replace file export with dashboards, databases, or alerting workflows. |
| Sticky Note4 | Sticky Note | Documentation / use cases | — | — | ## Usecases  \n- SEO performance monitoring\n- Executive and client reporting\n- Keyword opportunity discovery\n- Competitive domain analysis\n- Scalable SEO automation |
| Sticky Note5 | Sticky Note | Branding image | — | — | ![Logo](https://s3-eu-west-1.amazonaws.com/tpd/logos/538f1575000064000578dee6/0x0.png) |
| Sticky Note6 | Sticky Note | Documentation / output | — | — | ## Output  \n- Structured keyword ranking data\n-  AI-generated summaries and insights\n-  Export-ready JSON for reporting and automation |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: “Domain Keyword Ranking + AI Summarization with SE Ranking + OpenAI GPT 4.1-mini” (or your preferred name).
   - (Optional) Add tags: Engineering, Building Blocks, AI, SERP.

2. **Add trigger**
   - Add node: **Manual Trigger**
   - Name: “When clicking ‘Execute workflow’”

3. **Add parameter setup node**
   - Add node: **Set**
   - Name: “Set the Input Fields”
   - Add fields (recommended types in parentheses):
     - `target_site` (string): e.g., `seranking.com`
     - `type` (string): `organic` (or `paid` if you want paid results)
     - `limit` (number recommended): `100`
     - `filter` (string): e.g., `sitelinks`
     - `source` (string): e.g., `us`
     - `columns` (string): `keyword,cpc,volume,snippet_title,competition,difficulty,url,total_sites,traffic,traffic_percent,price,serp_features,intents`
   - Connect: **Manual Trigger → Set**

4. **Configure SE Ranking API request**
   - Add node: **HTTP Request**
   - Name: “SE Ranking Request”
   - Method: GET (default when only URL/query is used)
   - URL: `https://api.seranking.com/v1/domain/keywords`
   - Enable **Send Query Parameters**
   - Add query params (use expressions):
     - `source` = `{{$json.source}}`
     - `url` = `{{$json.target_site}}`
     - `type` = `{{$json.type}}`
     - `limit` = `{{$json.limit}}`
     - `filter[serp_features]` = `{{$json.filter}}`
     - `cols` = `{{$json.columns}}`
   - Authentication:
     - Choose **Header Auth** (HTTP Header Authentication).
     - Create credential “SE Ranking” that sends the required API token header (as per SE Ranking API docs; commonly a Bearer token or an API key header).
   - Connect: **Set → HTTP Request**
   - Turn on **Retry on Fail** (optional but matches the workflow).

5. **Add OpenAI model provider**
   - Add node: **OpenAI Chat Model** (LangChain)
   - Name: “OpenAI Chat Model”
   - Model: `gpt-4.1-mini`
   - Credentials: Configure **OpenAI API** credential (API key).
   - This node will connect via the special **AI Language Model** connection to the extractor node.

6. **Add the summarization/extraction node**
   - Add node: **Information Extractor** (LangChain)
   - Name: “SE Ranking AI Summarizer”
   - Text / Prompt:
     - Use a prompt similar to:
       - `Use the following JSON to come up with an overview. Provide a human friendly descrptive and comprehensive summary`
       - Then include the JSON: `{{ $json.toJsonString() }}`
   - Schema type: **Manual**
   - Schema (object):
     - `comprehensive_summary`: string
     - `abstract_summary`: string
   - Enable **Retry on Fail** (recommended).
   - Connect:
     - **HTTP Request (main) → Information Extractor (main)**
     - **OpenAI Chat Model (ai_languageModel) → Information Extractor (ai_languageModel)**

7. **Extract summary into a stable field**
   - Add node: **Code**
   - Name: “Extract Summary”
   - Code:
     - Set `summary` to the extractor output (commonly `json.output` in this workflow):
       - `return [{ summary: $input.first().json.output }];`
   - Connect: **Information Extractor → Extract Summary**

8. **Extract raw API response**
   - Add node: **Code**
   - Name: “Extract Raw Response”
   - Code (node reference approach):
     - `return [{ domain_keywords_response: $('SE Ranking Request').first().json }];`
   - Connect: **Information Extractor → Extract Raw Response**
   - Note: If you rename “SE Ranking Request”, update this reference.

9. **Merge summary + raw response**
   - Add node: **Merge**
   - Name: “Merge”
   - Connect:
     - **Extract Summary → Merge (Input 1)**
     - **Extract Raw Response → Merge (Input 2)**

10. **Aggregate into one consolidated payload**
    - Add node: **Aggregate**
    - Name: “Aggregate”
    - Operation: **Aggregate All Item Data**
    - Connect: **Merge → Aggregate**

11. **Persist into n8n DataTable**
    - Add node: **DataTable**
    - Name: “Persist on DataTable”
    - Select/create a DataTable (e.g., “DomainKeywordRanking”).
    - Map a column `data` to:
      - `={{ $json.data.toJsonString() }}`
    - Connect: **Aggregate → DataTable**

12. **Persist into Google Sheets**
    - Add node: **Google Sheets**
    - Name: “Append or update row in sheet”
    - Credentials: Google Sheets OAuth2.
    - Operation: **Append or update**
    - Pick document and sheet/tab.
    - Ensure a column header exists: `json_data`
    - Map:
      - `json_data` = `={{ $json.data.toJsonString() }}`
    - Connect: **Aggregate → Google Sheets**

13. **Prepare exportable file (optional branch)**
    - Add node: **Function**
    - Name: “Create a Binary Data”
    - Use safer Buffer usage:
      - Convert JSON to a binary property named `data`:
        - `items[0].binary = { data: { data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64') } }; return items;`
    - Connect: **Aggregate → Create a Binary Data**
    - Add node: **Convert to File**
    - Name: “Convert to File”
    - Connect: **Create a Binary Data → Convert to File**
    - (Optional) Add a “Write Binary File” / S3 / Email node afterwards; the provided workflow ends here.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Logo image | https://s3-eu-west-1.amazonaws.com/tpd/logos/538f1575000064000578dee6/0x0.png |
| Sticky note mentions verifying a “Write File to Disk” node, but none exists in the current workflow | If you need a real disk export, add “Write Binary File” after “Convert to File” and set a filesystem path (self-hosted n8n only). |
| Matching rows in Google Sheets/DataTable by full JSON string is fragile | Prefer adding stable keys like `domain`, `source`, `type`, `run_date` and match on those instead. |
| Risk of LLM context overflow when sending full API JSON | Reduce `limit`, reduce `cols`, or pre-aggregate metrics before summarization. |