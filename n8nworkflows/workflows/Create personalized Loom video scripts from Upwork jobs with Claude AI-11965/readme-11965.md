Create personalized Loom video scripts from Upwork jobs with Claude AI

https://n8nworkflows.xyz/workflows/create-personalized-loom-video-scripts-from-upwork-jobs-with-claude-ai-11965


# Create personalized Loom video scripts from Upwork jobs with Claude AI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (provided):** Create personalized Loom video scripts from Upwork jobs with Claude AI  
**Workflow name (in n8n):** Generate Loom Outreach Assets from Upwork Jobs with AI

**Purpose:**  
Convert an Upwork job post (title/description/client name/URL) into a complete set of outreach assets for a Loom video: a flow diagram, before/after transformation, a 90–120s Loom script, proposal snippet, visual prompts, and a reference card—then save results to Google Docs, log the lead in Google Sheets, and notify via Slack.

**Target use cases:**
- Rapidly producing personalized outreach content from Upwork job posts
- Standardizing pre-sales assets for automation/ops freelancers or agencies
- Creating a repeatable pipeline for lead capture → AI analysis → asset generation → logging/notification

### Logical Blocks
**1.1 Input Reception (Form Intake)**  
Captures the Upwork job details via an n8n Form Trigger.

**1.2 AI Job Analysis (Claude → JSON extraction + validation)**  
Sends job text to Anthropic Claude to extract structured fields; validates and normalizes the JSON.

**1.3 Asset Generation (Claude → long-form content)**  
Uses the validated analysis to prompt Claude to generate the complete Loom outreach package; structures output and computes token usage.

**1.4 Save & Notify (Docs + Sheets + Slack)**  
Creates a Google Doc, inserts the generated assets, appends a row to Google Sheets, and sends a Slack success message.

**1.5 Error Path (Slack)**  
If the AI analysis JSON is invalid/unparseable, sends a Slack alert including the raw response.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Form Intake)

**Overview:**  
Collects the job title, client name (optional), full job description, and an optional Upwork URL via an n8n-hosted form endpoint. Immediately returns a “processing” message to the submitter.

**Nodes involved:**
- Sticky Note1
- Job Intake Form

#### Node: Sticky Note1
- **Type / role:** Sticky Note; documentation overlay.
- **Configuration:** Text: “1. Capture Job…”
- **Connections:** None.
- **Edge cases:** None.

#### Node: Job Intake Form
- **Type / role:** `formTrigger`; entry point via a form + webhook path.
- **Key configuration choices:**
  - **Path / webhookId:** `upwork-loom-generator` (path is also set to the same).
  - **Form UI:**
    - Title: “Upwork Job → Loom Assets”
    - Description: explains to paste an Upwork job description
    - Fields:
      - **Job Title** (required)
      - **Client Name (if known)** (optional)
      - **Job Description** (textarea, required)
      - **Upwork Job URL (Optional)** (optional)
  - **Response behavior:** Responds with: “✅ Analyzing job description... Assets will be ready in ~45 seconds. Check Slack!”
- **Outputs / connections:**
  - Main output → **Analyze Job with AI**
- **Version-specific notes:** Node typeVersion `2.1` (Form Trigger behavior and UI fields depend on n8n version that supports Forms).
- **Edge cases / failures:**
  - Missing required fields prevents submission (Job Title, Job Description).
  - Large descriptions may increase downstream token usage or request size.

---

### 2.2 AI Job Analysis (Claude → JSON extraction + validation)

**Overview:**  
Sends a carefully constructed prompt to Anthropic Messages API asking for strict JSON output, then validates/parses it in Code. Routes invalid output to an error Slack alert.

**Nodes involved:**
- Sticky Note2
- Analyze Job with AI
- Validate AI Response
- Analysis Valid?
- Sticky Note5 (error path note)
- Send Error Alert (error route target)

#### Node: Sticky Note2
- **Type / role:** Sticky Note; documentation overlay.
- **Configuration:** “2. Analyze Upwork Job Description…”
- **Connections:** None.

