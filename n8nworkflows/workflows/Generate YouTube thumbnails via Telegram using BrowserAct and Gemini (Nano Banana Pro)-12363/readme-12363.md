Generate YouTube thumbnails via Telegram using BrowserAct and Gemini (Nano Banana Pro)

https://n8nworkflows.xyz/workflows/generate-youtube-thumbnails-via-telegram-using-browseract-and-gemini--nano-banana-pro--12363


# Generate YouTube thumbnails via Telegram using BrowserAct and Gemini (Nano Banana Pro)

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Generate YouTube thumbnails via Telegram using BrowserAct and Gemini (Nano Banana Pro)  
**Purpose:** A Telegram bot that (1) classifies a user’s message as a thumbnail request vs casual chat, (2) scrapes YouTube thumbnail examples via BrowserAct, (3) uses an image-capable LLM to “forensically” analyze those thumbnails and store descriptions in Google Sheets, (4) asks the user for explicit approval (to avoid spending image-generation credits), then (5) generates a final 16:9 YouTube thumbnail using Google Gemini image generation and sends it back in Telegram.

### Logical blocks
1.1 **Telegram entrypoint & routing (message vs callback button)**  
1.2 **Intent analysis & branching (Start / Chat / NoData)**  
1.3 **Session setup in Google Sheets (create/clear a per-keyword sheet + save sheet id)**  
1.4 **YouTube scraping via BrowserAct + data cleanup**  
1.5 **Thumbnail example analysis (LLM vision) + persistence + approval loop**  
1.6 **Callback “Yes” branch: load research from Sheets → craft cinematic prompt → generate image → send to Telegram**  
1.7 **Conversational fallback (Chat/NoData)**

---

## 2. Block-by-Block Analysis

### 1.1 Telegram entrypoint & routing (message vs callback button)
**Overview:** Receives Telegram updates and routes execution depending on whether the update is a normal message or an inline keyboard callback.  
**Nodes involved:** `User Sends Message to Bot`, `Check for Query Callback`

#### Node: User Sends Message to Bot
- **Type / role:** Telegram Trigger (`telegramTrigger`) — entry point.
- **Config (interpreted):**
  - Listens for updates: `message`, `callback_query`.
- **Outputs:** Sends the update payload to `Check for Query Callback`.
- **Edge cases / failures:**
  - Telegram credential invalid → trigger won’t receive updates.
  - If bot privacy settings restrict messages, updates may not arrive.

#### Node: Check for Query Callback
- **Type / role:** Switch — splits flow by existence of `callback_query`.
- **Config:**
  - Route 1 (true): if `{{$json.callback_query}}` exists.
  - Route 2 (false): if it does not exist (regular message).
- **Outputs:**
  - Callback branch → `Check If User Wants to Continue`
  - Message branch → `Validate user input`
- **Edge cases:**
  - Telegram updates can contain non-text messages (photos, stickers). In that case `message.text` may be missing later.

---

### 1.2 Intent analysis & branching (Start / Chat / NoData)
**Overview:** Classifies user text into one of three categories and extracts a short keyword when the user asks for a thumbnail topic.  
**Nodes involved:** `Validate user input`, `Validate input`, `Structured Output Parser`, `Check For Input Type`, `Inform user`

#### Node: Validate user input
- **Type / role:** LangChain Agent — “input classification engine”.
- **Config:**
  - Input text: `={{ $json.message.text }}`
  - System message enforces JSON-only output with schema:
    - `{"Type":"Start","Keyword":"..."}` for thumbnail topic requests
    - `{"Type":"Chat","Keyword":"Null"}`
    - `{"Type":"NoData","Keyword":"Null"}`
  - `hasOutputParser: true` → expects structured parsing.
- **AI connections:**
  - **Language model:** `Validate input` (Gemini chat model)
  - **Output parser:** `Structured Output Parser`
- **Outputs:** Parsed JSON available as `item.json.output` → `Check For Input Type`.
- **Key variables:**
  - `{{$json.output.Type}}`, `{{$json.output.Keyword}}`
- **Edge cases:**
  - If `message.text` is undefined (user sends non-text), model may output wrong classification or fail.
  - Overly long or multilingual text may yield parsing issues (mitigated by parser auto-fix).

