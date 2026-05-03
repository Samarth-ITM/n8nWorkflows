Compare products and generate visual scorecards in Telegram with BrowserAct and Gemini

https://n8nworkflows.xyz/workflows/compare-products-and-generate-visual-scorecards-in-telegram-with-browseract-and-gemini-12357


# Compare products and generate visual scorecards in Telegram with BrowserAct and Gemini

## 1. Workflow Overview

**Purpose:**  
This workflow listens to Telegram messages, detects when a user asks to compare products, automatically gathers product data from a best-fit review/commerce site via BrowserAct, runs a deep AI comparison, then generates a **visual “winner scorecard”** image using **Google Gemini image generation (Nano Banana Pro / gemini-3-pro-image-preview)** and sends it back to Telegram. If the message is not a comparison request, it falls back to a conversational response.

**Target use cases:**
- “Compare iPhone 15 vs Galaxy S24”
- “Which is better: Notion or ClickUp?”
- “Compare NordVPN vs ExpressVPN”

### 1.1 Input Reception (Telegram)
Receives inbound Telegram messages and forwards the raw text to the classification agent.

### 1.2 Intent Classification + Routing
An AI agent classifies the message as either:
- **Comparison** (extract products + decide best source site), or
- **Chat** (fallback conversation)

### 1.3 Comparison Pipeline: Initialize + Prepare Storage
If comparison:
- Sends a “please wait” status message to the user
- Clears a Google Sheet used as a temporary database for scraped product data

### 1.4 Product Data Collection (BrowserAct) + Storage
Splits extracted product list into items, loops through them:
- Calls BrowserAct workflow to scrape product details from the chosen target site
- Appends each scrape result into Google Sheets (“Product Data” column)

### 1.5 Data Aggregation + Deep AI Analysis
Loads all rows from the sheet, aggregates them into one payload, then uses Gemini to produce:
- summarized product_1/product_2 blocks
- a final verdict comparison (materials/composition, performance, economics, experience)

### 1.6 Prompt Engineering + Image Generation + Telegram Delivery
Uses another AI agent to:
- generate a Telegram caption (HTML-safe)
- generate a detailed image prompt for Nano Banana Pro
Then Gemini image generation creates the infographic and Telegram sends it as a photo with caption.

### 1.7 Chat Fallback
If classified as Chat, responds with plain text back to Telegram.

---

## 2. Block-by-Block Analysis

### Block A — Telegram Reception
**Overview:** Receives Telegram messages and starts the workflow with the user’s text and chat metadata.

**Nodes involved:**
- **User Sends Message to Bot** (Telegram Trigger)

#### Node: User Sends Message to Bot
- **Type / Role:** `telegramTrigger` — entry point (webhook-based) for Telegram updates.
- **Configuration choices:**
  - Listens to `message` updates only.
- **Key data used downstream:**
  - `{{$json.message.text}}` (user text)
  - `{{$json.message.chat.id}}` (chat ID for replies)
- **Outputs / Connections:**
  - → **Validate user Input**
- **Failure / Edge cases:**
  - Bot not authorized or webhook misconfigured (Telegram credential error).
  - Non-text messages (stickers/photos) may have no `message.text`, causing expressions to evaluate to empty/undefined later.

---

### Block B — Intent Classification + Structured Output Parsing
**Overview:** Classifies the user message into Comparison vs Chat, extracts product names, and selects the best target site.

**Nodes involved:**
- **Validate user Input** (LangChain Agent)
- **Google Gemini** (Chat LLM)
- **Structured Output Parser** (Structured parser)
- **Validation Type Switch** (Switch)

#### Node: Google Gemini
- **Type / Role:** `lmChatGoogleGemini` — provides the LLM for the classification agent.
- **Configuration choices:**
  - Model: `models/gemini-2.5-pro`
- **Connections:**
  - Provides `ai_languageModel` to **Validate user Input** and **Structured Output Parser**
- **Failure / Edge cases:**
  - Invalid/expired Google PaLM/Gemini API credential.
  - Model access not enabled in the Google project.
  - Rate limits.

#### Node: Structured Output Parser
- **Type / Role:** `outputParserStructured` — enforces JSON output schema for the classifier.
- **Configuration choices:**
  - `autoFix: true` (attempts to repair near-valid JSON)
  - JSON schema example: `{ Type, Products[], BestSite }`
- **Connections:**
  - Used as `ai_outputParser` by **Validate user Input**
- **Failure / Edge cases:**
  - If the model outputs non-JSON or severely malformed content, auto-fix may fail.
  - Schema mismatch (missing keys) can cause downstream switch to misroute.