#### Node: Analyze Job with AI
- **Type / role:** `httpRequest`; calls Anthropic Messages API to extract structured fields.
- **Endpoint:** `POST https://api.anthropic.com/v1/messages`
- **Authentication:** Generic credential type → **HTTP Header Auth**
  - Requires `x-api-key` header via credential configuration (not embedded in workflow JSON).
- **Headers explicitly set:**
  - `anthropic-version: 2023-06-01`
  - `content-type: application/json`
- **Body construction (expression):**
  - Builds sanitized strings from form fields:
    - Removes newlines (`\n\r`), replaces `"` with `'`, trims whitespace.
  - Creates a long prompt instructing Claude to:
    - Infer **industry** and **business_function** from enumerated categories
    - Extract pain points, tools, opportunities, signals (budget/urgency/competition)
    - Return **ONLY** a JSON object (no markdown)
  - Sends:
    - `model: "claude-sonnet-4-20250514"`
    - `max_tokens: 1500`
    - `messages: [{ role: "user", content: prompt }]`
- **Outputs / connections:**
  - Main output → **Validate AI Response**
- **Version-specific notes:** Node typeVersion `4.2` (HTTP Request node has evolved; authentication options may vary across n8n versions).
- **Edge cases / failures:**
  - Auth failures if Anthropic credential missing/invalid (`401/403`).
  - Anthropic rate limits / quota (`429`) or model availability changes.
  - Response not in expected structure (`content[0].text`) would break parsing unless handled downstream.
  - Prompt may elicit non-JSON output; downstream Code node attempts to sanitize.

#### Node: Validate AI Response
- **Type / role:** `code`; parses Claude output, validates required fields, normalizes into a stable schema.
- **Key logic:**
  - Reads first input item.
  - Extracts `responseText` from `item.content[0].text`; errors if missing.
  - Removes markdown fences if response begins with ``` or ```json.
  - `JSON.parse()` the cleaned text.
  - Validates required fields: `prospect_name`, `industry`, `business_function`, `pain_point`.
  - Outputs a normalized object with:
    - `valid: true`
    - Standardized fields like `known_tools` (from `tools_string`) and fallbacks for missing optional fields
    - `timestamp: new Date().toISOString()`
  - On error, outputs:
    - `valid: false`
    - `error` message
    - `raw_response` (best-effort)
- **Outputs / connections:**
  - Main output → **Analysis Valid?**
- **Version-specific notes:** Code node typeVersion `2` (n8n code execution environment depends on instance settings; some environments restrict certain modules—this script uses only standard JS).
- **Edge cases / failures:**
  - Claude returning trailing commentary breaks JSON parsing.
  - `content` array not present (API format change) triggers “No content…” error.
  - Missing required fields triggers explicit validation error.

#### Node: Analysis Valid?
- **Type / role:** `if`; routes based on whether parsing/validation succeeded.
- **Condition:**
  - Boolean equals: `{{ $json.valid }}` == `true`
- **Outputs / connections:**
  - **True** → Generate Loom Script with AI
  - **False** → Send Error Alert
- **Edge cases / failures:**
  - If upstream returns `valid` as string `"true"` instead of boolean, strict validation might fail; current validator sets boolean explicitly.

#### Node: Sticky Note5
- **Type / role:** Sticky Note; documents error path.
- **Configuration:** “Error Path… Notifies via Slack if job parsing fails”
- **Connections:** None.

#### Node: Send Error Alert
- **Type / role:** `slack`; sends an alert when parsing fails.
- **Message text:** Includes:
  - Error: `{{ $json.error }}`
  - Raw response inside triple backticks: `{{ $json.raw_response }}`
- **Destination:** Channel by ID: `YOUR_SLACK_CHANNEL_ID`
- **Outputs / connections:** No downstream nodes.
- **Version-specific notes:** Slack node typeVersion `2.1`.
- **Edge cases / failures:**
  - Missing/invalid Slack credential or channel ID.
  - Raw response can be very large; Slack message limits may truncate.

---

### 2.3 Asset Generation (Claude → long-form content)