#### Node: Validate input
- **Type / role:** Google Gemini Chat Model (`lmChatGoogleGemini`) — LLM provider for classification.
- **Config:** Default options (no model specified in node parameters; uses credential defaults).
- **Outputs:** Feeds `Validate user input` and `Structured Output Parser`.
- **Edge cases:** Model/region quota, API key restrictions, or billing errors.

#### Node: Structured Output Parser
- **Type / role:** Structured output parser — enforces JSON schema.
- **Config:**
  - `autoFix: true`
  - Example schema: `{"Type":"File Type","Keyword":"kling 2.6"}`
- **Outputs:** Provides final parsed `output` object to `Validate user input`.
- **Edge cases:** If the LLM returns text that cannot be repaired into JSON, the node errors.

#### Node: Check For Input Type
- **Type / role:** Switch — routes by `output.Type`.
- **Config:**
  - If `Start` → goes to `Create Database sheet` and `Inform user`
  - If `Chat` → `Chat with User`
  - If `NoData` → `Chat with User`
- **Edge cases:** If `output.Type` missing → no route matched.

#### Node: Inform user
- **Type / role:** Telegram — informs user that generation will proceed.
- **Config:**
  - Text: `Ok, I will generate thumbnail for {{ $json.output.Keyword }}`
  - chatId: `={{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - HTML parse mode enabled.
- **Edge cases:**
  - If message came from callback branch, `message.chat.id` path differs (but this node is only on message branch).
  - Telegram rate limits.

---

### 1.3 Session setup in Google Sheets (create/clear per-keyword sheet + save sheet id)
**Overview:** Creates a new sheet named after the keyword, clears it (keeping header), and writes the active sheet ID + keyword into a central “Database” sheet (row 2).  
**Nodes involved:** `Create Database sheet`, `Wait for Google Sheet Creation`, `Clear Database sheet`, `Save Sheet ID and Keyword`

#### Node: Create Database sheet
- **Type / role:** Google Sheets — creates a new sheet tab.
- **Config:**
  - Operation: **create sheet**
  - Title: `={{ $json.output.Keyword }}`
  - Spreadsheet: “Thumbnail Data base” (documentId `11H5l1...74ic`)
  - **onError:** `continueErrorOutput` (workflow continues even if creation fails)
- **Outputs:** `sheetId` returned (used later).
- **Edge cases:**
  - If a sheet with same name exists, creation may fail; because errors continue, downstream nodes may receive no `sheetId` and fail later.
  - Permissions/scopes insufficient.

#### Node: Wait for Google Sheet Creation
- **Type / role:** Wait — ensures sheet creation/availability before follow-up operations.
- **Config:** Default wait (resume via internal mechanism).
- **Outputs:** Pass-through to `Clear Database sheet` and `Save Sheet ID and Keyword`.
- **Edge cases:** If the wait node is resumed unexpectedly or execution is stopped, the run may remain paused.

#### Node: Clear Database sheet
- **Type / role:** Google Sheets — clears the newly created sheet.
- **Config:**
  - Operation: **clear**
  - Sheet: by id `={{ $json.sheetId }}`
  - keepFirstRow: true
- **Outputs:** Sends to `Run a workflow`.
- **Edge cases:** If `sheetId` missing (sheet creation failed), this node errors.

#### Node: Save Sheet ID and Keyword
- **Type / role:** Google Sheets — updates the central “Database” sheet with the session metadata.
- **Config:**
  - Operation: **update**
  - Sheet: `Database` (gid=0)
  - Matching column: `row_number`
  - Writes at `row_number = 2`:
    - `Keyword = {{ $('Validate user input').item.json.output.Keyword }}`
    - `Current Workflow Sheet ID = {{ $json.sheetId }}`
- **Dependencies:** Requires `sheetId` from `Wait for Google Sheet Creation`.
- **Edge cases:**
  - Hard-coding row 2 means only one active session can be tracked correctly at a time (concurrency issue).
  - If the Database sheet lacks required columns, mapping fails.

---

### 1.4 YouTube scraping via BrowserAct + data cleanup
**Overview:** Invokes a BrowserAct “WORKFLOW” to scrape YouTube data for the keyword, then transforms returned JSON into individual image items and filters out non-image/low-quality URLs.  
**Nodes involved:** `Run a workflow`, `Splitting Image Items`, `Filter Low-Quality Images`

#### Node: Run a workflow
- **Type / role:** BrowserAct — runs a saved BrowserAct workflow.
- **Config:**
  - Type: `WORKFLOW`
  - Timeout: 7200s
  - BrowserAct workflowId: `68750895894137580`
  - Input mapping: `input-Keyword = {{ $('Validate user input').item.json.output.Keyword }}`
- **Outputs:** Expected to contain a JSON string at `output.string` representing an array of items.
- **Edge cases:**
  - BrowserAct credential invalid / workflowId wrong.
  - Scraper changes (YouTube layout) → output schema changes.
  - Timeout on long scraping runs.

#### Node: Splitting Image Items
- **Type / role:** Code node — parses and splits JSON-string array into multiple n8n items.
- **Config logic:**
  - Reads: `$input.first().json.output.string`
  - `JSON.parse()` → must be an array
  - Returns `[{json: item1}, {json: item2}, ...]`
- **Failure modes:**
  - Throws explicit error if string missing/empty.
  - Throws if JSON malformed or not an array.

#### Node: Filter Low-Quality Images
- **Type / role:** Code node — flattens nested structures and filters URLs by extension.
- **Config logic:**
  - Flattens: `$input.all().flatMap(item => item.json.item || [])`
  - Keeps only `.jpg/.jpeg/.png/.webp/.gif`
  - Cleans URL by removing query string (`split('?')[0]`)
  - Outputs items with `json.Cover` as cleaned image URL.
- **Edge cases:**
  - If BrowserAct output shape differs (no `item` array, no `Cover` field) → empty results.
  - Some valid images may be served without extensions → filtered out.

---

### 1.5 Thumbnail example analysis (LLM vision) + persistence + approval loop
**Overview:** Iterates through filtered thumbnails, uses a vision-capable LLM to describe each example, appends descriptions to the newly created keyword sheet, and then asks the user for approval to continue.  
**Nodes involved:** `Loop Over Items`, `Analyze Image Content`, `OpenRouter Chat Model`, `Save Image Descriptions to Database`, `Await Continuation Approval`

#### Node: Loop Over Items
- **Type / role:** Split In Batches — batch/loop mechanism.
- **Config:** Default batch options (not explicitly set).
- **Connections:**
  - Output 1 (loop item) → `Analyze Image Content`
  - Output 2 (done) → `Await Continuation Approval`
- **Edge cases:**
  - If no items, it may go directly to “done” path and still ask approval with no examples stored.

#### Node: Analyze Image Content
- **Type / role:** LangChain Agent — vision analysis of each thumbnail URL.
- **Config:**
  - Sends a multimodal message (text + image_url) with `detail: high`
  - Image URL: `{{ $json.Cover }}`
  - System message forces **raw descriptive text only** (no headers/formatting).
- **AI connection:** Uses `OpenRouter Chat Model` as language model.
- **Outputs:** Produces a text description in node output (commonly in `json.output` depending on agent behavior).
- **Edge cases:**
  - OpenRouter model must support image input; if not, analysis fails.
  - Hotlink restrictions: some image URLs may block fetching by the model provider.

#### Node: OpenRouter Chat Model
- **Type / role:** OpenRouter chat model provider.
- **Config:**
  - Model: `openai/gpt-4o`
- **Usage:** Connected as the language model for `Analyze Image Content`.
- **Edge cases:** OpenRouter API limits, model availability, image input billing.

#### Node: Save Image Descriptions to Database
- **Type / role:** Google Sheets — appends each analysis row to the keyword sheet.
- **Config:**
  - Operation: **append**
  - Sheet: `={{ $('Create Database sheet').first().json.sheetId }}`
  - Mapping: auto-map all input fields.
- **Connections:** After append → back into `Loop Over Items` to continue.
- **Edge cases:**
  - If `Create Database sheet` failed or returned no `sheetId`, append fails.
  - Auto-mapping may create inconsistent columns if input fields vary.

#### Node: Await Continuation Approval
- **Type / role:** Telegram — sends inline keyboard and waits (executeOnce) for user click.
- **Config:**
  - Text: “Okay, I grabbed a few examples. Do you want to Continue? (Cost Credits)”
  - Inline keyboard callback_data:
    - yes → `"yes"`
    - no → `"no"`
  - chatId: `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
  - `executeOnce: true` (helps avoid duplicate approvals within a single run)