#### Node: Validate user Input
- **Type / Role:** `langchain.agent` — strict input classification engine.
- **Configuration choices (interpreted):**
  - Input text: `={{ $json.message.text }}`
  - System message enforces:
    - Output **only raw JSON**
    - Two categories: `Comparison` (extract Products + choose BestSite) or `Chat` (null-ish fields)
    - BestSite logic:
      - Physical product → Amazon
      - Service/website → Trustpilot
      - SaaS → G2
      - Video game → Steam
  - `hasOutputParser: true` (uses **Structured Output Parser**)
- **Outputs / Connections:**
  - Main output includes `output` object (parsed JSON)
  - → **Validation Type Switch**
- **Failure / Edge cases:**
  - Missing `message.text` leads to poor classification.
  - User provides >2 products; agent may still output more items; later steps assume looping is okay but analysis prompt expects two.
  - Ambiguous product category may select a suboptimal `BestSite`.

#### Node: Validation Type Switch
- **Type / Role:** `switch` — routes based on `{{$json.output.Type}}`.
- **Configuration choices:**
  - Rule 1: equals `"Comparison"` → comparison branch
  - Rule 2: equals `"Chat"` → chat branch
- **Outputs / Connections:**
  - Comparison path → **Process Initialization Alert** and **Clear Database**
  - Chat path → **Conversational AI**
- **Failure / Edge cases:**
  - If `output.Type` is missing/invalid, nothing matches and workflow may silently stop (no default route configured).

---

### Block C — Comparison Initialization + Database Reset
**Overview:** Acknowledges the user and resets the Google Sheet used to store scraped product data.

**Nodes involved:**
- **Process Initialization Alert** (Telegram)
- **Clear Database** (Google Sheets)

