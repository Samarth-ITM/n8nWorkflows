Analyze website SEO and log AI recommendations to Google Sheets with OpenAI

https://n8nworkflows.xyz/workflows/analyze-website-seo-and-log-ai-recommendations-to-google-sheets-with-openai-12767


# Analyze website SEO and log AI recommendations to Google Sheets with OpenAI

## 1. Workflow Overview

**Purpose:** This workflow receives a website URL via a POST webhook, fetches the page HTML, extracts key SEO-related elements (title, meta description, headings, images, links), sends the extracted data to OpenAI for an SEO assessment, then logs results to Google Sheets and optionally notifies via Slack and Gmail based on the SEO score.

**Primary use cases:**
- Fast SEO triage for agencies/consultants
- Automated site audits and centralized logging
- Alerting when a page falls below a quality threshold

### 1.1 Input Reception & Normalization
Receives the target URL and adds a timestamp to standardize downstream usage.

### 1.2 Website Fetch & SEO Element Extraction
Downloads the page HTML and parses core on-page SEO signals.

### 1.3 AI SEO Assessment
Sends extracted signals to OpenAI and requests a structured JSON response.

### 1.4 Formatting, Routing, Logging & Notifications
Prepares a consolidated record, routes by SEO severity, writes to Google Sheets, and sends alerts/reports.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Normalization
**Overview:** Accepts inbound URL requests and stores a normalized `websiteUrl` plus a timestamp for consistent downstream references.  
**Nodes involved:**  
- Webhook Trigger - Input URL  
- Workflow Configuration

#### Node: Webhook Trigger - Input URL
- **Type / role:** `Webhook` — workflow entry point.
- **Key configuration:**
  - Method: **POST**
  - Path: **/seo-analyzer**
  - Response mode: **Last Node** (returns the output of the final executed node back to the webhook caller)
- **Input/Output:**
  - **No inputs** (trigger node).
  - Output to **Workflow Configuration**.
- **Key fields expected:** `body.url` (e.g., `{ "url": "https://example.com" }`)
- **Edge cases / failures:**
  - Missing `body.url` → downstream expressions will produce empty/invalid URL and HTTP request will fail.
  - If the workflow branches, “responseMode: lastNode” can return an unexpected branch’s output depending on execution path.

#### Node: Workflow Configuration
- **Type / role:** `Set` — maps incoming payload to internal fields.
- **Configuration choices:**
  - Creates:
    - `websiteUrl` = `{{$json.body.url}}`
    - `analysisTimestamp` = `{{$now.toISO()}}`
  - **Include Other Fields:** enabled (preserves the rest of the webhook payload).
- **Input/Output:**
  - Input from **Webhook Trigger - Input URL**
  - Output to **Fetch Website HTML**
- **Edge cases / failures:**
  - If `body.url` is not provided or not a valid URL, the HTTP request node will error.

---

### Block 2 — Website Fetch & SEO Element Extraction
**Overview:** Downloads the target page and extracts basic SEO-relevant elements using CSS selectors.  
**Nodes involved:**  
- Fetch Website HTML  
- Extract SEO Elements

#### Node: Fetch Website HTML
- **Type / role:** `HTTP Request` — fetches the website content.
- **Configuration choices:**
  - URL: `{{ $('Workflow Configuration').first().json.websiteUrl }}`
  - Redirect and response options are present but not specifically customized (defaults apply).
- **Input/Output:**
  - Input from **Workflow Configuration**
  - Output to **Extract SEO Elements**
- **Important integration detail:** The next node (“HTML”) is configured to read from **binary**. This HTTP node must therefore be configured to return binary data (or the HTML node will not find expected binary content).
- **Edge cases / failures:**
  - 4xx/5xx responses, DNS failures, timeouts.
  - Sites that block bots/unknown user agents.
  - Non-HTML responses (PDF, JS app shell, etc.) may produce weak/empty extractions.
  - If HTTP node returns JSON/text (not binary) while HTML node expects binary, extraction will fail.