- **Edge cases:**
  - If user clicked button from a different chat or the original message object differs, chatId resolution may fail.
  - “No” path is not implemented later (see block 1.6).

---

### 1.6 Callback “Yes” branch: load research → craft prompt → generate image → send to Telegram
**Overview:** When the user clicks “Yes”, the workflow loads the saved sheet ID/keyword from the Database sheet, reads stored thumbnail analyses, aggregates them, uses Gemini to generate a single “Apex Architect” cinematic prompt, generates the final thumbnail image, and sends it to Telegram.  
**Nodes involved:** `Check If User Wants to Continue`, `Get Keywords`, `Get Images Details`, `Aggregate Google Sheets data`, `Google Gemini1`, `Structured Output`, `Generate Image Prompt`, `Generate an image`, `Send Thumbnail Back to Bot`, plus shared `Create Database sheet` reference.

#### Node: Check If User Wants to Continue
- **Type / role:** IF — checks callback_data.
- **Config:** condition `={{ $json.callback_query.data }} == "yes"`
- **Outputs:**
  - True → `Get Keywords`
  - False → nothing connected (no explicit “no” handling)
- **Edge cases:**
  - Users clicking “no” will not receive a follow-up (silent stop).

#### Node: Get Keywords
- **Type / role:** Google Sheets — reads the central Database sheet.
- **Config:**
  - Spreadsheet: “Thumbnail Data base”
  - Sheet: “Database” (gid=0)
  - Range is configured via “specifyRange” (exact range not visible here), intended to fetch `Keyword` and `Current Workflow Sheet ID`.