**Overview:**  
Uses the structured analysis to prompt Claude to generate a full outreach “asset bundle”. Then extracts the text and token usage and merges it with the validated metadata.

**Nodes involved:**
- Sticky Note3
- Generate Loom Script with AI
- Structure Generated Content

#### Node: Sticky Note3
- **Type / role:** Sticky Note; documents asset generation stage.
- **Configuration:** “3. Generate Assets…”
- **Connections:** None.

#### Node: Generate Loom Script with AI
- **Type / role:** `httpRequest`; calls Anthropic Messages API to generate the final assets.
- **Endpoint:** `POST https://api.anthropic.com/v1/messages`
- **Authentication & headers:** Same pattern as analysis call:
  - HTTP Header Auth credential (x-api-key)
  - `anthropic-version: 2023-06-01`
  - `content-type: application/json`
- **Body construction (expression):**
  - Pulls from validated fields: prospect name, industry, business function, pain point, tools, hook angle, opportunities.
  - Includes an internal instruction section (“INTERNAL REASONING… do NOT include in output”) plus:
    - Industry-specific time savings and hourly values tables
    - Strict constraints: “EXACTLY: 1 Trigger → 4 Process Steps → 1 Output”, conversational, Upwork context
  - Generates sections:
    1) Automation flow diagram
    2) Before vs After
    3) Why this matters
    4) Loom script
    5) Proposal snippet
    6) Visual prompts
    7) Quick reference card
  - Uses placeholders for “MY BACKGROUND” to be customized.
  - Sends:
    - `model: "claude-sonnet-4-20250514"`
    - `max_tokens: 4500`
- **Outputs / connections:**
  - Main output → Structure Generated Content
- **Edge cases / failures:**
  - Same API risks as earlier (auth, rate limits, model changes).
  - Long prompts + long job descriptions can exceed context limits or increase cost.
  - Claude may not follow the “exactly 1 trigger → 4 steps” constraint; no downstream validator enforces this.

#### Node: Structure Generated Content
- **Type / role:** `code`; normalizes the final payload and captures token usage.
- **Key logic:**
  - Reads Claude generation response `item.content[0].text` into `generated_content`.
  - Computes `tokensUsed` from `item.usage.input_tokens + item.usage.output_tokens` if present.
  - Pulls the validated analysis data using node reference:
    - `const inputData = $('Validate AI Response').first().json;`
  - Outputs a single structured JSON object containing:
    - prospect/job metadata + signals
    - `generated_content`
    - `tokens_used`
    - `timestamp`
- **Outputs / connections:**
  - Main output → Create Output Doc
- **Edge cases / failures:**
  - If upstream response does not contain `.usage`, tokens_used becomes 0 (handled).
  - If `Validate AI Response` node has no item (unusual execution paths), the node reference can fail.

---

### 2.4 Save & Notify (Docs + Sheets + Slack)

**Overview:**  
Creates a Google Doc, inserts the generated content, logs key fields to a Google Sheet, then posts a formatted Slack message with links and signals.

**Nodes involved:**
- Sticky Note4
- Create Output Doc
- Add Content to Doc
- Log Lead to Sheets
- Send Success Notification

#### Node: Sticky Note4
- **Type / role:** Sticky Note; documents persistence + notification stage.
- **Configuration:** “4. Save & Notify…”
- **Connections:** None.

#### Node: Create Output Doc
- **Type / role:** `googleDocs`; creates a new Google Doc for the asset bundle.
- **Key configuration:**
  - **Title:** `{{ $('Structure Generated Content').item.json.prospect_name }} - Upwork Loom Assets`
  - **Folder ID:** `YOUR_GOOGLE_DRIVE_FOLDER_ID` (must be replaced)
- **Outputs / connections:**
  - Main output → Add Content to Doc
- **Credentials:** Google Docs OAuth2 (must be configured).
- **Edge cases / failures:**
  - Folder ID invalid or not shared with the OAuth user → permission errors.
  - Google API quota/permission issues.

