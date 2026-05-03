Generate employee performance review summaries with GPT-4, Gmail and Sheets

https://n8nworkflows.xyz/workflows/generate-employee-performance-review-summaries-with-gpt-4--gmail-and-sheets-11745


# Generate employee performance review summaries with GPT-4, Gmail and Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** AI Performance Review SummaryGenerator  
**Purpose:** Automate the creation and delivery of employee performance review summaries. The workflow receives structured review data via a webhook, computes derived metrics, uses GPT‑4 to generate an HR-style summary, renders a polished HTML “review card”, converts it to an image, emails the employee with the image attached, logs the event to Google Sheets, posts a Slack notification for HR, and returns a JSON response to the caller.

### 1.1 Logical blocks
1. **Input Reception (Webhook)**  
   Receives a POST payload containing employee identity, manager info, scores, and qualitative feedback.
2. **Data Processing & Normalization (Code)**  
   Calculates averages/percentages, derives performance labels/colors, creates IDs and flags.
3. **AI Review Generation (OpenAI / LangChain node + Parsing)**  
   Sends a structured prompt to GPT‑4 requesting *JSON-only output*, then parses/normalizes the response.
4. **Formatting & Asset Generation (HTML + HTML→Image + Download)**  
   Builds a full HTML page, converts it to a PNG via an external API, then downloads the PNG as a binary file.
5. **Delivery & Auditing (Gmail + Sheets + Slack + Webhook response)**  
   Emails the employee (with image attachment), appends a log row to Google Sheets, alerts HR in Slack, and responds to the original webhook request.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Accepts inbound review payloads from any system (HRIS, form, internal tool) via HTTP POST. The workflow is configured to respond only after downstream processing completes.

**Nodes involved:**  
- Receive Review Data (Webhook)

#### Node: Receive Review Data
- **Type / role:** `n8n-nodes-base.webhook` — entry point; receives HTTP request payload.
- **Key configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `performance-review`
  - **Response mode:** `responseNode` (the response is sent by a later “Respond to Webhook” node).
- **Inputs / Outputs:**
  - **Input:** External HTTP POST request.
  - **Output:** One item with the webhook request data (notably `json.body` expected by later code).
  - **Next node:** Process Employee Data
- **Edge cases / failures:**
  - Missing or malformed `body` will break later computations (e.g., missing `scores` or `feedback`).
  - If the workflow errors before “Send Webhook Response”, the requester may receive a timeout/5xx depending on n8n settings.
- **Version-specific notes:** node typeVersion `2.1` (standard Webhook node behavior).

---

### Block 2 — Data Processing & Normalization
**Overview:** Converts raw webhook payload into a normalized object: calculated averages, percentages for display bars, performance level and color, and flags like low performer/high performer. Also generates a unique review ID and timestamps.

**Nodes involved:**  
- Process Employee Data (Code)

#### Node: Process Employee Data
- **Type / role:** `n8n-nodes-base.code` — transforms data, calculates metrics, prepares a clean JSON structure for AI and downstream nodes.
- **Key configuration choices:**
  - Reads inbound payload from `const webhookData = $input.item.json.body;`
  - Computes:
    - **Average score** from `scores.technical`, `scores.communication`, `scores.leadership`, `scores.productivity`, `scores.teamwork`
    - **Performance level** by `overallRating` thresholds:
      - ≥ 4.5: Outstanding
      - ≥ 4.0: Exceeds Expectations
      - ≥ 3.0: Meets Expectations
      - else: Needs Improvement
    - **Performance color** (hex) from the same thresholds
    - **Percent values** for each score (`score/5 * 100`) for HTML progress bars
  - Adds flags:
    - `isLowPerformer: overallRating < 3.0`
    - `isHighPerformer: overallRating >= 4.5`
    - `requiresManagerAlert: overallRating < 3.0` (computed but not used later in this workflow)
  - Generates `reviewId: REV-{employeeId}-{timestamp}` and `processedAt` ISO string
- **Inputs / Outputs:**
  - **Input:** Item from Webhook node (expects `json.body`).
  - **Output:** One item with `json` containing the normalized `output` object.
  - **Next node:** AI Summary Generator
