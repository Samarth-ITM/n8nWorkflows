Monitor product prices with Decodo, Google Sheets, Gemini and Gmail

https://n8nworkflows.xyz/workflows/monitor-product-prices-with-decodo--google-sheets--gemini-and-gmail-13377


# Monitor product prices with Decodo, Google Sheets, Gemini and Gmail

## 1. Workflow Overview

**Purpose:** Monitor product prices from a list of URLs stored in Google Sheets. For each URL, the workflow fetches the page via Decodo, extracts the HTML `<body>`, uses an AI Agent (Google Gemini) to identify the product name and current price, compares it against a “Desired price” from the sheet, and emails an alert when the current price is **less than or equal** to the desired price.

**Target use cases:**
- Price drop alerts for e-commerce products
- Monitoring multiple product pages on a schedule
- Converting raw web page HTML into structured data (name/price) with an LLM

### 1.1 Scheduling & Input Reception
Runs daily (or on the configured schedule), reads rows from a Google Sheet containing product URLs and desired price thresholds.

### 1.2 Iteration (Row-by-row processing)
Processes sheet rows one at a time to prevent mixing data between products.

### 1.3 Raw Content Retrieval & HTML Extraction
Uses Decodo to retrieve page content, reshapes the response for the HTML node, then extracts `<body>`.

### 1.4 AI Extraction & Parsing
Gemini-powered agent extracts “product name” and “price” from the HTML body; a Code node converts the agent’s text output into structured JSON fields.

### 1.5 Validation & Alerting + Loop Continuation
Compares extracted price to the sheet’s desired price; if condition passes, sends Gmail and continues to the next row; otherwise continues without alert.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Sheet Data Load
**Overview:** Triggers the workflow on a schedule and loads product monitoring rows from Google Sheets.

**Nodes involved:**
- **Daily Run**
- **Get row(s) in sheet**

#### Node: Daily Run
- **Type / role:** `Schedule Trigger` — entry point that starts the workflow automatically.
- **Config (interpreted):** Runs on an interval schedule (`rule.interval`). The JSON shows an interval array with an empty object; in n8n this typically means it was configured via UI and may default to “every day” depending on the editor state.
- **Outputs:** To **Get row(s) in sheet**.
- **Edge cases / failures:**
  - Misconfigured schedule interval (may not run as intended).
  - Timezone assumptions (instance timezone vs expected business timezone).

#### Node: Get row(s) in sheet
- **Type / role:** `Google Sheets` — fetches rows from the “Price monitoring” spreadsheet, Sheet1.
- **Config (interpreted):**
  - **Document:** “Price monitoring” (Google Spreadsheet ID `1QAMgjU-...`).
  - **Sheet:** Sheet1 (`gid=0`).
  - Operation implied by node name: fetch rows (no explicit filters visible).
  - `alwaysOutputData: true` so the workflow continues even if the node returns no rows (it will output an empty item set rather than stopping).
- **Key fields expected downstream:**
  - A product URL column (the sticky note says “product link”; exact column name is not shown in node params).
  - A numeric column **`Desired price`** (used later in the If node expression).
- **Connections:**
  - Output → **Loop Over Items**
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - OAuth token expired / insufficient permissions.
  - Sheet has different column headers (e.g., `Desired Price` vs `Desired price`) causing expression lookups to fail later.
  - Desired price stored as text with currency symbols/commas → numeric compare issues.

---

### Block 2 — Iteration Control (Batch/Loop)
**Overview:** Ensures each sheet row (each product) is processed independently in a loop.

**Nodes involved:**
- **Loop Over Items**

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — loops through input items.
- **Config (interpreted):**
  - Batch size not specified in parameters (defaults to n8n’s standard for this node, typically **1** unless changed in UI).
  - `alwaysOutputData: true` to keep flow control stable.
- **Connections:**
  - **Input:** from **Get row(s) in sheet**
  - **Output (index 1):** to **Decodo** (this is the “next batch / continue” path in this workflow’s wiring)
  - **Receives back edges:** from **If (false)** and from **Send a message** to continue looping.
- **Edge cases / failures:**
  - If the loop wiring is incorrect, it can stop after first item or create an infinite loop. Here, the design is: after each item, route back to Loop Over Items to get the next item.

---

### Block 3 — Raw Content Retrieval & HTML Body Extraction
**Overview:** Fetches the product page content and extracts the full HTML `<body>` so the AI has enough context.

**Nodes involved:**
- **Decodo**
- **Code in JavaScript1**
- **HTML**

#### Node: Decodo
- **Type / role:** `Decodo` (community node `@decodo/n8n-nodes-decodo.decodo`) — acts as the web data retrieval/handling bridge for each product URL.
- **Config (interpreted):**
  - Parameters are empty in JSON, implying defaults or UI-configured fields not captured here. Conceptually, it should take the current row’s product link as input and return fetched HTML.
