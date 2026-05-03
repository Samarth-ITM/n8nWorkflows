Import Faire products to Shopify using BrowserAct, Gemini, and Telegram

https://n8nworkflows.xyz/workflows/import-faire-products-to-shopify-using-browseract--gemini--and-telegram-12358


# Import Faire products to Shopify using BrowserAct, Gemini, and Telegram

## 1. Workflow Overview

**Purpose:**  
This workflow lets a user send a **Faire product URL** to a **Telegram bot**. The workflow validates the message, scrapes product data via **BrowserAct** (including handling “human verification”/CAPTCHA pauses), uses **Google Gemini** to rewrite/structure the product content for Shopify (SEO title + HTML description + images + price), then creates the product in **Shopify**, adds a variant/price, and uploads all images.

**Target use cases:**
- Rapidly importing Faire product listings into a Shopify store
- Automatically generating SEO-friendly Shopify content (title + HTML description)
- Supporting a conversational fallback if users don’t provide a valid Faire URL

### 1.1 Input Reception (Telegram)
Receives inbound Telegram messages and forwards message text into AI-based validation.

### 1.2 Input Classification (Gemini + Structured JSON)
Uses a LangChain agent backed by Gemini to classify the message into:
- `Create_Product` with a Faire URL
- `Chat`
- `NoData`

### 1.3 Product Scraping Orchestration (BrowserAct + status handling)
Starts a BrowserAct scraping workflow for the provided Faire link, then routes execution based on task status:
- finished → continue
- paused → ask user to complete verification + wait + poll task data
- failed → alert the user

### 1.4 AI Content Synthesis (Gemini + Structured JSON)
Transforms BrowserAct’s scraped raw product JSON (list) into **one** structured Shopify-ready JSON object (only first product).

### 1.5 Shopify Publishing (Product + Variant/Price + Images)
Creates the Shopify product, adds variant price via Admin API, and uploads images.

### 1.6 Conversational Fallback
If the user didn’t provide a valid Faire link (or is chatting), responds via Gemini-powered agent and sends a Telegram message back.

---

## 2. Block-by-Block Analysis

### Block 1 — Telegram Intake
**Overview:** Receives the user’s Telegram message and makes it available to downstream nodes.  
**Nodes involved:** `User Sends Message to Bot`

#### Node: User Sends Message to Bot
- **Type / role:** Telegram Trigger (`n8n-nodes-base.telegramTrigger`) – entry point.
- **Configuration (interpreted):**
  - Listens to `message` updates.
- **Key data used later:**
  - `{{$json.message.text}}` (user message)
  - `{{$json.message.chat.id}}` (chat routing for replies)
- **Connections:**
  - **Output →** `Validate user Input`
- **Failure/edge cases:**
  - Telegram credential revoked/invalid → trigger won’t fire
  - Bot not added / blocked by user
  - Non-text messages (photos, stickers) will not contain `message.text` (expressions may break downstream unless handled)

---

### Block 2 — Input Classification (Faire link vs chat vs missing data)
**Overview:** Uses an AI agent to classify the message and extract a Faire URL, producing structured JSON output used by routing switches.  
**Nodes involved:** `Validate user Input`, `Google Gemini`, `Structured Output`, `Validation Type Switch`

#### Node: Validate user Input
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) – classification engine.
- **Configuration choices:**
  - Input text: `={{ $json.message.text }}`
  - System message enforces strict JSON-only output with rules:
    - If requesting product/blog creation and contains a URL **and** it’s from Faire (`www.faire.com`) → `{"Type":"Create_Product","Link":"..." }`
    - If greeting/small talk → `{"Type":"Chat","Link":"Null"}`
    - Else → `{"Type":"NoData","Link":"Null"}`
  - `hasOutputParser: true` (paired with structured parser node)
- **Key expressions/variables:**
  - Uses incoming Telegram message text.
- **Connections:**
  - **AI Language Model input ←** `Google Gemini`
  - **AI Output Parser input ←** `Structured Output`
  - **Main output →** `Validation Type Switch` (expects parsed output at `$json.output`)
- **Version-specific notes:**
  - Node typeVersion `3` (LangChain agent behavior and output structure can differ across versions; relies on `$json.output` being populated after parsing).
