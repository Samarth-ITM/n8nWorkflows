Generate content ideas from social trends with Apify, Gemini, and Google Sheets

https://n8nworkflows.xyz/workflows/generate-content-ideas-from-social-trends-with-apify--gemini--and-google-sheets-13525


# Generate content ideas from social trends with Apify, Gemini, and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Trend2Content  
**Purpose:** Generate content ideas (blog title ideas + tweet hooks) from current “top” social posts for a user-provided topic, using **Apify (X/Twitter scraping)**, **Google Gemini (via LangChain AI Agent)**, and **Google Sheets (storage)**.

### 1.1 Input Reception (Topic capture)
Collects a single input (`Topic`) via an n8n Form trigger.

### 1.2 Trend Ingestion (Scrape + normalize)
Calls an Apify Actor to fetch top posts for the topic, then normalizes each returned item into a single text field called `Content`.

### 1.3 Aggregation + AI Generation (Combine → Agent → Structured output)
Aggregates multiple scraped posts into a consolidated input, sends it to a LangChain AI Agent powered by **Google Gemini**, and enforces a structured JSON response via a structured output parser.

### 1.4 Formatting + Persistence (Format → Append to Sheets)
Formats arrays (titles/hooks) into bullet-like multi-line strings and appends a new row into a Google Sheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Topic capture)

**Overview:** Receives the user’s topic from a simple form. This topic becomes the query used by Apify and is also injected into the AI system message for contextual generation.

**Nodes involved:**
- **On form submission** (Form Trigger)

#### Node: On form submission
- **Type / role:** `n8n-nodes-base.formTrigger` — Entry point; hosts an n8n Form and triggers executions on submit.
- **Key configuration (interpreted):**
  - Form title: **“Trend Extraction and AI Content Generation”**
  - One required field:
    - Label: **Topic**
    - Placeholder: **Web3**
- **Key variables/expressions:**
  - Produces `{{$json.Topic}}` (used downstream in the Apify request and AI system message via cross-node reference).
- **Connections:**
  - **Output →** `X Scraper`
- **Edge cases / failure types:**
  - Empty topic prevented by required field, but user input may still be too broad/narrow (leading to no scrape results).
  - If the form is not reachable (n8n URL / webhook exposure), submissions won’t trigger.
- **Version-specific notes:** Type version `2.3` (Form trigger behavior can differ across n8n versions; ensure your instance supports Form Trigger nodes).

---

### Block 2 — Trend Ingestion (Scrape + normalize)

**Overview:** Sends the topic to Apify to scrape top X posts, then maps the returned text into a clean `Content` field per item.

**Nodes involved:**
- **X Scraper** (HTTP Request)
- **Edit Fields** (Set)