#### Node: Extract SEO Elements
- **Type / role:** `HTML` — parses the fetched HTML and extracts values.
- **Configuration choices:**
  - Operation: **Extract HTML Content**
  - Source data: **Binary**
  - Extraction map (CSS selectors):
    - `title` from `title`
    - `metaDescription` from `meta[name="description"]` attribute `content`
    - `h1` from `h1`
    - `h2` from `h2`
    - `images` from `img` attribute `src`
    - `links` from `a` attribute `href`
- **Input/Output:**
  - Input from **Fetch Website HTML**
  - Output to **AI SEO Analysis**
- **Edge cases / failures:**
  - Missing tags (no meta description, no H1) → empty/null outputs.
  - Multiple H1/H2 → typically returned as arrays; formatting downstream assumes `.length` exists (guarded with `?.length || 0` in later node).
  - Relative URLs in `src`/`href` are not normalized; analysis may misinterpret them.

---

### Block 3 — AI SEO Assessment
**Overview:** Sends extracted page signals to OpenAI with instructions to return a JSON assessment (score, issues, recommendations).  
**Nodes involved:**  
- AI SEO Analysis

#### Node: AI SEO Analysis
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI call via n8n’s LangChain-based node.
- **Credentials:** OpenAI API credential (“n8n free OpenAI API credits”).
- **Configuration choices (interpreted):**
  - Provides “instructions” to act as an expert SEO analyst.
  - Requests a strict JSON schema in the response including:
    - `seoScore` (0–100)
    - `criticalIssues` (array)
    - `recommendations` (object with title/meta/headings/keywords/internalLinking/images)
    - `summary`
  - Prompt content includes extracted values and counts:
    - URL from `Workflow Configuration`
    - Title, meta description, H1/H2, number of images, number of links
- **Input/Output:**
  - Input from **Extract SEO Elements**
  - Output to **Format AI Output**
- **Edge cases / failures:**
  - Model may return non-JSON or JSON wrapped in markdown → downstream parsing is not implemented, so routing by score will break.
  - Token limits if the extracted content becomes large (not likely here since only key elements are included).
  - Credential quota/auth errors.

---

### Block 4 — Formatting, Routing, Logging & Notifications
**Overview:** Consolidates AI output, attempts to route based on SEO score thresholds, appends/updates Google Sheets, and sends Slack/Gmail notifications for issues.  
**Nodes involved:**  
- Format AI Output  
- Route by SEO Score  
- Log to Google Sheets  
- Send Slack Alert  
- Send Email Report  
- Final Report Storage

#### Node: Format AI Output
- **Type / role:** `Set` — prepares a unified record for storage/notifications.
- **Configuration choices:**
  - `aiAnalysis` = `{{$json.message.content}}` (raw model response text)
  - `websiteUrl` / `timestamp` pulled from **Workflow Configuration**
  - `extractedElements` object composed from **Extract SEO Elements**:
    - `title`, `metaDescription`
    - counts: `h1Count`, `h2Count`, `imageCount`, `linkCount` using optional chaining guards
  - **Include Other Fields:** enabled
- **Input/Output:**
  - Input from **AI SEO Analysis**
  - Output to **Route by SEO Score**
- **Key issue (important):**
  - The workflow never parses the AI JSON into fields like `$json.seoScore`. It only stores the response as a string in `aiAnalysis`. This makes the next switch node’s numeric comparisons unreliable unless another node parses `aiAnalysis` into JSON and maps `seoScore`.
- **Edge cases / failures:**
  - If OpenAI node output structure differs (e.g., no `message.content`), `aiAnalysis` becomes empty.

#### Node: Route by SEO Score
- **Type / role:** `Switch` — routes execution based on score thresholds.
- **Configuration choices:**
  - Two named outputs:
    - **Critical Issues** when:
      - `aiAnalysis` contains `"seoScore"` (string contains check)
      - AND `$json.seoScore < 60` (numeric check)
    - **Minor Issues** when:
      - `aiAnalysis` contains `"seoScore"`
      - AND `$json.seoScore >= 60`
  - Fallback output renamed to **No Score**
- **Input/Output:**
  - Input from **Format AI Output**
  - Outputs:
    - Output 0 → **Log to Google Sheets**
    - Output 1 → **Send Slack Alert** and **Send Email Report** (both in parallel)