- **Failure/edge cases:**
  - `message.text` missing → empty prompt; may classify wrongly or fail parsing
  - Model returns invalid JSON → structured parser must fix; if cannot, downstream switch conditions may fail

#### Node: Google Gemini
- **Type / role:** Google Gemini Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) – LLM for validation agent.
- **Configuration choices:**
  - Uses Google PaLM/Gemini credentials.
- **Connections:**
  - **Output (ai_languageModel) →** `Validate user Input`, `Structured Output`
- **Failure/edge cases:**
  - API key invalid / quota exceeded / model errors → validation stops

#### Node: Structured Output
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) – enforces JSON schema-like structure.
- **Configuration choices:**
  - `autoFix: true` (attempt to repair malformed JSON)
  - Example schema: `{"Type":"Article_Request","Link":"extracted_link"}`
    - Note: Example uses `Article_Request`, but the agent’s system prompt expects `Create_Product`, `Chat`, `NoData`. The example is only illustrative but may confuse autofix in edge cases.
- **Connections:**
  - **Output (ai_outputParser) →** `Validate user Input`
- **Failure/edge cases:**
  - If the LLM output deviates heavily, auto-fix may fail; `$json.output.Type` may be absent

#### Node: Validation Type Switch
- **Type / role:** Switch (`n8n-nodes-base.switch`) – routes flow by classification result.
- **Configuration choices:**
  - Rules check `={{ $json.output.Type }}`
    - If equals `Create_Product` → proceed to scraping + alert
    - If equals `Chat` → go to `Answering Agent`
    - If equals `NoData` → go to `Answering Agent`
- **Connections:**
  - **Create_Product branch →** `Process Initialization Alert` and `Scrape Product Data` (two parallel outputs)
  - **Chat branch →** `Answering Agent`
  - **NoData branch →** `Answering Agent`
- **Failure/edge cases:**
  - Missing `$json.output.Type` → no rule matches (workflow may end silently unless a default route is configured; none shown)

---

### Block 3 — Start Scrape + Human Verification Handling (BrowserAct)
**Overview:** Launches a BrowserAct workflow to scrape the Faire product page, then handles task states: finished / paused (CAPTCHA) / failed.  
**Nodes involved:** `Process Initialization Alert`, `Scrape Product Data`, `Human verification Switch`, `Ask User for Verification`, `Give Time to Complete Verification`, `Get Data From BrowserAct`, `Send Failure Alert`

#### Node: Process Initialization Alert
- **Type / role:** Telegram Send Message (`n8n-nodes-base.telegram`) – acknowledges request.
- **Configuration choices:**
  - Text: “Ok, I will do it please give me a moment.”
  - `chatId`: `={{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - `parse_mode: HTML` (though message is plain text)
- **Connections:** none (it runs in parallel with scraping start)
- **Failure/edge cases:**
  - Chat ID expression depends on trigger node item availability; usually OK in same execution path

#### Node: Scrape Product Data
- **Type / role:** BrowserAct node (`n8n-nodes-browseract.browserAct`) – runs an external scraping workflow.
- **Configuration choices:**
  - Mode: `WORKFLOW`
  - Timeout: `7200` seconds (2 hours)
  - BrowserAct `workflowId`: `69706501323865842`
  - Passes input variable:
    - `faire_Target_Link` = `={{ $json.output.Link }}`
  - `open_incognito_mode: false`
- **Connections:**
  - **Output →** `Human verification Switch`
- **Sub-workflow reference:**
  - This node depends on an existing BrowserAct workflow/template in BrowserAct (not an n8n sub-workflow). Sticky note indicates: **“Product Importer Bot for Shopify”** template is mandatory.
- **Failure/edge cases:**
  - BrowserAct auth/workflowId wrong → task creation fails
  - Faire page changes or blocks scraping
  - If BrowserAct returns `paused`, workflow must wait and poll (handled downstream)

#### Node: Human verification Switch
- **Type / role:** Switch (`n8n-nodes-base.switch`) – routes based on BrowserAct task status.
- **Configuration choices:**
  - Checks `={{ $json.status }}`
    - `finished` → proceed to `Analyze Products and Generate Data` (direct)
    - `paused` → ask user to complete verification
    - `failed` → send failure alert
- **Connections:**
  - finished → `Analyze Products and Generate Data`
  - paused → `Ask User for Verification`
  - failed → `Send Failure Alert`
- **Failure/edge cases:**
  - If BrowserAct returns other statuses (e.g., “running”), no route matches and execution may stop

#### Node: Ask User for Verification
- **Type / role:** Telegram Send Message – prompts user for CAPTCHA/human verification.
- **Configuration choices:**
  - Text: “Please complete the human varification.”
  - Chat ID from Telegram trigger node.
- **Connections:**
  - **Output →** `Give Time to Complete Verification`
- **Failure/edge cases:**
  - User may not have access to BrowserAct session; message alone may be insufficient without a link/instructions (not provided)

#### Node: Give Time to Complete Verification
- **Type / role:** Wait (`n8n-nodes-base.wait`) – delays to allow user action.
- **Configuration choices:**
  - Wait 10 minutes
- **Connections:**
  - **Output →** `Get Data From BrowserAct`
- **Failure/edge cases:**
  - If verification takes longer than 10 minutes, polling may still show paused/running and no further handling exists

#### Node: Get Data From BrowserAct
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) – polls BrowserAct task state/results.
- **Configuration choices:**
  - GET `https://api.browseract.com/v2/workflow/get-task`
  - Query param: `task_id = {{ $('Scrape Product Data').item.json.id }}`
  - Auth: predefined `browserActApi`
