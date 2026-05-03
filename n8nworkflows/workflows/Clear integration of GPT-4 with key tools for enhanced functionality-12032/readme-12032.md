Clear integration of GPT-4 with key tools for enhanced functionality

https://n8nworkflows.xyz/workflows/clear-integration-of-gpt-4-with-key-tools-for-enhanced-functionality-12032


# Clear integration of GPT-4 with key tools for enhanced functionality

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *AI Proposal Generator - Typeform to PandaDoc with GPT-4*  
**Stated title:** *Clear integration of GPT-4 with key tools for enhanced functionality*

**Purpose:** Automatically turn a Typeform “discovery call” submission into a professional proposal/quote using GPT-4, generate a realistic project timeline, create a PandaDoc document from a chosen template, and notify via Slack. The workflow includes validation and error notifications for missing fields and AI parsing failures.

### 1.1 Capture & Configuration
- Stores company identity, template IDs, thresholds, and Slack webhook.
- Receives Typeform submissions and normalizes them into a consistent internal schema.

### 1.2 Validate & Route
- Ensures the submission has required fields (email, company).
- Routes to either “Quick Quote” or “Standard Proposal” generation based on budget and complexity.

### 1.3 AI Proposal + AI Timeline
- Uses GPT-4 to produce proposal content as JSON.
- Parses/cleans the model response into structured fields.
- Generates milestone timeline JSON and parses it with fallback defaults.

### 1.4 Document Assembly (PandaDoc)
- Merges proposal + milestones and ensures parsing succeeded.
- Creates the PandaDoc document using tokens and a pricing table.
- Verifies a document ID exists.

### 1.5 Notifications & Error Handling
- Sends Slack webhook notifications for success.
- Sends Slack webhook notifications for validation failures, AI parsing errors, and PandaDoc failures.

---

## 2. Block-by-Block Analysis

### Block 1 — Capture & Extract

**Overview:** Provides configuration, receives Typeform payloads, and converts them into a normalized proposal-ready object (including budget parsing, template selection, and dates).

**Nodes involved:**
- ⚙️ Config
- 📥 Typeform Trigger
- 🔄 Extract & Transform Data

#### Node: ⚙️ Config
- **Type / role:** `Set` node — static configuration provider.
- **Key configuration:**
  - Uses **Raw JSON mode** to emit a single JSON object containing:
    - Sender identity fields (email/name/company/title/phone)
    - PandaDoc template IDs (`quickQuoteTemplateId`, `standardProposalTemplateId`)
    - Threshold (`quickQuoteThreshold` as a string `"2500"`)
    - Slack webhook URL (`slackWebhookUrl`)
    - Case studies object (`caseStudies.default`)
- **Key variables used later:**
  - `quickQuoteThreshold`, `quickQuoteTemplateId`, `standardProposalTemplateId`, `slackWebhookUrl`, `caseStudies.default`
- **Connections:** Not connected as a main path input; referenced by expression from other nodes via `$('⚙️ Config').first().json`.
- **Version notes:** Set node v3.4; raw JSON output behavior depends on n8n versions supporting `jsonOutput`.
- **Failure/edge cases:**
  - Missing/invalid template IDs will cause PandaDoc creation to fail.
  - `quickQuoteThreshold` is a string; later converted with `Number(...)`—non-numeric values fall back to `2500`.
  - Missing Slack webhook URL results in notification HTTP node failures (but those nodes use `continueOnFail`).

#### Node: 📥 Typeform Trigger
- **Type / role:** `Typeform Trigger` — workflow entry point on form submission.
- **Key configuration:** `formId = YOUR_TYPEFORM_FORM_ID` (must be replaced).
- **Output:** Emits a JSON representation of the submission (field labels used as keys).
- **Connections:** Outputs to **🔄 Extract & Transform Data**.
- **Failure/edge cases:**
  - Credential/auth issues with Typeform.
  - If Typeform question labels change, downstream extraction that relies on exact strings will produce empty values.