#### Node: Add Content to Doc
- **Type / role:** `googleDocs`; inserts generated content into the created doc.
- **Operation:** Update document with action “insert”.
- **Key configuration:**
  - **Document URL/ID field:** `{{ $('Create Output Doc').item.json.id }}`
    - Note: despite parameter name `documentURL`, it’s being set to a document ID; this works in some configurations but may break if the node expects a full URL in your n8n version. If it fails, use the full URL: `https://docs.google.com/document/d/<id>/edit`.
  - **Insert text:** `{{ $('Structure Generated Content').item.json.generated_content }}`
- **Outputs / connections:**
  - Main output → Log Lead to Sheets
- **Edge cases / failures:**
  - Very long insert text may hit Google Docs API limits.
  - If the doc creation output differs, the ID reference may fail.

#### Node: Log Lead to Sheets
- **Type / role:** `googleSheets`; appends a row to a “Lead Log” sheet for tracking.
- **Operation:** Append to sheet.
- **Key configuration:**
  - **Spreadsheet ID:** `YOUR_GOOGLE_SHEET_ID` (must be replaced)
  - **Sheet name:** “Lead Log”
  - **Columns written:**
    - Version: `v2.2`
    - Industry, Prospect Name, Business Function, Pain Point
    - Timestamp: `{{ $now.format('yyyy-MM-dd HH:mm:ss') }}`
    - Tokens Used: computed from `Generate Loom Script with AI` usage fields
    - Google Doc Link: formula-like string:  
      `https://docs.google.com/document/d/{{ $('Create Output Doc').item.json.id }}/edit`
- **Outputs / connections:**
  - Main output → Send Success Notification
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - If `usage` is missing in the generation response, the “Tokens Used” expression may error (it directly references `.usage.input_tokens`). Consider switching to `Structure Generated Content.tokens_used`.
  - Sheet must exist and have matching columns; otherwise append may fail or mis-map.

#### Node: Send Success Notification
- **Type / role:** `slack`; posts the success summary and links.
- **Destination:** Channel ID: `YOUR_SLACK_CHANNEL_ID`
- **Message content includes:**
  - Job title, prospect name, industry, function
  - Signals (budget/urgency/competition)
  - Google Doc link
  - Job URL (fallback “Not provided”)
  - Recommended approach
  - Tokens used
- **Outputs / connections:** No downstream nodes.
- **Edge cases / failures:**
  - Slack auth/channel issues.
  - Slack formatting or message length constraints if fields are long.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Workflow description, setup checklist, customization notes |  |  | ## Generate Loom Outreach Assets from Upwork Jobs with AI… (setup/customize checklist incl. credentials, folder/sheet/channel IDs, sheet columns) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documents Block 1: capture job |  |  | ## 1. Capture Job… |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documents Block 2: analyze job |  |  | ## 2. Analyze Upwork Job Description… |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documents Block 3: generate assets |  |  | ## 3. Generate Assets… |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documents Block 4: save & notify |  |  | ## 4. Save & Notify… |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documents error path |  |  | ## Error Path… Notifies via Slack if job parsing fails |
| Job Intake Form | n8n-nodes-base.formTrigger | Collect Upwork job details via form | (Entry) | Analyze Job with AI | ## 1. Capture Job… |
| Analyze Job with AI | n8n-nodes-base.httpRequest | Call Anthropic to extract structured JSON fields | Job Intake Form | Validate AI Response | ## 2. Analyze Upwork Job Description… |
| Validate AI Response | n8n-nodes-base.code | Parse/validate AI JSON; normalize fields | Analyze Job with AI | Analysis Valid? | ## 2. Analyze Upwork Job Description… |
| Analysis Valid? | n8n-nodes-base.if | Route success vs parse error | Validate AI Response | Generate Loom Script with AI; Send Error Alert | ## 2. Analyze Upwork Job Description… |
| Generate Loom Script with AI | n8n-nodes-base.httpRequest | Call Anthropic to generate Loom asset bundle | Analysis Valid? (true) | Structure Generated Content | ## 3. Generate Assets… |
| Structure Generated Content | n8n-nodes-base.code | Extract generated text + tokens; merge metadata | Generate Loom Script with AI | Create Output Doc | ## 3. Generate Assets… |
| Create Output Doc | n8n-nodes-base.googleDocs | Create a Google Doc in target Drive folder | Structure Generated Content | Add Content to Doc | ## 4. Save & Notify… |
| Add Content to Doc | n8n-nodes-base.googleDocs | Insert generated content into the doc | Create Output Doc | Log Lead to Sheets | ## 4. Save & Notify… |
| Log Lead to Sheets | n8n-nodes-base.googleSheets | Append lead row to “Lead Log” sheet | Add Content to Doc | Send Success Notification | ## 4. Save & Notify… |
| Send Success Notification | n8n-nodes-base.slack | Post success summary to Slack | Log Lead to Sheets |  | ## 4. Save & Notify… |
| Send Error Alert | n8n-nodes-base.slack | Post parse error + raw response to Slack | Analysis Valid? (false) |  | ## Error Path… Notifies via Slack if job parsing fails |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: “Generate Loom Outreach Assets from Upwork Jobs with AI” (or your preferred name).
   - Keep workflow **Inactive** until credentials/IDs are set.

