Track competitor prices and email AI insights with Google Sheets, Groq and Gmail

https://n8nworkflows.xyz/workflows/track-competitor-prices-and-email-ai-insights-with-google-sheets--groq-and-gmail-14311


# Track competitor prices and email AI insights with Google Sheets, Groq and Gmail

# 1. Workflow Overview

This workflow monitors competitor product pages listed in Google Sheets, extracts live prices and promotional offers using Groq-based AI, compares them against historical values, generates a market strategy summary, emails that summary through Gmail, and finally rolls current values into historical columns for the next run.

Its primary use cases are:
- Daily competitor price tracking
- Detecting price drops or offer changes
- Producing AI-generated strategic pricing insights
- Sending internal market intelligence emails
- Maintaining a rolling history in Google Sheets

## 1.1 Input Reception and Source Data Load
The workflow starts on a schedule and reads all competitor records from a Google Sheet. Each row is expected to contain at least a competitor name and product URL, plus historical/current pricing columns.

## 1.2 Per-Competitor Scraping and Content Cleaning
Each sheet row is processed one by one. The workflow fetches the competitor product page through ScraperAPI and strips scripts, styles, tags, and excess whitespace to reduce token usage before AI extraction.

## 1.3 AI Price and Offer Extraction
A Groq-powered AI agent receives cleaned page text and is prompted to extract:
- product name
- current live selling price
- offer summary

The workflow then parses the AI response into structured JSON.

## 1.4 Change Detection and Sheet Update
The extracted current values are checked against the row’s historical columns:
- If this is the first run for that competitor, the workflow initializes historical values.
- If current values differ from historical values, it writes current price/offer into the “Current” columns.
- If no change is detected, it still updates current values.

## 1.5 Market Aggregation and AI Strategy Analysis
After item iteration, the workflow reloads the sheet, aggregates all competitor rows into a market intelligence text block, and asks another Groq AI agent to produce a strategic summary.

## 1.6 Reporting and Reset
The summary is saved back to Google Sheets, converted into HTML email content, sent with Gmail, and then the workflow reloads the sheet again to:
- copy current values into historical columns
- clear current columns

This prepares the data model for the next scheduled run.

---

# 2. Block-by-Block Analysis

## Block 1: Scheduled Start and Initial Google Sheets Load

### Overview
This block starts the workflow on a daily schedule and retrieves the competitor tracking dataset from Google Sheets. It establishes the base row set used throughout the rest of the workflow.

### Nodes Involved
- Schedule Trigger
- Get row(s) in sheet

### Node Details

#### Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; entry-point node that launches the workflow automatically.
- **Configuration choices:** Configured with an interval rule triggering at hour `9`. The exact timezone depends on the workflow/server timezone unless explicitly set in n8n settings.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `Get row(s) in sheet`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or failure types:**
  - Unexpected execution time if instance timezone differs from business timezone
  - Workflow inactive or instance offline at trigger time
- **Sub-workflow reference:** None.

#### Get row(s) in sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; reads the spreadsheet rows that define competitors and tracked products.
- **Configuration choices:** Uses a Google Sheets document identified as `YOUR_GOOGLE_SHEET_ID`, sheet `gid=0`. No additional filters are configured, so it appears intended to retrieve all rows.
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:** Input from `Schedule Trigger`; output to `Loop Over Items`.
- **Version-specific requirements:** Type version `4.7`; requires valid Google Sheets credentials.
- **Edge cases or failure types:**
  - Authentication or permission errors
  - Spreadsheet ID/sheet mismatch
  - Empty sheet causing downstream no-op behavior
  - Header/schema mismatch with downstream references such as `Competitor Name`, `Product URL`, `Last Known Price`, `Last Known Offer`
- **Sub-workflow reference:** None.

---

## Block 2: Iteration, Web Scraping, and Content Cleanup

### Overview
This block processes each competitor row individually, fetches the live product page HTML via ScraperAPI, and converts the page into compact plain text suitable for AI extraction.

### Nodes Involved
- Loop Over Items
- HTTP Request3
- Clean Content

### Node Details

#### Loop Over Items
- **Type and role:** `n8n-nodes-base.splitInBatches`; iterates through rows one item at a time.
- **Configuration choices:** Uses default options with no custom batch size shown, effectively handling sequential item looping.
- **Key expressions or variables used:** Downstream nodes refer back to current loop item via expressions such as `$('Loop Over Items').item.json[...]`.
- **Input and output connections:** Input from `Get row(s) in sheet`, and loops back from `If2` and `If1`. Outputs to both `Get row(s) in sheet1` and `HTTP Request3`.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or failure types:**
  - If no rows are provided, downstream branches may not execute
  - Loop control can be confusing in reproduction; incorrect reconnection may break iteration or cause early aggregation
- **Sub-workflow reference:** None.

#### HTTP Request3
- **Type and role:** `n8n-nodes-base.httpRequest`; fetches live product page HTML through ScraperAPI.
- **Configuration choices:** Uses a URL of the form:
  `https://api.scraperapi.com?api_key=YOUR_SCRAPERAPI_KEY&url={{ encodeURIComponent($json['Product URL'])}}`
  This wraps the sheet’s `Product URL` inside a ScraperAPI request.