- **Critical flaw / edge case:**
  - `$json.seoScore` is **not created anywhere** in this workflow. Unless n8n magically merges parsed fields (it doesn’t here), both numeric conditions can evaluate incorrectly or as `NaN`, pushing most runs into **No Score**.
  - The “contains seoScore” check only proves the string includes the text, not that it is valid JSON.
- **Recommendation to fix (structural):**
  - Add a parsing step (e.g., Code node `JSON.parse($json.aiAnalysis)` or “Structured Output Parser” approach) and map `seoScore` into a numeric field before this switch.

#### Node: Log to Google Sheets
- **Type / role:** `Google Sheets` — persistent logging (append or update).
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account 3”).
- **Configuration choices:**
  - Operation: **Append or Update**
  - Matching column(s): `websiteUrl` (used as the upsert key)
  - Mapping mode: **Auto map input data**
  - Document ID: placeholder (`<Google Sheets Document ID>`)
  - Sheet name: placeholder (`<Sheet Name (e.g., SEO Analysis)>`)
- **Input/Output:**
  - Input from **Route by SEO Score** (Critical Issues path)
  - Output to **Final Report Storage**
- **Edge cases / failures:**
  - Placeholders must be replaced or node will fail.
  - The sheet must contain a `websiteUrl` column for matching; otherwise upsert may error or behave unexpectedly.
  - Auto-mapping may not create a clean column layout for nested objects (e.g., `extractedElements` object). This may end up as `[object Object]` unless flattened.

#### Node: Send Slack Alert
- **Type / role:** `Slack` — alerting for “Critical Issues” path.
- **Credentials:** Not shown in node, but Slack node requires Slack credentials or a configured connection in n8n.
- **Configuration choices:**
  - Channel selection: by `channelId` placeholder (`<Slack Channel ID or Name>`)
  - Message includes:
    - Website URL, timestamp, and raw `aiAnalysis`
- **Input/Output:**
  - Input from **Route by SEO Score** (Minor Issues path per current wiring)
  - Output to **Final Report Storage**
- **Edge cases / failures:**
  - If channel placeholder not replaced, message won’t deliver.
  - Long `aiAnalysis` can exceed Slack message limits; may truncate or fail.
- **Note on routing logic:** As wired, Slack/Email are triggered on the **second** switch output (“Minor Issues”). That conflicts with the message text (“Critical SEO Issues Found”). Either the outputs are miswired or the switch rules/outputs are misinterpreted.

#### Node: Send Email Report
- **Type / role:** `Gmail` — sends an HTML email report.
- **Credentials:** Gmail OAuth2 (“Gmail account 4”).
- **Configuration choices:**
  - To: placeholder (`<Recipient Email Address>`)
  - Subject: `SEO Analysis Report - {{websiteUrl}}`
  - HTML body includes extracted elements and a `<pre>` block with `aiAnalysis`
- **Input/Output:**
  - Input from **Route by SEO Score** (Minor Issues path per current wiring)
  - Output to **Final Report Storage**
- **Edge cases / failures:**
  - Recipient placeholder not replaced.
  - Gmail API send limits/quota.
  - Very long AI output may cause email size issues.

#### Node: Final Report Storage
- **Type / role:** `Set` — final payload consolidation (useful for webhook response and/or downstream storage).
- **Configuration choices:**
  - `finalReport` object:
    - `websiteUrl`, `timestamp`, `extractedElements`, `aiAnalysis`, `status: 'completed'`
  - `reportId` = `{{$now.toMillis()}}`
  - **Include Other Fields:** enabled
- **Input/Output:**
  - Inputs from **Log to Google Sheets**, **Send Slack Alert**, and **Send Email Report** (multiple possible upstreams)
  - No downstream nodes; due to webhook “responseMode: lastNode”, this likely becomes the HTTP response body.
- **Edge cases / failures:**
  - Because multiple branches converge here, the returned webhook response depends on which branch executed last and what data shape arrived.

---

