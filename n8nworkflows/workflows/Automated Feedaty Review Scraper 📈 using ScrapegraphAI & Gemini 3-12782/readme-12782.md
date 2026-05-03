Automated Feedaty Review Scraper 📈 using ScrapegraphAI & Gemini 3

https://n8nworkflows.xyz/workflows/automated-feedaty-review-scraper----using-scrapegraphai---gemini-3-12782


# Automated Feedaty Review Scraper 📈 using ScrapegraphAI & Gemini 3

## 1. Workflow Overview

**Purpose:** Automatically scrape the latest customer reviews from a **Feedaty** company profile, run **per-review sentiment classification** (Positive/Neutral/Negative) with **Gemini 3**, aggregate all reviews, generate a **management-ready reputation report**, convert it into **HTML → PDF** via **ConvertAPI**, and **upload the PDF to Google Drive**.

**Primary use cases:** periodic reputation monitoring, executive reporting, CX audits, stakeholder distribution via shared Drive folder.

### 1.1 Input & Configuration
Manual trigger + parameters defining *which Feedaty company* and *how many pages* to scrape.

### 1.2 Review Scraping (ScrapeGraphAI)
ScrapeGraphAI extracts review objects (date/vote/text/id) across multiple pages.

### 1.3 Review Normalization, Limiting, and Per-Review Sentiment
Flatten results into individual review items, optionally limit volume, loop per item, run sentiment analysis, and normalize fields.

### 1.4 Company-Level Reputation Report (AI Agent)
Aggregate all enriched reviews, then a dedicated agent produces a structured company-level report.

### 1.5 Rendering, PDF Conversion, and Google Drive Delivery
Convert the report text into HTML, wrap it in a full HTML document, convert to PDF, then upload to a configured Drive folder.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Setup

**Overview:** Starts the workflow manually and sets the company identifier and pagination limit used downstream by the scraper.  
**Nodes involved:**  
- When clicking ‘Test workflow’  
- Set Parameters  

#### Node: When clicking ‘Test workflow’
- **Type / role:** `Manual Trigger` (entry point)
- **Configuration:** No parameters; execution starts on manual test.
- **Outputs:** 1 item to **Set Parameters**
- **Failure/edge cases:** None (manual-only). In production, you’d replace with Cron/Webhook.

#### Node: Set Parameters
- **Type / role:** `Set` (creates configuration fields)
- **Key fields set:**
  - `company_id` (string): currently `"XXX"` (must be replaced, e.g. `maxisport`)
  - `max_page` (number): `2`
- **Key expressions:** none (static values)
- **Input:** From Manual Trigger  
- **Output:** To ScrapeGraphAI node
- **Edge cases / failures:**
  - `company_id` left as `"XXX"` → scraper likely returns empty/404 page content.
  - `max_page` too high → longer runtime, higher API cost, possible rate limits/timeouts.

---

### Block 2 — Automated Review Scraping (ScrapeGraphAI)

**Overview:** Scrapes Feedaty reviews from the company page with pagination enabled, enforcing a target output schema.  
**Nodes involved:**  
- Autonomously extract live data from any website perfect for e commerce job boards lead capture and more (ScrapeGraphAI)
- Get Result

#### Node: Autonomously extract live data from any website perfect for e commerce job boards lead capture and more
- **Type / role:** `scrapegraphAi` (ScrapeGraphAI extraction + pagination)
- **Configuration choices (interpreted):**
  - **websiteUrl:** `https://www.feedaty.com/recensioni/{{company_id}}`
  - **totalPages:** from `{{$json.max_page}}`
  - **enablePagination:** enabled
  - **userPrompt:** extract all user reviews with:
    - date (converted to `dd/MM/yyyy`)
    - score/vote
    - review text
    - review id from an anchor tag pattern: `<a href="#" name="12133072"></a>`
  - **useOutputSchema:** true, with array schema requiring `date`, `vote`, `review` (and optional `id`)
- **Credentials:** ScrapeGraphAI API credential required
- **Input:** From **Set Parameters**
- **Output:** To **Get Result**
- **Edge cases / failures:**
  - Auth failure (invalid/expired ScrapeGraphAI key)
  - Feedaty page layout changes → schema extraction breaks or fields missing
  - Pagination mismatch (pages not discoverable) → fewer reviews than expected
  - Date conversion may fail if Feedaty locale/date format changes