- **Key expressions or variables used:**
  - `{{$json['Product URL']}}`
  - `{{ encodeURIComponent(...) }}`
- **Input and output connections:** Input from `Loop Over Items`; output to `Clean Content`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or failure types:**
  - Invalid or missing ScraperAPI key
  - Invalid or empty `Product URL`
  - Target site blocking or returning anti-bot pages
  - Timeout or non-HTML responses
  - Large page payloads
- **Sub-workflow reference:** None.

#### Clean Content
- **Type and role:** `n8n-nodes-base.code`; sanitizes HTML and truncates text for token efficiency.
- **Configuration choices:** JavaScript removes:
  - `<script>` blocks
  - `<style>` blocks
  - all remaining HTML tags
  - repeated whitespace
  It then returns the first 15,000 characters as `cleaned_content`.
- **Key expressions or variables used:**
  - Reads from `$input.first().json.data`
  - Returns `{ cleaned_content: ... }`
- **Input and output connections:** Input from `HTTP Request3`; output to `AI Agent1`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - If HTTP response body is not in `json.data`, extraction will fail semantically
  - If `data` is missing, returns `"No data found"`
  - Important pricing text may be truncated if page content is long
  - Removing HTML may collapse structured pricing info into ambiguous plain text
- **Sub-workflow reference:** None.

---

## Block 3: AI Extraction of Current Price and Offers

### Overview
This block uses a Groq chat model through an AI Agent to extract structured product, pricing, and promotional information from cleaned page text, then parses the output into usable JSON fields.

### Nodes Involved
- AI Agent1
- Groq Chat Model1
- current Price and offer

### Node Details

#### AI Agent1
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; LLM orchestration node for extraction.
- **Configuration choices:** Prompt instructs the AI to act as a “Global Retail Data Scientist” and extract:
  - `product_name`
  - `current_price`
  - `current_offer`
  
  The prompt contains specialized logic:
  - choose the “middle” price in common e-commerce price clusters
  - exclude “Buy at”, “with Bank offer”, “Effective price”, “Special price”, and “Exchange offer”
  - return only valid JSON
  - no Markdown fences
- **Key expressions or variables used:**
  - `{{ $json.cleaned_content }}`
- **Input and output connections:** Main input from `Clean Content`; AI language model input from `Groq Chat Model1`; output to `current Price and offer`.
- **Version-specific requirements:** Type version `3`; requires n8n LangChain-compatible setup.
- **Edge cases or failure types:**
  - Model may still output malformed JSON
  - Truncated page content may omit real price
  - Product pages with dynamic JS-loaded prices may not expose the desired number in raw HTML
  - Multi-variant pages may confuse price selection
- **Sub-workflow reference:** None.

#### Groq Chat Model1
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGroq`; language model provider for extraction.
- **Configuration choices:** Uses model `llama-3.3-70b-versatile`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_languageModel` to `AI Agent1`.
- **Version-specific requirements:** Type version `1`; requires Groq API credentials.
- **Edge cases or failure types:**
  - Invalid API key
  - Rate limits or request throttling
  - Model availability changes
  - Token/context limits on very large inputs
- **Sub-workflow reference:** None.