### Sticky Notes (documentation-only nodes)
These nodes don’t execute logic but provide context on the canvas.

- **Sticky Note:** “## Webhook & Config”
- **Sticky Note1:** “## Website Data”
- **Sticky Note2:** “## SEO Analysis”
- **Sticky Note3:** “## Log & Notify”
- **Sticky Note4:** Contains overall description, setup steps, and contact email (see section 5).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger - Input URL | Webhook | Entry point (POST URL submission) | — | Workflow Configuration | ## Webhook & Config |
| Workflow Configuration | Set | Normalize fields (websiteUrl, timestamp) | Webhook Trigger - Input URL | Fetch Website HTML | ## Webhook & Config |
| Fetch Website HTML | HTTP Request | Download page HTML | Workflow Configuration | Extract SEO Elements | ## Website Data |
| Extract SEO Elements | HTML | Extract title/meta/H1/H2/images/links | Fetch Website HTML | AI SEO Analysis | ## Website Data |
| AI SEO Analysis | OpenAI (LangChain) | Produce SEO score + recommendations | Extract SEO Elements | Format AI Output | ## SEO Analysis |
| Format AI Output | Set | Consolidate AI output + extracted counts | AI SEO Analysis | Route by SEO Score | ## SEO Analysis |
| Route by SEO Score | Switch | Branch by score threshold | Format AI Output | Log to Google Sheets; Send Slack Alert; Send Email Report | ## Log & Notify |
| Log to Google Sheets | Google Sheets | Append/update analysis record | Route by SEO Score | Final Report Storage | ## Log & Notify |
| Send Slack Alert | Slack | Send alert message | Route by SEO Score | Final Report Storage | ## Log & Notify |
| Send Email Report | Gmail | Send HTML email report | Route by SEO Score | Final Report Storage | ## Log & Notify |
| Final Report Storage | Set | Build final response object | Log to Google Sheets; Send Slack Alert; Send Email Report | — | ## Log & Notify |
| Sticky Note | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note1 | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note2 | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note3 | Sticky Note | Canvas annotation | — | — |  |
| Sticky Note4 | Sticky Note | Canvas annotation (overview/setup/credits) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: **“AI SEO Analyzer & Suggestions”**.

2. **Add Trigger: Webhook**
   - Node: **Webhook**
   - Name: **Webhook Trigger - Input URL**
   - Method: **POST**
   - Path: **seo-analyzer**
   - Response: **Last Node**
   - Expected request body: `{ "url": "https://example.com" }`

3. **Add Set node for configuration**
   - Node: **Set**
   - Name: **Workflow Configuration**
   - Add fields:
     - `websiteUrl` (String): `{{$json.body.url}}`
     - `analysisTimestamp` (String): `{{$now.toISO()}}`
   - Turn on **Include Other Fields**.
   - Connect: **Webhook → Workflow Configuration**

4. **Add HTTP Request node to fetch HTML**
   - Node: **HTTP Request**
   - Name: **Fetch Website HTML**
   - URL: `{{ $('Workflow Configuration').first().json.websiteUrl }}`
   - Ensure response is compatible with the next HTML node:
     - Either configure HTTP Request to **download** / **return binary** (recommended if you keep HTML node “Source Data: Binary”),
     - or change the HTML node to read from **JSON/text** accordingly.
   - Connect: **Workflow Configuration → Fetch Website HTML**

5. **Add HTML extraction node**
   - Node: **HTML**
   - Name: **Extract SEO Elements**
   - Operation: **Extract HTML Content**
   - Source Data: **Binary** (must match previous node output)
   - Add extraction values:
     - `title` selector `title`
     - `metaDescription` selector `meta[name="description"]`, return attribute `content`
     - `h1` selector `h1`
     - `h2` selector `h2`
     - `images` selector `img`, return attribute `src`
     - `links` selector `a`, return attribute `href`
   - Connect: **Fetch Website HTML → Extract SEO Elements**