- **Connections:**
  - Input: from **Loop Over Items**
  - Output: to **Code in JavaScript1**
- **Credentials:** none shown (may be required depending on Decodo setup; could be token-based configured in node UI).
- **Edge cases / failures:**
  - Target site blocks scraping (403/429, bot protection, CAPTCHA).
  - Timeout or large HTML payload.
  - Node returns a different schema than expected (`item.json.data` and `item.json.link` are assumed downstream).

#### Node: Code in JavaScript1
- **Type / role:** `Code` — transforms Decodo output into the schema expected by the HTML node.
- **Config (interpreted):**
  - For each item, it returns:
    - `json.html = item.json.data`
    - `json.link = item.json.link`
- **Key variables/expressions:**
  - Assumes Decodo output contains `data` with raw HTML and `link`.
- **Connections:**
  - Input: from **Decodo**
  - Output: to **HTML**
- **Edge cases / failures:**
  - If Decodo returns HTML under another key (e.g., `body`, `content`, `html`), `$json.data` will be undefined and HTML extraction will fail.
  - If `data` is not a string, HTML node extraction may error.

#### Node: HTML
- **Type / role:** `HTML` — parses the HTML string and extracts content using CSS selectors.
- **Config (interpreted):**
  - Operation: “Extract HTML Content”
  - Source property: `html` (expects input item has `json.html`)
  - Extraction rule: `body` via selector `body`
  - Outputs a field named `body`
- **Connections:**
  - Output: to **AI Agent**
- **Edge cases / failures:**
  - Invalid/empty HTML input leads to empty `body`.
  - Extremely large HTML could hit memory/time limits.

---

### Block 4 — AI Intelligence & Structured Parsing
**Overview:** Uses a Gemini-backed agent to extract product name and price from the HTML body, then parses the agent response into structured fields.

**Nodes involved:**
- **AI Agent**
- **Google Gemini Chat Model**
- **Code in JavaScript2**

#### Node: Google Gemini Chat Model
- **Type / role:** `LangChain Chat Model (Google Gemini)`
- **Config (interpreted):**
  - Provides the LLM backend for the AI Agent via the `ai_languageModel` connection.
  - Default options (no explicit temperature/max tokens shown).
- **Connections:**
  - `ai_languageModel` output → **AI Agent**
- **Credentials:** Google Gemini / PaLM API credentials.
- **Edge cases / failures:**
  - Invalid API key / project not enabled for Gemini.
  - Rate limits / quota exhaustion.
  - Model output variance (format changes) affecting regex parsing later.

#### Node: AI Agent
- **Type / role:** `LangChain Agent`
- **Config (interpreted):**
  - Prompt (define mode):  
    `extract the price and name of the product from this: | {{ $json.body }}`
  - Expects the HTML node to provide `$json.body` (extracted `<body>`).
- **Connections:**
  - Input: from **HTML**
  - Uses language model: **Google Gemini Chat Model**
  - Output: to **Code in JavaScript2**
- **Edge cases / failures:**
  - Prompt does not enforce a strict output schema; LLM may respond differently across pages.
  - If the page contains multiple prices (discounted + original), agent may choose wrong one.
  - Non-Rs currencies or alternate formats will break the downstream regex.

#### Node: Code in JavaScript2
- **Type / role:** `Code` — converts the agent’s natural language output into `name` and numeric `price`.
- **Config (interpreted):**
  - Reads `const text = $json.output;`
  - Regex extraction:
    - Name: `/Product Name:\*\*\s*(.*)/`
    - Price: `/Price:\*\*\s*Rs\.?([\d,\.]+)/`
  - Produces:
    - `json.name` (string or null)
    - `json.price` (number or null)
- **Connections:**
  - Output: to **If**
- **Edge cases / failures:**
  - If the AI Agent output doesn’t contain exactly `Product Name:**` and `Price:** Rs`, both matches return null.
  - Price parsing allows commas and dots, but assumes “Rs” currency marker.
  - If `price` becomes `null`, the If node numeric comparison may behave unexpectedly or fail strict validation.

---

### Block 5 — Price Validation, Email Alert, and Loop Continuation
**Overview:** Checks if current price is <= desired price; if yes, emails an alert; either way continues to next row.

**Nodes involved:**
- **If**
- **Send a message**

#### Node: If
- **Type / role:** `If` — branching logic for threshold comparison.
- **Config (interpreted):**
  - Condition: `{{$json.price}} <= {{$('Get row(s) in sheet').item.json['Desired price']}}`
  - Strict type validation enabled (`typeValidation: "strict"`)
- **Connections:**
  - True branch (output 0) → **Send a message**
  - False branch (output 1) → **Loop Over Items** (continue loop without alert)