#### Node: Process Initialization Alert
- **Type / Role:** `telegram` — sends a status message.
- **Configuration choices:**
  - Text: “Okay, I will do it. Please give me a moment.”
  - `chatId`: `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
  - parse mode: HTML
  - `executeOnce: true` (sends only once even if multiple items pass)
- **Connections:** No downstream connections (notification only).
- **Failure / Edge cases:**
  - Invalid chat ID expression if trigger payload structure differs.
  - Telegram HTML parse errors are rare here (text is plain).

#### Node: Clear Database
- **Type / Role:** `googleSheets` — clears previous run’s stored data.
- **Configuration choices:**
  - Operation: Clear
  - Document: `Product Comparision` (Spreadsheet ID present)
  - Sheet: `Sheet1` (gid=0)
  - `keepFirstRow: true` (preserves header row)
- **Outputs / Connections:**
  - → **Split Out Extracted Products**
- **Failure / Edge cases:**
  - Google OAuth token expired / insufficient permissions.
  - Spreadsheet renamed/deleted, wrong sheet, or protected range.

---

### Block D — Product List Split + Looping
**Overview:** Splits the extracted product array into individual items, then iterates: scrape → save → next item, and finally proceeds to analysis when looping completes.

**Nodes involved:**
- **Split Out Extracted Products** (Split Out)
- **Loop Over Items** (Split In Batches)

#### Node: Split Out Extracted Products
- **Type / Role:** `splitOut` — converts `output.Products` array into multiple items.
- **Configuration choices:**
  - Field to split: `output.Products`
  - Includes other field: `output.BestSite` (so each item retains chosen target site)
- **Outputs / Connections:**
  - → **Loop Over Items**
- **Failure / Edge cases:**
  - If `output.Products` is empty/not an array, produces zero items → nothing to loop/analyze.

#### Node: Loop Over Items
- **Type / Role:** `splitInBatches` — controls iterative execution.
- **Configuration choices:**
  - Uses default batch options (batch size default in node UI; not explicitly set here).
- **Connections:**
  - Main output 1 → **Search for each product's data.** (per-item scraping)
  - Main output 0 (when done) → **Retrieve Data from Database** (post-loop continuation)
  - Also receives back edge from **Save Product Information to Database** to continue loop.
- **Failure / Edge cases:**
  - If BrowserAct or Sheets append fails mid-loop, later analysis may run with partial data.

---

### Block E — Scrape Product Data (BrowserAct) + Store in Google Sheets
**Overview:** For each product, calls a BrowserAct sub-workflow to collect structured product details, then appends the result into a Google Sheet.

**Nodes involved:**
- **Search for each product's data.** (BrowserAct)
- **Save Product Information to Database** (Google Sheets append)

#### Node: Search for each product's data.
- **Type / Role:** `browserAct.browserAct` — runs a BrowserAct workflow (“WORKFLOW” execution type).
- **Configuration choices:**
  - Timeout: 7200 seconds (2 hours)
  - BrowserAct workflowId: `69517518588721138`
  - Workflow inputs mapping:
    - `input-Product` = `={{ $json["output.Products"] }}`
    - `input-Target_Site` = `={{ $json["output.BestSite"] }}`
- **Outputs / Connections:**
  - → **Save Product Information to Database**
- **Sub-workflow reference:**
  - External: BrowserAct workflow template referenced in sticky note as **“Product Comparison & Visualize Bo”** (ensure the workflow ID matches your saved template).
- **Failure / Edge cases:**
  - BrowserAct API key invalid.
  - Workflow ID not found / not accessible.
  - Target site blocks scraping, CAPTCHAs, layout changes.
  - Output shape differences; downstream expects `{{$json.output.string}}` to exist.

#### Node: Save Product Information to Database
- **Type / Role:** `googleSheets` — appends scraped data into a “database” sheet.
- **Configuration choices:**
  - Operation: Append
  - Target spreadsheet: `Product Comparision`
  - Column mapping:
    - `Product Data` = `={{ $json.output.string }}`
  - Matching columns configured but effectively used for mapping (not for update).
- **Outputs / Connections:**
  - → **Loop Over Items** (to fetch next product)
- **Failure / Edge cases:**
  - If BrowserAct returns large text, Sheet cell size limits may truncate.
  - If `output.string` is undefined, appends blank row; analysis later degrades.

---

### Block F — Retrieve + Aggregate Sheet Data
**Overview:** Once all products are scraped, reads the sheet and aggregates all item rows into a single combined dataset.

**Nodes involved:**
- **Retrieve Data from Database** (Google Sheets read)
- **Aggregate Google Sheet Data** (Aggregate)

#### Node: Retrieve Data from Database
- **Type / Role:** `googleSheets` — reads rows from the sheet.
- **Configuration choices:**
  - Data location: specifies a range (configured via “specifyRange” option; exact range not shown, so it likely reads used range).
  - Spreadsheet: `Product Comparision`, sheet `Sheet1`.
- **Outputs / Connections:**
  - → **Aggregate Google Sheet Data**
- **Failure / Edge cases:**
  - Empty sheet (if scraping failed) → aggregate yields empty dataset.
  - Range misconfiguration can omit rows.

#### Node: Aggregate Google Sheet Data
- **Type / Role:** `aggregate` — merges all rows into one item payload.
- **Configuration choices:**
  - Mode: `aggregateAllItemData` (collects all incoming items together)
- **Outputs / Connections:**
  - → **Analyze the data**
- **Failure / Edge cases:**
  - Very large combined payload can exceed LLM context size later.

---

### Block G — Deep Comparison Analysis (Gemini + Structured Output)
**Overview:** Uses Gemini to analyze scraped data and produce a strict JSON result with product summaries and a final verdict.

**Nodes involved:**
- **Analyze the data** (LangChain Agent)
- **Google Gemini 2** (Chat LLM)
- **Structured Output** (Structured parser)

#### Node: Google Gemini 2
- **Type / Role:** `lmChatGoogleGemini` — LLM provider for analysis.
- **Configuration choices:**
  - Uses Gemini via the same Google PaLM/Gemini credential (model not explicitly set here; node default applies).
- **Connections:**
  - Provides `ai_languageModel` to **Analyze the data** and **Structured Output**
- **Failure / Edge cases:**
  - Default model may differ across environments; for deterministic behavior, explicitly set a model.

#### Node: Structured Output
- **Type / Role:** `outputParserStructured` — enforces the analysis JSON schema.
- **Configuration choices:**
  - Manual schema:
    - `product_1`: “Name + Summary (specs, pros, cons)”
    - `product_2`: same
    - `comparison`: deep analysis + verdict
  - `autoFix: true`
- **Connections:**
  - Used as `ai_outputParser` by **Analyze the data**
- **Failure / Edge cases:**
  - If the analysis agent returns content too long or malformed, parsing may fail.

#### Node: Analyze the data
- **Type / Role:** `langchain.agent` — product comparison engine.
- **Configuration choices:**
  - Input text includes:
    - `Data : {{ $json.data }}` (from Aggregate)
    - `User Inputs : {{ $('User Sends Message to Bot').first().json.message.text }}`
    - `compare these products only : {{ $('Validate user Input').first().json.output.Products }}`
  - System message enforces 4-step framework: Composition, Performance, Economics, Experience
  - Output: raw JSON only
  - `maxTries: 2`, `retryOnFail: true`
  - `hasOutputParser: true` (uses **Structured Output**)
- **Outputs / Connections:**
  - → **Generate Image and Description**
- **Failure / Edge cases:**
  - If scraped data is sparse, agent “fills gaps” with internal knowledge (can introduce inaccuracies).
  - If more/less than two products were scraped, prompt still frames “product_1/product_2”, potentially misrepresenting.

---

### Block H — Caption + Image Prompt Generation (Gemini + Structured Output)
**Overview:** Converts analysis JSON into (1) a Telegram HTML caption under 1000 chars and (2) a highly structured image prompt.

**Nodes involved:**
- **Generate Image and Description** (LangChain Agent)
- **Google Gemini 3** (Chat LLM)
- **Structured Output1** (Structured parser)

#### Node: Google Gemini 3
- **Type / Role:** `lmChatGoogleGemini` — LLM provider for prompt engineering.
- **Configuration choices:** default model for this node (not explicitly set).
- **Connections:**
  - Provides `ai_languageModel` to **Generate Image and Description** and **Structured Output1**
- **Failure / Edge cases:**
  - As with Gemini 2, default model variance across instances.

#### Node: Structured Output1
- **Type / Role:** `outputParserStructured` — enforces JSON keys: caption + prompt.
- **Configuration choices:**
  - Manual schema:
    - `telegram_caption`
    - `prompt`
  - `autoFix: true`
- **Connections:**
  - Used as `ai_outputParser` by **Generate Image and Description**

#### Node: Generate Image and Description
- **Type / Role:** `langchain.agent` — transforms analysis into publishable caption + image prompt.
- **Configuration choices:**
  - Input: `={{ $json.output }}` (the analysis JSON from prior step)
  - System message constraints:
    - Output ONLY JSON with keys `telegram_caption` and `prompt`
    - Caption: HTML tags allowed only `<b><i><u><a>`; under 1000 chars
    - Prompt: must specify aspect ratio `--aspect_ratio 16:9` or `9:16`, demand 4K, layout coordinates, comparison table logic, winner seal, etc.
  - `maxTries: 2`, `retryOnFail: true`
  - `hasOutputParser: true` (uses **Structured Output1**)
- **Outputs / Connections:**
  - → **Generate Comparison Image**
- **Failure / Edge cases:**
  - Caption may exceed Telegram limit or include disallowed HTML → Telegram sendPhoto may fail.
  - If the agent outputs wrong key casing, downstream expressions will break.

---

### Block I — Image Generation (Gemini Image) + Telegram Photo Delivery
**Overview:** Generates the comparison infographic image and sends it to the user in Telegram with the generated caption.

**Nodes involved:**
- **Generate Comparison Image** (Google Gemini Image)
- **Send Photo Message to Bot** (Telegram)

#### Node: Generate Comparison Image
- **Type / Role:** `@n8n/n8n-nodes-langchain.googleGemini` (resource: image) — generates an image from the prompt.
- **Configuration choices:**
  - Model: `models/gemini-3-pro-image-preview` (labelled “Nano Banana Pro” in the node UI)
  - Prompt: `={{ $json.output.prompt }}`
  - Output binary property: `data`
- **Outputs / Connections:**
  - → **Send Photo Message to Bot**
- **Failure / Edge cases:**
  - Image model access not enabled.
  - Prompt policy refusals or safety filtering.
  - Large outputs / binary handling issues if n8n instance limits are tight.

#### Node: Send Photo Message to Bot
- **Type / Role:** `telegram` — sends generated image.
- **Configuration choices:**
  - Operation: `sendPhoto`
  - `chatId`: `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
  - `binaryData: true` (expects image binary from previous node)
  - Caption: `={{ $('Generate Image and Description').first().json.output["telegram_caption"] }}`
  - parse mode: HTML