#### current Price and offer
- **Type and role:** `n8n-nodes-base.code`; parses model output into structured fields.
- **Configuration choices:** Reads `rawText = $json.output`, strips optional ```json fences, attempts `JSON.parse`, and returns:
  - `product_name`
  - `current_price`
  - `current_offer`
  
  On parse failure, returns:
  - `error`
  - `raw_output`
- **Key expressions or variables used:**
  - `$json.output`
- **Input and output connections:** Input from `AI Agent1`; output to `If2`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - If the AI returns explanatory text instead of JSON, parse fails
  - If `current_price` is returned as formatted currency string rather than pure digits, downstream numeric comparisons may behave unexpectedly
  - Error branch is not separately handled later, so parse failures may silently propagate into updates
- **Sub-workflow reference:** None.

---

## Block 4: Initialization and Change Detection

### Overview
This block decides whether a competitor row is being processed for the first time or already has historical data. It then checks whether price or offer changed since the last stored snapshot.

### Nodes Involved
- If2
- First time price and offer added
- If1
- Updated current price and offer in sheet
- If No changes then update

### Node Details

#### If2
- **Type and role:** `n8n-nodes-base.if`; detects first-run initialization case.
- **Configuration choices:** Uses AND logic:
  - `Last Known Price` is empty
  - `Last Known Offer` is empty
- **Key expressions or variables used:**
  - `{{ $('Get row(s) in sheet').item.json['Last Known Price'] }}`
  - `{{ $('Get row(s) in sheet').item.json['Last Known Offer'] }}`
- **Input and output connections:** Input from `current Price and offer`.
  - True branch → `First time price and offer added`
  - False branch → `If1`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or failure types:**
  - Empty detection depends on exact cell content; whitespace or unexpected types may cause false negatives
  - References `Get row(s) in sheet` instead of current loop item more generally; item linking must remain intact
- **Sub-workflow reference:** None.

#### First time price and offer added
- **Type and role:** `n8n-nodes-base.googleSheets`; initializes historical columns on first run.
- **Configuration choices:** Updates row matched by `Competitor Name`, setting:
  - `Competitor Name` = current loop item competitor
  - `Last Known Price` = extracted `current_price`
  - `Last Known Offer` = extracted `current_offer`
- **Key expressions or variables used:**
  - `{{ $('Loop Over Items').item.json['Competitor Name'] }}`
  - `{{ $json.current_price }}`
  - `{{ $json.current_offer }}`
- **Input and output connections:** Input from `If2` true branch; output loops back to `Loop Over Items`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failure types:**
  - Matching by `Competitor Name` assumes uniqueness
  - If competitor names are duplicated or inconsistently spelled, wrong rows may be updated
  - If extraction fields are missing, blank history may be written
- **Sub-workflow reference:** None.

#### If1
- **Type and role:** `n8n-nodes-base.if`; checks whether live values differ from historical values.
- **Configuration choices:** Uses OR logic:
  - historical `Last Known Price` != `current_price`
  - historical `Last Known Offer` != `current_offer`
- **Key expressions or variables used:**
  - `{{ $('Get row(s) in sheet').item.json['Last Known Price'] }}`
  - `{{ $json.current_price }}`
  - `{{ $('Get row(s) in sheet').item.json['Last Known Offer'] }}`
  - `{{ $json.current_offer }}`
- **Input and output connections:** Input from `If2` false branch.
  - True branch → `Updated current price and offer in sheet`
  - False branch → `If No changes then update`
  Both branches also loop back to `Loop Over Items`.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or failure types:**
  - Numeric comparison may fail if price is stored as text with symbols/commas
  - Offer text comparisons are exact string comparisons; small formatting changes can count as a change
- **Sub-workflow reference:** None.

#### Updated current price and offer in sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; stores detected changed values in current columns.
- **Configuration choices:** Updates row matched by `Competitor Name`, setting:
  - `Current Price`
  - `Current Offer`
  - `Competitor Name`
- **Key expressions or variables used:**
  - `{{ $json.current_price }}`
  - `{{ $json.current_offer }}`
  - `{{ $('Loop Over Items').item.json['Competitor Name'] }}`
- **Input and output connections:** Input from `If1` true branch; output back to `Loop Over Items`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failure types:**
  - Duplicate competitor names can overwrite incorrect rows
  - Missing extracted values may replace valid current data with blanks
- **Sub-workflow reference:** None.

#### If No changes then update
- **Type and role:** `n8n-nodes-base.googleSheets`; intended to update current columns even when no difference exists.
- **Configuration choices:** Updates:
  - `Current Offer` = extracted `current_offer`
  - `Current Price` = extracted `current_price`
  - `Competitor Name` = `"="`
  
  This `Competitor Name` mapping appears misconfigured.
- **Key expressions or variables used:**
  - `{{ $json.current_offer }}`
  - `{{ $json.current_price }}`
  - competitor field is set to literal `=`
- **Input and output connections:** Input from `If1` false branch; output back to `Loop Over Items`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failure types:**
  - Very likely update failure because the matching column is `Competitor Name` but the assigned value is malformed
  - Could target no row or wrong row depending on n8n evaluation
  - This is the most obvious functional defect in the workflow JSON
- **Sub-workflow reference:** None.

---

## Block 5: Post-Loop Reload and Market Data Aggregation

### Overview
After per-item processing, this block reloads the full sheet and compiles all rows into a structured market intelligence text report, including historical/current prices, offer evolution, and computed price deltas.

### Nodes Involved
- Get row(s) in sheet1
- Data Aggregator

### Node Details

#### Get row(s) in sheet1
- **Type and role:** `n8n-nodes-base.googleSheets`; reloads the spreadsheet after updates so downstream analysis uses the latest sheet state.
- **Configuration choices:** Same document and sheet as the initial read; retrieves all rows.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Loop Over Items`; output to `Data Aggregator`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failure types:**
  - Because this node is connected directly from `Loop Over Items`, aggregation timing depends on loop completion behavior
  - If loop wiring is reproduced incorrectly, it may run too early or multiple times
- **Sub-workflow reference:** None.

#### Data Aggregator
- **Type and role:** `n8n-nodes-base.code`; builds a consolidated text report from all competitor rows.
- **Configuration choices:** For each item, it:
  - reads competitor and product fields
  - parses previous and current prices
  - calculates absolute and percentage change
  - labels movement as stable, price drop, or price increase
  - adds offer history and a strategic gap statement
  
  It returns:
  - `aggregated_market_data`
  - `product_name`
- **Key expressions or variables used:**
  - `$input.all()`
  - `items[0].json["Product Name"]`
  - row fields such as `Competitor Name`, `Last Known Price`, `Current Price`, `Last Known Offer`, `Current Offer`
