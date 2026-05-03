Generate recipe variations from ingredients using Google Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/generate-recipe-variations-from-ingredients-using-google-gemini-and-google-sheets-12121


# Generate recipe variations from ingredients using Google Gemini and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Collect ingredients (and mood + email) via an n8n Form, ask **Google Gemini** to generate **3 recipe variations** (Speed / Healthy / Creative) as JSON, enrich each recipe with an **image URL** via **Google Custom Search (Images)**, store results in **Google Sheets**, then send a **single aggregated email** containing the 3 options.

**Target use cases:**
- “What can I cook with leftovers?” quick generation of multiple recipe styles.
- Capturing generated recipes into a lightweight database (Google Sheets).
- Automated personal delivery (Gmail email).

### Logical Blocks
1. **Input Reception & Configuration**
   - Form input + static configuration (Sheet ID, Custom Search API key, Search Engine ID)
2. **AI Processing**
   - Prompt Gemini to return a JSON array of recipes, then split into items
3. **Enrichment Loop (per recipe item)**
   - Image search + append results to Google Sheets
4. **Delivery**
   - Aggregate items into an HTML email body and send via Gmail

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Configuration

**Overview:** Captures user input from a Form Trigger and injects required identifiers/keys used downstream (Sheets + Custom Search). This block is the entry point and ensures later nodes can reference consistent settings.

**Nodes Involved:**
- Sticky Note Main (comment)
- Sticky Note Config (comment)
- Ingredient Form
- Configuration

#### Node: Sticky Note Main
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Contains a high-level explanation + setup checklist (sheet headers, credentials, where to configure IDs/keys).
- **Connections:** None (notes do not connect).
- **Edge cases:** None.

#### Node: Sticky Note Config
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Labels this as “1. Configuration”.
- **Connections:** None.
- **Edge cases:** None.

#### Node: Ingredient Form
- **Type / role:** `formTrigger` (entry point UI form)
- **Key configuration:**
  - Form title: “Lazy Chef - Recipe Generator”
  - Description: explains it generates Speed/Healthy/Creative recipes
  - Fields:
    - **Ingredients** (required, free text)
    - **Current Mood** (required, default “Hungry!”)
    - **Your Email** (required, email type)
  - Button label: “Generate Recipes”
- **Outputs:** Runs workflow with submitted form fields in JSON (e.g., `$json['Ingredients']`).
- **Connections:**
  - Output → **Configuration**
- **Potential failures / edge cases:**
  - Missing required fields (handled by the form UI)
  - Users may enter very long ingredients lists; may impact LLM output length/quality.

#### Node: Configuration
- **Type / role:** `set` node (injects constants)
- **Configuration choices:**
  - Adds fields while **including other incoming fields**:
    - `sheetId` (Spreadsheet ID placeholder)
    - `googleSearchApiKey` (API key placeholder)
    - `googleSearchEngineId` (Custom Search Engine cx placeholder)
- **Key expressions/variables:** None (static strings in this workflow; typically replaced with real values).
- **Connections:**
  - Input ← **Ingredient Form**
  - Output → **Gemini: Generate Recipes**
- **Potential failures / edge cases:**
  - Misconfigured/invalid IDs or API keys will cause downstream HTTP/Sheets failures.
  - Storing API keys directly in workflow data is risky; prefer n8n credentials or environment variables.

---

### 2.2 AI Processing

**Overview:** Sends a structured prompt to Gemini requesting **ONLY a JSON array** with 3 recipe objects, then parses and converts that JSON array into individual n8n items for per-recipe processing.

**Nodes Involved:**
- Sticky Note AI (comment)
- Gemini: Generate Recipes
- Split into Items

#### Node: Sticky Note AI
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Labels this as “2. AI Processing”.
- **Connections:** None.
- **Edge cases:** None.

#### Node: Gemini: Generate Recipes
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` (LLM call)
- **Configuration choices:**
  - Model: `models/gemini-1.5-flash` (fast/cheap general model)
  - Prompt instructs:
    - Role: “creative chef”
    - Produce 3 styles (Speed 5 min, Healthy low calorie, Creative unique)
    - Output **ONLY a JSON Array** of objects with:
      - `style`
      - `recipe_name`
      - `search_query`
      - `recipe_text`
  - Injects user inputs using expressions:
    - `Ingredients: {{ $json['Ingredients'] }}`
    - `Mood: {{ $json['Current Mood'] }}`
- **Credentials:** Google Gemini(PaLM) API account (must be configured in n8n)
- **Connections:**
  - Input ← **Configuration**
  - Output → **Split into Items**
- **Potential failures / edge cases:**
  - Auth/quota errors from Gemini credentials.
  - Model may still wrap JSON in code fences (```json) or include commentary despite instructions.
  - Output schema drift (missing keys) can break later expressions.

