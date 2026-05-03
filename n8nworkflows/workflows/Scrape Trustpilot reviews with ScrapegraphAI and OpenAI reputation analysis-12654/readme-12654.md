Scrape Trustpilot reviews with ScrapegraphAI and OpenAI reputation analysis

https://n8nworkflows.xyz/workflows/scrape-trustpilot-reviews-with-scrapegraphai-and-openai-reputation-analysis-12654


# Scrape Trustpilot reviews with ScrapegraphAI and OpenAI reputation analysis

## 1. Workflow Overview

**Title:** Scrape Trustpilot reviews with ScrapegraphAI and OpenAI reputation analysis  
**Workflow name (in JSON):** Automated Trustpilot Review Scraper & Reputation Analysis

**Purpose:**  
This workflow collects the most recent Trustpilot reviews for a given company, extracts structured fields from each review using an LLM, runs sentiment classification, generates a sentiment pie chart, then produces an AI-generated **company-level** reputation report and emails it as HTML.

**Typical use cases:**
- Reputation monitoring for a brand/service (not a single product)
- Periodic executive reporting (Marketing, Support, Operations, Product)
- Quickly summarizing themes, risks, and actions from fresh Trustpilot feedback

### 1.1 Logical Blocks

1. **Input & Parameters**
   - Manual start; user defines Trustpilot company identifier and max pages to scrape.
2. **Trustpilot Scraping (Index Pages → Review Links)**
   - Fetch review listing pages with pagination; parse HTML to collect review page URLs.
3. **Per-Review Extraction (Review Page → Structured Fields)**
   - Convert each review page to Markdown (JS-rendered); extract author/rating/date/title/text/etc via LLM.
4. **Sentiment Classification & Normalization**
   - Classify review text into Positive/Neutral/Negative; map into normalized fields for downstream use.
5. **Visualization (Sentiment Pie Chart)**
   - Aggregate sentiment counts; build QuickChart config; request chart from QuickChart.
6. **Company-Level Reputation Report & Email Delivery**
   - Aggregate all processed reviews; agent generates structured management report; convert to HTML; send via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Parameters

**Overview:**  
Starts the workflow manually and sets the two primary controls: the Trustpilot company identifier and the number of review listing pages to scrape.

**Nodes involved:**
- When clicking ‘Test workflow’
- Set Parameters
- Sticky Note (STEP 1)
- Sticky Note1 (workflow description + setup)
- Sticky Note9 (YouTube promo)

#### Node: When clicking ‘Test workflow’
- **Type / role:** Manual Trigger; entry point for ad-hoc execution.
- **Key configuration:** No parameters.
- **Outputs:** Sends a single empty item to **Set Parameters**.
- **Failure modes:** None (unless workflow is disabled or n8n UI execution is blocked).

#### Node: Set Parameters
- **Type / role:** Set node; defines variables used by the HTTP pagination request.
- **Configuration choices:**
  - Sets `company_id` (string): placeholder `TRUSTPILOT_COMOPANY_ID` (note spelling).
  - Sets `max_page` (number): `2` by default.
- **Key expressions/variables:** Downstream uses `$json.company_id` and `$json.max_page`.
- **Connections:**
  - Input: Manual Trigger
  - Output: Get reviews
- **Edge cases / failures:**
  - If `company_id` is not replaced with a real Trustpilot slug/path, scraping fails (404 or incorrect HTML).
  - Non-numeric `max_page` breaks pagination expression evaluation or yields unexpected paging.

#### Sticky Note (STEP 1)
- Content:
  - “## STEP 1 - Setup  
    Setup The Trustpilot company identifier and The maximum number of review pages to scrape”
- Applies visually to the parameter setup area (documented per-node in the summary table).

#### Sticky Note1 (overall description + setup)
- Contains full workflow description and setup guidance (credentials, Limit node, email config).

#### Sticky Note9 (promo)
- Content includes links and an image preview:
  - `https://youtube.com/@n3witalia`
  - Image: `https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg`