#### Node: 🔄 Extract & Transform Data
- **Type / role:** `Code` node — transforms Typeform raw answers into a structured schema.
- **Key configuration choices:**
  - Reads config: `const config = $('⚙️ Config').first().json;`
  - Reads submission: `const formData = $input.first().json;`
  - Extracts fields by exact Typeform label text (e.g., `"What's your company name?"`, `"Email address"`).
  - Parses budget label into a numeric `estimatedValue` via `parseBudget()`.
  - Determines template selection:
    - `threshold = Number(config.quickQuoteThreshold) || 2500`
    - `isComplex = projectComplexity includes 'complex' (case-insensitive)`
    - `selectedTemplate = quick_quote` only if `estimatedValue < threshold` AND not complex.
  - Computes:
    - `depositAmount = round(estimatedValue * 0.5)`
    - `balanceAmount = estimatedValue - depositAmount`
    - Dates: `documentDate` (today), `proposalExpiryDate` (today + 7 days for quick quote, +14 for standard)
  - Embeds config and case study:
    - `extractedData.config = config`
    - `extractedData.caseStudy = config.caseStudies?.default || {}`
- **Output:** One item with normalized JSON (company/client fields, budget numbers, template selection, dates, config).
- **Connections:** Outputs to **✅ Check Required Fields**.
- **Failure/edge cases:**
  - If Typeform keys don’t match exact labels, values become empty strings; validation may fail.
  - Budget parsing defaults to `3000` for unknown labels (can route incorrectly).
  - `projectComplexity` “complex” detection is simplistic (only checks substring `"complex"`).

---

### Block 2 — Validate & Route to AI

**Overview:** Ensures minimum required lead data exists, then routes to the appropriate GPT prompt for either a quick quote or a full proposal.

**Nodes involved:**
- ✅ Check Required Fields
- 🔀 Route: Quick Quote or Standard?
- 🤖 AI: Generate Quick Quote
- 🤖 AI: Generate Standard Proposal
- 🚨 Notify: Missing Required Fields (error branch)

#### Node: ✅ Check Required Fields
- **Type / role:** `IF` node — data validation gate.
- **Conditions:**
  - `clientEmail` is not empty
  - `companyName` is not empty
- **Outputs:**
  - **True** → 🔀 Route: Quick Quote or Standard?
  - **False** → 🚨 Notify: Missing Required Fields
- **Failure/edge cases:**
  - If extraction failed due to renamed Typeform fields, this branch will fire frequently.
  - “Strict” type validation is enabled; if unexpected types occur, comparisons may behave unexpectedly.

#### Node: 🔀 Route: Quick Quote or Standard?
- **Type / role:** `IF` node — routing by `selectedTemplate`.
- **Condition:** `$json.selectedTemplate === 'quick_quote'`
- **Outputs:**
  - **True** → 🤖 AI: Generate Quick Quote
  - **False** → 🤖 AI: Generate Standard Proposal
- **Failure/edge cases:**
  - If `selectedTemplate` is missing/empty, it will route to the “false” output (standard).

#### Node: 🤖 AI: Generate Quick Quote
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat completion via LangChain wrapper.
- **Key configuration choices:**
  - Temperature `0.7` (more creative).
  - System prompt: proposal writer, concise, benefit-focused.
  - User prompt injects: company, industry, challenge, desired outcome, tools, budget.
  - Requires output **ONLY valid JSON** with keys like `title`, `challengeSummary`, `solutionType`, impact bullets, `timeSavings`, `costSavings`.
  - `jsonOutput: true` enabled (node expects JSON mode behavior).
- **Connections:** Output → 📋 Parse AI Proposal Response
- **Failure/edge cases:**
  - Missing OpenAI credentials/model selection (`modelId` is empty in JSON export; must be set).
  - Model may still return non-JSON or wrap in markdown; handled later by parser.

#### Node: 🤖 AI: Generate Standard Proposal
- **Type / role:** same OpenAI node — longer structured proposal JSON.
- **Key configuration choices:**
  - Temperature `0.7`.
  - System prompt: detailed, ROI-focused.
  - User prompt includes more fields (businessDescription, complexity, additional notes).
  - Output JSON includes extra keys: `outcomeProcess1..3` in addition to impact bullets, etc.
- **Connections:** Output → 📋 Parse AI Proposal Response
- **Failure/edge cases:** Same as quick quote; also more tokens/latency due to longer response.

#### Node: 🚨 Notify: Missing Required Fields
- **Type / role:** `HTTP Request` — Slack incoming webhook notification (validation error).
- **Key configuration choices:**
  - POST to `$('⚙️ Config').first().json.slackWebhookUrl`
  - Body is built via `JSON.stringify({ text, blocks })` (Slack Block Kit payload).
  - Uses the current item’s `$json` for `clientEmail`, `companyName`.
  - `continueOnFail: true`