#### Node: Get Result
- **Type / role:** `Set` (re-maps scraper output into a predictable field)
- **Configuration:** sets:
  - `result` (array) = `{{$json.result.pages}}`
- **Input:** From ScrapeGraphAI node  
- **Output:** To **Extract reviews**
- **Edge cases / failures:**
  - If ScrapeGraphAI response does not include `result.pages`, expression evaluates to `undefined`, downstream code may produce no items.

---

### Block 3 — Review Flattening, Limiting, and Sentiment Enrichment

**Overview:** Flattens the paginated scraping output into individual review items, optionally limits count, loops over reviews, runs sentiment classification with Gemini, and normalizes fields into a consistent structure.  
**Nodes involved:**  
- Extract reviews (Code)  
- Limit reviews  
- Loop Over Items (Split in Batches)  
- Sentiment Analysis1  
- Google Gemini Chat Model2  
- Set fields  

#### Node: Extract reviews
- **Type / role:** `Code` (flatten/normalize raw scrape output)
- **Logic (interpreted):**
  - Iterates over all incoming items.
  - Expects each item’s `json.result` to be an array of “pages”.
  - For each page, tries to read reviews from:
    - `page.result.recensioni` **or** `page.result.reviews`
  - Pushes each review object into `allReviews`
  - Outputs **one n8n item per review**
- **Input:** From **Get Result**
- **Output:** To **Limit reviews**
- **Edge cases / failures:**
  - If `data.result` is not an array → outputs empty.
  - If the actual structure differs (e.g., `result.pages` already flattened) → may miss reviews.
  - Mixed languages: node checks both Italian `recensioni` and English `reviews`, which is helpful but still assumes nested `page.result`.