- **Input and output connections:** Input from `Get row(s) in sheet1`; output to `AI Agent`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - If `Current Price` is empty for many rows, calculations may default to `0`
  - If first item lacks `Product Name`, fallback becomes `"Target Product"`
  - Multi-product sheets are not handled separately; all rows are treated as one report
- **Sub-workflow reference:** None.

---

## Block 6: AI Strategy Summary Generation

### Overview
This block asks a second Groq-powered AI agent to interpret the aggregated market data and produce a concise business-oriented strategy summary.

### Nodes Involved
- AI Agent
- Groq Chat Model
- Simple Memory

### Node Details

#### AI Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; generates business analysis from aggregated market text.
- **Configuration choices:** Prompt defines the role as “Retail Strategist” and requests:
  - Summary of price leaders and drops
  - Trend classification: stable or discounting
  - Strategy comparison: price matching vs value-add
  - Alert threshold and recommended action
  
  It also forbids Markdown.
- **Key expressions or variables used:**
  - `{{ $json.product_name }}`
  - `{{ $json.aggregated_market_data }}`
- **Input and output connections:** Main input from `Data Aggregator`; AI language model input from `Groq Chat Model`; AI memory input from `Simple Memory`; output to `Update row in sheet`.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or failure types:**
  - Summary quality depends heavily on aggregation quality
  - If aggregated data includes zeroed values, the strategic output may be misleading
  - `executeOnce` is enabled, so this block should run once per workflow execution
- **Sub-workflow reference:** None.

#### Groq Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGroq`; language model backing the strategic analysis.
- **Configuration choices:** Uses `llama-3.3-70b-versatile`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_languageModel` to `AI Agent`.
- **Version-specific requirements:** Type version `1`; requires Groq credentials.
- **Edge cases or failure types:**
  - API/authentication errors
  - Rate limits
  - Latency/timeouts with long prompts
- **Sub-workflow reference:** None.

#### Simple Memory
- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; memory store attached to the analysis agent.
- **Configuration choices:** Uses:
  - `sessionKey = history`
  - `sessionIdType = customKey`
  
  However, no explicit custom session ID value is visible in the JSON.
- **Key expressions or variables used:** None shown.
- **Input and output connections:** Connected as `ai_memory` to `AI Agent`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or failure types:**
  - If session ID is not properly set by node defaults, memory behavior may be inconsistent
  - Memory is not essential for a single-run strategic summary and may be unnecessary
- **Sub-workflow reference:** None.

---

## Block 7: Save Summary, Format Email, and Send Gmail Report

### Overview
This block stores the generated strategy summary in Google Sheets, formats it as HTML, and emails it to a recipient.

### Nodes Involved
- Update row in sheet
- Edit Fields1
- Send a message

### Node Details

#### Update row in sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; writes the AI-generated strategic summary into the sheet.
- **Configuration choices:** Updates:
  - `Summary` = AI output
  - `Competitor Name` = value from `Get row(s) in sheet`
  
  Matching is done by `Competitor Name`.
- **Key expressions or variables used:**
  - `{{ $json.output }}`
  - `{{ $('Get row(s) in sheet').item.json['Competitor Name'] }}`
- **Input and output connections:** Input from `AI Agent`; output to `Edit Fields1`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failure types:**
  - The AI summary is global, but the row match is by one competitor name from the initial sheet context; this likely writes the same summary to only one matched row rather than all rows
  - Ambiguous item-linking may produce non-deterministic target row behavior
- **Sub-workflow reference:** None.

#### Edit Fields1
- **Type and role:** `n8n-nodes-base.set`; creates an HTML-formatted email body from the sheet summary.
- **Configuration choices:** Assigns `formatted_summary` as a styled HTML block containing:
  - title
  - summary content via `{{ $json.Summary }}`
  - report date/time formatted in `Asia/Kolkata`
- **Key expressions or variables used:**
  - `{{ $json.Summary }}`
  - `{{ $now.setZone('Asia/Kolkata').toFormat('dd MMM yyyy') }}`
  - `{{ $now.setZone('Asia/Kolkata').toFormat('hh:mm a') }}`
- **Input and output connections:** Input from `Update row in sheet`; output to `Send a message`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or failure types:**
  - If incoming item lacks `Summary` field, email body will be empty
  - HTML rendering varies slightly across mail clients
- **Sub-workflow reference:** None.

#### Send a message
- **Type and role:** `n8n-nodes-base.gmail`; sends the final report email.
- **Configuration choices:**
  - Recipient: `user@example.com`
  - Subject: `Market Strategy Report: {{ $('Data Aggregator').item.json.product_name }}`
  - HTML message body includes `{{ $json.formatted_summary }}`
- **Key expressions or variables used:**
  - `{{ $json.formatted_summary }}`
  - `{{ $('Data Aggregator').item.json.product_name }}`
- **Input and output connections:** Input from `Edit Fields1`; output to `Get row(s) in sheet2`.
- **Version-specific requirements:** Type version `2.2`; requires Gmail OAuth2 credentials.
- **Edge cases or failure types:**
  - Gmail auth or token expiration
  - Sending limits or blocked account
  - Hardcoded recipient requiring manual replacement
  - If product name is null, subject becomes incomplete
- **Sub-workflow reference:** None.

---

## Block 8: Reset Current Values into History

### Overview
This block reloads the sheet after reporting and prepares it for the next cycle by moving current values into historical columns and clearing current fields.

### Nodes Involved
- Get row(s) in sheet2
- Update row in sheet1

### Node Details

#### Get row(s) in sheet2
- **Type and role:** `n8n-nodes-base.googleSheets`; reloads all rows before the reset/update pass.
- **Configuration choices:** Same spreadsheet and sheet as previous Google Sheets nodes.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Send a message`; output to `Update row in sheet1`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failure types:**
  - If email sending fails, reset will not occur
  - Any stale or partially updated data from previous steps will be carried into history
