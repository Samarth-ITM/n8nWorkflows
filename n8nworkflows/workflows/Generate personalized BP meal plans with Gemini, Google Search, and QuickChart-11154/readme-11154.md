Generate personalized BP meal plans with Gemini, Google Search, and QuickChart

https://n8nworkflows.xyz/workflows/generate-personalized-bp-meal-plans-with-gemini--google-search--and-quickchart-11154


# Generate personalized BP meal plans with Gemini, Google Search, and QuickChart

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow acts as an “AI nutritionist” for blood pressure (BP) management. Every Monday at 7:00 AM, it pulls the last 7 days of BP readings from Google Sheets, classifies the user’s status (High/Normal/Optimal), asks **Gemini 1.5 Flash** to pick the best Japanese recipe-search query, fetches real recipe candidates via **Google Custom Search**, generates a BP trend chart via **QuickChart**, then asks Gemini to synthesize a **5-day (Mon–Fri) dinner plan** and emails the report with the chart.

**Target use cases:**
- Weekly BP-aware meal planning (especially low-sodium planning when BP is elevated)
- Automated health summary + visual chart + recipe links delivered by email
- Template workflow that can be adapted to other biometrics or nutrition constraints

### 1.1 Input Reception & Scheduling
- Triggered weekly (Monday 07:00).
- Loads configuration placeholders (Sheet ID, API keys, etc.).

### 1.2 BP Data Collection & Aggregation
- Reads BP rows from Google Sheets.
- Computes 7-day averages and classifies BP status.

### 1.3 AI Strategy (Query Decision)
- Gemini generates **one Japanese search query string** tailored to BP status.

### 1.4 Parallel Execution (Recipe Search + Chart)
- Uses the Gemini query to search recipes (Cookpad / Rakuten Recipes).
- Builds a QuickChart URL and downloads the chart image.

### 1.5 Synthesis & Delivery
- Merges recipe search results + chart branch.
- Gemini creates the weekly narrative + 5-day dinner plan with URLs.
- Sends an email report (text) (chart is downloaded, but attachment handling depends on node configuration—see notes).

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Scheduling
**Overview:** Starts the workflow weekly and injects user-specific configuration values used by later nodes (Sheets identifiers and Google Search credentials).  
**Nodes involved:**  
- Weekly Schedule (Mon 7AM)  
- Workflow Configuration  

#### Node: Weekly Schedule (Mon 7AM)
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Configuration (interpreted):** Runs every week on **Monday** at **07:00**.
- **Inputs/outputs:** No input. Output → **Workflow Configuration**.
- **Version-specific notes:** Uses schedule trigger `typeVersion 1.2`; UI options can vary slightly across n8n versions.
- **Failure modes / edge cases:**
  - n8n instance timezone affects “7AM”; verify instance timezone settings.
  - If instance is down at trigger time, run is missed unless you implement catch-up externally.

#### Node: Workflow Configuration
- **Type / role:** `Set` — central parameter store.
- **Configuration choices:**
  - Adds fields:
    - `spreadsheetId` (placeholder)
    - `sheetName` (placeholder)
    - `googleSearchApiKey` (placeholder)
    - `googleSearchEngineId` (placeholder)
  - **Include other fields: true** (passes through incoming fields, though trigger has none).
- **Key variables used later:**
  - `$('Workflow Configuration').first().json.googleSearchApiKey`
  - `$('Workflow Configuration').first().json.googleSearchEngineId`
- **Inputs/outputs:** Input ← Schedule Trigger. Output → **Get Blood Pressure Data**.
- **Failure modes / edge cases:**
  - Leaving placeholders unchanged will cause downstream API calls to fail (Sheets/search).
  - Storing API keys in node fields is functional but not ideal; credentials/environment variables are safer.

---

### Block 2 — BP Data Collection & Aggregation
**Overview:** Retrieves BP readings from Google Sheets and computes 7-day averages and a status label used for both search strategy and reporting.  
**Nodes involved:**  
- Get Blood Pressure Data  
- Calculate Averages  

#### Node: Get Blood Pressure Data
- **Type / role:** `Google Sheets` — reads spreadsheet data.
- **Configuration (interpreted):**
  - Resource set to **Spreadsheet**.
  - The JSON does not show the specific operation/range; it is expected to be configured in the UI to read rows from the given sheet.
  - Uses OAuth2 credentials: **googleSheetsOAuth2Api**.