---

### Block 2 — Trustpilot Scraping (Index Pages → Review Links)

**Overview:**  
Fetches Trustpilot review listing pages (sorted by recency) with pagination, then extracts review detail page links from the HTML.

**Nodes involved:**
- Get reviews
- Extract
- Split Out
- Sticky Note2 (STEP 2)

#### Node: Get reviews
- **Type / role:** HTTP Request; downloads Trustpilot review listing pages.
- **Configuration choices:**
  - **URL:** `https://it.trustpilot.com/review/{{ $json.company_id }}`
  - **Query:** `sort=recency`
  - **Pagination (n8n built-in pagination):**
    - Adds query parameter `page = {{$pageCount + 1}}`
    - `maxRequests = {{$json.max_page}}`
    - `requestInterval = 5000ms`
    - “limitPagesFetched” enabled
- **Connections:**
  - Input: Set Parameters
  - Output: Extract
- **Edge cases / failures:**
  - Trustpilot may block/ratelimit scraping (403/429), change markup, or require additional headers/cookies.
  - Locale is fixed to `it.trustpilot.com`; a non-Italian company page may redirect and alter selectors.
  - Pagination assumes page indexing starts at 1; if Trustpilot changes behavior, results may repeat or skip.

#### Node: Extract
- **Type / role:** HTML node (extractHtmlContent); parses listing page HTML and extracts link attributes.
- **Configuration choices:**
  - Extracts key `recensioni` from selector: `article section a`
  - Returns **href attribute**; returns an **array**
- **Connections:**
  - Input: Get reviews (HTML content)
  - Output: Split Out
- **Edge cases / failures:**
  - CSS selector may match extra links (non-review links) or miss review links if Trustpilot markup changes.
  - If page is blocked (captcha/consent), extracted hrefs may be empty or irrelevant.

#### Node: Split Out
- **Type / role:** Split Out; turns an array field into multiple items (one per review link).
- **Configuration choices:** Splits field `recensioni`.
- **Connections:**
  - Input: Extract
  - Output: Limit
- **Edge cases / failures:**
  - If `recensioni` is missing or not an array, node can error or output zero items depending on n8n behavior.

#### Sticky Note2 (STEP 2)
- Content:
  - “## STEP 2 - Automated Review Scraping  
    Reviews are fetched directly from Trustpilot, ordered by recency. Pagination is handled automatically to respect the configured page limit.”

---

### Block 3 — Per-Review Extraction (Review Page → Structured Fields)

**Overview:**  
Iterates each review URL, converts the review page into clean Markdown using ScrapegraphAI (with JS rendering), then uses an LLM-based extractor to output structured review fields.

**Nodes involved:**
- Limit
- Loop Over Items
- Convert a webpage or article to clean markdown useful for blogs dev docs and more1 (ScrapegraphAI)
- OpenAI Chat Model1
- Information Extractor1
- Sticky Note3 (STEP 3)

#### Node: Limit
- **Type / role:** Limit; caps number of reviews processed per run (cost/rate-limit control).
- **Configuration choices:** `maxItems = 5`
- **Connections:**
  - Input: Split Out
  - Output: Loop Over Items
- **Edge cases / failures:** None; but can hide issues by processing too few items.

#### Node: Loop Over Items
- **Type / role:** Split In Batches; controls per-item processing and also provides a “done” output branch.
- **Configuration choices:** Defaults (batch size not explicitly set in JSON).
- **Connections (important):**
  - **Output 1 (items batch):** → ScrapegraphAI markdownify node
  - **Output 0 (when finished / aggregated path in this workflow):** → Set vars for chart **and** Aggregate reviews
- **Edge cases / failures:**
  - Misunderstanding of outputs can lead to charts/reports running before all items are processed if wired incorrectly. Here, the “done” branch is used for aggregation/charting.
  - If batch size defaults to 1, it will loop per item; if changed, downstream nodes must handle batch arrays.