- **Connections:** Terminal (no outputs)
- **Failure/edge cases:**
  - Slack webhook URL missing/invalid → node fails but workflow continues (because continueOnFail).
  - Slack may reject malformed blocks JSON (rare, but possible if string building breaks).

---

### Block 3 — Parse, Timeline & Merge

**Overview:** Cleans and parses the GPT proposal JSON, generates a timeline with GPT, parses it with fallback defaults, then merges proposal and milestones into one document payload.

**Nodes involved:**
- 📋 Parse AI Proposal Response
- 🤖 AI: Generate Project Milestones
- 📋 Parse Milestone Response
- 🔗 Combine Proposal + Milestones

#### Node: 📋 Parse AI Proposal Response
- **Type / role:** `Code` node — robust-ish JSON extraction + merge.
- **Key configuration choices:**
  - Retrieves extracted baseline data from: `$('🔄 Extract & Transform Data').first().json`
  - Reads model output from `$input.first().json` and tries:
    - `raw.message?.content ?? raw.content ?? raw`
  - If string:
    - Strips markdown fences: ```json, ```
    - `JSON.parse(content)`
  - Merges:
    - `{ ...extractedData, ...content, proposalParseOk: true }`
  - On error:
    - `{ ...extractedData, proposalParseOk: false, proposalParseError: e.message }`
- **Connections:**
  - Output goes to:
    1) 🤖 AI: Generate Project Milestones (index 0)
    2) 🔗 Combine Proposal + Milestones (index 1)
- **Failure/edge cases:**
  - If GPT returns valid JSON but with unexpected keys, PandaDoc tokens may be blank later.
  - If GPT returns JSON but types mismatch (numbers vs strings), downstream formatting may be affected.
  - If parsing fails, workflow still proceeds to merge and later gates on `proposalParseOk`.

#### Node: 🤖 AI: Generate Project Milestones
- **Type / role:** OpenAI node — timeline generation.
- **Key configuration choices:**
  - Temperature `0.5` (more deterministic).
  - Prompt uses:
    - `{{ $json.solutionType }}`, `{{ $json.scopeDescription }}`
    - Complexity taken directly from extraction node: `{{ $('🔄 Extract & Transform Data').first().json.projectComplexity }}`
  - Returns JSON with `milestone1..4` and `timeline1..4`
  - `jsonOutput: true`
- **Connections:** Output → 📋 Parse Milestone Response
- **Failure/edge cases:**
  - If proposal parsing failed, `solutionType`/`scopeDescription` may be missing; milestones may degrade or become generic.

#### Node: 📋 Parse Milestone Response
- **Type / role:** `Code` node — parses timeline JSON, provides defaults on error.
- **Key configuration choices:**
  - Same markdown-fence stripping + JSON.parse strategy.
  - On parse error: returns a default 4-phase milestone set and flags `milestonesParseOk: false`.
- **Connections:** Output → 🔗 Combine Proposal + Milestones
- **Failure/edge cases:**
  - Even on failure, it returns usable default milestones, so document creation can proceed.

#### Node: 🔗 Combine Proposal + Milestones
- **Type / role:** `Merge` node — combines two streams into one item.
- **Key configuration:**
  - Mode: `combine`
  - Combine by: `position` (combineByPosition)
  - Inputs:
    - Input 1: from 📋 Parse AI Proposal Response
    - Input 2: from 📋 Parse Milestone Response
- **Output:** Single combined JSON containing proposal + timeline fields.
- **Connections:** Output → ✅ Check: AI Parse Successful?
- **Failure/edge cases:**
  - If one branch produces no item, combine-by-position may yield empty output or mismatched merges.
  - Execution order matters; n8n merge semantics can vary if items counts differ.

---

### Block 4 — Create PandaDoc

**Overview:** Ensures proposal JSON parsing succeeded, then creates a PandaDoc document using template tokens and a pricing table, and verifies a document ID was returned.

**Nodes involved:**
- ✅ Check: AI Parse Successful?
- 📄 Create PandaDoc Document
- ✅ Check: Document Created?
- 🚨 Notify: GPT Parse Failed (error branch)