#### Node: Limit reviews
- **Type / role:** `Limit` (throttle the number of review items processed)
- **Configuration:** `maxItems: 5`
- **Input:** From **Extract reviews**
- **Output:** To **Loop Over Items**
- **Edge cases / failures:** None; but limiting can bias analysis (non-representative sample).

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` (iterate per review)
- **Configuration:** default batch settings (not explicitly set)
- **Connections (important):**
  - Main output goes to **Sentiment Analysis1**
  - Also connected to **Aggregate reviews** (see caveat below)
- **Input:** From **Limit reviews**
- **Output:** To **Sentiment Analysis1** and **Aggregate reviews**
- **Edge cases / failures:**
  - Misconfigured batch size could affect performance; defaults are usually fine for small review volumes.
  - **Design caveat:** In this workflow, **Aggregate reviews** is connected directly to the loop output while sentiment enrichment happens in a parallel branch. Depending on execution timing and how n8n merges items, the aggregation may receive items **before** sentiment fields are added (see “Aggregate reviews” notes).

#### Node: Sentiment Analysis1
- **Type / role:** `Sentiment Analysis` (LangChain node) using an attached LLM
- **Model binding:** Receives its language model from **Google Gemini Chat Model2** via `ai_languageModel` connection
- **Configuration choices:**
  - **Categories:** `Positive, Neutral, Negative`
  - **System prompt:** strict sentiment analyzer, “Only output the JSON.”
  - **inputText expression:**
    ```
    Vote: {{$json.vote}}/5
    Review: {{$json.review}}
    ```
- **Input:** Each review item
- **Output:** To **Set fields**
- **Edge cases / failures:**
  - Missing `$json.vote` or `$json.review` → weak classification or node error depending on runtime.
  - LLM may return invalid JSON despite instructions; downstream expressions referencing `sentimentAnalysis.category` may fail.
  - Reviews with non-text/HTML noise may reduce accuracy.

#### Node: Google Gemini Chat Model2
- **Type / role:** `lmChatGoogleGemini` (LLM provider for sentiment)
- **Configuration:** model not specified here (defaults apply)
- **Credentials:** Google Gemini (PaLM) API credential required
- **Output connection:** Provides `ai_languageModel` to **Sentiment Analysis1**
- **Edge cases / failures:** invalid key, quota exceeded, model availability changes.

#### Node: Set fields
- **Type / role:** `Set` (normalizes fields for downstream reporting)
- **Fields:**
  - `date` = `{{$json.date}}`
  - `description` = `{{$json.review}}`
  - `vote` = `{{$json.vote}}`
  - `sentiment` = `{{$json.sentimentAnalysis.category}}`
- **Input:** From **Sentiment Analysis1**
- **Output:** Back to **Loop Over Items**
- **Edge cases / failures:**
  - If `sentimentAnalysis.category` missing (invalid JSON from model or node failure), sentiment becomes empty and report logic may classify as “Other” or fail expectations.
  - Note the workflow’s report agent expects `review` field, but this node stores review text as `description` (see next block for impact).

---

### Block 4 — Aggregation & Company Reputation Report (AI Agent)

**Overview:** Aggregates all (ideally enriched) reviews into an array and prompts a dedicated agent to produce a strictly structured reputation report at the company level.  
**Nodes involved:**  
- Aggregate reviews (Code)  
- Company Reputation Management (Agent)  
- Google Gemini Chat Model (LLM for agent)

#### Node: Aggregate reviews
- **Type / role:** `Code` (collects all incoming items into one array)
- **Logic:** `const reviews = $input.all().map(item => item.json);` then outputs `{ reviews }`
- **Input:** From **Loop Over Items**
- **Output:** To **Company Reputation Management**
- **Important integration caveats:**
  - **Race/branching risk:** Because aggregation is connected to the loop directly while enrichment happens through **Sentiment Analysis1 → Set fields → Loop Over Items**, the aggregation may capture items that **don’t include `sentiment`** (and/or may still have `review` instead of `description` depending on which branch wins).  
  - If you intend the agent to always see sentiment + normalized fields, the safer pattern is: **Limit → SplitInBatches → Sentiment → Set fields → (collect) Aggregate reviews**.
- **Edge cases / failures:**
  - Large review counts → huge JSON payload → LLM context limits or slow execution.

#### Node: Company Reputation Management
- **Type / role:** `LangChain Agent`
- **Model binding:** Uses **Google Gemini Chat Model** via `ai_languageModel` connection
- **Input text expression:**
  - `Reviews: {{ JSON.stringify($json.reviews) }}`
- **System message:** Long, strict instruction set enforcing:
  - company-level focus (not product-level)
  - no invention/assumption, mask emails
  - mandatory 7-section structure (Executive Summary → Next Steps)
  - calculation rules (percent rounding, etc.)
  - handle sentiments outside Positive/Neutral/Negative as “Other”
- **Output:** To **HTML Converter**
- **Edge cases / failures:**
  - If reviews array items don’t contain the expected keys (`review`, `sentiment`, `vote`), the agent may report “missing data” extensively.
  - Very large input array can exceed model context or raise costs.
  - If review text contains HTML, the agent is instructed to strip tags; results depend on model compliance.

#### Node: Google Gemini Chat Model
- **Type / role:** `lmChatGoogleGemini` (LLM provider for the agent)
- **Configuration:** `modelName: models/gemini-3-pro-preview`
- **Credentials:** Google Gemini (PaLM) API
- **Output connection:** `ai_languageModel` → **Company Reputation Management**
- **Edge cases / failures:** preview model may change/retire; quota/rate limiting.

---

### Block 5 — HTML/PDF Generation & Google Drive Upload

**Overview:** Converts the agent’s report text into HTML via Gemini, wraps it into a full HTML document, converts to PDF using ConvertAPI, then uploads the resulting PDF to Google Drive.  
**Nodes involved:**  
- HTML Converter (Chain LLM)  
- Google Gemini Chat Model1  
- HTML  
- From Json to Binary (Code)  
- HTML to PDF (HTTP Request, ConvertAPI)  
- Base64 to File (Convert to File)  
- Upload file (Google Drive)

#### Node: HTML Converter
- **Type / role:** `Chain LLM` (LLM transforms report text → HTML)
- **Model binding:** **Google Gemini Chat Model1**
- **Input text:** `{{$json.output}}` (agent output)
- **Prompt:** “Translate into HTML… I only need the HTML, not the opening tags ```html and closing ```.”
- **Output:** To **HTML**
- **Edge cases / failures:**
  - If agent output field name differs (not `output`) the HTML conversion receives empty text.
  - Model may still include markdown fences; downstream HTML wrapper will embed them.

#### Node: Google Gemini Chat Model1
- **Type / role:** `lmChatGoogleGemini` (LLM provider for HTML conversion)
- **Configuration:** `modelName: models/gemini-3-pro-preview`
- **Output connection:** `ai_languageModel` → **HTML Converter**
- **Edge cases / failures:** same as other Gemini nodes.

#### Node: HTML
- **Type / role:** `HTML` node (wraps content in an HTML template)
- **Configuration:** Static HTML skeleton with:
  - `{{$json.text}}` inserted in `<body>`
  - basic CSS styles and a trivial `<script>`
- **Input expectation:** The upstream node should provide `text` containing HTML body content.  
  **Potential mismatch:** `HTML Converter` typically outputs its generated text under a field that is not guaranteed to be named `text` unless configured/known; if it outputs under another key, the body will be empty.
- **Output:** To **From Json to Binary**
- **Edge cases / failures:** Missing `$json.text` → empty PDF content.

#### Node: From Json to Binary
- **Type / role:** `Code` (creates binary HTML file from JSON field)
- **Logic:** Reads `const htmlContent = $input.item.json.html;`, converts to base64, outputs binary `data` with:
  - `mimeType: text/html`
  - `fileName: page.html`
- **Input expectation:** JSON field named `html` containing the full HTML string.
- **Potential mismatch:** The **HTML** node typically outputs the rendered HTML in a field that may not be called `html`. If it outputs `html` then this is correct; if not, this node will generate an empty/undefined file.
- **Output:** To **HTML to PDF**
- **Edge cases / failures:** If `htmlContent` is `undefined`, Buffer conversion yields `"undefined"` content, producing a broken PDF.

#### Node: HTML to PDF
- **Type / role:** `HTTP Request` (ConvertAPI conversion)
- **Configuration choices:**
  - POST `https://v2.convertapi.com/convert/html/to/pdf`
  - `multipart-form-data`
  - Form binary field: `File` from input binary property `data`
  - Auth: predefined credential type `convertApi`