- **Failure / Edge cases:**
  - If `data` binary property is missing (image generation failed), Telegram node fails.
  - Caption HTML errors can reject message.

---

### Block J — Chat Fallback (Conversational Reply)
**Overview:** If input is classified as Chat, uses an LLM to generate a plain text response and sends it back.

**Nodes involved:**
- **Conversational AI** (LangChain Agent)
- **Google Gemini 1** (Chat LLM)
- **Answer the User** (Telegram)

#### Node: Google Gemini 1
- **Type / Role:** `lmChatGoogleGemini` — LLM provider for chat responses.
- **Configuration choices:** default model (not explicitly set).
- **Connections:**
  - Provides `ai_languageModel` to **Conversational AI**

#### Node: Conversational AI
- **Type / Role:** `langchain.agent` — generates a natural language response.
- **Configuration choices:**
  - Input: `Input type : {{ $json.output.Type }} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
  - System message:
    - Only respond if type is chat
    - Output raw text only, avoid tags/code fences
- **Outputs / Connections:**
  - → **Answer the User**
- **Failure / Edge cases:**
  - If classification is wrong (Comparison misclassified as Chat), the user won’t get the intended scorecard flow.

#### Node: Answer the User
- **Type / Role:** `telegram` — sends the fallback message.
- **Configuration choices:**
  - Text: `={{ $json.output }}`
  - `chatId`: `={{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - parse mode: HTML (even though system says “avoid tags”; generally fine)
- **Failure / Edge cases:**
  - If LLM outputs characters interpreted as HTML, Telegram may format unexpectedly (low risk).

---

### Block K — Documentation / Notes (Sticky Notes)
**Overview:** Non-executing notes embedded in the canvas; they describe prerequisites and the conceptual steps.

**Nodes involved (sticky notes):**
- Documentation
- Step 1 Explanation
- Step 2 Explanation
- Step 3 Explanation
- Step 4 Explanation
- Sticky Note (YouTube)
- Sticky Note1 (Sheet requirements)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Entry point: receive Telegram message | — | Validate user Input | ### 🔍 Step 1: Intent Analysis<br><br>The workflow intercepts Telegram messages to identify product comparison requests. An AI agent classifies the input and determines the best source (e.g., Amazon, G2, Trustpilot) to fetch data from based on the product type (Physical, Software, Service). |
| Validate user Input | langchain.agent | Classify request into Comparison vs Chat and extract products/site | User Sends Message to Bot; Google Gemini (LLM); Structured Output Parser (parser) | Validation Type Switch | ### 🔍 Step 1: Intent Analysis<br><br>The workflow intercepts Telegram messages to identify product comparison requests. An AI agent classifies the input and determines the best source (e.g., Amazon, G2, Trustpilot) to fetch data from based on the product type (Physical, Software, Service). |
| Google Gemini | lmChatGoogleGemini | LLM for intent classification | — | Validate user Input; Structured Output Parser | ### 🔍 Step 1: Intent Analysis<br><br>The workflow intercepts Telegram messages to identify product comparison requests. An AI agent classifies the input and determines the best source (e.g., Amazon, G2, Trustpilot) to fetch data from based on the product type (Physical, Software, Service). |
| Structured Output Parser | outputParserStructured | Enforce classifier JSON schema | Google Gemini (LLM) | Validate user Input | ### 🔍 Step 1: Intent Analysis<br><br>The workflow intercepts Telegram messages to identify product comparison requests. An AI agent classifies the input and determines the best source (e.g., Amazon, G2, Trustpilot) to fetch data from based on the product type (Physical, Software, Service). |
| Validation Type Switch | switch | Route Comparison vs Chat | Validate user Input | Process Initialization Alert; Clear Database; Conversational AI | ### 🔍 Step 1: Intent Analysis<br><br>The workflow intercepts Telegram messages to identify product comparison requests. An AI agent classifies the input and determines the best source (e.g., Amazon, G2, Trustpilot) to fetch data from based on the product type (Physical, Software, Service). |
| Process Initialization Alert | telegram | Notify user workflow started | Validation Type Switch | — |  |
| Clear Database | googleSheets | Clear sheet rows (keep header) | Validation Type Switch | Split Out Extracted Products | ### 📊 Step 2: Data Extraction & Analysis<br><br>BrowserAct scrapes detailed specs and reviews for both products. A specialized "Comparison Engine" AI then analyzes this data across four dimensions: Composition, Performance, Economics, and Experience to determine an objective winner. |
| Split Out Extracted Products | splitOut | Convert product array into per-product items | Clear Database | Loop Over Items | ### 📊 Step 2: Data Extraction & Analysis<br><br>BrowserAct scrapes detailed specs and reviews for both products. A specialized "Comparison Engine" AI then analyzes this data across four dimensions: Composition, Performance, Economics, and Experience to determine an objective winner. |
| Loop Over Items | splitInBatches | Iterate scraping per product; then continue to analysis | Split Out Extracted Products; Save Product Information to Database | Search for each product's data.; Retrieve Data from Database | ### 📊 Step 2: Data Extraction & Analysis<br><br>BrowserAct scrapes detailed specs and reviews for both products. A specialized "Comparison Engine" AI then analyzes this data across four dimensions: Composition, Performance, Economics, and Experience to determine an objective winner. |
| Search for each product's data. | browserAct.browserAct | Scrape product info from selected target site | Loop Over Items | Save Product Information to Database | ### 📊 Step 2: Data Extraction & Analysis<br><br>BrowserAct scrapes detailed specs and reviews for both products. A specialized "Comparison Engine" AI then analyzes this data across four dimensions: Composition, Performance, Economics, and Experience to determine an objective winner. |
| Save Product Information to Database | googleSheets | Append scraped product data to sheet | Search for each product's data. | Loop Over Items | ### 📊 Sheet Processing Requirements<br><br>**File Name:** `Product Comparision`<br>**Target Column:** `Product Data` (Cell A1) |
| Retrieve Data from Database | googleSheets | Read collected rows after loop | Loop Over Items | Aggregate Google Sheet Data | ### 📊 Step 2: Data Extraction & Analysis<br><br>BrowserAct scrapes detailed specs and reviews for both products. A specialized "Comparison Engine" AI then analyzes this data across four dimensions: Composition, Performance, Economics, and Experience to determine an objective winner. |
| Aggregate Google Sheet Data | aggregate | Merge all rows into one combined payload | Retrieve Data from Database | Analyze the data | ### 📊 Step 2: Data Extraction & Analysis<br><br>BrowserAct scrapes detailed specs and reviews for both products. A specialized "Comparison Engine" AI then analyzes this data across four dimensions: Composition, Performance, Economics, and Experience to determine an objective winner. |
| Analyze the data | langchain.agent | Deep product comparison + verdict in JSON | Aggregate Google Sheet Data; Google Gemini 2 (LLM); Structured Output (parser) | Generate Image and Description | ### 📊 Step 2: Data Extraction & Analysis<br><br>BrowserAct scrapes detailed specs and reviews for both products. A specialized "Comparison Engine" AI then analyzes this data across four dimensions: Composition, Performance, Economics, and Experience to determine an objective winner. |
| Google Gemini 2 | lmChatGoogleGemini | LLM for deep analysis | — | Analyze the data; Structured Output | ### 📊 Step 2: Data Extraction & Analysis<br><br>BrowserAct scrapes detailed specs and reviews for both products. A specialized "Comparison Engine" AI then analyzes this data across four dimensions: Composition, Performance, Economics, and Experience to determine an objective winner. |
| Structured Output | outputParserStructured | Enforce analysis JSON schema | Google Gemini 2 (LLM) | Analyze the data | ### 📊 Step 2: Data Extraction & Analysis<br><br>BrowserAct scrapes detailed specs and reviews for both products. A specialized "Comparison Engine" AI then analyzes this data across four dimensions: Composition, Performance, Economics, and Experience to determine an objective winner. |
| Generate Image and Description | langchain.agent | Produce Telegram caption + Nano Banana prompt JSON | Analyze the data; Google Gemini 3 (LLM); Structured Output1 (parser) | Generate Comparison Image | ### 🎨 Step 3: Visual Scorecard Generation<br><br>Using the analysis results, a "Generative Information Architect" AI creates a detailed image prompt. This prompt instructs the Nano Banana Pro model to render a high-fidelity, 4K comparison infographic, complete with a "Winner" seal and visual spec breakdown. |
| Google Gemini 3 | lmChatGoogleGemini | LLM for caption + prompt engineering | — | Generate Image and Description; Structured Output1 | ### 🎨 Step 3: Visual Scorecard Generation<br><br>Using the analysis results, a "Generative Information Architect" AI creates a detailed image prompt. This prompt instructs the Nano Banana Pro model to render a high-fidelity, 4K comparison infographic, complete with a "Winner" seal and visual spec breakdown. |
| Structured Output1 | outputParserStructured | Enforce caption+prompt JSON schema | Google Gemini 3 (LLM) | Generate Image and Description | ### 🎨 Step 3: Visual Scorecard Generation<br><br>Using the analysis results, a "Generative Information Architect" AI creates a detailed image prompt. This prompt instructs the Nano Banana Pro model to render a high-fidelity, 4K comparison infographic, complete with a "Winner" seal and visual spec breakdown. |
| Generate Comparison Image | googleGemini (image) | Generate infographic image from prompt | Generate Image and Description | Send Photo Message to Bot | ### 🎨 Step 3: Visual Scorecard Generation<br><br>Using the analysis results, a "Generative Information Architect" AI creates a detailed image prompt. This prompt instructs the Nano Banana Pro model to render a high-fidelity, 4K comparison infographic, complete with a "Winner" seal and visual spec breakdown. |
| Send Photo Message to Bot | telegram | Send generated image + caption to user | Generate Comparison Image | — |  |
| Conversational AI | langchain.agent | Fallback chat response | Validation Type Switch; Google Gemini 1 (LLM) | Answer the User | ### 💬 Step 2-2: Conversational Fallback<br><br>This branch engages the user in natural conversation and chats with them if needed. |
| Google Gemini 1 | lmChatGoogleGemini | LLM for chat fallback | — | Conversational AI | ### 💬 Step 2-2: Conversational Fallback<br><br>This branch engages the user in natural conversation and chats with them if needed. |
| Answer the User | telegram | Send fallback text message | Conversational AI | — | ### 💬 Step 2-2: Conversational Fallback<br><br>This branch engages the user in natural conversation and chats with them if needed. |
| Documentation | stickyNote | Canvas note: requirements + links | — | — | ## ⚡ Workflow Overview & Setup<br><br>**Summary:** This automation takes a user request to compare two products (via Telegram), scrapes their details using BrowserAct, performs a deep AI analysis, and generates a visual "Winner Scorecard" image using the Nano Banana Pro model.<br><br>### Requirements<br>* **Credentials:** Telegram, BrowserAct, Google Gemini (PaLM), Google Sheets.<br>* **Mandatory:** BrowserAct API (Template: **Product Comparison & Visualize Bo**)<br><br>### How to Use<br>1. **Credentials:** Set up your Telegram, BrowserAct, and AI model credentials in n8n.<br>2. **BrowserAct Template:** Ensure you have the **Product Comparison & Visualize Bo** template saved in your BrowserAct account.<br>3. **Operation:** Send a message like "Compare iPhone 15 vs Galaxy S24" to your Telegram bot. The system will research, analyze, and generate a visual comparison.<br><br>### Need Help?<br>https://docs.browseract.com |
| Step 1 Explanation | stickyNote | Canvas note: intent analysis | — | — | ### 🔍 Step 1: Intent Analysis<br><br>The workflow intercepts Telegram messages to identify product comparison requests. An AI agent classifies the input and determines the best source (e.g., Amazon, G2, Trustpilot) to fetch data from based on the product type (Physical, Software, Service). |
| Step 2 Explanation | stickyNote | Canvas note: scraping + analysis | — | — | ### 📊 Step 2: Data Extraction & Analysis<br><br>BrowserAct scrapes detailed specs and reviews for both products. A specialized "Comparison Engine" AI then analyzes this data across four dimensions: Composition, Performance, Economics, and Experience to determine an objective winner. |
| Step 3 Explanation | stickyNote | Canvas note: scorecard generation | — | — | ### 🎨 Step 3: Visual Scorecard Generation<br><br>Using the analysis results, a "Generative Information Architect" AI creates a detailed image prompt. This prompt instructs the Nano Banana Pro model to render a high-fidelity, 4K comparison infographic, complete with a "Winner" seal and visual spec breakdown. |
| Step 4 Explanation | stickyNote | Canvas note: chat fallback | — | — | ### 💬 Step 2-2: Conversational Fallback<br><br>This branch engages the user in natural conversation and chats with them if needed. |
| Sticky Note | stickyNote | Canvas note: video reference | — | — | @[youtube](MCKLEF0m9ps) (https://www.youtube.com/watch?v=MCKLEF0m9ps) |
| Sticky Note1 | stickyNote | Canvas note: sheet requirements | — | — | ### 📊 Sheet Processing Requirements<br><br>**File Name:** `Product Comparision`<br>**Target Column:** `Product Data` (Cell A1) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram Trigger**
   - Node: **Telegram Trigger** (“User Sends Message to Bot”)
   - Updates: `message`
   - Credential: Telegram Bot token (Telegram API credential in n8n)

2) **Create LLM + Structured Parser for classification**
   - Node: **Google Gemini Chat Model** (“Google Gemini”)
     - Model: `models/gemini-2.5-pro`
     - Credential: Google Gemini(PaLM) API
   - Node: **Structured Output Parser** (“Structured Output Parser”)
     - Enable Auto-fix
     - Provide schema example with keys `Type`, `Products` (array), `BestSite`
   - Node: **AI Agent** (“Validate user Input”)
     - Text: `{{$json.message.text}}`
     - System message: implement the strict JSON-only classification rules (Comparison vs Chat + BestSite mapping)
     - Enable “Use Output Parser” and select the structured output parser node
     - Connect the Gemini chat model to this agent as its language model

3) **Add Switch routing**
   - Node: **Switch** (“Validation Type Switch”)
   - Rules:
     - If `{{$json.output.Type}}` equals `Comparison` → output 1
     - If `{{$json.output.Type}}` equals `Chat` → output 2
   - Connect:
     - Telegram Trigger → Validate user Input → Switch

4) **Comparison branch: send “processing” message**
   - Node: **Telegram** (“Process Initialization Alert”)
   - Operation: sendMessage
   - Chat ID: `{{$('User Sends Message to Bot').first().json.message.chat.id}}`
   - Text: “Okay, I will do it. Please give me a moment.”
   - parse_mode: HTML
   - (Optional) set “Execute Once”