#### Node: ✅ Check: AI Parse Successful?
- **Type / role:** `IF` node — blocks PandaDoc creation if GPT proposal parsing failed.
- **Condition:** `$json.proposalParseOk === true`
- **Outputs:**
  - **True** → 📄 Create PandaDoc Document
  - **False** → 🚨 Notify: GPT Parse Failed
- **Failure/edge cases:**
  - This gate checks only proposal parse success, not milestone parse success (milestones have defaults anyway).

#### Node: 📄 Create PandaDoc Document
- **Type / role:** `HTTP Request` — PandaDoc API call to create a document from template.
- **Authentication:**
  - `genericCredentialType` + `httpHeaderAuth` (you must configure PandaDoc API key header in credentials).
  - Also sets `Content-Type: application/json` header.
- **Request:**
  - POST `https://api.pandadoc.com/public/v1/documents`
  - Body includes:
    - `name`: `{{ $json.title }} - {{ $json.companyName }}`
    - `template_uuid`: `{{ $json.pandaDocTemplateId }}`
    - `recipients`: uses client email/name, role `Client`
    - `tokens`: many template tokens like `Client.Company`, `Timeline.Phase1`, `Impact.Bullet1`, etc.
    - `pricing_tables`: builds a single row priced at `estimatedValue`, quantity 1, currency USD.
- **Operational choices:**
  - `continueOnFail: true` (workflow won’t hard-stop; downstream IF decides success/failure).
- **Connections:** Output → ✅ Check: Document Created?
- **Failure/edge cases:**
  - Missing/incorrect PandaDoc API key → 401/403.
  - Invalid `template_uuid` → 404/validation error.
  - Token names must match PandaDoc template tokens exactly; otherwise values won’t populate.
  - If `estimatedValue` is not numeric, the pricing table `price` may be invalid JSON (expression injects raw number).
  - PandaDoc may return success but with different ID field; this workflow checks `$json.id`.

#### Node: ✅ Check: Document Created?
- **Type / role:** `IF` node — verifies PandaDoc returned an ID.
- **Condition:** `$json.id` is not empty.
- **Outputs:**
  - **True** → 💬 Notify: Proposal Ready
  - **False** → 🚨 Notify: PandaDoc Failed
- **Failure/edge cases:**
  - If PandaDoc returns the ID in a different field (unlikely for this endpoint, but API changes happen), it will incorrectly route to failure.

#### Node: 🚨 Notify: GPT Parse Failed
- **Type / role:** `HTTP Request` — Slack webhook error notification for invalid AI JSON.
- **Key configuration:**
  - POST to Slack webhook from config.
  - Uses current `$json` fields from the parse node output: `companyName`, `clientEmail`, `selectedTemplate`, `proposalParseError`.
  - `continueOnFail: true`
- **Connections:** Terminal
- **Failure/edge cases:** Same Slack webhook concerns as other notify nodes.

---

### Block 5 — Send Notifications

**Overview:** Notifies Slack on successful PandaDoc creation with a direct document link, or on PandaDoc failure with error context.

**Nodes involved:**
- 💬 Notify: Proposal Ready
- 🚨 Notify: PandaDoc Failed

#### Node: 💬 Notify: Proposal Ready
- **Type / role:** `HTTP Request` — Slack webhook success message.
- **Key configuration:**
  - POST to Slack webhook URL from config.
  - Block Kit message includes:
    - Client name, estimated value, template type, expiry date (pulled from **🔄 Extract & Transform Data** via node reference).
    - Button link: `https://app.pandadoc.com/a/#/documents/` + `$json.id`
  - `continueOnFail: true`
- **Connections:** Terminal
- **Failure/edge cases:**
  - If PandaDoc document URL format changes, the link may be incorrect.
  - Uses values from Extract node rather than the combined payload; if upstream changes, keep references consistent.

#### Node: 🚨 Notify: PandaDoc Failed
- **Type / role:** `HTTP Request` — Slack webhook failure message.
- **Key configuration:**
  - POST to Slack webhook URL from config.
  - Shows client, email, selectedTemplate (from Extract node reference) and error from `$json.message || $json.error || 'Unknown error...'`
  - `continueOnFail: true`
- **Connections:** Terminal
- **Failure/edge cases:**
  - PandaDoc error structure may not include `message`/`error`; you may want to include `$json` dump or `$json.body` depending on n8n’s HTTP node error shape.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Visual documentation | — | — | ## 📄 AI Proposal Generator  \nAutomatically generate professional, personalized proposals from discovery call intake forms.  \n… (Setup, Customize) |