#### Node: Split into Items
- **Type / role:** `code` node (JS transformation)
- **Configuration choices / logic:**
  - Reads Gemini response text from:  
    `const text = $input.first().json.content.parts[0].text;`
  - Removes code fences and parses JSON.
  - If parsing fails, returns a single fallback item:
    - `style: "Error"`, `recipe_text: "Failed to parse AI output."`
- **Inputs/outputs:**
  - Input: 1 item from Gemini
  - Output: 3 items (expected) or 1 “Error” item (fallback)
- **Connections:**
  - Input ← **Gemini: Generate Recipes**
  - Output → **Google Image Search** (parallel) AND **Save to Google Sheets** (parallel)
- **Potential failures / edge cases:**
  - If Gemini output structure changes, the path `.json.content.parts[0].text` may be undefined → runtime error.
  - If parsed JSON is not an array, returning it may create unexpected item structure.
  - Fallback item uses `search_query: "Error"`; downstream image search may still run and fail.

---

### 2.3 Enrichment Loop (per recipe item)

**Overview:** For each recipe item, the workflow retrieves one image via Google Custom Search and writes a row to Google Sheets with the recipe data and image URL.

**Nodes Involved:**
- Sticky Note Process (comment)
- Google Image Search
- Save to Google Sheets

#### Node: Sticky Note Process
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Labels this as “3. Enrichment Loop”.
- **Connections:** None.
- **Edge cases:** None.

#### Node: Google Image Search
- **Type / role:** `httpRequest` (calls Google Custom Search JSON API)
- **Configuration choices:**
  - Method: implicit GET (via URL + query parameters)
  - URL: `https://www.googleapis.com/customsearch/v1`
  - Query params:
    - `q` = `{{$json.search_query}}`
    - `cx` = `{{ $('Configuration').first().json.googleSearchEngineId }}`
    - `key` = `{{ $('Configuration').first().json.googleSearchApiKey }}`
    - `searchType` = `image`
    - `num` = `1`
- **Connections:**
  - Input ← **Split into Items**
  - Output → **Save to Google Sheets**
- **Potential failures / edge cases:**
  - Invalid API key / cx, quota exceeded, or Custom Search API not enabled.
  - No results: `items` may be missing or empty → later expression `items[0].link` will fail.
  - Rate limits if many executions occur.

#### Node: Save to Google Sheets
- **Type / role:** `googleSheets` (data persistence)
- **Operation:** `appendOrUpdate`
- **Document / sheet:**
  - Spreadsheet ID: `={{ $('Configuration').first().json.sheetId }}`
  - Sheet name: `Recipes`
- **Column mapping (define below):**
  - `date` = `{{ $now.format('yyyy-MM-dd HH:mm') }}`
  - `ingredients` = `{{ $('Ingredient Form').first().json['Ingredients'] }}`
  - `style` = `{{ $json.style }}`
  - `recipe_name` = `{{ $json.recipe_name }}`
  - `recipe_text` = `{{ $json.recipe_text }}`
  - `image_url` = `{{ $('Google Image Search').item.json.items[0].link }}`
- **Credentials:** Google Sheets OAuth2
- **Connections:**
  - Inputs:
    - From **Split into Items** (direct)
    - From **Google Image Search** (direct)
  - Output → **Aggregate to HTML**
- **Important behavior note (data dependency risk):**
  - This node references `$('Google Image Search').item...` but it can also be triggered directly from **Split into Items**. If it executes before the HTTP node for that item (or without its data), `image_url` resolution can fail.
  - The workflow also connects **Google Image Search → Save to Google Sheets**, which is the correct dependency chain; the extra **Split → Save** connection is a potential misconfiguration.
- **Potential failures / edge cases:**
  - Sheet/tab name mismatch (“Recipes” must exist).
  - Headers must match exactly (`date`, `ingredients`, `style`, `recipe_name`, `recipe_text`, `image_url`).
  - Empty `items[0]` from search results will break expression unless guarded.
  - OAuth token expiration / insufficient permissions.

---

### 2.4 Delivery

**Overview:** Collects all processed recipe items, generates a single HTML email body, and emails it to the address provided in the form.

**Nodes Involved:**
- Sticky Note Delivery (comment)
- Aggregate to HTML
- Send Email