2) **Add the documentation sticky notes (optional but recommended)**
   - Add 6 Sticky Note nodes and paste the corresponding texts:
     - Main overview/setup checklist (credentials, folder/sheet/channel IDs, sheet columns).
     - Block labels: “1. Capture Job”, “2. Analyze…”, “3. Generate Assets”, “4. Save & Notify”, and “Error Path”.

3) **Create the intake node**
   - Add **Form Trigger** node named **Job Intake Form**.
   - Configure:
     - Path: `upwork-loom-generator`
     - Form title: “Upwork Job → Loom Assets”
     - Form description: “Paste an Upwork job description…”
     - Fields:
       - Job Title (required)
       - Client Name (optional)
       - Job Description (textarea, required)
       - Upwork Job URL (optional)
     - Response text on submit: “✅ Analyzing job description... Assets will be ready in ~45 seconds. Check Slack!”

4) **Create Anthropic credential**
   - In n8n Credentials, create **HTTP Header Auth** credential (for HTTP Request nodes):
     - Header name: `x-api-key`
     - Value: your Anthropic API key
   - You will select this credential in both Anthropic HTTP Request nodes.

5) **Add “Analyze Job with AI” (Anthropic extraction call)**
   - Add **HTTP Request** node named **Analyze Job with AI**.
   - Configure:
     - Method: POST
     - URL: `https://api.anthropic.com/v1/messages`
     - Authentication: Generic → HTTP Header Auth (select your Anthropic credential)
     - Send Headers: enabled
     - Additional headers:
       - `anthropic-version` = `2023-06-01`
       - `content-type` = `application/json`
     - Body: JSON (as an expression) that:
       - Uses model `claude-sonnet-4-20250514`
       - `max_tokens: 1500`
       - `messages: [{role:"user", content: <prompt>}]`
       - Prompt instructs Claude to return strict JSON with keys like `prospect_name`, `industry`, `business_function`, `pain_point`, etc.
     - Include sanitization (remove newlines, replace quotes) for the job fields.

6) **Add “Validate AI Response” (parse + normalize)**
   - Add **Code** node named **Validate AI Response**.
   - Paste logic that:
     - Reads `item.content[0].text`
     - Strips ``` fences if present
     - `JSON.parse()` it
     - Validates required fields: prospect_name/industry/business_function/pain_point
     - Outputs `{valid:true, ...normalized fields...}` else `{valid:false, error, raw_response}`

7) **Add router “Analysis Valid?”**
   - Add an **IF** node named **Analysis Valid?**
   - Condition: Boolean equals → `{{$json.valid}}` is `true`
   - True path continues to generation; false path goes to Slack error.

8) **Create Slack credential + error alert**
   - Add Slack credential in n8n (Slack API).
   - Add **Slack** node named **Send Error Alert**.
     - Post to channel by ID (set `YOUR_SLACK_CHANNEL_ID`).
     - Message includes `{{$json.error}}` and `{{$json.raw_response}}` in a code block.

9) **Add “Generate Loom Script with AI” (Anthropic generation call)**
   - Add **HTTP Request** node named **Generate Loom Script with AI**.
   - Configure similarly to the analysis call:
     - POST `https://api.anthropic.com/v1/messages`
     - Same Anthropic auth + headers
     - Body JSON:
       - model `claude-sonnet-4-20250514`
       - `max_tokens: 4500`
       - Prompt includes:
         - Industry-specific time/hourly tables
         - Constraint: “EXACTLY 1 Trigger → 4 Process Steps → 1 Output”
         - “MY BACKGROUND” section you must customize
         - Output sections 1–7 (diagram, before/after, Loom script, proposal snippet, etc.)
     - Prompt variables come from the validated fields (prospect_name, pain_point, known_tools, etc.)