| Sticky Note1 | Sticky Note | Visual documentation | — | — | ## 1. Capture & Extract  \nThis section handles the intake process… |
| Sticky Note2 | Sticky Note | Visual documentation | — | — | ## 2. Validate & Route to AI  \nEnsures data quality before AI processing… |
| Sticky Note3 | Sticky Note | Visual documentation | — | — | ## 3. Parse, Timeline & Merge  \nProcesses AI output and adds project schedule… |
| Sticky Note4 | Sticky Note | Visual documentation | — | — | ## 4. Create PandaDoc  \nAssembles the final proposal document… |
| Sticky Note5 | Sticky Note | Visual documentation | — | — | ## 5. Send Notifications  \nKeeps you informed of every submission… |
| ⚙️ Config | Set | Central configuration store | — | (Referenced by expressions) | ## 1. Capture & Extract  \nThis section handles the intake process… |
| 📥 Typeform Trigger | Typeform Trigger | Entry point: capture discovery form submission | — | 🔄 Extract & Transform Data | ## 1. Capture & Extract  \nThis section handles the intake process… |
| 🔄 Extract & Transform Data | Code | Normalize fields, parse budget, choose template, compute dates | 📥 Typeform Trigger | ✅ Check Required Fields | ## 1. Capture & Extract  \nThis section handles the intake process… |
| ✅ Check Required Fields | IF | Validate presence of email and company | 🔄 Extract & Transform Data | 🔀 Route: Quick Quote or Standard?; 🚨 Notify: Missing Required Fields | ## 2. Validate & Route to AI  \nEnsures data quality before AI processing… |
| 🔀 Route: Quick Quote or Standard? | IF | Route to quick quote vs standard proposal | ✅ Check Required Fields | 🤖 AI: Generate Quick Quote; 🤖 AI: Generate Standard Proposal | ## 2. Validate & Route to AI  \nEnsures data quality before AI processing… |
| 🤖 AI: Generate Quick Quote | OpenAI (LangChain) | Generate concise proposal JSON | 🔀 Route: Quick Quote or Standard? | 📋 Parse AI Proposal Response | ## 2. Validate & Route to AI  \nEnsures data quality before AI processing… |
| 🤖 AI: Generate Standard Proposal | OpenAI (LangChain) | Generate detailed proposal JSON | 🔀 Route: Quick Quote or Standard? | 📋 Parse AI Proposal Response | ## 2. Validate & Route to AI  \nEnsures data quality before AI processing… |
| 📋 Parse AI Proposal Response | Code | Parse GPT JSON, clean markdown, merge with extracted data | 🤖 AI: Generate Quick Quote / 🤖 AI: Generate Standard Proposal | 🤖 AI: Generate Project Milestones; 🔗 Combine Proposal + Milestones | ## 3. Parse, Timeline & Merge  \nProcesses AI output and adds project schedule… |
| 🤖 AI: Generate Project Milestones | OpenAI (LangChain) | Generate 4-phase timeline JSON | 📋 Parse AI Proposal Response | 📋 Parse Milestone Response | ## 3. Parse, Timeline & Merge  \nProcesses AI output and adds project schedule… |
| 📋 Parse Milestone Response | Code | Parse timeline JSON; fallback to defaults on failure | 🤖 AI: Generate Project Milestones | 🔗 Combine Proposal + Milestones | ## 3. Parse, Timeline & Merge  \nProcesses AI output and adds project schedule… |
| 🔗 Combine Proposal + Milestones | Merge | Combine proposal payload with milestone payload | 📋 Parse AI Proposal Response; 📋 Parse Milestone Response | ✅ Check: AI Parse Successful? | ## 3. Parse, Timeline & Merge  \nProcesses AI output and adds project schedule… |
| ✅ Check: AI Parse Successful? | IF | Gate: only proceed if proposal JSON parsed | 🔗 Combine Proposal + Milestones | 📄 Create PandaDoc Document; 🚨 Notify: GPT Parse Failed | ## 4. Create PandaDoc  \nAssembles the final proposal document… |
| 📄 Create PandaDoc Document | HTTP Request | Create PandaDoc doc from template with tokens/pricing | ✅ Check: AI Parse Successful? | ✅ Check: Document Created? | ## 4. Create PandaDoc  \nAssembles the final proposal document… |
| ✅ Check: Document Created? | IF | Verify PandaDoc returned document ID | 📄 Create PandaDoc Document | 💬 Notify: Proposal Ready; 🚨 Notify: PandaDoc Failed | ## 4. Create PandaDoc  \nAssembles the final proposal document… |
| 💬 Notify: Proposal Ready | HTTP Request | Slack success message with PandaDoc link | ✅ Check: Document Created? | — | ## 5. Send Notifications  \nKeeps you informed of every submission… |
| 🚨 Notify: PandaDoc Failed | HTTP Request | Slack failure message with error details | ✅ Check: Document Created? | — | ## 5. Send Notifications  \nKeeps you informed of every submission… |
| 🚨 Notify: GPT Parse Failed | HTTP Request | Slack error if GPT JSON parsing fails | ✅ Check: AI Parse Successful? | — | ## ⚠️ Error Handling  \nCaptures issues at every stage… |
| 🚨 Notify: Missing Required Fields | HTTP Request | Slack error if form lacks email/company | ✅ Check Required Fields | — | ## ⚠️ Error Handling  \nCaptures issues at every stage… |
| Sticky Note Error | Sticky Note | Visual documentation | — | — | ## ⚠️ Error Handling  \nCaptures issues at every stage… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: `AI Proposal Generator - Typeform to PandaDoc with GPT-4`
- (Optional) Set execution order to `v1` (Workflow Settings → Execution order).