#### Node: Sticky Note Delivery
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Labels this as “4. Delivery”.
- **Connections:** None.
- **Edge cases:** None.

#### Node: Aggregate to HTML
- **Type / role:** `code` node (aggregation + HTML templating)
- **Configuration choices / logic:**
  - Reads all incoming items: `const items = $input.all();`
  - Builds an HTML string with 3 sections, each showing:
    - style + recipe name
    - image tag using `item.json.image_url` (or empty string)
    - instructions text
  - Outputs one item with:
    - `emailBody` (HTML string)
    - `recipient` = `$('Ingredient Form').first().json["Your Email"]`
- **Connections:**
  - Input ← **Save to Google Sheets**
  - Output → **Send Email**
- **Potential failures / edge cases:**
  - If upstream produced only 1 “Error” item, the email will contain that single entry.
  - If `image_url` missing, `<img src=''>` may render broken.
  - HTML includes a header with an emoji; some email clients may render differently.

#### Node: Send Email
- **Type / role:** `gmail` (send message)
- **Key configuration:**
  - To: `={{ $json.recipient }}`
  - Subject: “🍳 Your 3 Recipe Proposals”
  - Message: `={{ $json.emailBody }}`
  - Email type set as **text** (despite HTML content)
- **Credentials:** Gmail OAuth2
- **Connections:**
  - Input ← **Aggregate to HTML**
  - Output: end
- **Potential failures / edge cases:**
  - Gmail OAuth not authorized or missing scopes.
  - Because **emailType is “text”**, HTML may be delivered as raw markup. If the intention is rich rendering, this should be set to HTML (depending on node options/version).
  - User-provided email address could be invalid; Gmail send will fail.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note Main | stickyNote | Documentation / setup notes |  |  | # Generate Recipe Variations; Setup steps: create Google Sheet headers; set credentials for Gemini/Custom Search/Sheets/Gmail; configure Spreadsheet ID, Search Engine ID, API Key in “1. Configuration”. |
| Sticky Note Config | stickyNote | Documentation / block label |  |  | ## 1. Configuration Accepts user input and holds API settings. |
| Sticky Note AI | stickyNote | Documentation / block label |  |  | ## 2. AI Processing Generates 3 recipes and splits them for processing. |
| Sticky Note Process | stickyNote | Documentation / block label |  |  | ## 3. Enrichment Loop Fetches images and saves data for each recipe. |
| Sticky Note Delivery | stickyNote | Documentation / block label |  |  | ## 4. Delivery Combines results and sends an email. |
| Ingredient Form | formTrigger | Collect ingredients/mood/email |  | Configuration | ## 1. Configuration Accepts user input and holds API settings. |
| Configuration | set | Store sheetId + search API settings | Ingredient Form | Gemini: Generate Recipes | ## 1. Configuration Accepts user input and holds API settings. |
| Gemini: Generate Recipes | googleGemini (LangChain) | Generate 3 recipes JSON | Configuration | Split into Items | ## 2. AI Processing Generates 3 recipes and splits them for processing. |
| Split into Items | code | Parse Gemini JSON and emit items | Gemini: Generate Recipes | Google Image Search; Save to Google Sheets | ## 2. AI Processing Generates 3 recipes and splits them for processing. |
| Google Image Search | httpRequest | Fetch 1 image URL per recipe | Split into Items | Save to Google Sheets | ## 3. Enrichment Loop Fetches images and saves data for each recipe. |
| Save to Google Sheets | googleSheets | Persist each recipe row | Split into Items; Google Image Search | Aggregate to HTML | ## 3. Enrichment Loop Fetches images and saves data for each recipe. |
| Aggregate to HTML | code | Combine items into one email payload | Save to Google Sheets | Send Email | ## 4. Delivery Combines results and sends an email. |
| Send Email | gmail | Send aggregated email | Aggregate to HTML |  | ## 4. Delivery Combines results and sends an email. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheet**
   1. Create a spreadsheet.
   2. Add a tab named **`Recipes`**.
   3. Add headers in row 1 exactly:  
      `date`, `ingredients`, `style`, `recipe_name`, `recipe_text`, `image_url`.

2. **Create the Form Trigger**
   1. Add node: **Form Trigger** (name: *Ingredient Form*).
   2. Set:
      - Title: *Lazy Chef - Recipe Generator*
      - Description: *Enter ingredients. AI will generate Speed, Healthy, and Creative recipes.*
      - Button label: *Generate Recipes*
   3. Add fields:
      - Text: **Ingredients** (required; placeholder like “e.g., Eggs, Milk, Rice”)
      - Text: **Current Mood** (required; default “Hungry!”)
      - Email: **Your Email** (required)