#### Node: X Scraper
- **Type / role:** `n8n-nodes-base.httpRequest` — Calls Apify Actor run endpoint and returns dataset items synchronously.
- **Key configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.apify.com/v2/acts/nfp1fpt5gUlBwPcor/run-sync-get-dataset-items`
  - **Body (JSON):**
    - `maxItems: 20`
    - `searchTerms: ["{{ $json.Topic }}"]`
    - `sort: "Top"`
  - **Headers:**
    - `Accept: application/json`
    - `Authorization: Bearer <API Key>` (must be replaced with a real Apify token; ideally stored in n8n credentials/env vars)
- **Key variables/expressions:**
  - Uses the form output: `{{ $json.Topic }}`
- **Connections:**
  - **Input ←** `On form submission`
  - **Output →** `Edit Fields`
- **Edge cases / failure types:**
  - **401/403** if Apify token invalid/expired.
  - **429** rate limiting if Apify quota exceeded.
  - **Timeout/large payload** if actor run is slow or returns large items.
  - Schema drift: returned items may not contain `fullText` (breaking the next Set node).
- **Version-specific notes:** Type version `4.2` (HTTP node options/fields vary by version).

#### Node: Edit Fields
- **Type / role:** `n8n-nodes-base.set` — Normalizes scraped items to a single field used for aggregation.
- **Key configuration (interpreted):**
  - Creates/overwrites:
    - `Content = {{$json.fullText}}`
- **Connections:**
  - **Input ←** `X Scraper`
  - **Output →** `Aggregate`
- **Edge cases / failure types:**
  - If Apify response items lack `fullText`, `Content` becomes empty/null; downstream AI input quality degrades.
  - Non-string `fullText` values could cause unexpected formatting.
- **Version-specific notes:** Type version `3.4`.

---

### Block 3 — Aggregation + AI Generation (Combine → Agent → Structured output)

**Overview:** Combines multiple scraped posts into one aggregated payload, then uses a LangChain Agent (powered by Google Gemini) to generate structured content ideas. A structured output parser enforces a predictable JSON schema.

**Nodes involved:**
- **Aggregate**
- **Google Gemini Chat Model**
- **Structured Output Parser**
- **AI Agent**

#### Node: Aggregate
- **Type / role:** `n8n-nodes-base.aggregate` — Combines multiple incoming items into a single item containing aggregated fields.
- **Key configuration (interpreted):**
  - Aggregates the field: `Content`
  - Output becomes one item where `Content` is a collection of all `Content` values (n8n aggregate output shape can be array-like; the AI Agent then receives it as text via expression).
- **Connections:**
  - **Input ←** `Edit Fields`
  - **Output →** `AI Agent`
- **Edge cases / failure types:**
  - If there are **0 items** (no scrape results), aggregation may output empty content or no item (depending on node behavior/version).
  - If `Content` is very large, the AI call may exceed model limits.
- **Version-specific notes:** Type version `1`.

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — Provides the LLM backend for the agent.
- **Key configuration (interpreted):**
  - Uses Google Gemini credentials (`googlePalmApi`).
  - Default model/options (no explicit parameters shown; relies on node defaults).
- **Connections:**
  - **AI language model output →** `AI Agent` (special LangChain connection type: `ai_languageModel`)
- **Edge cases / failure types:**
  - Auth misconfiguration (API key/OAuth) → request failures.
  - Model availability/region limitations.
  - Safety filters can refuse some outputs depending on content.
- **Version-specific notes:** Type version `1`.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Forces the AI to return valid JSON matching a schema.
- **Key configuration (interpreted):**
  - Schema example requires:
    - `topic_summary` (string)
    - `blog_post_titles` (array of strings)
    - `tweet_hooks` (array of strings)
- **Connections:**
  - **AI output parser output →** `AI Agent` (special LangChain connection type: `ai_outputParser`)
- **Edge cases / failure types:**
  - Model may return non-JSON or schema-mismatching JSON → parser/agent errors.
  - Missing keys (`tweet_hooks`, etc.) breaks downstream Code node expectations.
- **Version-specific notes:** Type version `1.3`.

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Orchestrates prompt + LLM + parser to generate structured content ideas.
- **Key configuration (interpreted):**
  - **Input text:** `={{ $json.Content }}`
  - **System message:** Dynamic, specializing the assistant on the submitted topic, referencing the trigger node explicitly:
    - `{{ $('On form submission').item.json.Topic }}`
  - **Prompt mode:** “define” (custom instructions provided)
  - **Output parser enabled:** yes (`hasOutputParser: true`)
- **Connections:**
  - **Main input ←** `Aggregate`
  - **AI language model ←** `Google Gemini Chat Model`
  - **AI output parser ←** `Structured Output Parser`
  - **Main output →** `Code in JavaScript`
- **Edge cases / failure types:**
  - If the cross-node reference `$('On form submission').item.json.Topic` is missing (e.g., partial executions, manual runs without trigger data), the system message may fail or be empty.
  - Large aggregated content can cause token/context overflow.
  - Parser mismatch leads to failure or incomplete `output`.
- **Version-specific notes:** Type version `2.2`.

---

### Block 4 — Formatting + Persistence (Format → Append to Sheets)

**Overview:** Converts the structured arrays returned by the AI into multi-line strings, then appends them into a Google Sheet.

**Nodes involved:**
- **Code in JavaScript**
- **Append row in sheet**

#### Node: Code in JavaScript
- **Type / role:** `n8n-nodes-base.code` — Transforms the AI agent result into a shape suitable for Google Sheets.
- **Key configuration (interpreted):**
  - JavaScript returns:
    - `topic_summary: $json.output.topic_summary`
    - `blog_post_titles: $json.output.blog_post_titles.join('\n• ')`
    - `tweet_hooks: $json.output.tweet_hooks.join('\n• ')`
  - This effectively creates bullet-style lines (note: the first entry will not start with `• ` unless you prepend it separately).
- **Connections:**
  - **Input ←** `AI Agent`
  - **Output →** `Append row in sheet`
- **Edge cases / failure types:**
  - If `output` is missing (AI/parsing failure), this will throw (cannot read properties of undefined).
  - If `blog_post_titles` or `tweet_hooks` are not arrays, `.join()` will throw.
- **Version-specific notes:** Type version `2`.

#### Node: Append row in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — Appends a new row with generated ideas.
- **Key configuration (interpreted):**
  - **Operation:** Append
  - **Spreadsheet:** “Trend2Content” (document ID `1ewthiWelucJgbn1V3xizT_eeQ0gROcaHaJr0WNLX5sE`)
  - **Sheet tab:** “Sheet1” (gid=0)
  - **Columns mapped:**
    - `blog_post_titles = {{$json.blog_post_titles}}`
    - `tweet_hooks = {{$json.tweet_hooks}}`
  - Type conversion disabled (`attemptToConvertTypes: false`)
- **Connections:**
  - **Input ←** `Code in JavaScript`
  - **Output:** none (end of workflow)
- **Credentials:**
  - Google Sheets OAuth2 (`Google Sheets account`)
- **Edge cases / failure types:**
  - OAuth expired/insufficient permissions → auth errors.
  - Sheet column headers must match exactly (`blog_post_titles`, `tweet_hooks`) or mapping may fail / write to wrong places.
  - Append can fail if sheet is protected or quota limits reached.
- **Version-specific notes:** Type version `4.7`.

---

## 3. Summary Table

> Sticky notes in this workflow act as documentation overlays. The large “Trend2Content” note describes the whole pipeline; section notes label the three major stages.

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Collect `Topic` from user and start workflow | — | X Scraper | # Trend2Content … (pipeline + links) \| # Input & Ingest (Trigger → Scrape → Normalize) |
| X Scraper | HTTP Request | Call Apify actor to fetch top X posts for the topic | On form submission | Edit Fields | # Trend2Content … (pipeline + links) \| # Input & Ingest (Trigger → Scrape → Normalize) |
| Edit Fields | Set | Map scraped `fullText` to `Content` | X Scraper | Aggregate | # Trend2Content … (pipeline + links) \| # Input & Ingest (Trigger → Scrape → Normalize) |
| Aggregate | Aggregate | Combine many `Content` items into one aggregated input | Edit Fields | AI Agent | # Trend2Content … (pipeline + links) \| # Aggregate & Generate (Combine → LangChain Agent → Parser) |
| Google Gemini Chat Model | LangChain LLM (Google Gemini) | Provide the LLM used by the AI Agent | — | AI Agent (ai_languageModel) | # Trend2Content … (pipeline + links) \| # Aggregate & Generate (Combine → LangChain Agent → Parser) |
| Structured Output Parser | LangChain Structured Parser | Enforce JSON schema for AI output | — | AI Agent (ai_outputParser) | # Trend2Content … (pipeline + links) \| # Aggregate & Generate (Combine → LangChain Agent → Parser) |
| AI Agent | LangChain Agent | Generate summary + titles + hooks from aggregated posts | Aggregate | Code in JavaScript | # Trend2Content … (pipeline + links) \| # Aggregate & Generate (Combine → LangChain Agent → Parser) |
| Code in JavaScript | Code | Format arrays into multi-line strings for Sheets | AI Agent | Append row in sheet | # Trend2Content … (pipeline + links) \| # Format, Store & Observe (JS Format → Google Sheets → Notes) |
| Append row in sheet | Google Sheets | Append formatted output to Google Sheet | Code in JavaScript | — | # Trend2Content … (pipeline + links) \| # Format, Store & Observe (JS Format → Google Sheets → Notes) |
| Sticky Note | Sticky Note | Documentation (overview + links) | — | — | # Trend2Content (contains: Demo & Setup Video, Sheet Template, Course links) |
| Sticky Note1 | Sticky Note | Section label | — | — | # Input & Ingest (Trigger → Scrape → Normalize) |
| Sticky Note2 | Sticky Note | Section label | — | — | # Aggregate & Generate (Combine → LangChain Agent → Parser) |
| Sticky Note3 | Sticky Note | Section label | — | — | # Format, Store & Observe (JS Format → Google Sheets → Notes) |

**Sticky note “Trend2Content” content (verbatim links preserved):**
- Demo & Setup Video: https://drive.google.com/file/d/1HeypybNWR-AHu1Odtt0kgcsK-uRPB0JP/view?usp=sharing  
- Sheet Template: https://docs.google.com/spreadsheets/d/1ewthiWelucJgbn1V3xizT_eeQ0gROcaHaJr0WNLX5sE/edit?usp=sharing  
- Course: https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC  

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n
   - Name it: **Trend2Content**
   - (Optional) Set execution order to default (`v1`), as in the original.

2. **Add trigger: “On form submission”**
   - Node type: **Form Trigger**
   - Form title: `Trend Extraction and AI Content Generation`
   - Add one field:
     - Label: `Topic`
     - Placeholder: `Web3`
     - Required: enabled
   - This node is the main entry point.

3. **Add “X Scraper” (HTTP Request)**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL: `https://api.apify.com/v2/acts/nfp1fpt5gUlBwPcor/run-sync-get-dataset-items`
   - Body content type: **JSON**
   - JSON body:
     - `maxItems`: `20`
     - `searchTerms`: array with the topic from the trigger, e.g. `["{{ $json.Topic }}"]`
     - `sort`: `"Top"`
   - Headers:
     - `Accept: application/json`
     - `Authorization: Bearer <YOUR_APIFY_API_TOKEN>`
   - Connect: **On form submission → X Scraper**