- **Outputs:** Rows that include `Current Workflow Sheet ID` used by next node.
- **Edge cases:** If range excludes row 2 or columns, next step fails.

#### Node: Get Images Details
- **Type / role:** Google Sheets — reads the keyword sheet using stored sheet ID.
- **Config:**
  - sheetName (by id): `={{ $json["Current Workflow Sheet ID"] }}`
  - Spreadsheet: same documentId.
- **Outputs:** Rows of previously stored image analyses.
- **Edge cases:** If sheet ID is wrong/empty, read fails.

#### Node: Aggregate Google Sheets data
- **Type / role:** Aggregate — concatenates all item JSON into one field for prompting.
- **Config:**
  - Mode: aggregate all item data
  - Destination field: `AggreagtedData` (note misspelling is intentional and used later)
- **Outputs:** Single item with `json.AggreagtedData`.
- **Edge cases:** Very large aggregated payload can exceed LLM context limits.

#### Node: Google Gemini1
- **Type / role:** Gemini chat model provider for prompt-generation agent.
- **Config:** model `models/gemini-2.5-pro`
- **Connections:** Feeds `Generate Image Prompt` and `Structured Output` as their language model.

#### Node: Structured Output
- **Type / role:** Structured output parser for prompt generation.
- **Config:** Example schema `{"Prompt":"Prompt"}`, autoFix enabled.
- **Connections:** `Structured Output` → `Generate Image Prompt` (as ai_outputParser).

#### Node: Generate Image Prompt
- **Type / role:** LangChain Agent — generates the final cinematic thumbnail prompt.
- **Config:**
  - Input text combines:
    - `Thumbnai Examples Descroptions :{{ $json.AggreagtedData }}`
    - `Keyword = {{ $('Get Keywords').item.json.Keyword }}`
  - System message defines the “Nano Banana Pro Apex Architect” rules, and mandates final phrase:
    - `image in 16:9 aspect ratio, wide YouTube thumbnail format.`
  - `hasOutputParser: true` → expected output contains `output.Prompt`.
- **Outputs:** `json.output.Prompt` used by image generation.
- **Edge cases:** If aggregated descriptions are noisy, the prompt may become too long or off-style.

#### Node: Generate an image
- **Type / role:** Google Gemini Image generation node.
- **Config:**
  - Resource: image
  - Model: `models/gemini-3-pro-image-preview (Nano Banana Pro)`
  - Prompt: `={{ $json.output.Prompt }}`
  - Binary output property: `data`
- **Outputs:** Binary image in `binary.data`.
- **Edge cases:** Gemini image model access may require special enablement; credit/billing failures.

#### Node: Send Thumbnail Back to Bot
- **Type / role:** Telegram sendPhoto — sends generated binary image.
- **Config:**
  - operation: sendPhoto
  - binaryData: true (uses incoming binary)
  - chatId: `={{ $('User Sends Message to Bot').first().json.callback_query.message.chat.id}}`