- **Output:** To **Base64 to File**
- **Edge cases / failures:**
  - ConvertAPI auth errors / quota exceeded
  - If sent HTML is invalid or empty → PDF may be blank or API returns error
  - ConvertAPI response shape must include `Files[0].FileData` for next step

#### Node: Base64 to File
- **Type / role:** `Convert to File` (turn ConvertAPI base64 output into binary)
- **Operation:** `toBinary`
- **Source property:** `Files[0].FileData`
- **Output:** To **Upload file**
- **Edge cases / failures:**
  - If ConvertAPI response differs (no `Files`, different casing) this fails.

#### Node: Upload file
- **Type / role:** `Google Drive` (upload binary PDF)
- **Configuration:**
  - **File name expression:** `{{$now.format('yyyyLLddHHiiss')}}_{{ $binary.data.fileName }}`
  - Drive: “My Drive”
  - Folder ID: `1tkCr7xdraoZwsHqeLm7FZ4aRWY94oLbZ` (cached name “n8n”)
- **Credentials:** Google Drive OAuth2
- **Input:** binary from **Base64 to File**
- **Edge cases / failures:**
  - OAuth token expired/invalid
  - Folder ID not accessible to the authenticated Drive user
  - If binary is not present (`$binary.data`) → upload fails

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Test workflow’ | manualTrigger | Manual entry point | — | Set Parameters |  |
| Set Parameters | set | Define company_id and max_page | When clicking ‘Test workflow’ | Autonomously extract live data from any website… | ## STEP 1 - Setup<br>Setup The Feedaty company identifier and The maximum number of review pages to scrape (eg. maxisport) |
| Autonomously extract live data from any website perfect for e commerce job boards lead capture and more | scrapegraphAi | Scrape Feedaty reviews with pagination + schema | Set Parameters | Get Result | ## STEP 2 - Automated Review Scraping<br>Reviews are fetched directly from Feedaty using ScrapeGraphAI, ordered by recency. Pagination is handled automatically to respect the configured page limit. If you want, you can Limit the number of reviews |
| Get Result | set | Map scraper pages to `result` array | Autonomously extract live data from any website… | Extract reviews | ## STEP 2 - Automated Review Scraping<br>Reviews are fetched directly from Feedaty using ScrapeGraphAI, ordered by recency. Pagination is handled automatically to respect the configured page limit. If you want, you can Limit the number of reviews |
| Extract reviews | code | Flatten paginated scrape results into review items | Get Result | Limit reviews | ## STEP 2 - Automated Review Scraping<br>Reviews are fetched directly from Feedaty using ScrapeGraphAI, ordered by recency. Pagination is handled automatically to respect the configured page limit. If you want, you can Limit the number of reviews |
| Limit reviews | limit | Reduce number of reviews processed | Extract reviews | Loop Over Items | ## STEP 2 - Automated Review Scraping<br>Reviews are fetched directly from Feedaty using ScrapeGraphAI, ordered by recency. Pagination is handled automatically to respect the configured page limit. If you want, you can Limit the number of reviews |
| Loop Over Items | splitInBatches | Iterate per review item | Limit reviews / Set fields | Aggregate reviews; Sentiment Analysis1 | ## STEP 3 - Content Extraction & Sentiment Analysis<br>Each review page is converted into clean Markdown, including content rendered via JavaScript.<br>Each review’s text is analyzed and classified as Positive, Neutral, or Negative.<br>Results are normalized into consistent, machine-readable fields. |
| Google Gemini Chat Model2 | lmChatGoogleGemini | LLM provider for sentiment node | — | Sentiment Analysis1 (ai_languageModel) | ## STEP 3 - Content Extraction & Sentiment Analysis<br>Each review page is converted into clean Markdown, including content rendered via JavaScript.<br>Each review’s text is analyzed and classified as Positive, Neutral, or Negative.<br>Results are normalized into consistent, machine-readable fields. |
| Sentiment Analysis1 | sentimentAnalysis | Classify sentiment per review | Loop Over Items | Set fields | ## STEP 3 - Content Extraction & Sentiment Analysis<br>Each review page is converted into clean Markdown, including content rendered via JavaScript.<br>Each review’s text is analyzed and classified as Positive, Neutral, or Negative.<br>Results are normalized into consistent, machine-readable fields. |
| Set fields | set | Normalize fields (date/description/vote/sentiment) | Sentiment Analysis1 | Loop Over Items | ## STEP 3 - Content Extraction & Sentiment Analysis<br>Each review page is converted into clean Markdown, including content rendered via JavaScript.<br>Each review’s text is analyzed and classified as Positive, Neutral, or Negative.<br>Results are normalized into consistent, machine-readable fields. |
| Aggregate reviews | code | Collect all items into `reviews[]` array | Loop Over Items | Company Reputation Management | ## STEP 4 - AI-Powered Reputation Report<br>A specialized AI agent analyzes all reviews at a company level, not product level.<br>It generates a structured management report<br>The report is automatically sent via email to stakeholders. |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM provider for reputation agent | — | Company Reputation Management (ai_languageModel) | ## STEP 4 - AI-Powered Reputation Report<br>A specialized AI agent analyzes all reviews at a company level, not product level.<br>It generates a structured management report<br>The report is automatically sent via email to stakeholders. |
| Company Reputation Management | agent | Generate structured company reputation report | Aggregate reviews | HTML Converter | ## STEP 4 - AI-Powered Reputation Report<br>A specialized AI agent analyzes all reviews at a company level, not product level.<br>It generates a structured management report<br>The report is automatically sent via email to stakeholders. |
| Google Gemini Chat Model1 | lmChatGoogleGemini | LLM provider for HTML conversion | — | HTML Converter (ai_languageModel) | ## STEP 5 -Convert to HTML/PDF & Upload Google Drive |
| HTML Converter | chainLlm | Convert report text to HTML | Company Reputation Management | HTML | ## STEP 5 -Convert to HTML/PDF & Upload Google Drive |
| HTML | html | Wrap generated HTML into full document template | HTML Converter | From Json to Binary | ## STEP 5 -Convert to HTML/PDF & Upload Google Drive |
| From Json to Binary | code | Create binary HTML file for ConvertAPI | HTML | HTML to PDF | ## STEP 5 -Convert to HTML/PDF & Upload Google Drive |
| HTML to PDF | httpRequest | Convert HTML file to PDF via ConvertAPI | From Json to Binary | Base64 to File | ## STEP 5 -Convert to HTML/PDF & Upload Google Drive |
| Base64 to File | convertToFile | Convert ConvertAPI base64 output to binary file | HTML to PDF | Upload file | ## STEP 5 -Convert to HTML/PDF & Upload Google Drive |
| Upload file | googleDrive | Upload resulting PDF to Google Drive folder | Base64 to File | — | ## STEP 5 -Convert to HTML/PDF & Upload Google Drive |
| Sticky Note | stickyNote | Comment block (setup) | — | — | ## STEP 1 - Setup<br>Setup The Feedaty company identifier and The maximum number of review pages to scrape (eg. maxisport) |
| Sticky Note1 | stickyNote | Workflow description block | — | — | # Automated Feedaty Review Scraper using ScrapegraphAI & Reputation Analysis<br>…(contains ScrapeGraphAI/ConvertAPI links and full description) |
| Sticky Note2 | stickyNote | Comment block (scraping) | — | — | ## STEP 2 - Automated Review Scraping<br>… |
| Sticky Note3 | stickyNote | Comment block (sentiment) | — | — | ## STEP 3 - Content Extraction & Sentiment Analysis<br>… |
| Sticky Note4 | stickyNote | Comment block (report) | — | — | ## STEP 4 - AI-Powered Reputation Report<br>… |
| Sticky Note5 | stickyNote | Comment block (pdf/upload) | — | — | ## STEP 5 -Convert to HTML/PDF & Upload Google Drive |
| Sticky Note8 | stickyNote | Channel promo note | — | — | ## MY NEW YOUTUBE CHANNEL<br>👉 https://youtube.com/@n3witalia<br>Image: https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1) Add node **Manual Trigger** named *When clicking ‘Test workflow’*.