5) **Comparison branch: Google Sheets “database” reset**
   - Node: **Google Sheets** (“Clear Database”)
   - Operation: Clear
   - Spreadsheet: create/select **Product Comparision**
   - Sheet: `Sheet1`
   - Keep first row: true
   - Credential: Google Sheets OAuth2

6) **Split products into items**
   - Node: **Split Out** (“Split Out Extracted Products”)
   - Field to split out: `output.Products`
   - Include other fields: include `output.BestSite`
   - Connect: Clear Database → Split Out

7) **Loop over products**
   - Node: **Split In Batches** (“Loop Over Items”)
   - Default options are acceptable (or set batch size = 1)
   - Connect: Split Out → Split In Batches

8) **BrowserAct scrape per product**
   - Node: **BrowserAct** (“Search for each product's data.”)
   - Type: WORKFLOW
   - Workflow ID: your BrowserAct workflow id (example in JSON: `69517518588721138`)
   - Map inputs:
     - `input-Product` = `{{$json["output.Products"]}}`
     - `input-Target_Site` = `{{$json["output.BestSite"]}}`
   - Timeout: 7200 seconds
   - Credential: BrowserAct API key
   - Connect: Split In Batches (loop output) → BrowserAct

9) **Append scrape results to Google Sheets**
   - Node: **Google Sheets** (“Save Product Information to Database”)
   - Operation: Append
   - Spreadsheet: `Product Comparision`
   - Sheet: `Sheet1`
   - Ensure header row contains column **Product Data** (cell A1)
   - Map:
     - `Product Data` = `{{$json.output.string}}`
   - Connect: BrowserAct → Save to Sheets → back to Split In Batches (to continue the loop)