- **Edge cases:**
  - If the execution that generates the image is not in the same run context as the callback update, referencing `User Sends Message to Bot` may not resolve as expected.
  - Telegram file size limits.

---

### 1.7 Conversational fallback (Chat/NoData)
**Overview:** If the user’s message is casual chat or insufficient, the bot responds conversationally, asking for a clearer video niche/topic.  
**Nodes involved:** `Chat with User`, `Chat`, `Answer the User`

#### Node: Chat with User
- **Type / role:** LangChain Agent — produces a user-facing message (HTML allowed).
- **Config:**
  - Input includes type and original user text:
    - `Input type : {{ $json.output.Type }} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
  - System message forces raw text only and limits formatting to Telegram HTML tags.
- **AI connection:** Uses `Chat` (Gemini chat model).
- **Outputs:** `json.output` (message text) → `Answer the User`.

#### Node: Chat
- **Type / role:** Gemini chat model provider.
- **Config:** Default options (no explicit model in parameters; uses credential defaults).

#### Node: Answer the User
- **Type / role:** Telegram send message.
- **Config:**
  - Text: `={{ $json.output }}`
  - chatId: `={{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - parse_mode HTML enabled.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Entry point for Telegram messages/callbacks | — | Check for Query Callback | ## ⚡ Workflow Overview & Setup Summary… (BrowserAct/Gemini/Sheets/Telegram requirements) |
| Check for Query Callback | switch | Route callback_query vs message | User Sends Message to Bot | Check If User Wants to Continue; Validate user input | ## ⚡ Workflow Overview & Setup Summary… |
| Validate user input | langchain agent | Classify intent + extract keyword | Check for Query Callback (message path) | Check For Input Type | ### 🧠 Step 1: Intent Analysis … |
| Validate input | Gemini chat model | LLM for intent classification | — (AI provider) | Validate user input; Structured Output Parser | ### 🧠 Step 1: Intent Analysis … |
| Structured Output Parser | structured output parser | Enforce `{Type, Keyword}` JSON | — (AI parser) | Validate user input | ### 🧠 Step 1: Intent Analysis … |
| Check For Input Type | switch | Branch Start vs Chat/NoData | Validate user input | Create Database sheet + Inform user; Chat with User | ### 🧠 Step 1: Intent Analysis … |
| Inform user | telegram | Confirm keyword to user | Check For Input Type (Start) | — | ### 🧠 Step 1: Intent Analysis … |
| Create Database sheet | googleSheets | Create per-keyword sheet tab | Check For Input Type (Start) | Wait for Google Sheet Creation; Clear Database sheet | ### 🧠 Step 1: Intent Analysis … |
| Wait for Google Sheet Creation | wait | Pause to ensure sheet exists | Create Database sheet | Clear Database sheet; Save Sheet ID and Keyword | ### 🧠 Step 1: Intent Analysis … |
| Save Sheet ID and Keyword | googleSheets | Store keyword + sheetId in Database row 2 | Wait for Google Sheet Creation | — | ### 📊 Google Sheets Requirements Spreadsheet Name: Thumbnail Data base / Sheet Name: Database / Required Columns: Keyword, Current Workflow Sheet ID |
| Clear Database sheet | googleSheets | Clear per-keyword sheet (keep header) | Create Database sheet; Wait for Google Sheet Creation | Run a workflow | ### 🕵️ Step 2: Visual Research … |
| Run a workflow | browserAct | Scrape YouTube thumbnails via BrowserAct workflow | Clear Database sheet | Splitting Image Items | ### 🕵️ Step 2: Visual Research … |
| Splitting Image Items | code | Parse `output.string` JSON array into items | Run a workflow | Filter Low-Quality Images | ### 🕵️ Step 2: Visual Research … |
| Filter Low-Quality Images | code | Clean URLs + keep valid image extensions | Splitting Image Items | Loop Over Items | ### 🕵️ Step 2: Visual Research … |
| Loop Over Items | splitInBatches | Iterate thumbnails; then approval | Filter Low-Quality Images; Save Image Descriptions to Database | Analyze Image Content; Await Continuation Approval | ### 🕵️ Step 2: Visual Research … |
| Analyze Image Content | langchain agent | Vision analysis of each thumbnail URL | Loop Over Items | Save Image Descriptions to Database | ### 🕵️ Step 2: Visual Research … |
| OpenRouter Chat Model | OpenRouter chat model | Vision-capable LLM provider (GPT-4o) | — (AI provider) | Analyze Image Content | ### 🕵️ Step 2: Visual Research … |
| Save Image Descriptions to Database | googleSheets | Append analysis rows to per-keyword sheet | Analyze Image Content | Loop Over Items | ### 🕵️ Step 2: Visual Research … |
| Await Continuation Approval | telegram | Ask user to approve spending credits (inline buttons) | Loop Over Items (done path) | — (user responds via trigger) | ### ⏸️ Step 3: Human-in-the-Loop … |
| Check If User Wants to Continue | if | Continue only if callback_data == "yes" | Check for Query Callback (callback path) | Get Keywords | ### 🎨 Step 4: Callback & Generation … |
| Get Keywords | googleSheets | Read Database sheet to fetch current sheet id/keyword | Check If User Wants to Continue | Get Images Details | ### 🎨 Step 4: Callback & Generation … |
| Get Images Details | googleSheets | Read per-keyword sheet by stored sheetId | Get Keywords | Aggregate Google Sheets data | ### 🎨 Step 4: Callback & Generation … |
| Aggregate Google Sheets data | aggregate | Combine analysis rows into `AggreagtedData` | Get Images Details | Generate Image Prompt | ### 🎨 Step 4: Callback & Generation … |
| Google Gemini1 | Gemini chat model | LLM provider for prompt crafting | — (AI provider) | Generate Image Prompt; Structured Output | ### 🎨 Step 4: Callback & Generation … |
| Structured Output | structured output parser | Enforce `{Prompt}` output | — (AI parser) | Generate Image Prompt | ### 🎨 Step 4: Callback & Generation … |
| Generate Image Prompt | langchain agent | Create cinematic prompt from research + keyword | Aggregate Google Sheets data | Generate an image | ### 🎨 Step 4: Callback & Generation … |
| Generate an image | Gemini image | Generate final thumbnail (binary) | Generate Image Prompt | Send Thumbnail Back to Bot | ### 🎨 Step 4: Callback & Generation … |
| Send Thumbnail Back to Bot | telegram | Send generated image to Telegram chat | Generate an image | — | ### 🎨 Step 4: Callback & Generation … |
| Chat with User | langchain agent | Fallback conversation for Chat/NoData | Check For Input Type | Answer the User | ### 💬 Step 2-2: Conversational Fallback … |
| Chat | Gemini chat model | LLM provider for conversation | — (AI provider) | Chat with User | ### 💬 Step 2-2: Conversational Fallback … |
| Answer the User | telegram | Send fallback response | Chat with User | — | ### 💬 Step 2-2: Conversational Fallback … |
| Sticky Note | stickyNote | Comment node (Sheets requirements) | — | — | (Contains Google Sheets Requirements + required columns) |
| Documentation | stickyNote | Comment node (overview/setup/links) | — | — | (Contains setup notes + BrowserAct links) |
| Step 1 Explanation | stickyNote | Comment node (intent analysis) | — | — | (Intent analysis explanation) |
| Step 2 Explanation | stickyNote | Comment node (visual research) | — | — | (Visual research explanation) |
| Step 3 Explanation | stickyNote | Comment node (approval loop) | — | — | (Human-in-the-loop explanation) |
| Step 4 Explanation | stickyNote | Comment node (callback & generation) | — | — | (Callback & generation explanation) |
| Step 4 Explanation1 | stickyNote | Comment node (fallback chat) | — | — | (Conversational fallback explanation) |
| Sticky Note1 | stickyNote | Comment node (video reference) | — | — | `@[youtube](m0N91nN4ElA)` |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials (n8n → Credentials)**
   1. Telegram API credential (connect your bot token).
   2. Google Sheets OAuth2 credential (with access to the target spreadsheet).
   3. Google Gemini (PaLM) API credential (enabled for Gemini chat + image models).
   4. OpenRouter API credential (for GPT‑4o vision analysis).
   5. BrowserAct API credential.

2) **Create the Google Spreadsheet**
   1. Create (or reuse) a spreadsheet named **“Thumbnail Data base”**.
   2. Ensure it has a sheet tab named **“Database”** with columns:
      - `Keyword`
      - `Current Workflow Sheet ID`
   3. Decide how you will manage concurrency (this workflow writes to **row 2**).

3) **Telegram trigger**
   1. Add node: **Telegram Trigger** → name it `User Sends Message to Bot`.
   2. Updates: enable `message` and `callback_query`.
   3. Select Telegram credentials.