#### Node: Convert a webpage or article to clean markdown useful for blogs dev docs and more1
- **Type / role:** ScrapegraphAI node (`markdownify`); fetches page and converts to Markdown.
- **Configuration choices:**
  - **Website URL:** `https://it.trustpilot.com{{ $json.recensioni }}`
  - **renderHeavyJs:** enabled (important for dynamic content).
  - Requires ScrapegraphAI API credentials.
- **Connections:**
  - Input: Loop Over Items (per review href)
  - Output: Information Extractor1
- **Failure modes / edge cases:**
  - API auth errors (invalid key, quota exceeded).
  - Heavy JS rendering is slower and more failure-prone (timeouts).
  - If `$json.recensioni` is already an absolute URL, concatenation will break (double host).
  - Trustpilot bot protections may reduce content quality or block access.

#### Node: OpenAI Chat Model1
- **Type / role:** LangChain Chat Model (OpenAI); provides the LLM for Information Extractor.
- **Configuration choices:**
  - Model: `gpt-5-mini`
  - Credentials: OpenAI account
- **Connections:**
  - Connected via `ai_languageModel` to **Information Extractor1**
- **Failure modes:**
  - OpenAI auth/billing/quota errors.
  - Model availability differences across regions/accounts.

#### Node: Information Extractor1
- **Type / role:** LangChain Information Extractor; extracts structured fields from Markdown.
- **Configuration choices:**
  - **Input text prompt:** `You need to extract the review from the following MD: {{ $json.result }}`
  - **System prompt:** review expert; “extract only required information without changing anything”
  - **Schema attributes (all required):**
    - `autore` (author name)
    - `valutazione` (number 1–5)
    - `data` (YYYY-MM-DD)
    - `titolo`
    - `testo`
    - `n_recensioni` (number)
    - `nazione` (two-character country code)
- **Connections:**
  - Input: ScrapegraphAI markdownify output
  - Output: Sentiment Analysis1
  - Uses OpenAI Chat Model1 via `ai_languageModel`
- **Edge cases / failures:**
  - If Trustpilot page layout changes, Markdown may omit fields; extractor may fail validation due to required fields.
  - Date format normalization may fail (non-parseable locale strings).
  - Country “two characters” may not be present; may cause extraction failure or hallucination risk (prompt says don’t change anything, but required fields force a value).
  - Large Markdown can cause token limit issues.

#### Sticky Note3 (STEP 3)
- Content:
  - “## STEP 3 - Content Extraction & Sentiment Analysis  
    Each review page is converted into clean Markdown, including content rendered via JavaScript.  
    Each review’s text is analyzed and classified as Positive, Neutral, or Negative.  
    Results are normalized into consistent, machine-readable fields.”

---

### Block 4 — Sentiment Classification & Normalization

**Overview:**  
Runs sentiment analysis on the extracted review text, then maps relevant fields into a simplified, consistent structure used by downstream aggregation and reporting.

**Nodes involved:**
- OpenAI Chat Model2
- Sentiment Analysis1
- Set fields

#### Node: OpenAI Chat Model2
- **Type / role:** LangChain Chat Model (OpenAI); model provider for sentiment node.
- **Configuration choices:**
  - Model: `gpt-5-mini` (string form)
- **Connections:**
  - Provides `ai_languageModel` to Sentiment Analysis1
- **Failure modes:** Same as other OpenAI model nodes (auth/quota/model availability).

#### Node: Sentiment Analysis1
- **Type / role:** LangChain Sentiment Analysis; classifies text.
- **Configuration choices:**
  - **Input text:** `{{ $json.output.testo }}`
  - Categories: `Positive, Neutral, Negative`
  - System prompt: “Only output the JSON.”
- **Connections:**
  - Input: Information Extractor1
  - Output: Set fields (note: JSON shows three parallel main outputs all going to Set fields; functionally they converge to the same next node)
  - Uses OpenAI Chat Model2 via `ai_languageModel`