- **Inputs/outputs:** Input ← Workflow Configuration. Output → Calculate Averages.
- **Version-specific requirements:** `typeVersion 4.7` (recent Google Sheets node). Field names/operations differ between v3 and v4+.
- **Failure modes / edge cases:**
  - OAuth token expired/revoked → auth error.
  - Spreadsheet ID / sheet name mismatch.
  - Data types: if `date`, `systolic`, `diastolic` columns are missing or named differently, the Code node will misbehave.
  - If the node returns no rows, averages become 0 and status may become “Optimal” due to logic (see next node).

#### Node: Calculate Averages
- **Type / role:** `Code` — transforms rows into a single weekly summary object.
- **Configuration (logic summary):**
  - Takes all incoming rows (`$input.all()`).
  - Filters to entries whose `item.json.date` is within last 7 days.
  - Computes:
    - `avgSystolic` = rounded mean of `systolic`
    - `avgDiastolic` = rounded mean of `diastolic`
    - `dataCount` = number of recent entries
  - Status logic:
    - Start: `Normal`
    - If `avgSystolic > 130 OR avgDiastolic > 80` → `High`
    - If `avgSystolic < 120 AND avgDiastolic < 80` → `Optimal` (this can override “High” if averages are 0)
  - Outputs one item with:
    - `avgSystolic`, `avgDiastolic`, `dataCount`, `status`, `rawData` (array of recent row JSON)
- **Key expressions/vars:** Uses `item.json.date`, `item.json.systolic`, `item.json.diastolic`.
- **Inputs/outputs:**
  - Input ← Get Blood Pressure Data
  - Output → (two branches)
    - **Message a model** (Gemini decide query)
    - **Generate Chart Config** (QuickChart)
- **Failure modes / edge cases:**
  - **Empty or invalid dates:** `new Date(item.json.date)` may produce `Invalid Date` and filter incorrectly.
  - **No data in last 7 days:** `dataCount = 0` → averages = 0 → status becomes **Optimal** due to `<120/<80` condition. If undesired, add a guard (e.g., if dataCount===0 then status = 'NoData').
  - Non-numeric BP values become `NaN` if not parseable; current code falls back to `0` only when field is falsy, not when it is a non-numeric string.

---

### Block 3 — AI Strategy (Search Query Decision)
**Overview:** Gemini decides the most appropriate Japanese search query string based on the computed BP status.  
**Nodes involved:**  
- Message a model (Gemini decide)

#### Node: Message a model (Gemini decide)
- **Type / role:** `Google Gemini (LangChain)` — LLM call to produce a search query.
- **Configuration (interpreted):**
  - Model: `models/gemini-1.5-flash`
  - Prompt includes:
    - Avg systolic/diastolic + status
    - Instruction: “Output ONLY the search query string.”
    - Examples: High → “減塩 夕食 レシピ”, Normal → “バランスの良い 夕食 レシピ”
- **Key expressions/variables:** Injects `{{ $json.avgSystolic }}`, `{{ $json.avgDiastolic }}`, `{{ $json.status }}` from Calculate Averages output.
- **Inputs/outputs:** Input ← Calculate Averages. Output → Google Custom Search.
- **Credentials:** `googlePalmApi` credential (used by this node type for Gemini in n8n).
- **Failure modes / edge cases:**
  - Model may output extra text despite instruction; downstream Google Search uses `{{$json.content.parts[0].text}}` and expects the response structure and that the query is in that field.
  - API quota / billing issues.
  - If Gemini response structure differs (node version changes), the path `content.parts[0].text` may not exist.

---

### Block 4 — Parallel Execution (Recipe Search + Chart)
**Overview:** Executes two tasks in parallel: searches for recipe candidates using Google Custom Search and builds a BP trend chart image through QuickChart.  
**Nodes involved:**  
- Google Custom Search  
- Generate Chart Config  
- Download Chart Image  

#### Node: Google Custom Search
- **Type / role:** `HTTP Request` — calls Google Custom Search JSON API.
- **Configuration (interpreted):**
  - URL: `https://www.googleapis.com/customsearch/v1`
  - Sends query parameters:
    - `key` = from Workflow Configuration (`googleSearchApiKey`)
    - `cx` = from Workflow Configuration (`googleSearchEngineId`)
    - `q` = Gemini output + `site:cookpad.com OR site:recipe.rakuten.co.jp`
    - `num` = `5`