4. **Add “Edit Fields” (Set)**
   - Node type: **Set**
   - Add field:
     - Name: `Content`
     - Value expression: `{{$json.fullText}}`
   - Connect: **X Scraper → Edit Fields**

5. **Add “Aggregate”**
   - Node type: **Aggregate**
   - Configure “Fields to aggregate” to include:
     - `Content`
   - Connect: **Edit Fields → Aggregate**

6. **Add “Google Gemini Chat Model”**
   - Node type: **Google Gemini Chat Model** (LangChain LLM node)
   - Credentials:
     - Create/select **Google Gemini API** credentials (listed as `googlePalmApi` in this workflow).
   - Keep default options unless you need a specific model/version.

7. **Add “Structured Output Parser”**
   - Node type: **Structured Output Parser** (LangChain)
   - Provide a JSON schema example requiring:
     - `topic_summary` (string)
     - `blog_post_titles` (string array)
     - `tweet_hooks` (string array)

8. **Add “AI Agent”**
   - Node type: **AI Agent** (LangChain Agent)
   - Set **Text/Input** to: `{{$json.Content}}`
   - Set **System Message** (dynamic) to reference the trigger’s topic, e.g.:
     - `You are a helpful AI assistant specialized in {{ $('On form submission').item.json.Topic }} content creation...`
   - Enable/attach the structured output parser.
   - Connect special LangChain ports:
     - **Google Gemini Chat Model → AI Agent** via **ai_languageModel**
     - **Structured Output Parser → AI Agent** via **ai_outputParser**
   - Connect main flow:
     - **Aggregate → AI Agent**