10) **After loop completes: read and aggregate sheet data**
   - Node: **Google Sheets** (“Retrieve Data from Database”)
     - Operation: Read/Get (configure to read the relevant range/rows)
   - Node: **Aggregate** (“Aggregate Google Sheet Data”)
     - Aggregate mode: “Aggregate All Item Data”
   - Connect: Split In Batches (done output) → Retrieve Data → Aggregate

11) **Deep comparison analysis (LLM agent + parser)**
   - Node: **Google Gemini Chat Model** (“Google Gemini 2”) using your Gemini credential (set a specific model if desired)
   - Node: **Structured Output Parser** (“Structured Output”)
     - Schema: `product_1`, `product_2`, `comparison`
   - Node: **AI Agent** (“Analyze the data”)
     - Text should include aggregated data and the original user input and product list:
       - `Data: {{$json.data}}`
       - `User Inputs: {{$('User Sends Message to Bot').first().json.message.text}}`
       - `you need to compare these products only: {{$('Validate user Input').first().json.output.Products}}`
     - System message: enforce the 4-dimension evaluation and JSON-only output
     - Enable output parser and attach the structured output parser
   - Connect: Aggregate → Analyze the data

12) **Generate Telegram caption + image prompt**
   - Node: **Google Gemini Chat Model** (“Google Gemini 3”)
   - Node: **Structured Output Parser** (“Structured Output1”)
     - Schema: `telegram_caption`, `prompt`
   - Node: **AI Agent** (“Generate Image and Description”)
     - Input: `{{$json.output}}`
     - System message: enforce caption rules (HTML tag whitelist, <1000 chars) and Nano Banana prompt blueprint requirements
     - Enable output parser and attach Structured Output1
   - Connect: Analyze the data → Generate Image and Description