- **Sub-workflow reference:** None.

#### Update row in sheet1
- **Type and role:** `n8n-nodes-base.googleSheets`; archives current values into historical columns and clears current columns.
- **Configuration choices:** Updates row matched by `Competitor Name`, setting:
  - `Competitor Name` = current row competitor
  - `Last Known Offer` = current row `Current Offer`
  - `Last Known Price` = current row `Current Price`
  - `Current Offer` = `null`
  - `Current Price` = `null`
- **Key expressions or variables used:**
  - `{{ $json["Competitor Name"] }}`
  - `{{ $json["Current Offer"] }}`
  - `{{ $json["Current Price"] }}`
  - `{{ null }}`
- **Input and output connections:** Input from `Get row(s) in sheet2`; no downstream node.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failure types:**
  - If `Current Price`/`Current Offer` were never written, this can overwrite historical data with empty values
  - Null handling may differ depending on Google Sheets node behavior/version
  - Multi-row updates rely again on `Competitor Name` uniqueness
- **Sub-workflow reference:** None.

---

## Block 9: Documentation Sticky Notes

### Overview
These nodes are visual documentation elements inside the workflow canvas. They do not execute but are important context for users rebuilding or maintaining the workflow.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; overall workflow description and setup instructions.
- **Configuration choices:** Contains operational description, setup steps, and customization ideas.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None; documentation only.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and role:** sticky note for Step 1.
- **Configuration choices:** Labels Google Sheets sync stage.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.

#### Sticky Note2
- **Type and role:** sticky note for Step 2.
- **Configuration choices:** Labels scraping stage.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.

#### Sticky Note3
- **Type and role:** sticky note for Step 3.
- **Configuration choices:** Labels AI extraction stage.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.

#### Sticky Note4
- **Type and role:** sticky note for Step 4.
- **Configuration choices:** Labels analysis stage.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.

#### Sticky Note5
- **Type and role:** sticky note for Step 5.
- **Configuration choices:** Labels reporting stage.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.