4) **Route callback vs message**
   1. Add node: **Switch** → `Check for Query Callback`.
   2. Rule 1: “exists” on expression `{{$json.callback_query}}`.
   3. Rule 2: “not exists” on expression `{{$json.callback_query}}`.
   4. Connect: Trigger → Switch.

5) **Intent classification (message path)**
   1. Add node: **Google Gemini Chat Model** → `Validate input` (use your Gemini credential).
   2. Add node: **Structured Output Parser** → `Structured Output Parser`
      - Example schema: `{"Type":"Start","Keyword":"kling 2.6"}`
      - Enable auto-fix.
   3. Add node: **LangChain Agent** → `Validate user input`
      - Text: `={{ $json.message.text }}`
      - System message: paste the classification rules (Start/Chat/NoData) and “JSON only”.
      - Enable “Has output parser”.
      - Connect AI model: `Validate input` → `Validate user input`
      - Connect parser: `Structured Output Parser` → `Validate user input`
   4. Connect: Switch (message output) → `Validate user input`.

6) **Branch by input type**
   1. Add node: **Switch** → `Check For Input Type`
      - Case `Start` when `={{ $json.output.Type }}` equals `Start`
      - Case `Chat` equals `Chat`
      - Case `NoData` equals `NoData`
   2. Connect: `Validate user input` → `Check For Input Type`.