- **Edge cases / failures:**
  - If `output.testo` is missing (extractor failure), expression resolves to empty → poor classification or error.
  - Model may return unexpected schema; node expects structured JSON.

#### Node: Set fields
- **Type / role:** Set node; normalizes final per-review record.
- **Configuration choices (fields created):**
  - `title` = `{{$json.output.titolo}}`
  - `description` = `{{$json.output.testo}}`
  - `vote` = `{{$json.output.valutazione}}`
  - `sentiment` = `{{$json.sentimentAnalysis.category}}`
- **Connections:**
  - Input: Sentiment Analysis1
  - Output: Loop Over Items (feeds back for next batch / loop continuation)
- **Edge cases / failures:**
  - If sentiment output path differs (e.g., `sentimentAnalysis` absent), `sentiment` becomes null.
  - Type coercion: `vote` expects number; if extractor returns a string, it may coerce or error depending on n8n version.

---

### Block 5 — Visualization (Sentiment Pie Chart)

**Overview:**  
After all items are processed, aggregates sentiment counts and generates a QuickChart pie chart configuration, then calls QuickChart to render it.

**Nodes involved:**
- Set vars for chart
- QuickChart
- Sticky Note5 (STEP 5)

#### Node: Set vars for chart
- **Type / role:** Code node; computes sentiment distribution and builds chart config JSON.
- **Logic highlights:**
  - Iterates `items` and counts `item.json.sentiment`
  - Color mapping:
    - Positive `#4CAF50`
    - Negative `#F44336`
    - Neutral `#FFC107`
    - default `#9E9E9E`
  - Builds a QuickChart config for a pie chart and returns:
    - `labels`, `data`, `colors`
    - `chartConfig` as JSON string
- **Connections:**
  - Input: Loop Over Items (done branch)
  - Output: QuickChart
- **Edge cases / failures:**
  - If sentiment values differ in capitalization/spelling, they’ll fall into default color and separate slices.
  - If no items exist, chart config will contain empty arrays.

#### Node: QuickChart
- **Type / role:** HTTP Request; calls QuickChart endpoint.
- **Configuration choices:**
  - URL: `https://quickchart.io/chart`
  - Query parameter: `c={{$json.chartConfig}}`
- **Connections:**
  - Input: Set vars for chart
  - Output: (none connected in JSON)
- **Edge cases / failures:**
  - URL length limits: large configs can exceed practical URL/query size (usually safe for small pie charts).
  - To embed image in email, you’d typically use QuickChart image URL or fetch binary; currently not wired into the email/report chain.

#### Sticky Note5 (STEP 5)
- Content:
  - “## STEP 5 - Visualization  
    A dynamic pie chart is generated using QuickChart for immediate visual insight.”

---

### Block 6 — Company-Level Reputation Report & Email Delivery

**Overview:**  
Collects all normalized reviews, asks an AI agent to produce a structured management report focused on the company, converts the result to HTML, and emails it.

**Nodes involved:**
- Aggregate reviews
- OpenAI Chat Model
- Company Reputation Analyst
- HTML Converter
- Send a message
- Sticky Note4 (STEP 4)

#### Node: Aggregate reviews
- **Type / role:** Code node; aggregates all incoming items into a single JSON array.
- **Configuration choices / logic:**
  - `const reviews = $input.all().map(item => item.json);`
  - Outputs `{ reviews }` as a single item
- **Connections:**
  - Input: Loop Over Items (done branch)
  - Output: Company Reputation Analyst
- **Edge cases / failures:**
  - If upstream normalized fields are incomplete, the agent receives partial review objects.
  - If many reviews are processed, the JSON string can exceed model context limits downstream.

#### Node: OpenAI Chat Model
- **Type / role:** LangChain Chat Model (OpenAI); model provider for the agent and the HTML converter chain.
- **Configuration choices:**
  - Model: `gpt-5-mini`
- **Connections:**
  - `ai_languageModel` to Company Reputation Analyst
  - `ai_languageModel` to HTML Converter
- **Failure modes:** OpenAI auth/quota/model availability.