#### Sticky Note6
- **Type and role:** sticky note for Step 6.
- **Configuration choices:** Labels reset stage.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts the workflow daily |  | Get row(s) in sheet | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 1: Database Sync<br>Fetch competitor names and product URLs from Google Sheets. |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Loads source competitor rows from Google Sheets | Schedule Trigger | Loop Over Items | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 1: Database Sync<br>Fetch competitor names and product URLs from Google Sheets. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterates through competitor rows | Get row(s) in sheet, First time price and offer added, Updated current price and offer in sheet, If No changes then update | Get row(s) in sheet1, HTTP Request3 | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 2: Scraping<br>Pull raw product page data using ScraperAPI. |
| HTTP Request3 | n8n-nodes-base.httpRequest | Scrapes competitor product page through ScraperAPI | Loop Over Items | Clean Content | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 2: Scraping<br>Pull raw product page data using ScraperAPI. |
| Clean Content | n8n-nodes-base.code | Removes HTML noise and truncates content | HTTP Request3 | AI Agent1 | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 2: Scraping<br>Pull raw product page data using ScraperAPI.<br>## Step 3: Price Extraction<br>AI extracts the real selling price and offer details. |
| AI Agent1 | @n8n/n8n-nodes-langchain.agent | Extracts product name, live price, and offers from cleaned text | Clean Content | current Price and offer | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 3: Price Extraction<br>AI extracts the real selling price and offer details. |
| Groq Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGroq | Provides LLM for extraction agent |  | AI Agent1 | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 3: Price Extraction<br>AI extracts the real selling price and offer details. |
| current Price and offer | n8n-nodes-base.code | Parses AI JSON output into structured fields | AI Agent1 | If2 | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 3: Price Extraction<br>AI extracts the real selling price and offer details. |
| If2 | n8n-nodes-base.if | Detects whether historical price/offer fields are empty | current Price and offer | First time price and offer added, If1 | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 3: Price Extraction<br>AI extracts the real selling price and offer details. |
| First time price and offer added | n8n-nodes-base.googleSheets | Initializes historical price and offer fields on first run | If2 | Loop Over Items | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 3: Price Extraction<br>AI extracts the real selling price and offer details. |
| If1 | n8n-nodes-base.if | Compares historical vs current values to detect changes | If2 | Updated current price and offer in sheet, If No changes then update | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 3: Price Extraction<br>AI extracts the real selling price and offer details. |
| Updated current price and offer in sheet | n8n-nodes-base.googleSheets | Writes changed live price and offer to current columns | If1 | Loop Over Items | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 3: Price Extraction<br>AI extracts the real selling price and offer details. |
| If No changes then update | n8n-nodes-base.googleSheets | Updates current columns even when no change is detected | If1 | Loop Over Items | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 3: Price Extraction<br>AI extracts the real selling price and offer details. |
| Get row(s) in sheet1 | n8n-nodes-base.googleSheets | Reloads updated sheet for market-wide analysis | Loop Over Items | Data Aggregator | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 4: Analysis<br>Compare current vs historical data to detect changes. |
| Data Aggregator | n8n-nodes-base.code | Builds a consolidated market intelligence text block | Get row(s) in sheet1 | AI Agent | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 4: Analysis<br>Compare current vs historical data to detect changes. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generates strategic summary from aggregated market data | Data Aggregator | Update row in sheet | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 4: Analysis<br>Compare current vs historical data to detect changes. |
| Groq Chat Model | @n8n/n8n-nodes-langchain.lmChatGroq | Provides LLM for strategy analysis |  | AI Agent | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 4: Analysis<br>Compare current vs historical data to detect changes. |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Supplies conversational memory to the analysis agent |  | AI Agent | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 4: Analysis<br>Compare current vs historical data to detect changes. |
| Update row in sheet | n8n-nodes-base.googleSheets | Saves AI strategy summary into the sheet | AI Agent | Edit Fields1 | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 4: Analysis<br>Compare current vs historical data to detect changes. |
| Edit Fields1 | n8n-nodes-base.set | Formats summary as HTML email content | Update row in sheet | Send a message | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 5: Reporting<br>Generate and email the market intelligence report. |
| Send a message | n8n-nodes-base.gmail | Emails the formatted market intelligence report | Edit Fields1 | Get row(s) in sheet2 | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 5: Reporting<br>Generate and email the market intelligence report. |
| Get row(s) in sheet2 | n8n-nodes-base.googleSheets | Reloads rows before rolling current values into history | Send a message | Update row in sheet1 | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 6: Reset<br>Move current data to history and clear fields for next run. |
| Update row in sheet1 | n8n-nodes-base.googleSheets | Copies current values to historical fields and clears current fields | Get row(s) in sheet2 |  | # Daily Competitor Price & Offer Monitor<br>### How it works<br>This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report.<br>The system identifies price drops, offer changes, and competitive positioning shifts. It then summarizes market trends and recommends pricing actions, helping you respond quickly to competitor movements.<br>### Setup steps<br>1. Connect your Google Sheets account and add product URLs with competitor names.<br>2. Add your ScraperAPI key in the HTTP Request node.<br>3. Connect your AI provider (Groq, OpenAI, or Gemini).<br>4. Configure the Gmail node to send reports.<br>5. Set the Schedule Trigger to your preferred time.<br>### Customization tips<br>- Adjust AI prompts to refine pricing logic or strategy output.<br>- Modify schedule frequency for faster monitoring.<br>- Add more competitors directly in Google Sheets.<br>## Step 6: Reset<br>Move current data to history and clear fields for next run. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for Step 1 |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for Step 2 |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for Step 3 |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation for Step 4 |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation for Step 5 |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation for Step 6 |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   `Track competitor prices and email AI insights with Google Sheets, Groq and Gmail`.

2. **Prepare the Google Sheet** with at least these columns:
   - `Competitor Name`
   - `Product Name`
   - `Product URL`
   - `Last Known Price`
   - `Last Known Offer`
   - `Current Price`
   - `Current Offer`
   - `Summary`

3. **Add a Schedule Trigger node**.
   - Type: `Schedule Trigger`
   - Set it to run daily at hour `9`
   - Adjust instance/workflow timezone if needed

4. **Add the first Google Sheets node** named `Get row(s) in sheet`.
   - Operation: read/get rows
   - Select your spreadsheet
   - Select the main sheet
   - Use Google Sheets credentials with access to the file

5. **Connect** `Schedule Trigger -> Get row(s) in sheet`.

6. **Add a Split In Batches node** named `Loop Over Items`.
   - Keep default options unless you want a custom batch size of 1
   - This node is used as the per-row loop controller

7. **Connect** `Get row(s) in sheet -> Loop Over Items`.

8. **Add an HTTP Request node** named `HTTP Request3`.
   - Method: GET
   - URL:
     `=https://api.scraperapi.com?api_key=YOUR_SCRAPERAPI_KEY&url={{ encodeURIComponent($json['Product URL'])}}`
   - Replace `YOUR_SCRAPERAPI_KEY` with your actual ScraperAPI key or move it into credentials/environment variables
   - Ensure response format preserves page content in a field accessible to the next node

9. **Connect** `Loop Over Items -> HTTP Request3`.