- **Edge cases / failures:**
  - If any score is missing/non-numeric, average and percentages become `NaN`.
  - If `feedback` or nested keys are missing (`feedback.strengths`, etc.), properties become `undefined` and can degrade AI prompt quality.
  - `overallRating` and `scores.*` may be strings (common in forms). Without explicit coercion, addition may concatenate strings or produce unexpected results; here JS `+` can concatenate. This is a real risk if inputs are not numeric.
- **Version-specific notes:** Code node typeVersion `2`.

---

### Block 3 — AI Review Generation & Parsing
**Overview:** Sends a structured prompt to GPT‑4 to generate a performance summary strictly as JSON. Then cleans/parses the model output and merges the AI result with the previously computed employee data.

**Nodes involved:**  
- AI Summary Generator (OpenAI / LangChain)  
- Parse AI Response (Code)

#### Node: AI Summary Generator
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — calls OpenAI with a multi-message prompt.
- **Key configuration choices:**
  - **Model:** GPT‑4 (`modelId: gpt-4`)
  - **Temperature:** 0.7 (moderate creativity; may increase formatting drift risk)
  - **System message:** HR expert tone + “Always format your response as valid JSON.”
  - **User message:** Injects workflow variables:
    - `{{ $json.employeeName }}`, `{{ $json.department }}`, `{{ $json.reviewPeriod }}`, `{{ $json.overallRating }}`, etc.
    - Includes manager feedback fields: `strengthsFeedback`, `improvementsFeedback`, `managerComments`
  - **Hard instruction:** “Return ONLY valid JSON… no markdown… ensure strings escaped…”
- **Inputs / Outputs:**
  - **Input:** Processed employee data JSON.
  - **Output:** LangChain/OpenAI structured output (later node expects `json.output[0].content[0].text`)
  - **Next node:** Parse AI Response
- **Credentials:** OpenAI API credential referenced as `OpenAI account` (placeholder id in JSON).
- **Edge cases / failures:**
  - Auth/credit limits (401/429), timeouts, model unavailability.
  - Despite instructions, LLM may output extra text or invalid JSON; downstream parsing attempts to handle this but is not foolproof.
  - Output structure may differ depending on n8n node version; parsing assumes a specific schema.
- **Version-specific notes:** node typeVersion `2.1`. The output path used later (`output[0].content[0].text`) is sensitive to node changes.

#### Node: Parse AI Response
- **Type / role:** `n8n-nodes-base.code` — cleans and parses the AI response into usable fields; merges with prior employee data.
- **Key configuration choices:**
  - Reads AI text from:  
    `const openaiOutput = $input.item.json.output[0].content[0].text;`
  - Strips potential markdown fences:
    - Removes ```json and ```
  - JSON parsing strategy:
    1. `JSON.parse(cleanedResponse)`
    2. Fallback: regex extract first `{ ... }` block and parse
    3. Otherwise throws error
  - Merges AI fields into `previousData` fetched via:  
    `const previousData = $('Process Employee Data').item.json;`
  - Builds HTML list fragments:
    - `strengthsHTML` from `keyStrengths`
    - `developmentHTML` from `developmentAreas`
- **Inputs / Outputs:**
  - **Input:** Output from AI Summary Generator.
  - **Output:** Combined object (employee data + AI summary + helper HTML strings).
  - **Next node:** Generate Review Card HTML
- **Edge cases / failures:**
  - If the OpenAI node output schema changes, `openaiOutput` path breaks.
  - If the model returns JSON arrays/objects with unexpected keys, downstream HTML rendering may break (e.g., `keyStrengths` not an array).
  - Regex extraction can capture invalid JSON if there are multiple braces in the response.
  - Using `$('Process Employee Data').item.json` assumes that node executed and item indexing aligns (single-item flow). Multi-item batches may mismatch.
- **Version-specific notes:** Code node typeVersion `2`.

---

### Block 4 — Formatting & Asset Generation
**Overview:** Creates a visually rich HTML performance review card, converts it to a PNG image via the Htmlcsstoimg API node, and downloads that image as a binary file to attach to email.

**Nodes involved:**  
- Generate Review Card HTML (Code)  
- Convert HTML to Image (htmlcsstoimage)  
- Download Image (HTTP Request)