- **Connections:**
  - **Output →** `Analyze Products and Generate Data`
- **Failure/edge cases:**
  - If `Scrape Product Data` didn’t produce `id` (task creation failed), expression breaks
  - If task still paused/failed, workflow still proceeds to analysis node (no second status gate here)

#### Node: Send Failure Alert
- **Type / role:** Telegram Send Message – user-facing error notification.
- **Configuration choices:**
  - Text: “Sorry we facing problem right now.”
  - Chat ID from Telegram trigger node.
- **Connections:** none
- **Failure/edge cases:**
  - Same Telegram credential risks as other send nodes

---

### Block 4 — AI Processing: Convert scraped data into Shopify-ready JSON
**Overview:** Reads scraped JSON (expected to be a list of products), processes only the first item, and produces structured fields for Shopify product creation (title, type, HTML body, price, images).  
**Nodes involved:** `Analyze Products and Generate Data`, `Google Gemini3`, `Structured Output Parser`

#### Node: Analyze Products and Generate Data
- **Type / role:** LangChain Agent – SEO copywriting + data extraction.
- **Configuration choices:**
  - Input text: `={{ $json.output.string }}`
    - Assumes BrowserAct output is available under `$json.output.string` (likely a raw JSON string)
  - System message instructs:
    - Process only first product object in list (index 0)
    - Output only JSON with: `title`, `product_type`, `price`, `body_html` (inner HTML only), `images[]`
  - `executeOnce: true` (helps prevent repeated LLM calls with multiple items)
  - `hasOutputParser: true` (paired with `Structured Output Parser`)
- **Connections:**
  - **AI Language Model ←** `Google Gemini3`
  - **AI Output Parser ←** `Structured Output Parser`
  - **Main output →** `Create a product` and `Wait for All Inputs` (branch index 1)
- **Failure/edge cases:**
  - If `$json.output.string` is missing or not valid JSON list, LLM may hallucinate or parser may fail
  - Large payload may exceed model limits; consider reducing scraped content size

#### Node: Google Gemini3
- **Type / role:** Gemini Chat Model – LLM for product synthesis agent.
- **Connections:**
  - **Output →** `Analyze Products and Generate Data`, `Structured Output Parser`
- **Failure/edge cases:**
  - Quotas/timeouts; structured output parser may not fix incomplete content

#### Node: Structured Output Parser
- **Type / role:** Structured Output Parser – ensures JSON structure for Shopify fields.
- **Configuration choices:**
  - `autoFix: true`
  - Example schema includes `images` array and `price` numeric.
- **Connections:**
  - **Output (ai_outputParser) →** `Analyze Products and Generate Data`
- **Failure/edge cases:**
  - If HTML contains unescaped quotes, JSON can break; auto-fix may or may not repair