10. **Add a Code node** named `Clean Content`.
    - Paste this logic:
    ```javascript
    const html = $input.first().json.data;

    if (!html) return { text: "No data found" };

    const cleanText = html
      .replace(/<script\b[^>]*>([\s\S]*?)<\/script>/gim, "")
      .replace(/<style\b[^>]*>([\s\S]*?)<\/style>/gim, "")
      .replace(/<[^>]+>/g, " ")
      .replace(/\s+/g, " ")
      .trim();

    return {
      cleaned_content: cleanText.substring(0, 15000)
    };
    ```
    - If your HTTP node returns body content somewhere other than `json.data`, update the code accordingly

11. **Connect** `HTTP Request3 -> Clean Content`.

12. **Add a Groq Chat Model node** named `Groq Chat Model1`.
    - Provider: Groq
    - Model: `llama-3.3-70b-versatile`
    - Configure Groq API credentials

13. **Add an AI Agent node** named `AI Agent1`.
    - Prompt type: define
    - Use the extraction prompt from the workflow
    - Insert `{{ $json.cleaned_content }}` as the input payload
    - Keep the instruction to return valid JSON only

14. **Connect**:
    - `Clean Content -> AI Agent1`
    - `Groq Chat Model1 -> AI Agent1` using the AI language model connection

15. **Add a Code node** named `current Price and offer`.
    - Paste:
    ```javascript
    const rawText = $json.output;

    try {
      const cleanJson = rawText.replace(/```json|```/g, "").trim();
      const data = JSON.parse(cleanJson);

      return {
        product_name: data.product_name,
        current_price: data.current_price,
        current_offer: data.current_offer
      };
    } catch (error) {
      return {
        error: "Failed to parse Groq output",
        raw_output: rawText
      };
    }
    ```

16. **Connect** `AI Agent1 -> current Price and offer`.

17. **Add an IF node** named `If2` for first-run initialization.
    - Use AND logic
    - Condition 1: `Last Known Price` from the original sheet row is empty
    - Condition 2: `Last Known Offer` from the original sheet row is empty
    - Expressions:
      - `={{ $('Get row(s) in sheet').item.json['Last Known Price'] }}`
      - `={{ $('Get row(s) in sheet').item.json['Last Known Offer'] }}`

18. **Connect** `current Price and offer -> If2`.

19. **Add a Google Sheets update node** named `First time price and offer added`.
    - Operation: update row
    - Matching column: `Competitor Name`
    - Set:
      - `Competitor Name` = `{{ $('Loop Over Items').item.json['Competitor Name'] }}`
      - `Last Known Price` = `{{ $json.current_price }}`
      - `Last Known Offer` = `{{ $json.current_offer }}`

20. **Connect** `If2` true branch -> `First time price and offer added`.

21. **Add another IF node** named `If1`.
    - Use OR logic
    - Condition 1: historical price != extracted current price
    - Condition 2: historical offer != extracted current offer
    - Expressions:
      - `={{ $('Get row(s) in sheet').item.json['Last Known Price'] }}`
      - `={{ $json.current_price }}`
      - `={{ $('Get row(s) in sheet').item.json['Last Known Offer'] }}`
      - `={{ $json.current_offer }}`

22. **Connect** `If2` false branch -> `If1`.

23. **Add a Google Sheets update node** named `Updated current price and offer in sheet`.
    - Operation: update row
    - Matching column: `Competitor Name`
    - Set:
      - `Competitor Name` = `{{ $('Loop Over Items').item.json['Competitor Name'] }}`
      - `Current Price` = `{{ $json.current_price }}`
      - `Current Offer` = `{{ $json.current_offer }}`

24. **Connect** `If1` true branch -> `Updated current price and offer in sheet`.

25. **Add a Google Sheets update node** named `If No changes then update`.
    - Operation: update row
    - Matching column: `Competitor Name`
    - Intended values:
      - `Competitor Name` should be `{{ $('Loop Over Items').item.json['Competitor Name'] }}`
      - `Current Price` = `{{ $json.current_price }}`
      - `Current Offer` = `{{ $json.current_offer }}`
    - Important: the JSON contains a likely mistake where `Competitor Name` is `=`. Use the corrected expression above when rebuilding.

26. **Connect** `If1` false branch -> `If No changes then update`.

27. **Close the loop** by connecting:
    - `First time price and offer added -> Loop Over Items`
    - `Updated current price and offer in sheet -> Loop Over Items`
    - `If No changes then update -> Loop Over Items`

28. **Add a second Google Sheets read node** named `Get row(s) in sheet1`.
    - Read all rows from the same sheet

29. **Connect** `Loop Over Items -> Get row(s) in sheet1`.
    - In practice, ensure this path runs after loop completion, not per item. Test carefully because loop wiring in n8n can be sensitive.

30. **Add a Code node** named `Data Aggregator`.
    - Paste the provided JavaScript that:
      - reads all rows
      - computes price changes
      - builds `aggregated_market_data`
      - returns `product_name`

31. **Connect** `Get row(s) in sheet1 -> Data Aggregator`.

32. **Add a second Groq Chat Model node** named `Groq Chat Model`.
    - Model: `llama-3.3-70b-versatile`
    - Use Groq credentials