#### Node: Generate Review Card HTML
- **Type / role:** `n8n-nodes-base.code` — constructs a full HTML document using the combined data object.
- **Key configuration choices:**
  - Produces a complete HTML page with inline CSS (gradient header, score bars, sections).
  - Uses computed values:
    - `performanceColor` for label color
    - `technicalPercent`, etc. for bar widths
    - AI content inserted directly into HTML (`executiveSummary`, lists, `closingStatement`)
  - Returns `{ ...data, html }` in JSON.
- **Inputs / Outputs:**
  - **Input:** Parsed/merged data from Parse AI Response.
  - **Output:** Same JSON plus `html` string.
  - **Next node:** Convert HTML to Image
- **Edge cases / failures:**
  - If AI text contains unescaped HTML special characters, it may alter rendering (and is also an HTML injection risk). No sanitization is performed.
  - Very long AI text can overflow layout or exceed HTML-to-image API limits.

#### Node: Convert HTML to Image
- **Type / role:** `n8n-nodes-htmlcsstoimage.htmlCssToImage` — external rendering API call that converts HTML/CSS into an image.
- **Key configuration choices:**
  - `html_content`: uses `{{ $json.html }}`
  - Output format: PNG
- **Inputs / Outputs:**
  - **Input:** JSON with `html`.
  - **Output:** JSON containing an `image_url` (used later).
  - **Next node:** Download Image
- **Credentials:** Htmlcsstoimg account (placeholder).
- **Edge cases / failures:**
  - API auth failure, quota exceeded, payload too large, external rendering failures (fonts/CSS support).
  - If the API returns a different response key than `image_url`, subsequent nodes break.
- **Version-specific notes:** node typeVersion `1` (community/partner node behavior may differ from core nodes).

#### Node: Download Image
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads the generated image to binary for email attachment.
- **Key configuration choices:**
  - URL: `{{ $json.image_url }}`
  - Response format: **file** (binary)
- **Inputs / Outputs:**
  - **Input:** Convert HTML to Image output (expects `image_url`).
  - **Output:** Binary data property (default is typically `data` unless overridden by node settings).
  - **Next node:** Send Review to Employee
- **Edge cases / failures:**
  - `image_url` expired/invalid, network errors, large file size.
  - If binary property name differs, Gmail attachment configuration may not find it.
- **Version-specific notes:** node typeVersion `4.3`.

---

### Block 5 — Delivery, Logging, Notification, and Webhook Response
**Overview:** Emails the employee with an HTML email body and attaches the review card image, logs a structured record to Google Sheets, notifies HR via Slack with a link to the image, and returns a JSON success payload to the webhook caller.

**Nodes involved:**  
- Send Review to Employee (Gmail)  
- Log to Google Sheets (Google Sheets)  
- Notify HR Team (Slack)  
- Send Webhook Response (Respond to Webhook)

#### Node: Send Review to Employee
- **Type / role:** `n8n-nodes-base.gmail` — sends the performance review email.
- **Key configuration choices:**
  - **To:** `{{ $('Parse AI Response').item.json.employeeEmail }}`
  - **Subject:** `Your {reviewPeriod} Performance Review - {performanceLevel}`
  - **Body:** Large HTML email template referencing values from `Parse AI Response`.
  - **Attachment:** uses the binary property `data` via “attachmentsBinary” with `property: =data`
- **Inputs / Outputs:**
  - **Input:** From Download Image (binary) but body/subject reference Parse AI Response via node lookup.
  - **Output:** Gmail send result (message id, etc.), then continues to Log to Google Sheets.
- **Credentials:** Gmail OAuth2 (placeholder).
- **Edge cases / failures:**
  - If Download Image binary property isn’t named `data`, attachment will be missing or node may error.
  - Email HTML includes emoji in header/title; some corporate filters may alter rendering.
  - No CC is actually configured, but Slack message claims “Manager CC’d” (inconsistency).
  - Gmail API limits, invalid recipient address, OAuth token expired.