---

### Block 5 — Shopify Publishing (product, variant/price, images)
**Overview:** Creates the product in Shopify, then adds variant pricing via Admin API and uploads each image URL.  
**Nodes involved:** `Create a product`, `Add Price to Product`, `Wait for All Inputs`, `Split Out Images`, `Add Images to Product`

#### Node: Create a product
- **Type / role:** Shopify Node (`n8n-nodes-base.shopify`) – creates a Shopify product resource.
- **Configuration choices:**
  - Authentication: Access Token
  - Resource: `product`
  - Fields:
    - `title = {{ $json.output.title }}`
    - `body_html = {{ $json.output.body_html }}`
    - `product_type = {{ $json.output.product_type }}`
- **Connections:**
  - **Output →** `Add Price to Product` and `Wait for All Inputs`
- **Failure/edge cases:**
  - Missing required fields (e.g., title) → Shopify API error
  - HTML invalid or too long → may be rejected or truncated
  - Credential scope missing `write_products`

#### Node: Add Price to Product
- **Type / role:** HTTP Request – creates a variant with price (Shopify Admin API).
- **Configuration choices:**
  - URL: `https://browseract-2.myshopify.com/admin/api/2025-01/products/{{ $json.id }}/variants.json`
    - Uses product `id` from the **Create a product** output item
  - Method: POST
  - JSON body creates a `variant` with:
    - `option1` = product_type from analysis node
    - `price` = extracted price
    - `sku` = `"SKU-" + product_type` (note: uses a mix of `.first()` and `.item` referencing)
  - `onError: continueRegularOutput` (workflow continues even if this fails)
- **Connections:** none (but runs in parallel with other branch and then merged indirectly)
- **Failure/edge cases:**
  - Shopify typically creates a default variant automatically; adding another may cause duplicates or unwanted variant structure
  - SKU expression inconsistency:
    - `$('Analyze Products and Generate Data').item.json.output.product_type` may not exist depending on item context; safest is `.first()`
  - API version `2025-01` must be supported by store; otherwise 404/410 or version error
  - Requires `write_products` scope

#### Node: Wait for All Inputs
- **Type / role:** Merge (`n8n-nodes-base.merge`) – synchronizes branches.
- **Configuration choices:**
  - Mode: `chooseBranch`
  - `useDataOfInput: 2` (uses data from input 2)
  - Inputs come from:
    - Input 1: `Create a product`
    - Input 2: `Analyze Products and Generate Data`
- **Connections:**
  - **Output →** `Split Out Images`
- **Failure/edge cases:**
  - If `Create a product` fails, merge may not receive expected inputs and image path may not proceed

#### Node: Split Out Images
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) – iterates over image URLs.
- **Configuration choices:**
  - `fieldToSplitOut: output.images`
- **Connections:**
  - **Output →** `Add Images to Product`
- **Failure/edge cases:**
  - If `output.images` is empty/missing, no items emitted → no image uploads
  - If images contain non-URL strings, Shopify may reject

#### Node: Add Images to Product
- **Type / role:** HTTP Request – uploads images by URL to Shopify product.
- **Configuration choices:**
  - URL: `https://browseract-2.myshopify.com/admin/api/2025-01/products/{{ $('Create a product').first().json.id }}/images.json`
  - Method: POST
  - JSON body:
    - `image.src = {{ $json["output.images"] }}`
      - Because Split Out outputs each element into a field named `output.images` (flattened path), it is referenced via bracket notation.
- **Connections:** none
- **Failure/edge cases:**
  - External image URL must be publicly accessible by Shopify
  - API version constraints as above
  - Rate limits when uploading many images

---

### Block 6 — Conversational Fallback (Chat / NoData)
**Overview:** If no valid Faire link is provided, the workflow replies conversationally or asks the user to provide a Faire link.  
**Nodes involved:** `Answering Agent`, `Google Gemini2`, `Answer the User`