9. **Add “Code in JavaScript”**
   - Node type: **Code**
   - Paste code (adapt as desired):
     - Reads `output.topic_summary`, `output.blog_post_titles`, `output.tweet_hooks`
     - Joins arrays with `\n• ` for multi-line formatting
   - Connect: **AI Agent → Code in JavaScript**

10. **Add “Append row in sheet” (Google Sheets)**
   - Node type: **Google Sheets**
   - Credentials: create/select **Google Sheets OAuth2**
   - Operation: **Append**
   - Choose the target spreadsheet (or use the provided template):
     - Template link: https://docs.google.com/spreadsheets/d/1ewthiWelucJgbn1V3xizT_eeQ0gROcaHaJr0WNLX5sE/edit?usp=sharing
   - Select the sheet/tab: `Sheet1`
   - Map columns:
     - `blog_post_titles` → `{{$json.blog_post_titles}}`
     - `tweet_hooks` → `{{$json.tweet_hooks}}`
   - Connect: **Code in JavaScript → Append row in sheet**

11. **(Optional) Add Sticky Notes**
   - Add notes to label the three sections and include the links (video/template/course) for operators.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Demo & Setup Video | https://drive.google.com/file/d/1HeypybNWR-AHu1Odtt0kgcsK-uRPB0JP/view?usp=sharing |
| Sheet Template | https://docs.google.com/spreadsheets/d/1ewthiWelucJgbn1V3xizT_eeQ0gROcaHaJr0WNLX5sE/edit?usp=sharing |
| Course | https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC |
| Concept summary | “Topic → Trending Content → AI Ideas → Sheet” (end-to-end ideation pipeline) |