2) **Add “⚙️ Config” (Set node)**
- Node type: **Set**
- Mode: **Raw JSON**
- Paste and adjust values:
  - `senderEmail`, `senderFirstName`, `senderLastName`, `senderCompany`, `senderTitle`, `senderPhone`
  - `quickQuoteTemplateId`, `standardProposalTemplateId`
  - `quickQuoteThreshold` (e.g., `"2500"`)
  - `slackWebhookUrl` (Slack incoming webhook URL)
  - `caseStudies.default` (optional)
- No connections required (it will be referenced by expressions).

3) **Add “📥 Typeform Trigger”**
- Node type: **Typeform Trigger**
- Credential: **Typeform API** (configure in n8n Credentials)
- Set **Form ID** to your Typeform form ID.
- Connect: **📥 Typeform Trigger → 🔄 Extract & Transform Data**

4) **Add “🔄 Extract & Transform Data” (Code node)**
- Node type: **Code**
- Paste logic that:
  - Reads `$('⚙️ Config').first().json`
  - Extracts Typeform answers by label text
  - Parses budget to numeric `estimatedValue`
  - Sets `selectedTemplate`, `pandaDocTemplateId`, deposit/balance, `documentDate`, `proposalExpiryDate`
- Connect: **🔄 Extract & Transform Data → ✅ Check Required Fields**

5) **Add “✅ Check Required Fields” (IF node)**
- Node type: **IF**
- Conditions (AND):
  - String → `clientEmail` → **not empty**
  - String → `companyName` → **not empty**
- Connect:
  - **True** output → **🔀 Route: Quick Quote or Standard?**
  - **False** output → **🚨 Notify: Missing Required Fields**

6) **Add “🔀 Route: Quick Quote or Standard?” (IF node)**
- Condition:
  - String equals: `{{ $json.selectedTemplate }}` equals `quick_quote`
- Connect:
  - **True** → **🤖 AI: Generate Quick Quote**
  - **False** → **🤖 AI: Generate Standard Proposal**

7) **Add OpenAI nodes (LangChain)**
- Node type: **OpenAI (LangChain)** `@n8n/n8n-nodes-langchain.openAi`
- Credential: configure **OpenAI API** in n8n Credentials and select it in each node.
- Select a model (e.g., GPT-4 class model available in your account).
- Create:
  - **🤖 AI: Generate Quick Quote**
    - Temperature 0.7
    - System + user messages (ensure user message demands “ONLY valid JSON (no markdown backticks)”)
    - Enable JSON output (or equivalent structured output setting in your node version)
  - **🤖 AI: Generate Standard Proposal**
    - Temperature 0.7
    - Similar but more detailed JSON schema
- Connect each AI node output → **📋 Parse AI Proposal Response**