7) **Start branch: inform user**
   1. Add node: **Telegram** → `Inform user`
      - Text: `Ok, I will generate thumbnail for {{ $json.output.Keyword }}`
      - chatId: `={{ $('User Sends Message to Bot').item.json.message.chat.id }}`
      - parse_mode: HTML
   2. Connect: `Check For Input Type` (Start output) → `Inform user`.

8) **Start branch: create per-keyword sheet + wait + clear + save metadata**
   1. Add node: **Google Sheets** → `Create Database sheet`
      - Operation: create sheet
      - Title: `={{ $json.output.Keyword }}`
      - Spreadsheet: your “Thumbnail Data base”
      - Set **On Error** to “Continue” if you want the same behavior.
   2. Add node: **Wait** → `Wait for Google Sheet Creation`
      - Default settings.
   3. Add node: **Google Sheets** → `Clear Database sheet`
      - Operation: clear
      - Sheet: by ID `={{ $json.sheetId }}`
      - keepFirstRow: true
   4. Add node: **Google Sheets** → `Save Sheet ID and Keyword`
      - Operation: update
      - Sheet: `Database`
      - Match by: `row_number`
      - Set `row_number = 2`
      - Set `Keyword = {{ $('Validate user input').item.json.output.Keyword }}`
      - Set `Current Workflow Sheet ID = {{ $json.sheetId }}`
   5. Connect:
      - Start output → `Create Database sheet`
      - `Create Database sheet` → `Wait for Google Sheet Creation`
      - `Wait for Google Sheet Creation` → `Clear Database sheet`
      - `Wait for Google Sheet Creation` → `Save Sheet ID and Keyword`

9) **Scrape YouTube examples (BrowserAct)**
   1. Add node: **BrowserAct** → `Run a workflow`
      - Type: WORKFLOW
      - workflowId: `68750895894137580`
      - Timeout: 7200
      - Map `input-Keyword` to `={{ $('Validate user input').item.json.output.Keyword }}`
   2. Connect: `Clear Database sheet` → `Run a workflow`.

10) **Transform BrowserAct output → items; filter URLs**
   1. Add node: **Code** → `Splitting Image Items`
      - Parse `$input.first().json.output.string` as JSON array; return items.
   2. Add node: **Code** → `Filter Low-Quality Images`
      - Flatten `item.json.item`
      - Clean `Cover` query params
      - Filter by extensions
   3. Connect: `Run a workflow` → `Splitting Image Items` → `Filter Low-Quality Images`.