6. **Add OpenAI node (LangChain OpenAI)**
   - Node: **OpenAI** (`@n8n/n8n-nodes-langchain.openAi`)
   - Name: **AI SEO Analysis**
   - Credentials: configure an **OpenAI API** credential in n8n and select it.
   - Instructions: paste the SEO analyst instructions and required JSON schema (as in workflow).
   - Message/prompt content: include URL + extracted fields (title/meta/H1/H2 + counts).
   - Connect: **Extract SEO Elements → AI SEO Analysis**

7. **Add Set node to format AI output**
   - Node: **Set**
   - Name: **Format AI Output**
   - Fields:
     - `aiAnalysis` (String): `{{$json.message.content}}`
     - `websiteUrl` (String): `{{ $('Workflow Configuration').first().json.websiteUrl }}`
     - `timestamp` (String): `{{ $('Workflow Configuration').first().json.analysisTimestamp }}`
     - `extractedElements` (Object): construct object with title/meta and counts from **Extract SEO Elements**
   - Include Other Fields: enabled
   - Connect: **AI SEO Analysis → Format AI Output**

8. **(Strongly recommended) Add a parsing step for the AI JSON**
   - Add a **Code** node (or a structured output parser mechanism) to:
     - Parse `aiAnalysis` as JSON
     - Set `seoScore` as a **number** and optionally `criticalIssues`, `recommendations`, `summary`
   - Place it between **Format AI Output → Route by SEO Score**
   - Without this, the switch’s `$json.seoScore` comparisons won’t work reliably.

9. **Add Switch node for routing**
   - Node: **Switch**
   - Name: **Route by SEO Score**
   - Rules:
     - Output “Critical Issues”: `seoScore < 60`
     - Output “Minor Issues”: `seoScore >= 60`
   - Fallback: “No Score”
   - Connect: **Format AI Output (or Parse node) → Route by SEO Score**

10. **Add Google Sheets node for logging**
   - Node: **Google Sheets**
   - Name: **Log to Google Sheets**
   - Credentials: configure **Google Sheets OAuth2** and select it.
   - Operation: **Append or Update**
   - Document: select your spreadsheet
   - Sheet: select the target sheet (e.g., “SEO Analysis”)
   - Matching column: `websiteUrl` (make sure the sheet has this column header)
   - Connect: **Route by SEO Score (Critical path) → Log to Google Sheets**

11. **Add Slack node for alerts**
   - Node: **Slack**
   - Name: **Send Slack Alert**
   - Credentials: configure Slack in n8n and select the workspace.
   - Channel: choose a channel
   - Message: include websiteUrl, timestamp, and AI analysis
   - Connect: **Route by SEO Score (the branch you intend to alert on) → Send Slack Alert**
   - Verify your branch wiring matches your intent (critical vs minor).

12. **Add Gmail node for email report**
   - Node: **Gmail**
   - Name: **Send Email Report**
   - Credentials: configure **Gmail OAuth2**
   - To: set recipient email
   - Subject and HTML body: include extractedElements and aiAnalysis
   - Connect: **Route by SEO Score (the branch you intend) → Send Email Report**

13. **Add final Set node for response consolidation**
   - Node: **Set**
   - Name: **Final Report Storage**
   - Fields:
     - `finalReport` object with `websiteUrl`, `timestamp`, `extractedElements`, `aiAnalysis`, `status: "completed"`
     - `reportId` = `{{$now.toMillis()}}`
   - Connect:
     - **Log to Google Sheets → Final Report Storage**
     - **Send Slack Alert → Final Report Storage**
     - **Send Email Report → Final Report Storage**

14. **Activate the workflow**, then call the webhook:
   - POST to `/webhook/seo-analyzer` (URL depends on your n8n instance)
   - Body: `{ "url": "https://example.com" }`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically analyzes any website for SEO performance using AI and provides actionable suggestions…” | Sticky note “## Main” (canvas documentation) |
| Setup steps: configure URL input, Google Sheets credentials/sheet, Slack/Gmail notifications, OpenAI API key, customize Switch thresholds | Sticky note “## Setup” (canvas documentation) |
| Credit: “Created by Hyrum Hurst, AI Automation Engineer at QuarterSmart.” | Sticky note “## Main” |
| Contact: hyrum@quartersmart.com | Sticky note “## Setup” |