2. **Add configuration node**
   1) Add node **Set** named *Set Parameters*.  
   2) Add fields:
      - `company_id` (String) → set to your Feedaty slug (e.g., `maxisport`)
      - `max_page` (Number) → e.g., `2`
   3) Connect: **Manual Trigger → Set Parameters**

3. **Scrape reviews with ScrapeGraphAI**
   1) Add node **ScrapeGraphAI**.
   2) Configure:
      - Website URL: `https://www.feedaty.com/recensioni/{{$json.company_id}}`
      - Enable Pagination: on
      - Total Pages: `{{$json.max_page}}`
      - Enable “Use Output Schema”: on
      - Output schema: array of objects with `id` (string), `date` (string), `vote` (number), `review` (string); require at least `date`, `vote`, `review`.
      - Prompt instructing extraction (date in `dd/MM/yyyy`, score, text, id from anchor `name=""`)
   3) Set **ScrapeGraphAI API credential**.
   4) Connect: **Set Parameters → ScrapeGraphAI**

4. **Map scraper output pages**
   1) Add node **Set** named *Get Result*.
   2) Set field:
      - `result` (Array) = `{{$json.result.pages}}`
   3) Connect: **ScrapeGraphAI → Get Result**

5. **Flatten pages to individual reviews**
   1) Add node **Code** named *Extract reviews* with logic that:
      - iterates pages in `json.result`
      - extracts `page.result.recensioni` or `page.result.reviews`
      - outputs one item per review
   2) Connect: **Get Result → Extract reviews**