11) **Loop analysis + save to sheet**
   1. Add node: **Split In Batches** → `Loop Over Items`
   2. Add node: **OpenRouter Chat Model** → `OpenRouter Chat Model`
      - Model: `openai/gpt-4o`
   3. Add node: **LangChain Agent** → `Analyze Image Content`
      - Multimodal message referencing `{{ $json.Cover }}`
      - System message: “raw descriptive text only”.
      - Attach AI model: `OpenRouter Chat Model`.
   4. Add node: **Google Sheets** → `Save Image Descriptions to Database`
      - Operation: append
      - Sheet by ID: `={{ $('Create Database sheet').first().json.sheetId }}`
      - Auto-map input.
   5. Connect:
      - `Filter Low-Quality Images` → `Loop Over Items`
      - `Loop Over Items` (item output) → `Analyze Image Content` → `Save Image Descriptions to Database` → back to `Loop Over Items`
      - `Loop Over Items` (done output) → next step (approval)

12) **Ask for user approval**
   1. Add node: **Telegram** → `Await Continuation Approval`
      - Send inline keyboard with callback_data `yes` and `no`
      - chatId: `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
      - executeOnce: true
   2. Connect: `Loop Over Items` (done output) → `Await Continuation Approval`.

13) **Callback branch: proceed only if “yes”**
   1. From `Check for Query Callback` (callback output), add **IF** node → `Check If User Wants to Continue`
      - Condition: `={{ $json.callback_query.data }}` equals `yes`
   2. Connect: `Check for Query Callback` (callback path) → IF.

14) **Load stored sheetId + read research**
   1. Add **Google Sheets** → `Get Keywords`
      - Read from `Database` sheet (ensure range includes row 2 and needed columns).
   2. Add **Google Sheets** → `Get Images Details`
      - Sheet by ID: `={{ $json["Current Workflow Sheet ID"] }}`
   3. Add **Aggregate** → `Aggregate Google Sheets data`
      - Aggregate all items → destination field `AggreagtedData`
   4. Connect: IF (true) → `Get Keywords` → `Get Images Details` → `Aggregate Google Sheets data`.

15) **Generate final image prompt (Gemini 2.5 Pro + structured output)**
   1. Add **Gemini Chat Model** → `Google Gemini1` with model `models/gemini-2.5-pro`.
   2. Add **Structured Output Parser** → `Structured Output` with schema `{"Prompt":"Prompt"}`.
   3. Add **LangChain Agent** → `Generate Image Prompt`
      - Text references `{{$json.AggreagtedData}}` and `{{ $('Get Keywords').item.json.Keyword }}`
      - System message: “Apex Architect” rules and **mandatory 16:9 ending**.
      - Attach AI model: `Google Gemini1`
      - Attach output parser: `Structured Output`
   4. Connect: `Aggregate Google Sheets data` → `Generate Image Prompt`.

16) **Generate thumbnail image + send to Telegram**
   1. Add **Google Gemini (Image)** → `Generate an image`
      - Model: `models/gemini-3-pro-image-preview`
      - Prompt: `={{ $json.output.Prompt }}`
      - Binary property output: `data`
   2. Add **Telegram** → `Send Thumbnail Back to Bot`
      - Operation: sendPhoto
      - Binary data: true
      - chatId: `={{ $('User Sends Message to Bot').first().json.callback_query.message.chat.id}}`
   3. Connect: `Generate Image Prompt` → `Generate an image` → `Send Thumbnail Back to Bot`.

17) **Fallback chat (Chat/NoData)**
   1. Add **Gemini Chat Model** → `Chat` (use your Gemini credential).
   2. Add **LangChain Agent** → `Chat with User` with HTML-only rules.
   3. Add **Telegram** → `Answer the User`
      - text `={{ $json.output }}`
      - chatId `={{ $('User Sends Message to Bot').item.json.message.chat.id }}`
      - parse_mode HTML
   4. Connect: `Check For Input Type` (Chat and NoData outputs) → `Chat with User` → `Answer the User`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow overview, requirements, and usage notes (Telegram + BrowserAct + OpenRouter + Gemini + Sheets). | Included in the workflow “Documentation” sticky note |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | `@[youtube](m0N91nN4ElA)` |
| Google Sheets requirements: Spreadsheet “Thumbnail Data base”, sheet “Database”, required columns `Keyword`, `Current Workflow Sheet ID`. | Included in “Google Sheets Requirements” sticky note |