8) **Add “📋 Parse AI Proposal Response” (Code node)**
- Node type: **Code**
- Implement:
  - Read AI content from `raw.message.content` or similar
  - Strip ```json fences
  - JSON.parse
  - Merge with extracted data from `$('🔄 Extract & Transform Data').first().json`
  - Add `proposalParseOk` boolean and optional `proposalParseError`
- Connect outputs to:
  - **🤖 AI: Generate Project Milestones**
  - **🔗 Combine Proposal + Milestones** (as the *second* input of Merge or input 1—just be consistent with combine-by-position)

9) **Add “🤖 AI: Generate Project Milestones” (OpenAI LangChain node)**
- Temperature 0.5
- Prompt must output ONLY JSON with `milestone1..4` and `timeline1..4`
- Connect → **📋 Parse Milestone Response**

10) **Add “📋 Parse Milestone Response” (Code node)**
- Parse and fallback to defaults if JSON parsing fails
- Connect → **🔗 Combine Proposal + Milestones** (other merge input)

11) **Add “🔗 Combine Proposal + Milestones” (Merge node)**
- Node type: **Merge**
- Mode: **Combine**
- Combine by: **Position**
- Inputs:
  - One input from proposal parse
  - One input from milestone parse
- Connect → **✅ Check: AI Parse Successful?**

12) **Add “✅ Check: AI Parse Successful?” (IF node)**
- Condition: boolean equals `{{ $json.proposalParseOk }}` equals `true`
- Connect:
  - **True** → **📄 Create PandaDoc Document**
  - **False** → **🚨 Notify: GPT Parse Failed**

13) **Add “📄 Create PandaDoc Document” (HTTP Request node)**
- Node type: **HTTP Request**
- Method: POST
- URL: `https://api.pandadoc.com/public/v1/documents`
- Authentication: **HTTP Header Auth** credential
  - Create a credential that adds the PandaDoc API key header (commonly `Authorization: API-Key <key>`; use PandaDoc’s required header format).
- Headers: `Content-Type: application/json`
- Body: JSON with:
  - `template_uuid` = `{{ $json.pandaDocTemplateId }}`
  - `recipients` from client fields
  - `tokens` mapping to your PandaDoc template token names
  - `pricing_tables` with numeric `price: {{ $json.estimatedValue }}`
- Enable **Continue On Fail** to allow routing to failure notification.
- Connect → **✅ Check: Document Created?**

14) **Add “✅ Check: Document Created?” (IF node)**
- Condition: string `{{ $json.id }}` is not empty
- Connect:
  - **True** → **💬 Notify: Proposal Ready**
  - **False** → **🚨 Notify: PandaDoc Failed**

15) **Add Slack notification nodes (HTTP Request)**
- Create **💬 Notify: Proposal Ready**
  - POST to `{{ $('⚙️ Config').first().json.slackWebhookUrl }}`
  - JSON body as Slack Block Kit
  - Button URL uses PandaDoc doc id: `https://app.pandadoc.com/a/#/documents/` + `{{$json.id}}`
  - Continue On Fail = true
- Create **🚨 Notify: PandaDoc Failed**
  - POST to Slack webhook URL
  - Include error text from `$json.message || $json.error ...`
  - Continue On Fail = true
- Create **🚨 Notify: Missing Required Fields**
  - POST to Slack webhook URL
  - Include which fields are missing
  - Continue On Fail = true
- Create **🚨 Notify: GPT Parse Failed**
  - POST to Slack webhook URL
  - Include `proposalParseError`
  - Continue On Fail = true

16) **Add sticky notes (optional)**
- Add sticky notes for the five sections and error handling to match the documentation blocks.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Proposal Generator… Setup (15–20 minutes)… Customize: Adjust quickQuoteThreshold, edit GPT prompts, add case studies” | From sticky note: **AI Proposal Generator** (in-workflow documentation) |
| Error handling is Slack-webhook based and uses `continueOnFail` on HTTP nodes so the workflow doesn’t hard-stop on notification failures. | Operational characteristic of notification design |
| Typeform extraction depends on exact question label strings (e.g., `"What's your company name?"`, `"Email address"`). Renaming questions requires updating the Code node mappings. | Integration constraint (Typeform → Code node) |
| PandaDoc template tokens must match exactly the token names used in the HTTP request body (e.g., `Client.Company`, `Timeline.Phase1`, etc.). | Integration constraint (PandaDoc templating) |