6. **(Optional) Limit number of reviews**
   1) Add node **Limit** named *Limit reviews* with `Max Items` (e.g. `5`).
   2) Connect: **Extract reviews → Limit reviews**

7. **Iterate reviews**
   1) Add node **Split in Batches** named *Loop Over Items* (defaults OK).
   2) Connect: **Limit reviews → Loop Over Items**

8. **Sentiment analysis with Gemini**
   1) Add node **Google Gemini Chat Model** (LangChain) named *Google Gemini Chat Model2*.
      - Set credentials: **Google Gemini (PaLM) API**
   2) Add node **Sentiment Analysis** named *Sentiment Analysis1*.
      - Categories: `Positive, Neutral, Negative`
      - System prompt: strict JSON-only output
      - Input text:
        ```
        Vote: {{$json.vote}}/5
        Review: {{$json.review}}
        ```
   3) Connect model to sentiment node via **ai_languageModel**:  
      - **Google Gemini Chat Model2 → Sentiment Analysis1**
   4) Connect execution: **Loop Over Items → Sentiment Analysis1**

9. **Normalize fields**
   1) Add node **Set** named *Set fields*:
      - `date` = `{{$json.date}}`
      - `description` = `{{$json.review}}`
      - `vote` = `{{$json.vote}}`
      - `sentiment` = `{{$json.sentimentAnalysis.category}}`
   2) Connect: **Sentiment Analysis1 → Set fields**
   3) Connect loop continuation: **Set fields → Loop Over Items**