3. **Add a Set node for configuration**
   1. Add node: **Set** (name: *Configuration*).
   2. Enable **Include Other Fields**.
   3. Add string fields:
      - `sheetId` = your spreadsheet ID
      - `googleSearchApiKey` = your Google API key with Custom Search enabled
      - `googleSearchEngineId` = your Custom Search Engine cx
   4. Connect: **Ingredient Form → Configuration**.

4. **Add Google Gemini node**
   1. Add node: **Google Gemini** (LangChain) (name: *Gemini: Generate Recipes*).
   2. Select model: `gemini-1.5-flash` (or equivalent available).
   3. Set the user message content to request **ONLY JSON Array** with fields: `style`, `recipe_name`, `search_query`, `recipe_text`.
   4. Use expressions for inputs:
      - Ingredients: `{{ $json['Ingredients'] }}`
      - Mood: `{{ $json['Current Mood'] }}`
   5. Create/attach **Google Gemini(PaLM) API credentials** in n8n.
   6. Connect: **Configuration → Gemini: Generate Recipes**.

5. **Add Code node to split items**
   1. Add node: **Code** (name: *Split into Items*).
   2. Paste logic that:
      - Reads Gemini text output
      - Strips ```json fences
      - `JSON.parse` into an array
      - Returns the array (n8n will treat each element as an item)
      - On parse error, returns a single error item
   3. Connect: **Gemini: Generate Recipes → Split into Items**.

6. **Add HTTP Request for Google Custom Search image**
   1. Add node: **HTTP Request** (name: *Google Image Search*).
   2. URL: `https://www.googleapis.com/customsearch/v1`
   3. Enable “Send Query Parameters”.
   4. Add query parameters:
      - `q` = `{{ $json.search_query }}`
      - `cx` = `{{ $('Configuration').first().json.googleSearchEngineId }}`
      - `key` = `{{ $('Configuration').first().json.googleSearchApiKey }}`
      - `searchType` = `image`
      - `num` = `1`
   5. Connect: **Split into Items → Google Image Search**.

7. **Add Google Sheets node**
   1. Add node: **Google Sheets** (name: *Save to Google Sheets*).
   2. Credentials: configure **Google Sheets OAuth2**.
   3. Operation: **Append or Update**.
   4. Document ID: `{{ $('Configuration').first().json.sheetId }}`
   5. Sheet name: `Recipes`
   6. Map columns (by name):
      - `date` = `{{ $now.format('yyyy-MM-dd HH:mm') }}`
      - `ingredients` = `{{ $('Ingredient Form').first().json['Ingredients'] }}`
      - `style` = `{{ $json.style }}`
      - `recipe_name` = `{{ $json.recipe_name }}`
      - `recipe_text` = `{{ $json.recipe_text }}`
      - `image_url` = `{{ $('Google Image Search').item.json.items[0].link }}`
   7. Connect: **Google Image Search → Save to Google Sheets**.  
      (If you also connect Split → Save, be aware it can break `image_url` resolution.)

8. **Add Code node to aggregate into one email**
   1. Add node: **Code** (name: *Aggregate to HTML*).
   2. Build HTML from `$input.all()` and output one item containing:
      - `emailBody`
      - `recipient` = `{{ $('Ingredient Form').first().json["Your Email"] }}`
   3. Connect: **Save to Google Sheets → Aggregate to HTML**.

9. **Add Gmail node to send**
   1. Add node: **Gmail** (name: *Send Email*).
   2. Credentials: configure **Gmail OAuth2** in n8n.
   3. Set:
      - To: `{{ $json.recipient }}`
      - Subject: `🍳 Your 3 Recipe Proposals`
      - Message: `{{ $json.emailBody }}`
      - Ensure the node is configured to send **HTML** if you want HTML rendering (the provided workflow uses “text”).
   4. Connect: **Aggregate to HTML → Send Email**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Google Sheet with headers: `date`, `ingredients`, `style`, `recipe_name`, `recipe_text`, `image_url`. | Required for the Google Sheets mapping to work. |
| Set up credentials for Google Gemini, Custom Search, Sheets, and Gmail. | Needed for Gemini call, image search API, Sheets writes, and email sending. |
| Enter Spreadsheet ID, Search Engine ID, and API Key in the “Configuration” node. | Centralized configuration used by multiple nodes. |