10) **Add “Structure Generated Content”**
   - Add **Code** node named **Structure Generated Content**.
   - Logic:
     - Extract `item.content[0].text` as `generated_content`
     - Compute `tokens_used` from `item.usage`
     - Merge with the validated analysis data (reference the “Validate AI Response” node output).

11) **Set up Google credentials**
   - Create/configure **Google Docs OAuth2** credential.
   - Create/configure **Google Sheets OAuth2** credential.
   - Ensure the OAuth user has access to the Drive folder and the spreadsheet.

12) **Create Google Doc output**
   - Add **Google Docs** node named **Create Output Doc**:
     - Operation: create document
     - Title: `{{ $('Structure Generated Content').item.json.prospect_name }} - Upwork Loom Assets`
     - Folder ID: replace `YOUR_GOOGLE_DRIVE_FOLDER_ID` with your Drive folder ID.

13) **Insert content into the Doc**
   - Add **Google Docs** node named **Add Content to Doc**:
     - Operation: update
     - Action: insert text
     - Text: `{{ $('Structure Generated Content').item.json.generated_content }}`
     - Document identifier: reference the created doc ID (or use full URL if required by your node version).

14) **Log lead into Google Sheets**
   - Add **Google Sheets** node named **Log Lead to Sheets**:
     - Operation: append
     - Spreadsheet ID: replace `YOUR_GOOGLE_SHEET_ID`
     - Sheet name: `Lead Log`
     - Map columns (create them in the sheet first):
       - Timestamp, Prospect Name, Industry, Business Function, Pain Point, Tokens Used, Google Doc Link, Version
     - Prefer mapping **Tokens Used** from `Structure Generated Content.tokens_used` to avoid `.usage` missing errors.

15) **Send Slack success notification**
   - Add **Slack** node named **Send Success Notification**:
     - Channel ID: `YOUR_SLACK_CHANNEL_ID`
     - Message includes: job title, prospect, industry/function, signals, Google Doc link, job URL, recommended approach, tokens used.

16) **Wire the nodes in order**
   - Job Intake Form → Analyze Job with AI → Validate AI Response → Analysis Valid?
     - True → Generate Loom Script with AI → Structure Generated Content → Create Output Doc → Add Content to Doc → Log Lead to Sheets → Send Success Notification
     - False → Send Error Alert

17) **Create the Google Sheet structure**
   - Create a spreadsheet tab named **Lead Log** with columns:  
     `Timestamp | Prospect Name | Industry | Business Function | Pain Point | Tokens Used | Google Doc Link | Version`

18) **Test**
   - Run the form, paste a real Upwork job description, and verify:
     - Slack receives success message
     - Google Doc is created and populated
     - Sheets row is appended
   - Test a failure case (force invalid JSON by editing the analysis prompt) to verify Slack error path.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Setup checklist: Anthropic API credential (HTTP Header Auth, header `x-api-key`), Google Docs/Sheets OAuth2, Slack credential, set Drive folder ID, Sheet ID, Slack channel ID, create Lead Log columns | From main Sticky Note content |
| Customization: edit “MY BACKGROUND” in `Generate Loom Script with AI`; adjust hourly rates/time savings; modify pricing guidance and CTA | From main Sticky Note content |
| Workflow intent: generates complete outreach package from pasted job in ~60 seconds | From main Sticky Note content |