#### Node: Company Reputation Analyst
- **Type / role:** LangChain Agent; generates the management report with strict structure.
- **Configuration choices:**
  - **Input text:** `Reviews: {{JSON.stringify($json.reviews)}}`
  - **System message:** long, strict constraints:
    - Company-level analysis only
    - Remove HTML from review field if present
    - No invention; mask emails; professional style
    - Mandatory output structure (7 sections) and calculation rules
- **Connections:**
  - Input: Aggregate reviews
  - Output: HTML Converter
  - Uses OpenAI Chat Model via `ai_languageModel`
- **Edge cases / failures:**
  - Token/context overflow if too many reviews or long texts are included.
  - If reviews don’t include fields the agent expects (sentiment/vote), it will have to report missing info.
  - “review field may contain HTML” rule: in this workflow the normalized field is `description` (plain text). If HTML slips in, agent should handle it.

#### Node: HTML Converter
- **Type / role:** LangChain “chainLlm”; converts the agent output into HTML.
- **Configuration choices:**
  - **Text:** `{{$json.output}}` (expects agent output in `output`)
  - **Prompt:** “Translate into HTML… only the HTML, not the opening tags "html\n" and the closing \n.”
- **Connections:**
  - Input: Company Reputation Analyst
  - Output: Send a message
  - Uses OpenAI Chat Model via `ai_languageModel`
- **Edge cases / failures:**
  - If the agent output isn’t found at `$json.output`, HTML conversion will be empty.
  - The prompt asks not to include `<html>` wrapper; email clients may still need full HTML structure (optional improvement).

#### Node: Send a message
- **Type / role:** Gmail node; emails the final HTML report.
- **Configuration choices:**
  - To: `YOUR_EMAIL` (placeholder)
  - Subject: `Company review`
  - Message body: `{{$json.text}}` (expects HTML Converter output in `text`)
  - Credentials: Gmail OAuth2
- **Connections:**
  - Input: HTML Converter
- **Edge cases / failures:**
  - Gmail OAuth token expiration / missing scopes.
  - If message is HTML but node sends as plain text (depends on Gmail node options); you may need “HTML” option depending on n8n’s Gmail node capabilities/version.
  - Placeholder email must be replaced.

#### Sticky Note4 (STEP 4)
- Content:
  - “## STEP 4 - AI-Powered Reputation Report  
    A specialized AI agent analyzes all reviews at a company level, not product level.  
    It generates a structured management report  
    The report is automatically sent via email to stakeholders.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Test workflow’ | Manual Trigger | Manual entry point | — | Set Parameters |  |