- **Key expressions/variables:**
  - `q`: `={{ $json.content.parts[0].text }} site:cookpad.com OR site:recipe.rakuten.co.jp`
- **Inputs/outputs:** Input ← Gemini decide. Output → Merge Search & Chart (input index 0).
- **Failure modes / edge cases:**
  - Invalid API key / engine ID → 4xx.
  - Google CSE may return no `items` results; Gemini report must handle empty results (currently not explicitly handled).
  - If Gemini outputs quotes/newlines, query may be malformed or less effective.
  - Rate limiting / daily quota exceeded.

#### Node: Generate Chart Config
- **Type / role:** `Code` — generates a QuickChart URL from BP readings.
- **Configuration (logic summary):**
  - Reads `rawData` from **Calculate Averages** (`$('Calculate Averages').first().json.rawData`)
  - Builds `labels` as `M/D` from `date`
  - Builds datasets:
    - Systolic (red)
    - Diastolic (blue)
  - Produces:
    - `chartUrl` = `https://quickchart.io/chart?c=<encoded chartConfig JSON>`
- **Inputs/outputs:** Input ← Calculate Averages. Output → Download Chart Image.
- **Failure modes / edge cases:**
  - If `rawData` is empty, chart will have empty labels/data (still valid, but not informative).
  - If `date` parsing fails, labels become `NaN/NaN`.
  - If systolic/diastolic are strings, QuickChart will often coerce, but better to ensure numbers.

#### Node: Download Chart Image
- **Type / role:** `HTTP Request` — downloads the generated chart as a binary file.
- **Configuration (interpreted):**
  - URL: `={{ $json.chartUrl }}`
  - Response format: **File** (binary)
- **Inputs/outputs:** Input ← Generate Chart Config. Output → Merge Search & Chart (input index 1).
- **Failure modes / edge cases:**
  - QuickChart downtime or URL length issues (very large configs can exceed URL limits; here config is small).
  - File response stored as binary; later email node must explicitly attach it (see Block 5).

---

### Block 5 — Synthesis & Delivery
**Overview:** Combines recipe search results and chart output, asks Gemini to draft the Japanese weekly report and meal plan, and sends it via Gmail.  
**Nodes involved:**  
- Merge Search & Chart  
- Message a model2 (Gemini report)  
- Send Email Report  

#### Node: Merge Search & Chart
- **Type / role:** `Merge` — synchronizes parallel branches.
- **Configuration (interpreted):** Default merge behavior (JSON doesn’t specify mode). It receives:
  - Input 0: Google Custom Search results
  - Input 1: Chart image download result (binary)
- **Inputs/outputs:** Output → Gemini report.
- **Failure modes / edge cases:**
  - Merge mode matters: if it’s set to pair-by-index and one branch returns multiple items while the other returns one, results can be unexpected. Google Search often returns **one item** containing `items[]`, but depending on configuration it may also return multiple items.
  - If one branch fails, merge won’t emit combined output.

#### Node: Message a model2 (Gemini report)
- **Type / role:** `Google Gemini (LangChain)` — generates the final meal plan report in Japanese.
- **Configuration (interpreted):**
  - Model: `models/gemini-1.5-flash`
  - Prompt includes:
    - Full JSON summary from Calculate Averages
    - All JSON objects returned by Google Custom Search (`.all().map(i => i.json)`)
  - Task: Create:
    1) BP trend and advice  
    2) Mon–Fri 5 dinner menu items with URLs (must include URLs)  
    3) Weekly dietary points
  - `executeOnce: true` (node will run only once even with multiple incoming items, reducing duplication risk).
- **Key expressions/variables:**
  - `{{ JSON.stringify($('Calculate Averages').first().json) }}`
  - `{{ JSON.stringify($('Google Custom Search').all().map(i => i.json)) }}`
- **Inputs/outputs:** Input ← Merge. Output → Send Email Report.
- **Failure modes / edge cases:**
  - If Google Search returns no results, model may invent URLs unless constrained; prompt says “provided candidates” but does not forbid fabrication explicitly.
  - Token limits if Google Search JSON is large (usually fine with `num=5`).
  - Response structure must match the email node’s `{{$json.content.parts[0].text}}` expectation downstream.