#### Node: Log to Google Sheets
- **Type / role:** `n8n-nodes-base.googleSheets` — append an audit row with review metadata.
- **Key configuration choices:**
  - **Operation:** Append
  - **Document/Sheet:** placeholders (`YOUR_DOCUMENT_ID`, gid=0)
  - **Mapping:** “defineBelow” with explicit column values:
    - Pulls most values from Parse AI Response
    - Image URL from Convert HTML to Image
    - “Email Sent” hardcoded “Yes”
    - “Low Performer Alert” computed string Yes/No from `isLowPerformer`
- **Inputs / Outputs:**
  - **Input:** From Send Review to Employee.
  - **Output:** Append result, then continues to Notify HR Team.
- **Credentials:** Google Sheets OAuth2 (placeholder).
- **Edge cases / failures:**
  - Sheet missing columns or mismatched headers; append may fail or insert into wrong columns depending on configuration.
  - Document permissions, rate limits.
  - Types are not auto-converted (`attemptToConvertTypes: false`), so numeric values are stored as strings.

#### Node: Notify HR Team
- **Type / role:** `n8n-nodes-base.slack` — posts a status message to an HR Slack channel.
- **Key configuration choices:**
  - Posts a formatted message with employee, department, scores, email status, image link, review ID.
  - Channel selected by `channelId` (placeholder).
  - Includes conditional line: low performer alert vs normal range.
- **Inputs / Outputs:**
  - **Input:** From Log to Google Sheets.
  - **Output:** Slack post result, then continues to Send Webhook Response.
- **Credentials:** Slack API (placeholder).
- **Edge cases / failures:**
  - Slack auth/permissions, invalid channel ID.
  - Message states “Manager CC’d” but workflow doesn’t CC manager (data mismatch).
  - If image_url missing, Slack link becomes broken.

#### Node: Send Webhook Response
- **Type / role:** `n8n-nodes-base.respondToWebhook` — returns final JSON response to the original webhook request.
- **Key configuration choices:**
  - Responds with JSON:
    - `success: true`
    - `reviewId`, `employeeName`, `overallRating`, `performanceLevel`
    - `imageUrl` from Convert HTML to Image
    - `emailSent: true`
    - `timestamp` from `processedAt`
- **Inputs / Outputs:**
  - **Input:** From Notify HR Team.
  - **Output:** HTTP response to the caller (terminates the webhook request).
- **Edge cases / failures:**
  - If upstream nodes fail, this node won’t run, leaving the webhook request unresponded (timeout).
  - References multiple other nodes; if any are missing/renamed, expressions fail.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Review Data | Webhook (`n8n-nodes-base.webhook`) | Entry point; receives review payload | (External HTTP POST) | Process Employee Data | ## Input Collection; Webhook receives employee performance data including scores, feedback, and review period details. |