| Set Parameters | Set | Define `company_id` and `max_page` | When clicking ‘Test workflow’ | Get reviews | ## STEP 1 - Setup<br>Setup The Trustpilot company identifier and The maximum number of review pages to scrape |
| Get reviews | HTTP Request | Fetch Trustpilot listing pages with pagination | Set Parameters | Extract | ## STEP 2 - Automated Review Scraping<br>Reviews are fetched directly from Trustpilot, ordered by recency. Pagination is handled automatically to respect the configured page limit. |
| Extract | HTML (extractHtmlContent) | Extract review links (`href`) into array | Get reviews | Split Out | ## STEP 2 - Automated Review Scraping<br>Reviews are fetched directly from Trustpilot, ordered by recency. Pagination is handled automatically to respect the configured page limit. |
| Split Out | Split Out | Split `recensioni` array into items | Extract | Limit | ## STEP 2 - Automated Review Scraping<br>Reviews are fetched directly from Trustpilot, ordered by recency. Pagination is handled automatically to respect the configured page limit. |
| Limit | Limit | Cap number of reviews processed | Split Out | Loop Over Items |  |
| Loop Over Items | Split In Batches | Iterate reviews; provide “done” branch | Limit; Set fields (feedback) | Convert a webpage… (batch); Set vars for chart (done); Aggregate reviews (done) |  |
| Convert a webpage or article to clean markdown useful for blogs dev docs and more1 | ScrapegraphAI (markdownify) | Convert each review page to clean Markdown (JS-rendered) | Loop Over Items (batch output) | Information Extractor1 | ## STEP 3 - Content Extraction & Sentiment Analysis<br>Each review page is converted into clean Markdown, including content rendered via JavaScript.<br>Each review’s text is analyzed and classified as Positive, Neutral, or Negative.<br>Results are normalized into consistent, machine-readable fields. |
| OpenAI Chat Model1 | OpenAI Chat Model (LangChain) | LLM provider for structured extraction | — | Information Extractor1 (ai_languageModel) |  |
| Information Extractor1 | Information Extractor (LangChain) | Extract author/rating/date/title/text/etc from Markdown | Convert a webpage… | Sentiment Analysis1 | ## STEP 3 - Content Extraction & Sentiment Analysis<br>Each review page is converted into clean Markdown, including content rendered via JavaScript.<br>Each review’s text is analyzed and classified as Positive, Neutral, or Negative.<br>Results are normalized into consistent, machine-readable fields. |
| OpenAI Chat Model2 | OpenAI Chat Model (LangChain) | LLM provider for sentiment | — | Sentiment Analysis1 (ai_languageModel) |  |
| Sentiment Analysis1 | Sentiment Analysis (LangChain) | Classify review text into categories | Information Extractor1 | Set fields | ## STEP 3 - Content Extraction & Sentiment Analysis<br>Each review page is converted into clean Markdown, including content rendered via JavaScript.<br>Each review’s text is analyzed and classified as Positive, Neutral, or Negative.<br>Results are normalized into consistent, machine-readable fields. |
| Set fields | Set | Normalize fields: title/description/vote/sentiment | Sentiment Analysis1 | Loop Over Items |  |
| Set vars for chart | Code | Count sentiments and build QuickChart pie config | Loop Over Items (done output) | QuickChart | ## STEP 5 - Visualization<br>A dynamic pie chart is generated using QuickChart for immediate visual insight. |
| QuickChart | HTTP Request | Render sentiment chart via QuickChart | Set vars for chart | — | ## STEP 5 - Visualization<br>A dynamic pie chart is generated using QuickChart for immediate visual insight. |
| Aggregate reviews | Code | Merge all processed reviews into one array | Loop Over Items (done output) | Company Reputation Analyst | ## STEP 4 - AI-Powered Reputation Report<br>A specialized AI agent analyzes all reviews at a company level, not product level.<br>It generates a structured management report<br>The report is automatically sent via email to stakeholders. |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM provider for agent + HTML conversion | — | Company Reputation Analyst (ai_languageModel); HTML Converter (ai_languageModel) | ## STEP 4 - AI-Powered Reputation Report<br>A specialized AI agent analyzes all reviews at a company level, not product level.<br>It generates a structured management report<br>The report is automatically sent via email to stakeholders. |
| Company Reputation Analyst | Agent (LangChain) | Generate structured company reputation report | Aggregate reviews | HTML Converter | ## STEP 4 - AI-Powered Reputation Report<br>A specialized AI agent analyzes all reviews at a company level, not product level.<br>It generates a structured management report<br>The report is automatically sent via email to stakeholders. |
| HTML Converter | Chain LLM (LangChain) | Convert report text to HTML | Company Reputation Analyst | Send a message | ## STEP 4 - AI-Powered Reputation Report<br>A specialized AI agent analyzes all reviews at a company level, not product level.<br>It generates a structured management report<br>The report is automatically sent via email to stakeholders. |
| Send a message | Gmail | Email the HTML report to stakeholders | HTML Converter | — | ## STEP 4 - AI-Powered Reputation Report<br>A specialized AI agent analyzes all reviews at a company level, not product level.<br>It generates a structured management report<br>The report is automatically sent via email to stakeholders. |
| Sticky Note | Sticky Note | Comment block (Step 1) | — | — | ## STEP 1 - Setup<br>Setup The Trustpilot company identifier and The maximum number of review pages to scrape |
| Sticky Note1 | Sticky Note | Comment block (overview + setup) | — | — | # Automated Trustpilot Review Scraper using ScrapegraphAI & Reputation Analysis<br>This workflow automates the **collection, analysis, and reporting of Trustpilot reviews**… |
| Sticky Note2 | Sticky Note | Comment block (Step 2) | — | — | ## STEP 2 - Automated Review Scraping<br>Reviews are fetched directly from Trustpilot… |
| Sticky Note3 | Sticky Note | Comment block (Step 3) | — | — | ## STEP 3 - Content Extraction & Sentiment Analysis<br>Each review page is converted… |
| Sticky Note4 | Sticky Note | Comment block (Step 4) | — | — | ## STEP 4 - AI-Powered Reputation Report<br>A specialized AI agent analyzes… |
| Sticky Note5 | Sticky Note | Comment block (Step 5) | — | — | ## STEP 5 - Visualization<br>A dynamic pie chart is generated… |
| Sticky Note9 | Sticky Note | Comment block (promo) | — | — | ## MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new **YouTube channel**](https://youtube.com/@n3witalia)…<br>[![image](https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg)](https://youtube.com/@n3witalia) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add “Manual Trigger”** node  
   - Name: `When clicking ‘Test workflow’`