#### Node: Send Email Report
- **Type / role:** `Gmail` — sends the report email.
- **Configuration (interpreted):**
  - To: placeholder recipient email
  - Subject: Prefixes alert if status is High:
    - `⚠️ High BP Alert: ` + “Weekly Meal Plan & Health Report”
  - Body message: `={{ $json.content.parts[0].text }}`
  - Email type: text
- **Inputs/outputs:** Input ← Gemini report. Final node (no outgoing).
- **Credentials:** `gmailOAuth2`
- **Failure modes / edge cases:**
  - Recipient placeholder not replaced.
  - Gmail OAuth scope issues, revoked tokens.
  - **Chart attachment not configured:** Although the chart is downloaded as a file earlier, this node as shown does not explicitly attach binary data. If you want the chart attached, you must map the binary property in Gmail node options (see reproduction section).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note Main | Sticky Note | Visual documentation header & setup notes |  |  | # AI Nutritionist: BP Meal Planner / Overview, Steps, Setup, Requirements (Google Gemini API, Google Custom Search API, Google Sheets & Gmail) |
| Sticky Note Collection | Sticky Note | Documents “Data Collection” block |  |  | ## 1. Data Collection Fetches weekly blood pressure data from Google Sheets and calculates the average status (High/Normal/Optimal). |
| Sticky Note Strategy | Sticky Note | Documents “AI Strategy” block |  |  | ## 2. AI Strategy **Gemini (1.5-flash)** analyzes the health status to decide the best search query (e.g., "Low sodium recipes"). |
| Sticky Note Execution | Sticky Note | Documents “Parallel Execution” block |  |  | ## 3. Parallel Execution Simultaneously searches for real recipes on Google and generates a BP trend chart via QuickChart.io. |
| Sticky Note Report | Sticky Note | Documents “Synthesis & Report” block |  |  | ## 4. Synthesis & Report **Gemini** creates a personalized meal plan based on the search results and sends an email with the chart attached. |
| Weekly Schedule (Mon 7AM) | Schedule Trigger | Weekly entry point |  | Workflow Configuration |  |
| Workflow Configuration | Set | Stores IDs/keys used downstream | Weekly Schedule (Mon 7AM) | Get Blood Pressure Data |  |
| Get Blood Pressure Data | Google Sheets | Fetch BP rows from Sheets | Workflow Configuration | Calculate Averages |  |
| Calculate Averages | Code | Filter last 7 days, compute averages/status | Get Blood Pressure Data | Message a model; Generate Chart Config |  |
| Message a model | Google Gemini (LangChain) | Decide Japanese search query | Calculate Averages | Google Custom Search |  |
| Google Custom Search | HTTP Request | Fetch recipe candidates via Google CSE | Message a model | Merge Search & Chart |  |
| Generate Chart Config | Code | Build QuickChart URL from BP history | Calculate Averages | Download Chart Image |  |
| Download Chart Image | HTTP Request | Download chart as binary file | Generate Chart Config | Merge Search & Chart |  |
| Merge Search & Chart | Merge | Combine search results + chart branch | Google Custom Search; Download Chart Image | Message a model2 |  |
| Message a model2 | Google Gemini (LangChain) | Write the Japanese weekly report + 5-day plan | Merge Search & Chart | Send Email Report |  |
| Send Email Report | Gmail | Email the final report | Message a model2 |  |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node: **Schedule Trigger**
   2. Configure: Weekly → Monday → 07:00 (confirm timezone)

2) **Add configuration holder**
   1. Add node: **Set** (name it “Workflow Configuration”)
   2. Add string fields:
      - `spreadsheetId`
      - `sheetName`
      - `googleSearchApiKey`
      - `googleSearchEngineId`
   3. Connect: **Schedule Trigger → Workflow Configuration**

3) **Read BP data from Google Sheets**
   1. Add node: **Google Sheets** (name it “Get Blood Pressure Data”)
   2. Authenticate with **Google Sheets OAuth2** credential
   3. Configure to read rows from your spreadsheet:
      - Spreadsheet ID: use `{{$('Workflow Configuration').first().json.spreadsheetId}}` (or select directly)
      - Sheet/tab name: `{{$('Workflow Configuration').first().json.sheetName}}`
      - Ensure columns exist: `date`, `systolic`, `diastolic`
   4. Connect: **Workflow Configuration → Get Blood Pressure Data**