#### Node: Answering Agent
- **Type / role:** LangChain Agent – generates a single plain-text response.
- **Configuration choices:**
  - Input text:
    - `Input type : {{ $json.output.Type }} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
  - System message logic:
    - If `NoData` → ask for a Faire product link
    - If `Chat` → respond naturally
  - Output should be raw text (no code fences)
- **Connections:**
  - **AI Language Model ←** `Google Gemini2`
  - **Main output →** `Answer the User`
- **Failure/edge cases:**
  - If `$json.output.Type` missing, prompt becomes ambiguous
  - Might output HTML-sensitive characters; Telegram node uses `parse_mode: HTML` which can misrender if text includes `<` `&`

#### Node: Google Gemini2
- **Type / role:** Gemini Chat Model – LLM for conversational agent.
- **Connections:**
  - **Output →** `Answering Agent`
- **Failure/edge cases:** quota/auth issues

#### Node: Answer the User
- **Type / role:** Telegram Send Message – sends agent response to user.
- **Configuration choices:**
  - Text: `={{ $json.output }}`
  - Chat ID from trigger
  - `parse_mode: HTML`
- **Connections:** none
- **Failure/edge cases:**
  - If agent output contains invalid HTML, Telegram may reject or format unexpectedly

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Entry point: receive Telegram message | — | Validate user Input | ### 🔍 Step 1: Input Analysis & Security |
| Validate user Input | langchain.agent | Classify message + extract Faire URL | User Sends Message to Bot; (AI) Google Gemini; (Parser) Structured Output | Validation Type Switch | ### 🔍 Step 1: Input Analysis & Security |
| Google Gemini | lmChatGoogleGemini | LLM for input validation | — | Validate user Input; Structured Output | ### 🔍 Step 1: Input Analysis & Security |
| Structured Output | outputParserStructured | Enforce JSON output for classification | (AI model) Google Gemini | Validate user Input | ### 🔍 Step 1: Input Analysis & Security |
| Validation Type Switch | switch | Route by Type: Create_Product / Chat / NoData | Validate user Input | Process Initialization Alert; Scrape Product Data; Answering Agent | ### 🔍 Step 1: Input Analysis & Security |
| Process Initialization Alert | telegram | Notify user that processing started | Validation Type Switch | — | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Scrape Product Data | browserAct | Launch BrowserAct scraping workflow | Validation Type Switch | Human verification Switch | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Human verification Switch | switch | Route by BrowserAct status | Scrape Product Data | Analyze Products and Generate Data; Ask User for Verification; Send Failure Alert | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Ask User for Verification | telegram | Prompt user to complete CAPTCHA/verification | Human verification Switch | Give Time to Complete Verification | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Give Time to Complete Verification | wait | Wait 10 minutes before polling task | Ask User for Verification | Get Data From BrowserAct | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Get Data From BrowserAct | httpRequest | Poll BrowserAct task results | Give Time to Complete Verification | Analyze Products and Generate Data | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Send Failure Alert | telegram | Notify user of failure | Human verification Switch | — | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Analyze Products and Generate Data | langchain.agent | Convert scraped JSON into Shopify-ready JSON | Human verification Switch / Get Data From BrowserAct; (AI) Google Gemini3; (Parser) Structured Output Parser | Create a product; Wait for All Inputs | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Google Gemini3 | lmChatGoogleGemini | LLM for product synthesis | — | Analyze Products and Generate Data; Structured Output Parser | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Structured Output Parser | outputParserStructured | Enforce JSON schema for product fields | (AI model) Google Gemini3 | Analyze Products and Generate Data | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Create a product | shopify | Create Shopify product | Analyze Products and Generate Data | Add Price to Product; Wait for All Inputs | ### 📦 Step 3: Product Creation & Image Upload |
| Add Price to Product | httpRequest | Add variant/price via Shopify Admin API | Create a product | — | ### 📦 Step 3: Product Creation & Image Upload |
| Wait for All Inputs | merge | Sync product creation + AI data for images | Create a product; Analyze Products and Generate Data | Split Out Images | ### 📦 Step 3: Product Creation & Image Upload |
| Split Out Images | splitOut | Iterate over image URLs | Wait for All Inputs | Add Images to Product | ### 📦 Step 3: Product Creation & Image Upload |
| Add Images to Product | httpRequest | Upload images to Shopify product | Split Out Images | — | ### 📦 Step 3: Product Creation & Image Upload |
| Answering Agent | langchain.agent | Fallback: chat or ask for Faire link | Validation Type Switch; (AI) Google Gemini2 | Answer the User | ### 💬 Step 2-2: Conversational Fallback |
| Google Gemini2 | lmChatGoogleGemini | LLM for fallback responses | — | Answering Agent | ### 💬 Step 2-2: Conversational Fallback |
| Answer the User | telegram | Send fallback response to Telegram | Answering Agent | — | ### 💬 Step 2-2: Conversational Fallback |
| Documentation | stickyNote | Documentation panel | — | — | ## ⚡ Workflow Overview & Setup (plus BrowserAct docs links) |
| Step 1 Explanation | stickyNote | Comment: input analysis & security | — | — | ### 🔍 Step 1: Input Analysis & Security |
| Step 2 Explanation | stickyNote | Comment: scraping & synthesis | — | — | ### 🛍️ Step 2: Scraping & Content Synthesis |
| Step 3 Explanation | stickyNote | Comment: Shopify creation & upload | — | — | ### 📦 Step 3: Product Creation & Image Upload |
| Step 4 Explanation | stickyNote | Comment: fallback branch | — | — | ### 💬 Step 2-2: Conversational Fallback |
| Sticky Note | stickyNote | Video link | — | — | @[youtube](1Q9-XGlaoFA) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name: *Import products from Faire to Shopify using BrowserAct, Gemini and Telegram* (or your preferred name)

2) **Add Telegram Trigger**
   - Node: **Telegram Trigger**  
   - Updates: `message`  
   - Credentials: Telegram bot token credential (Telegram API)

3) **Add “Validate user Input” (LangChain Agent)**
   - Node: **LangChain Agent**
   - Text: `{{$json.message.text}}`
   - System message: implement the exact classification rules:
     - output JSON only
     - `Create_Product` only if URL exists and belongs to `www.faire.com`
     - else `Chat` or `NoData`
   - Enable output parsing (node option: “has output parser” / structured output)

4) **Add Gemini model for validation**
   - Node: **Google Gemini Chat Model**
   - Credentials: Google Gemini/PaLM API
   - Connect as **AI Language Model** to `Validate user Input`

5) **Add Structured Output parser for validation**
   - Node: **Structured Output Parser**
   - Enable `Auto-fix`
   - Provide a schema example like:
     - `{"Type":"Create_Product","Link":"https://www.faire.com/..."}`
   - Connect parser output to `Validate user Input` as **AI Output Parser**

6) **Add “Validation Type Switch”**
   - Node: **Switch**
   - Rules (string equals) on: `{{$json.output.Type}}`
     - `Create_Product` → branch 1
     - `Chat` → branch 2
     - `NoData` → branch 3
   - Connect `Validate user Input` → `Validation Type Switch`

7) **Create Product branch: Telegram “Process Initialization Alert”**
   - Node: **Telegram**
   - Chat ID: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
   - Text: “Ok, I will do it please give me a moment.”
   - Connect from `Validation Type Switch` (Create_Product output)

8) **Create Product branch: BrowserAct scrape**
   - Node: **BrowserAct**
   - Type: `WORKFLOW`
   - Timeout: `7200`
   - Workflow ID: your BrowserAct workflow id (in JSON: `69706501323865842`)
   - Workflow config input mapping:
     - `faire_Target_Link` = `{{$json.output.Link}}`
   - Credentials: BrowserAct API credential
   - Connect from `Validation Type Switch` (Create_Product output)

9) **Add “Human verification Switch”**
   - Node: **Switch**
   - Rules on `{{$json.status}}`:
     - `finished` → continue to AI synthesis
     - `paused` → verification flow
     - `failed` → send failure alert
   - Connect `Scrape Product Data` → `Human verification Switch`

10) **Paused path: Ask user + wait + poll**
   - Add **Telegram** node “Ask User for Verification”
     - Chat ID: from trigger
     - Text: “Please complete the human varification.”
   - Add **Wait** node “Give Time to Complete Verification”
     - Unit: minutes, Amount: 10
   - Add **HTTP Request** node “Get Data From BrowserAct”
     - GET `https://api.browseract.com/v2/workflow/get-task`
     - Query param: `task_id = {{ $('Scrape Product Data').item.json.id }}`
     - Auth: BrowserAct predefined credential
   - Connect: `Human verification Switch (paused)` → Ask → Wait → Get Data