13) **Generate image via Gemini Image (Nano Banana Pro)**
   - Node: **Google Gemini (Image)** (“Generate Comparison Image”)
   - Resource: image
   - Model: `models/gemini-3-pro-image-preview`
   - Prompt: `{{$json.output.prompt}}`
   - Binary output property: `data`
   - Connect: Generate Image and Description → Generate Comparison Image

14) **Send photo to Telegram**
   - Node: **Telegram** (“Send Photo Message to Bot”)
   - Operation: sendPhoto
   - Chat ID: `{{$('User Sends Message to Bot').first().json.message.chat.id}}`
   - Binary data: true (uses `data` from previous node)
   - Caption: `{{$('Generate Image and Description').first().json.output["telegram_caption"]}}`
   - parse_mode: HTML
   - Connect: Generate Comparison Image → Send Photo Message to Bot

15) **Chat fallback branch**
   - Node: **Google Gemini Chat Model** (“Google Gemini 1”)
   - Node: **AI Agent** (“Conversational AI”)
     - Text: `Input type : {{$json.output.Type}} | User Input : {{$('User Sends Message to Bot').item.json.message.text}}`
     - System message: respond only with raw text; avoid code fences/tags
   - Node: **Telegram** (“Answer the User”)
     - Operation: sendMessage
     - Chat ID: `{{$('User Sends Message to Bot').item.json.message.chat.id}}`
     - Text: `{{$json.output}}`
   - Connect: Switch (Chat output) → Conversational AI → Answer the User

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Summary: automation compares two products from Telegram, scrapes details with BrowserAct, runs deep AI analysis, generates a “Winner Scorecard” image with Nano Banana Pro. | From sticky note “Documentation” |
| Requirements: Telegram, BrowserAct, Google Gemini (PaLM), Google Sheets credentials. BrowserAct template: “Product Comparison & Visualize Bo”. | From sticky note “Documentation” |
| BrowserAct help links | https://docs.browseract.com |
| Video reference: @[youtube](MCKLEF0m9ps) (full URL: https://www.youtube.com/watch?v=MCKLEF0m9ps) | From sticky note “Sticky Note” |
| Google Sheet requirement: Spreadsheet name `Product Comparision`, column `Product Data` in cell A1 (header). | From sticky note “Sticky Note1” |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.