- **Edge cases / failures:**
  - If `Desired price` is text or empty, strict numeric compare can fail.
  - Cross-node item referencing: `$('Get row(s) in sheet').item` relies on correct item pairing. In loops, this usually works if the current item is preserved; but mismatches can occur if item linking is lost due to transformations.
  - If `price` is null, comparison may be invalid.

#### Node: Send a message
- **Type / role:** `Gmail` — sends email notification.
- **Config (interpreted):**
  - To: `user@example.com`
  - Subject: `\desired price is equal` (note the leading backslash; likely accidental)
  - Message body: `Product price is now equal to your desired price: {{ ...Desired price... }}`
- **Connections:**
  - Output → **Loop Over Items** (continue processing remaining rows)
- **Credentials:** Gmail OAuth2.
- **Edge cases / failures:**
  - Gmail OAuth expired / missing scopes.
  - Sending limits, spam/reputation issues.
  - Hardcoded recipient; not dynamic per row.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Run | Schedule Trigger | Starts workflow on schedule | — | Get row(s) in sheet | Data Sourcing & Iteration: This section manages the workflow timing and retrieves the target product list from Google Sheets. The Loop Over Items node ensures each URL is processed individually to prevent data collisions. |
| Get row(s) in sheet | Google Sheets | Loads product URLs + desired price rows | Daily Run | Loop Over Items | Data Sourcing & Iteration: This section manages the workflow timing and retrieves the target product list from Google Sheets. The Loop Over Items node ensures each URL is processed individually to prevent data collisions. |
| Loop Over Items | Split In Batches | Iterates through sheet rows one-by-one | Get row(s) in sheet; If (false); Send a message | Decodo | Data Sourcing & Iteration: This section manages the workflow timing and retrieves the target product list from Google Sheets. The Loop Over Items node ensures each URL is processed individually to prevent data collisions. |
| Decodo | Decodo (community) | Fetches/bridges raw web content per URL | Loop Over Items | Code in JavaScript1 | Raw Content Extraction & Prep: The Decodo node processes the input, followed by a JavaScript node that cleans the data for the HTML node. We extract the full <body> here to ensure the AI has the complete context of the page. |
| Code in JavaScript1 | Code | Maps Decodo output into `{html: ...}` | Decodo | HTML | Raw Content Extraction & Prep: The Decodo node processes the input, followed by a JavaScript node that cleans the data for the HTML node. We extract the full <body> here to ensure the AI has the complete context of the page. |
| HTML | HTML | Extracts `<body>` from HTML | Code in JavaScript1 | AI Agent | Raw Content Extraction & Prep: The Decodo node processes the input, followed by a JavaScript node that cleans the data for the HTML node. We extract the full <body> here to ensure the AI has the complete context of the page. |
| Google Gemini Chat Model | LangChain Chat Model (Google Gemini) | Provides LLM to the agent | — | AI Agent (ai_languageModel) | AI Intelligence & Data Parsing: The AI Agent uses Google Gemini to analyze the HTML text. It is prompted to find and return the Product Name and Current Price, transforming web data into usable information. |
| AI Agent | LangChain Agent | Extracts name/price from page body | HTML | Code in JavaScript2 | AI Intelligence & Data Parsing: The AI Agent uses Google Gemini to analyze the HTML text. It is prompted to find and return the Product Name and Current Price, transforming web data into usable information. |
| Code in JavaScript2 | Code | Regex-parses agent output into numeric fields | AI Agent | If | Validation & Alerting Logic: A Regex script extracts the numerical price from the AI’s response. The If Node compares this to the "Desired Price." If the price matches, a Gmail alert is triggered; otherwise, the loop continues. |
| If | If | Compares current price vs desired price | Code in JavaScript2 | Send a message (true); Loop Over Items (false) | Validation & Alerting Logic: A Regex script extracts the numerical price from the AI’s response. The If Node compares this to the "Desired Price." If the price matches, a Gmail alert is triggered; otherwise, the loop continues. |
| Send a message | Gmail | Sends email alert | If (true) | Loop Over Items | Validation & Alerting Logic: A Regex script extracts the numerical price from the AI’s response. The If Node compares this to the "Desired Price." If the price matches, a Gmail alert is triggered; otherwise, the loop continues. |
| Sticky Note | Sticky Note | Documentation block (global) | — | — | How it works: This workflow monitors prices by pulling URLs from a Google Sheet and processing them in a loop. It uses the Decodo node for data handling. JavaScript formats data for the HTML node. An AI Agent, powered by Google Gemini, analyzes the full page content to extract the product name and price. The workflow compares the current price against the "Desired Price" from the sheet and sends a Gmail notification if a match or price drop is found. Setup Steps: (1) Google Sheets columns for product link and "Desired price." (2) Configure Decodo node. (3) Enable Gemini credentials. (4) If node uses <= comparison. (5) Authorize Gmail. |
| Sticky Note1 | Sticky Note | Documentation block | — | — | Data Sourcing & Iteration (content as above). |
| Sticky Note2 | Sticky Note | Documentation block | — | — | Raw Content Extraction & Prep (content as above). |
| Sticky Note3 | Sticky Note | Documentation block | — | — | AI Intelligence & Data Parsing (content as above). |
| Sticky Note4 | Sticky Note | Documentation block | — | — | Validation & Alerting Logic (content as above). |