3. **Add “Set”** node  
   - Name: `Set Parameters`  
   - Add fields:
     - `company_id` (String): your Trustpilot company slug used in `/review/<slug>` (e.g., `example.com`)
     - `max_page` (Number): e.g., `2`
   - Connect: Manual Trigger → Set Parameters
4. **Add “HTTP Request”** node  
   - Name: `Get reviews`  
   - URL: `https://it.trustpilot.com/review/{{$json.company_id}}`  
   - Enable **Send Query Parameters**:
     - `sort` = `recency`
   - **Pagination**:
     - Add parameter `page` = `{{$pageCount + 1}}`
     - Max requests = `{{$json.max_page}}`
     - Request interval = `5000` ms
     - Enable “limit pages fetched” (or equivalent toggle in your n8n version)
   - Connect: Set Parameters → Get reviews
5. **Add “HTML”** node  
   - Name: `Extract`  
   - Operation: extract HTML content  
   - Extraction:
     - Key: `recensioni`
     - CSS selector: `article section a`
     - Return value: `attribute`
     - Attribute: `href`
     - Return array: enabled
   - Connect: Get reviews → Extract
6. **Add “Split Out”** node  
   - Name: `Split Out`  
   - Field to split out: `recensioni`  
   - Connect: Extract → Split Out
7. **Add “Limit”** node  
   - Name: `Limit`  
   - Max items: `5` (adjust for cost)  
   - Connect: Split Out → Limit
8. **Add “Split In Batches”** node  
   - Name: `Loop Over Items`  
   - Keep defaults (or set Batch Size = 1 for clear per-review looping)  
   - Connect: Limit → Loop Over Items
9. **Add ScrapegraphAI node** (community/installed package: `n8n-nodes-scrapegraphai`)  
   - Name: `Convert a webpage or article to clean markdown useful for blogs dev docs and more1`  
   - Resource/operation: `markdownify`  
   - Website URL: `https://it.trustpilot.com{{$json.recensioni}}`  
   - Enable `renderHeavyJs`  
   - **Credentials:** create/attach `ScrapegraphAI account` API key  
   - Connect: Loop Over Items (batch output) → ScrapegraphAI node
10. **Add OpenAI Chat Model (LangChain)**  
    - Name: `OpenAI Chat Model1`  
    - Model: `gpt-5-mini`  
    - **Credentials:** configure OpenAI API credential