| Process Employee Data | Code (`n8n-nodes-base.code`) | Normalizes payload; computes averages, labels, flags | Receive Review Data | AI Summary Generator | ## Input Collection; Webhook receives employee performance data including scores, feedback, and review period details. |
| AI Summary Generator | OpenAI LangChain (`@n8n/n8n-nodes-langchain.openAi`) | Generates HR performance summary JSON with GPT‑4 | Process Employee Data | Parse AI Response | ## AI Review Generation; OpenAI analyzes performance metrics and generates structured summaries with strengths, improvements, and closing statements. |
| Parse AI Response | Code (`n8n-nodes-base.code`) | Cleans/parses JSON output; merges with employee data | AI Summary Generator | Generate Review Card HTML | ## AI Review Generation; OpenAI analyzes performance metrics and generates structured summaries with strengths, improvements, and closing statements. |
| Generate Review Card HTML | Code (`n8n-nodes-base.code`) | Builds complete HTML performance review card | Parse AI Response | Convert HTML to Image | ## Formatting & Delivery; Converts AI summary to HTML, generates image via HTMLCSS to Image API, downloads it, and emails to employee with detailed metrics. |
| Convert HTML to Image | Htmlcsstoimg (`n8n-nodes-htmlcsstoimage.htmlCssToImage`) | Renders HTML into PNG; outputs image URL | Generate Review Card HTML | Download Image | ## Formatting & Delivery; Converts AI summary to HTML, generates image via HTMLCSS to Image API, downloads it, and emails to employee with detailed metrics. |
| Download Image | HTTP Request (`n8n-nodes-base.httpRequest`) | Downloads PNG to binary for attachment | Convert HTML to Image | Send Review to Employee | ## Formatting & Delivery; Converts AI summary to HTML, generates image via HTMLCSS to Image API, downloads it, and emails to employee with detailed metrics. |
| Send Review to Employee | Gmail (`n8n-nodes-base.gmail`) | Sends HTML email to employee with image attachment | Download Image | Log to Google Sheets | ## Formatting & Delivery; Converts AI summary to HTML, generates image via HTMLCSS to Image API, downloads it, and emails to employee with detailed metrics. |
| Log to Google Sheets | Google Sheets (`n8n-nodes-base.googleSheets`) | Appends audit record of review | Send Review to Employee | Notify HR Team | ## Workflow Complete; Logs review to Google Sheets, notifies HR via Slack, and sends webhook response confirming success. |
| Notify HR Team | Slack (`n8n-nodes-base.slack`) | Posts Slack status notification with key details and image link | Log to Google Sheets | Send Webhook Response | ## Workflow Complete; Logs review to Google Sheets, notifies HR via Slack, and sends webhook response confirming success. |
| Send Webhook Response | Respond to Webhook (`n8n-nodes-base.respondToWebhook`) | Returns JSON result to webhook caller | Notify HR Team | (Ends) | ## Workflow Complete; Logs review to Google Sheets, notifies HR via Slack, and sends webhook response confirming success. |
| Sticky Note | Sticky Note (`n8n-nodes-base.stickyNote`) | Documentation (canvas note) | (None) | (None) | ### How it works; This workflow automates AI-powered performance reviews… (includes setup and customization guidance) |
| Sticky Note2 | Sticky Note (`n8n-nodes-base.stickyNote`) | Documentation (canvas note) | (None) | (None) | ## Input Collection; Webhook receives employee performance data including scores, feedback, and review period details. |
| Sticky Note3 | Sticky Note (`n8n-nodes-base.stickyNote`) | Documentation (canvas note) | (None) | (None) | ## AI Review Generation; OpenAI analyzes performance metrics and generates structured summaries with strengths, improvements, and closing statements. |
| Sticky Note4 | Sticky Note (`n8n-nodes-base.stickyNote`) | Documentation (canvas note) | (None) | (None) | ## Formatting & Delivery; Converts AI summary to HTML, generates image via HTMLCSS to Image API, downloads it, and emails to employee with detailed metrics. |
| Sticky Note5 | Sticky Note (`n8n-nodes-base.stickyNote`) | Documentation (canvas note) | (None) | (None) | ## Workflow Complete; Logs review to Google Sheets, notifies HR via Slack, and sends webhook response confirming success. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: `AI Performance Review SummaryGenerator`
   - (Optional) Add sticky notes with the same headings as in the JSON for maintainability.

2. **Add “Receive Review Data” (Webhook)**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: `performance-review`
   - Response mode: **Respond with “Respond to Webhook” node** (responseNode)

3. **Add “Process Employee Data” (Code)**
   - Node type: **Code**
   - Paste logic that:
     - Reads `body` from webhook (`$input.item.json.body`)
     - Computes average scores and percent fields
     - Derives `performanceLevel`, `performanceColor`
     - Adds flags (`isLowPerformer`, etc.)
     - Generates `reviewId` and timestamps
   - Connect: **Receive Review Data → Process Employee Data**

4. **Add “AI Summary Generator” (OpenAI / LangChain)**
   - Node type: **OpenAI (LangChain)**
   - Credentials: configure **OpenAI API** credential in n8n (API key)
   - Model: **GPT‑4**
   - Temperature: **0.7**
   - Messages:
     - **System:** HR expert + JSON-only requirement
     - **User:** include employee fields and instructions to return *only* the JSON object with:
       - `executiveSummary` (string)
       - `keyStrengths` (array of strings)
       - `developmentAreas` (array of strings)
       - `closingStatement` (string)
     - Use expressions like `{{ $json.employeeName }}` etc.
   - Connect: **Process Employee Data → AI Summary Generator**