---

## 4. Reproducing the Workflow from Scratch

1. **Create trigger**
   1) Add node **Schedule Trigger** named **Daily Run**.  
   2) Configure it to run daily (choose the interval/time you want).

2. **Load product rows from Google Sheets**
   1) Add node **Google Sheets** named **Get row(s) in sheet**.  
   2) Authenticate with **Google Sheets OAuth2**.  
   3) Select spreadsheet **“Price monitoring”** and sheet **Sheet1**.  
   4) Configure to “get all rows” (or equivalent “read” operation).  
   5) Ensure your sheet has at least:
      - a URL column (e.g., `link` or `url`)
      - a numeric column named exactly **`Desired price`** (matching case and spacing).
   6) Connect **Daily Run → Get row(s) in sheet**.

3. **Loop over each row**
   1) Add node **Split In Batches** named **Loop Over Items**.  
   2) Set batch size to **1** (recommended for per-URL isolation).  
   3) Connect **Get row(s) in sheet → Loop Over Items**.

4. **Fetch page content with Decodo**
   1) Add node **Decodo** named **Decodo**.  
   2) Configure it to fetch the product URL from the current item (exact mapping depends on Decodo node fields). Typically you will map something like:
      - URL = `{{$json.link}}` or `{{$json.url}}` depending on your sheet column name.
   3) Connect **Loop Over Items → Decodo**.

5. **Transform Decodo output for HTML parsing**
   1) Add node **Code** named **Code in JavaScript1** with:
      - Map HTML into `json.html` from the Decodo response field that contains raw HTML (the provided workflow assumes `item.json.data`).
   2) Connect **Decodo → Code in JavaScript1**.

6. **Extract `<body>` with HTML node**
   1) Add node **HTML** named **HTML**.  
   2) Set operation to extract content from property **`html`**.  
   3) Add an extraction value:
      - Key: `body`
      - CSS selector: `body`
   4) Connect **Code in JavaScript1 → HTML**.

7. **Configure Gemini model and AI Agent**
   1) Add node **Google Gemini Chat Model**. Authenticate with **Google Gemini(PaLM) API** credentials.  
   2) Add node **AI Agent**.
      - Prompt text: `extract the price and name of the product from this:| {{ $json.body }}`
   3) Connect the model to the agent using the AI connection:
      - **Google Gemini Chat Model → AI Agent** via **ai_languageModel**
   4) Connect **HTML → AI Agent**.

8. **Parse AI output into structured fields**
   1) Add node **Code** named **Code in JavaScript2**.
   2) Implement regex extraction from `"$json.output"` and output `{ name, price }` as in the workflow.
   3) Connect **AI Agent → Code in JavaScript2**.

9. **Compare against desired price**
   1) Add node **If** named **If**.  
   2) Condition: **Number lte (<=)**  
      - Left: `{{$json.price}}`  
      - Right: `{{$('Get row(s) in sheet').item.json['Desired price']}}`
   3) Connect **Code in JavaScript2 → If**.

10. **Send Gmail alert**
   1) Add node **Gmail** named **Send a message**.  
   2) Authenticate with **Gmail OAuth2**.  
   3) Configure:
      - To: your recipient (or map from sheet)
      - Subject: choose a clear subject (remove the accidental leading `\` if desired)
      - Message: include desired price and optionally the extracted name/price.
   4) Connect **If (true) → Send a message**.

11. **Continue looping**
   1) Connect **If (false) → Loop Over Items**.  
   2) Connect **Send a message → Loop Over Items** (so it continues after sending).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow monitors prices by pulling URLs from a Google Sheet and processing them in a loop. It uses Decodo for data handling, formats content for HTML extraction, uses a Gemini-powered agent to extract product name/price, compares against “Desired Price,” and sends Gmail alerts. | From sticky note “How it works” |
| Sheet requirements: columns for product link and “Desired price.” Ensure exact header match for expressions. | Setup guidance from sticky notes |
| The parsing logic depends on the AI response format (expects “Product Name:** …” and “Price:** Rs…”). Consider enforcing a strict output format (e.g., JSON) to reduce regex failures. | Integration reliability note (implied by current design) |