10. **Aggregate all reviews for the report**
   1) Add node **Code** named *Aggregate reviews* that outputs a single item `{ reviews: [ ...itemsJson ] }`.
   2) **Recommended wiring (to avoid missing sentiment):** connect **Loop Over Items (done output)** after the sentiment path, or connect **Aggregate reviews** after *Set fields* completion logic.  
      - In the provided workflow JSON, aggregation is connected directly from Loop Over Items, which can risk aggregating before enrichment.
   3) Connect: **Loop Over Items → Aggregate reviews**

11. **Company-level report agent (Gemini 3)**
   1) Add node **Google Gemini Chat Model** named *Google Gemini Chat Model* with `modelName: models/gemini-3-pro-preview` and Gemini credentials.
   2) Add node **AI Agent** named *Company Reputation Management*:
      - Input text: `Reviews: {{JSON.stringify($json.reviews)}}`
      - System message: paste the provided structured reputation-report instructions (7 sections + rules).
   3) Connect:
      - **Aggregate reviews → Company Reputation Management**
      - **Google Gemini Chat Model → Company Reputation Management** via `ai_languageModel`

12. **Convert report text to HTML**
   1) Add node **Google Gemini Chat Model** named *Google Gemini Chat Model1* with `modelName: models/gemini-3-pro-preview`.
   2) Add node **Chain LLM** named *HTML Converter*:
      - Text input: `{{$json.output}}` (ensure this matches the agent output field in your n8n version)
      - Prompt: convert to HTML without markdown fences
   3) Connect:
      - **Company Reputation Management → HTML Converter**
      - **Google Gemini Chat Model1 → HTML Converter** via `ai_languageModel`

13. **Wrap into full HTML document**
   1) Add node **HTML** named *HTML* with a complete HTML skeleton and insert the converted content in the body (as in the workflow).
   2) Connect: **HTML Converter → HTML**

14. **Create binary HTML file**
   1) Add node **Code** named *From Json to Binary* to read HTML string and output binary `data` as `page.html`.
   2) Ensure the code reads the correct field name produced by the **HTML** node (the provided code expects `$input.item.json.html`).
   3) Connect: **HTML → From Json to Binary**

15. **Convert HTML to PDF (ConvertAPI)**
   1) Add node **HTTP Request** named *HTML to PDF*:
      - POST `https://v2.convertapi.com/convert/html/to/pdf`
      - Content-Type: `multipart/form-data`
      - Form field `File` = binary data field `data`
      - Authentication: **ConvertAPI credential**
   2) Connect: **From Json to Binary → HTML to PDF**

16. **Convert ConvertAPI base64 response to binary**
   1) Add node **Convert to File** named *Base64 to File*:
      - Operation: `toBinary`
      - Source property: `Files[0].FileData`
   2) Connect: **HTML to PDF → Base64 to File**

17. **Upload PDF to Google Drive**
   1) Add node **Google Drive** named *Upload file*:
      - Operation: Upload
      - Folder ID: choose destination folder (e.g., `1tkCr7xdraoZwsHqeLm7FZ4aRWY94oLbZ`)
      - Name: `{{$now.format('yyyyLLddHHiiss')}}_{{$binary.data.fileName}}`
   2) Set **Google Drive OAuth2** credential.
   3) Connect: **Base64 to File → Upload file**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ScrapeGraphAI dashboard | https://dashboard.scrapegraphai.com/?via=n3witalia |
| ConvertAPI | https://convertapi.com?ref=n3witalia |
| YouTube channel | https://youtube.com/@n3witalia |
| YouTube cover image | https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |
| Consulting/support contact | mailto:info@n3w.it |
| LinkedIn | https://www.linkedin.com/in/davideboizza/ |

**Disclaimer (provided):** *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* (content indicates lawful/public data handling and compliance).