5. **Add “Parse AI Response” (Code)**
   - Node type: **Code**
   - Implement:
     - Read AI output text from the OpenAI node output structure (ensure it matches your node version).
     - Strip ``` fences if present.
     - `JSON.parse` with fallback extraction.
     - Merge with `Process Employee Data` output using `$('Process Employee Data').item.json`.
     - Create `strengthsHTML` and `developmentHTML` list strings.
   - Connect: **AI Summary Generator → Parse AI Response**

6. **Add “Generate Review Card HTML” (Code)**
   - Node type: **Code**
   - Build an HTML document string using:
     - Employee fields, scores, bar percentages, AI summary, lists, closing statement, review id/date.
   - Return `{ ...data, html }`
   - Connect: **Parse AI Response → Generate Review Card HTML**

7. **Add “Convert HTML to Image” (Htmlcsstoimg)**
   - Node type: **HTML/CSS to Image** (Htmlcsstoimg integration)
   - Credentials: configure **Htmlcsstoimg API** credential
   - Input: `html_content = {{ $json.html }}`
   - Output format: PNG
   - Connect: **Generate Review Card HTML → Convert HTML to Image**

8. **Add “Download Image” (HTTP Request)**
   - Node type: **HTTP Request**
   - URL: `{{ $json.image_url }}`
   - Response: set **Response Format = File** (binary)
   - Connect: **Convert HTML to Image → Download Image**

9. **Add “Send Review to Employee” (Gmail)**
   - Node type: **Gmail**
   - Credentials: configure **Gmail OAuth2**
   - Operation: **Send**
   - To: `{{ $('Parse AI Response').item.json.employeeEmail }}`
   - Subject: `Your {{ reviewPeriod }} Performance Review - {{ performanceLevel }}`
   - Message/body: paste your HTML template and reference fields via expressions from `Parse AI Response`.
   - Attachments: attach the binary from **Download Image**:
     - Attachment binary property name: ensure it matches (commonly `data`)
   - Connect: **Download Image → Send Review to Employee**

10. **Add “Log to Google Sheets” (Google Sheets)**
    - Node type: **Google Sheets**
    - Credentials: configure **Google Sheets OAuth2**
    - Operation: **Append**
    - Select Spreadsheet + Sheet (gid=0 in the original)
    - Map columns explicitly (Review ID, Date, Employee ID/Name/Email, Dept, scores, performance level, image URL, etc.)
    - Connect: **Send Review to Employee → Log to Google Sheets**

11. **Add “Notify HR Team” (Slack)**
    - Node type: **Slack**
    - Credentials: configure **Slack API**
    - Operation: **Post message**
    - Channel: choose HR channel
    - Message: include employee info, ratings, image link `{{ $('Convert HTML to Image').item.json.image_url }}`
    - Connect: **Log to Google Sheets → Notify HR Team**

12. **Add “Send Webhook Response” (Respond to Webhook)**
    - Node type: **Respond to Webhook**
    - Respond with: **JSON**
    - Body fields: success, message, reviewId, employeeName, overallRating, performanceLevel, imageUrl, emailSent, timestamp
    - Connect: **Notify HR Team → Send Webhook Response**

13. **Activate and test**
    - Use the Webhook test URL in n8n.
    - Send a POST request with a body containing at least:
      - `employeeId, employeeName, employeeEmail`
      - `managerName, managerEmail`
      - `department, reviewPeriod, overallRating`
      - `scores: { technical, communication, leadership, productivity, teamwork }`
      - `feedback: { strengths, improvements, managerComments }`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it works” note: receives employee data via webhook, processes scores/feedback, uses OpenAI to generate summaries, creates visual review card image, emails employee, logs to Sheets, sends Slack notifications. | Sticky Note (workflow canvas) |
| Setup credentials required: OpenAI, HTMLCSS to Image, Gmail, Slack, Google Sheets. | Sticky Note (workflow canvas) |
| Customization: adjust AI prompt in “AI Summary Generator”; adjust thresholds in “Process Employee Data”; update HTML in “Generate Review Card HTML”. | Sticky Note (workflow canvas) |
| Inconsistency: Slack message claims “Manager CC’d”, but Gmail node does not set CC to managerEmail. | Implementation detail to correct if needed |
| Security note: AI text is inserted directly into HTML/email; consider sanitizing to avoid HTML injection. | Implementation hardening suggestion |