4) **Compute weekly averages + status**
   1. Add node: **Code** (name it “Calculate Averages”)
   2. Paste logic equivalent to:
      - filter last 7 days by `date`
      - compute avg systolic/diastolic
      - set `status` High/Normal/Optimal
      - output `rawData`
   3. Connect: **Get Blood Pressure Data → Calculate Averages**

5) **Gemini decides the search query**
   1. Add node: **Google Gemini (LangChain) – Message a model** (name it “Message a model”)
   2. Credential: configure **Gemini/Google AI** credential (shown as `googlePalmApi` in JSON)
   3. Model: `gemini-1.5-flash`
   4. Prompt: instruct to output only one Japanese query string based on averages/status
   5. Connect: **Calculate Averages → Message a model**

6) **Google Custom Search request**
   1. Add node: **HTTP Request** (name it “Google Custom Search”)
   2. Method: GET
   3. URL: `https://www.googleapis.com/customsearch/v1`
   4. Enable “Send Query Parameters”
   5. Add query parameters:
      - `key` = `{{$('Workflow Configuration').first().json.googleSearchApiKey}}`
      - `cx` = `{{$('Workflow Configuration').first().json.googleSearchEngineId}}`
      - `q` = `{{$json.content.parts[0].text}} site:cookpad.com OR site:recipe.rakuten.co.jp`
      - `num` = `5`
   6. Connect: **Message a model → Google Custom Search**

7) **Generate QuickChart URL**
   1. Add node: **Code** (name it “Generate Chart Config”)
   2. Build a line chart config using `$('Calculate Averages').first().json.rawData`
   3. Output `chartUrl`
   4. Connect: **Calculate Averages → Generate Chart Config**

8) **Download chart image**
   1. Add node: **HTTP Request** (name it “Download Chart Image”)
   2. URL: `{{$json.chartUrl}}`
   3. Response: set to **File** (binary)
   4. Connect: **Generate Chart Config → Download Chart Image**

9) **Merge search + chart branches**
   1. Add node: **Merge** (name it “Merge Search & Chart”)
   2. Connect:
      - **Google Custom Search → Merge Search & Chart** (Input 1 / index 0)
      - **Download Chart Image → Merge Search & Chart** (Input 2 / index 1)
   3. Ensure merge mode fits your data shape (commonly “Wait for both” / “Combine” depending on your n8n version).

10) **Gemini creates the weekly report + meal plan**
   1. Add node: **Google Gemini (LangChain) – Message a model** (name it “Message a model2”)
   2. Model: `gemini-1.5-flash`
   3. Prompt in Japanese:
      - include `JSON.stringify` of Calculate Averages summary
      - include search results JSON
      - require: advice + 5 dinners with URLs + weekly points
   4. Turn on **Execute Once** (to avoid multiple outputs if merge produces multiple items)
   5. Connect: **Merge Search & Chart → Message a model2**

11) **Send email**
   1. Add node: **Gmail** (name it “Send Email Report”)
   2. Authenticate with **Gmail OAuth2**
   3. Configure:
      - To: your recipient email
      - Subject: expression using status from Calculate Averages (same logic as JSON)
      - Body: `{{$json.content.parts[0].text}}`
      - Email type: Text
   4. Connect: **Message a model2 → Send Email Report**
   5. **(Optional but recommended) Attach chart image:** In Gmail node options, add an attachment pointing to the binary property produced by “Download Chart Image” (the exact field name depends on your HTTP Request node binary property; commonly `data`). If not set, the chart will not be attached despite being downloaded.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sheets must contain columns: `date`, `systolic`, `diastolic`. | From “Sticky Note Main” setup section |
| Update IDs/Keys in “Workflow Configuration”. | From “Sticky Note Main” setup section |
| Set recipient in “Send Email Report”. | From “Sticky Note Main” setup section |
| Requirements: Google Gemini API, Google Custom Search API, Google Sheets & Gmail. | From “Sticky Note Main” requirements section |
| Workflow claims the email includes the chart attached; ensure Gmail node attachments are configured because downloading the chart alone does not attach it automatically. | Derived from node behavior vs sticky note statement |
| QuickChart endpoint used: `https://quickchart.io/chart?c=...` | Implemented in “Generate Chart Config” |