33. **Add a Simple Memory node** named `Simple Memory`.
    - Session key: `history`
    - Session ID type: `customKey`
    - If n8n requires a concrete custom session ID, provide one such as a static string or execution-based identifier

34. **Add a second AI Agent node** named `AI Agent`.
    - Enable `executeOnce`
    - Prompt type: define
    - Use the strategist prompt with:
      - `{{ $json.product_name }}`
      - `{{ $json.aggregated_market_data }}`

35. **Connect**:
    - `Data Aggregator -> AI Agent`
    - `Groq Chat Model -> AI Agent` as language model
    - `Simple Memory -> AI Agent` as memory

36. **Add a Google Sheets update node** named `Update row in sheet`.
    - Operation: update row
    - Matching column: `Competitor Name`
    - Set:
      - `Summary` = `{{ $json.output }}`
      - `Competitor Name` = `{{ $('Get row(s) in sheet').item.json['Competitor Name'] }}`
    - Note: this design is imperfect because the summary is market-wide but matched to one competitor row. Consider instead storing summary in a dedicated report sheet or updating all rows.

37. **Connect** `AI Agent -> Update row in sheet`.

38. **Add a Set node** named `Edit Fields1`.
    - Create one field: `formatted_summary`
    - Type: string
    - Paste the HTML template from the workflow
    - Keep the timezone formatting if desired:
      - `Asia/Kolkata`

39. **Connect** `Update row in sheet -> Edit Fields1`.

40. **Add a Gmail node** named `Send a message`.
    - Operation: send email
    - Authenticate with Gmail OAuth2
    - To: replace `user@example.com` with the actual recipient
    - Subject:
      `=Market Strategy Report: {{ $('Data Aggregator').item.json.product_name }}`
    - HTML message:
      include greeting, `{{ $json.formatted_summary }}`, and signature

41. **Connect** `Edit Fields1 -> Send a message`.

42. **Add a third Google Sheets read node** named `Get row(s) in sheet2`.
    - Read all rows from the same sheet

43. **Connect** `Send a message -> Get row(s) in sheet2`.

44. **Add a final Google Sheets update node** named `Update row in sheet1`.
    - Operation: update row
    - Matching column: `Competitor Name`
    - Set:
      - `Competitor Name` = `{{ $json["Competitor Name"] }}`
      - `Last Known Price` = `{{ $json["Current Price"] }}`
      - `Last Known Offer` = `{{ $json["Current Offer"] }}`
      - `Current Price` = `{{ null }}`
      - `Current Offer` = `{{ null }}`

45. **Connect** `Get row(s) in sheet2 -> Update row in sheet1`.

46. **Add optional sticky notes** to document:
    - overall workflow purpose
    - step 1 database sync
    - step 2 scraping
    - step 3 price extraction
    - step 4 analysis
    - step 5 reporting
    - step 6 reset

47. **Configure credentials**:
    - Google Sheets OAuth2 or service account with sheet access
    - Groq API credentials for both model nodes
    - Gmail OAuth2 credentials
    - ScraperAPI key in the HTTP Request node or environment variable

48. **Test with a small sheet** containing 2–3 competitors first.

49. **Verify critical field names** match exactly across the sheet and expressions:
    - `Competitor Name`
    - `Product Name`
    - `Product URL`
    - `Last Known Price`
    - `Last Known Offer`
    - `Current Price`
    - `Current Offer`
    - `Summary`

50. **Validate and fix likely design issues before production**:
    - Correct `If No changes then update` so `Competitor Name` is not `=`
    - Confirm `Get row(s) in sheet1` executes after the loop completes
    - Decide whether the summary should be stored in one row, all rows, or another sheet
    - Prevent reset from overwriting history with blank current values

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Daily Competitor Price & Offer Monitor: This workflow automatically tracks competitor pricing and promotional offers from your Google Sheets database. It scrapes live product pages, extracts the real selling price using AI, compares it with historical data, and generates a strategic market report. | Workflow canvas note |
| Setup steps: Connect Google Sheets, add ScraperAPI key, connect AI provider, configure Gmail, set schedule. | Workflow canvas note |
| Customization tips: refine AI prompts, modify schedule frequency, add more competitors in Google Sheets. | Workflow canvas note |
| Step 1: Database Sync — Fetch competitor names and product URLs from Google Sheets. | Workflow canvas note |
| Step 2: Scraping — Pull raw product page data using ScraperAPI. | Workflow canvas note |
| Step 3: Price Extraction — AI extracts the real selling price and offer details. | Workflow canvas note |
| Step 4: Analysis — Compare current vs historical data to detect changes. | Workflow canvas note |
| Step 5: Reporting — Generate and email the market intelligence report. | Workflow canvas note |
| Step 6: Reset — Move current data to history and clear fields for next run. | Workflow canvas note |

## Additional implementation note
The provided workflow is functional in concept but contains at least two likely issues:
1. `If No changes then update` has an invalid `Competitor Name` mapping.
2. `Update row in sheet` stores a global summary while matching on a single competitor row, which may not reflect the intended reporting design.