11) **Failed path: Send Failure Alert**
   - Add **Telegram** node “Send Failure Alert”
     - Text: “Sorry we facing problem right now.”
     - Chat ID: from trigger
   - Connect: `Human verification Switch (failed)` → Send Failure Alert

12) **Add AI product synthesis agent**
   - Node: **LangChain Agent** “Analyze Products and Generate Data”
   - Text: `{{$json.output.string}}` (must match what BrowserAct returns; adjust if your BrowserAct output field differs)
   - System message: instruct to process only first product and output JSON with:
     - `title`, `product_type`, `body_html` (inner HTML only), `price` (number), `images` (array of URLs)
   - Enable output parsing

13) **Add Gemini model + structured parser for synthesis**
   - Add **Google Gemini Chat Model** “Google Gemini3” with same credentials
   - Add **Structured Output Parser** “Structured Output Parser” with auto-fix and schema example containing `images[]` and numeric `price`
   - Connect:
     - `Google Gemini3` → AI Language Model of `Analyze Products and Generate Data`
     - `Structured Output Parser` → AI Output Parser of `Analyze Products and Generate Data`

14) **Connect into synthesis**
   - Connect:
     - `Human verification Switch (finished)` → `Analyze Products and Generate Data`
     - `Get Data From BrowserAct` → `Analyze Products and Generate Data`