11. **Add “Information Extractor” (LangChain)**  
    - Name: `Information Extractor1`  
    - Text: `You need to extract the review from the following MD: {{$json.result}}`  
    - System prompt: (as in workflow) review expert; don’t change content  
    - Define attributes (all required):
      - `autore` (string)
      - `valutazione` (number)
      - `data` (string, YYYY-MM-DD)
      - `titolo` (string)
      - `testo` (string)
      - `n_recensioni` (number)
      - `nazione` (string, 2 chars)
    - Connect model: OpenAI Chat Model1 → Information Extractor1 (AI Language Model connection)
    - Connect: ScrapegraphAI → Information Extractor1
12. **Add OpenAI Chat Model (LangChain)**  
    - Name: `OpenAI Chat Model2`  
    - Model: `gpt-5-mini`  
    - Credentials: same OpenAI credential
13. **Add “Sentiment Analysis” (LangChain)**  
    - Name: `Sentiment Analysis1`  
    - Input text: `{{$json.output.testo}}`  
    - Categories: `Positive, Neutral, Negative`  
    - System prompt: ensure “Only output the JSON.”  
    - Connect model: OpenAI Chat Model2 → Sentiment Analysis1 (AI Language Model connection)  
    - Connect: Information Extractor1 → Sentiment Analysis1
14. **Add “Set”** node  
    - Name: `Set fields`  
    - Create:
      - `title` = `{{$json.output.titolo}}`
      - `description` = `{{$json.output.testo}}`
      - `vote` = `{{$json.output.valutazione}}`
      - `sentiment` = `{{$json.sentimentAnalysis.category}}`
    - Connect: Sentiment Analysis1 → Set fields
15. **Close the loop** for batching  
    - Connect: Set fields → Loop Over Items (so the batch iterator continues)
16. **Visualization branch (runs after loop completes)**  
    - Add **Code** node: `Set vars for chart` with the provided logic (count sentiments + build pie config).  
    - Add **HTTP Request** node: `QuickChart`  
      - URL: `https://quickchart.io/chart`  
      - Query parameter `c` = `{{$json.chartConfig}}`  
    - Connect: Loop Over Items (done output) → Set vars for chart → QuickChart
17. **Reporting branch (runs after loop completes)**  
    - Add **Code** node: `Aggregate reviews`  
      - Code: aggregate `$input.all()` into `{reviews: [...]}`  
    - Connect: Loop Over Items (done output) → Aggregate reviews
18. **Add OpenAI Chat Model (LangChain)**  
    - Name: `OpenAI Chat Model`  
    - Model: `gpt-5-mini`
19. **Add “Agent” (LangChain)**  
    - Name: `Company Reputation Analyst`  
    - Text: `Reviews: {{JSON.stringify($json.reviews)}}`  
    - System message: paste the full company-focused report constraints + mandatory structure from the workflow  
    - Connect model: OpenAI Chat Model → Company Reputation Analyst (AI Language Model connection)  
    - Connect: Aggregate reviews → Company Reputation Analyst
20. **Add “Chain LLM” (LangChain)**  
    - Name: `HTML Converter`  
    - Text: `{{$json.output}}`  
    - Prompt (define): “Translate into HTML… only the HTML…”  
    - Connect model: OpenAI Chat Model → HTML Converter (AI Language Model connection)  
    - Connect: Company Reputation Analyst → HTML Converter
21. **Add “Gmail”** node  
    - Name: `Send a message`  
    - To: your recipient address  
    - Subject: `Company review` (or customize)  
    - Message: `{{$json.text}}`  
    - **Credentials:** Gmail OAuth2 (connect Google account; ensure Gmail scopes)  
    - Connect: HTML Converter → Send a message
22. **Add Sticky Notes** (optional but matches the original)  
    - Create notes for STEP 1..5 and the overview/promo content; position them near relevant nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “# Automated Trustpilot Review Scraper using ScrapegraphAI & Reputation Analysis …” (full setup + explanation) | Sticky Note1 (embedded in workflow canvas) |
| “## MY NEW YOUTUBE CHANNEL 👉 Subscribe…” plus image link | https://youtube.com/@n3witalia (Sticky Note9) |
| Image preview used in sticky note | https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.