15) **Create Shopify product**
   - Node: **Shopify** “Create a product”
   - Auth: Access Token
   - Resource: Product, Operation: Create
   - Fields:
     - Title: `{{$json.output.title}}`
     - Body HTML: `{{$json.output.body_html}}`
     - Product type: `{{$json.output.product_type}}`
   - Credentials: Shopify Admin Access Token credential with `write_products`

16) **Add price/variant (HTTP Request)**
   - Node: **HTTP Request** “Add Price to Product”
   - POST `https://<your-shop>.myshopify.com/admin/api/2025-01/products/{{ $json.id }}/variants.json`
   - Body JSON:
     - variant.option1 = `{{ $('Analyze Products and Generate Data').first().json.output.product_type }}`
     - variant.price = `{{ $('Analyze Products and Generate Data').first().json.output.price }}`
     - variant.sku = `SKU-...`
   - Auth: predefined Shopify credential
   - Set **On Error** to “Continue” if you want the workflow to proceed even on variant failure (as in the JSON)

17) **Merge for images**
   - Node: **Merge** “Wait for All Inputs”
   - Mode: `chooseBranch`
   - “Use data of input”: 2 (the AI synthesis branch that contains `output.images`)
   - Connect:
     - `Create a product` → Merge input 1
     - `Analyze Products and Generate Data` → Merge input 2

18) **Split and upload images**
   - Node: **Split Out** “Split Out Images”
     - Field to split out: `output.images`
   - Node: **HTTP Request** “Add Images to Product”
     - POST `https://<your-shop>.myshopify.com/admin/api/2025-01/products/{{ $('Create a product').first().json.id }}/images.json`
     - Body JSON: `{ "image": { "src": "{{ $json['output.images'] }}" } }`
     - Auth: predefined Shopify credential
   - Connect: Merge → Split Out → Add Images

19) **Fallback branch: Answering Agent**
   - Node: **LangChain Agent** “Answering Agent”
   - Text: `Input type : {{$json.output.Type}} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
   - System message:
     - If NoData → ask for Faire product link
     - If Chat → respond naturally
     - Output raw text only
   - Add **Google Gemini Chat Model** “Google Gemini2” and connect it as AI Language Model
   - Add **Telegram** “Answer the User”
     - Text: `{{$json.output}}`
     - Chat ID from trigger
     - (Optional) Disable HTML parse mode if you want to avoid formatting issues
   - Connect `Validation Type Switch` Chat and NoData branches → Answering Agent → Answer the User

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Workflow overview & setup:** Requires credentials for Telegram, BrowserAct, Google Gemini (PaLM), Shopify. Mandatory BrowserAct template: **Product Importer Bot for Shopify**. | From sticky note “Documentation” |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video link | @[youtube](1Q9-XGlaoFA) |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.