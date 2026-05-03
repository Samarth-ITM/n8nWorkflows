Track Udemy course discounts with Airtop, Google Sheets and Telegram alerts

https://n8nworkflows.xyz/workflows/track-udemy-course-discounts-with-airtop--google-sheets-and-telegram-alerts-12248


# Track Udemy course discounts with Airtop, Google Sheets and Telegram alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Track Udemy course discounts with Airtop, Google Sheets and Telegram alerts

**Purpose:**  
This workflow automatically checks Udemy search results (example query: “n8n”), extracts course pricing details, computes discount percentages, and alerts you when a course has a **50%+ discount**. Qualified deals are also stored in **Google Sheets** for tracking.

**Target use cases:**
- Monitoring Udemy course discounts for a topic (e.g., n8n, Python, AWS).
- Building a lightweight “deal radar” with logging + instant notifications.
- Collecting deal history over time in a spreadsheet.

### Logical blocks
**1.1 Scheduling & Session Start**
- Runs on a schedule, then starts an Airtop browser session.

**1.2 Browser Automation & Data Extraction (Udemy via Airtop)**
- Opens Udemy search results and uses Airtop AI extraction to scrape course data.

**1.3 Data Processing & Offer Validation**
- Splits extracted array into per-course items and filters out items without an offer (no original price).

**1.4 Discount Computation & Threshold Filtering**
- Computes discount percentage and keeps only courses with discount ≥ 50%.

**1.5 Persistence & Notification**
- Appends qualifying deals to Google Sheets and sends a formatted Telegram message.

**1.6 Iteration & Cleanup**
- Loops through items and closes Airtop window + terminates session at the end.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Session Start

**Overview:**  
Triggers the workflow periodically and initializes Airtop, which is required for browser automation and scraping.

**Nodes involved:**
- Schedule Trigger
- Create a session

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration (interpreted):** Uses an interval rule (in the provided JSON the interval object is empty, meaning it may rely on UI defaults or is incomplete).
- **Connections:**
  - Output → **Create a session**
- **Edge cases / failures:**
  - Misconfigured schedule (empty interval) may result in not running as intended.
  - Timezone differences between n8n instance and expected schedule.

#### Node: Create a session
- **Type / role:** `n8n-nodes-base.airtop` (Airtop) — creates a browser automation session.
- **Configuration:**
  - Profile name: `udemy` (Airtop browser profile)
  - Timeout: 30 minutes
- **Key outputs used later:**
  - `sessionId` is referenced by cleanup nodes.
- **Connections:**
  - Input: Schedule Trigger
  - Output → **Create a window**
- **Edge cases / failures:**
  - Airtop credentials missing/invalid.
  - Profile `udemy` not existing in Airtop.
  - Session creation timeouts or quota limits.

---

### 2.2 Browser Automation & Data Extraction

**Overview:**  
Opens a Udemy search URL and uses Airtop’s extraction to return structured course information.

**Nodes involved:**
- Create a window
- Scrape course

#### Node: Create a window
- **Type / role:** `n8n-nodes-base.airtop` (window resource) — opens a page in the Airtop session.
- **Configuration:**
  - URL: `https://www.udemy.com/courses/search/?q=n8n&src=sac`
  - “Get live view”: enabled (useful for debugging/visibility)
- **Connections:**
  - Input: Create a session
  - Output → **Scrape course**
- **Edge cases / failures:**
  - Udemy blocking/consent walls/CAPTCHA can cause extraction failures.
  - Regional pricing/currency changes could change DOM and extracted strings.
  - Network/load delays; extraction may happen before content is fully loaded.

#### Node: Scrape course
- **Type / role:** `n8n-nodes-base.airtop` (extraction → query) — AI-based structured extraction from the page.
- **Configuration:**
  - Prompt requests: title, current price, original price (blank if not available), url, instructors name, rating, offerLeftTime.
  - Enforces a JSON schema:
    - Output object with `products: array`
    - Each product requires: `title`, `current_price`, `original_price`, `url`, `instructors_name`, `rating`, `offerLeftTime`
  - `parseJsonOutput: true` (expects valid JSON matching schema)
- **Connections:**
  - Input: Create a window
  - Output → **Split course data**
- **Edge cases / failures:**
  - Extraction output not matching schema (missing required fields) → parsing errors.
  - `original_price` may be empty, but schema marks it required; Airtop must still output it as `""` (string) to satisfy schema.
  - Udemy lazy-loading: not all course cards present without scrolling; extraction may capture only the first batch.

---

### 2.3 Data Processing & Offer Validation

**Overview:**  
Transforms the extracted `products` array into individual items, then filters to only those that appear to have an active discount (presence of original price).

**Nodes involved:**
- Split course data
- Check offer available or not
- No Operation, do nothing
- Loop Over Items

#### Node: Split course data
- **Type / role:** `n8n-nodes-base.splitOut` — splits an array field into separate items.
- **Configuration:**
  - Field to split: `output.products`
- **Connections:**
  - Input: Scrape course
  - Output → **Check offer available or not**
- **Edge cases / failures:**
  - If Airtop output path differs (e.g., `products` not nested under `output`), split will produce 0 items.
  - If `output.products` is missing/null, node may error depending on runtime behavior.

#### Node: Check offer available or not
- **Type / role:** `n8n-nodes-base.if` — filters items where `original_price` is not empty.
- **Configuration:**
  - Condition: `original_price` **is not empty**
  - Expression: `{{ $json.original_price }}`
- **Connections:**
  - **True** → **Loop Over Items**
  - **False** → **No Operation, do nothing**
- **Edge cases / failures:**
  - If `original_price` exists but contains non-price text (e.g., “Free”), it still passes “not empty” and may break later numeric parsing.
  - If extraction returns `null` instead of `""`, strict validation may behave unexpectedly.

#### Node: No Operation, do nothing
- **Type / role:** `n8n-nodes-base.noOp` — sink for non-offer items (explicitly discards).
- **Connections:**
  - Input: IF (false path)
- **Edge cases / failures:**
  - None (acts as terminator for that branch).

#### Node: Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — iterates items one-by-one (or by batch).
- **Configuration:** Default batch options (not explicitly set).
- **Connections:**
  - Input: IF (true path) and later re-entry from downstream nodes (looping)
  - Output 1 → **Close a window** (as wired, this fires when node runs; see note below)
  - Output 2 → **Find discount**
- **Important behavior note:**  
  This node is used as the loop controller. It is also the convergence point for “continue to next item” from:
  - Send notify course deal
  - Check discount percentage 50%> (false path)
- **Edge cases / failures:**
  - Because **Close a window** is directly connected to Loop Over Items, it may execute too early (depending on how batches/outputs are used). In many designs, cleanup should happen only after all items are processed. If you observe premature window closing, move cleanup to a separate “No more items” path or use SplitInBatches’ second output (when available in your n8n version/config).
  - If there are many items and Airtop session times out mid-loop, subsequent scraping-derived actions still run but cleanup may fail.

---

### 2.4 Discount Computation & Threshold Filtering

**Overview:**  
Computes a numeric discount percentage from the string prices, then checks whether the deal is at least 50% off.

**Nodes involved:**
- Find discount
- Check discount percentage 50%>

#### Node: Find discount
- **Type / role:** `n8n-nodes-base.code` — parses prices and computes discount percentage.
- **Configuration choices:**
  - Reads from first input item:
    - `current_price` and `original_price`
  - Extracts currency symbol as first character of `current_price`
  - Strips non-numeric characters, parses floats
  - Computes: `((original - current) / original) * 100`
  - Outputs:
    - `current_price` (normalized numeric with currency symbol)
    - `original_price` (normalized numeric with currency symbol)
    - `discount_percentage` (string with 2 decimals)
- **Connections:**
  - Input: Loop Over Items
  - Output → **Check discount percentage 50%>**
- **Edge cases / failures:**
  - If prices are like `"Free"` or missing digits, `parseFloat` returns `NaN` → discount becomes `NaN`.
  - Currency symbol assumption (first character) fails for formats like `"US$12.99"` or `"12,99 €"`.
  - If original price is `0` or parses to `0`, division by zero → `Infinity`.

#### Node: Check discount percentage 50%>
- **Type / role:** `n8n-nodes-base.if` — gates items based on computed discount.
- **Configuration:**
  - Condition: numeric `discount_percentage` **>= 50**
  - Uses loose type validation (explicitly enabled), so string numbers can compare as numbers.
- **Connections:**
  - **True** → **Append 50% up disc data**
  - **False** → **Loop Over Items** (continue loop without logging/notifying)
- **Edge cases / failures:**
  - If `discount_percentage` is `NaN`, numeric comparison fails and will route to false (silently skipping deals).
  - If discount is formatted unexpectedly (e.g., `"50.00%"`), loose parsing may not work.

---

### 2.5 Persistence & Notification

**Overview:**  
Saves qualifying deals to a Google Sheet and sends a Telegram alert formatted for quick reading.

**Nodes involved:**
- Append 50% up disc data
- Send notify course deal

#### Node: Append 50% up disc data
- **Type / role:** `n8n-nodes-base.googleSheets` — appends a row to a sheet.
- **Configuration:**
  - Operation: Append
  - Document: “Online Course Price Tracker with Coupon Detection”
  - Sheet tab: “Sheet1” (gid=0)
  - Column mapping (key fields):
    - Url, Title, Rating, Discount (`{{ $json.discount_percentage }}% OFF`)
    - Offer left, Current price, Original price
    - Instructor Name
  - Mixes data sources:
    - Most fields pulled from `$('Loop Over Items').item.json...` (the original scraped item)
    - Discount/current/original price from the Code node output (`$json...`)
- **Connections:**
  - Input: Check discount percentage 50%> (true path)
  - Output → **Send notify course deal**
- **Edge cases / failures:**
  - Google Sheets credentials missing/expired (OAuth).
  - Sheet structure mismatch: column headers must exist or be creatable depending on node settings.
  - Rate limits or API errors when many rows appended quickly.
  - Potential confusion from mixing `$('Loop Over Items').item` and `$json`: if the execution context changes, expressions can resolve unexpectedly. Generally OK here because the loop item is the canonical source.

#### Node: Send notify course deal
- **Type / role:** `n8n-nodes-base.telegram` — sends Telegram message to a chat.
- **Configuration:**
  - Chat ID: `chat_id` (placeholder; must be replaced)
  - Message text includes:
    - Instructor name, course title, original/current price, discount, rating, offer time left, URL
  - `appendAttribution: false`
- **Connections:**
  - Input: Append 50% up disc data
  - Output → **Loop Over Items** (continues to next item)
- **Edge cases / failures:**
  - Telegram bot token invalid.
  - Chat ID wrong or bot not allowed in the chat.
  - Message formatting: contains emoji and special characters; generally safe, but long titles could exceed Telegram limits in extreme cases.

---

### 2.6 Cleanup (Airtop)

**Overview:**  
Closes the Airtop window and terminates the Airtop session.

**Nodes involved:**
- Close a window
- Terminate a session

#### Node: Close a window
- **Type / role:** `n8n-nodes-base.airtop` (window → close) — closes the opened browser window.
- **Configuration:**
  - `windowId`: from `Create a window` output: `{{ $('Create a window').item.json.data.windowId }}`
  - `sessionId`: from session: `{{ $('Create a session').item.json.sessionId }}`
- **Connections:**
  - Input: Loop Over Items
  - Output → **Terminate a session**
- **Edge cases / failures:**
  - If window was already closed (or never opened), Airtop may return “not found”.
  - If called too early (before processing completes), subsequent extraction/loop steps may break.

#### Node: Terminate a session
- **Type / role:** `n8n-nodes-base.airtop` (terminate) — ends session to free resources.
- **Configuration:**
  - `sessionId`: `{{ $('Create a session').item.json.sessionId }}`
- **Connections:**
  - Input: Close a window
- **Edge cases / failures:**
  - Terminating an already-terminated session should be handled, but may throw errors depending on Airtop API behavior.
  - If this runs prematurely, later Airtop actions are impossible.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Scheduled entry point | — | Create a session |  |
| Create a session | Airtop | Start Airtop browser session | Schedule Trigger | Create a window | ## Browser Automation<br>Launches Airtop session, opens Udemy search page, and scrapes course data using AI extraction. |
| Create a window | Airtop | Open Udemy search page | Create a session | Scrape course | ## Browser Automation<br>Launches Airtop session, opens Udemy search page, and scrapes course data using AI extraction. |
| Scrape course | Airtop | AI extraction of course data | Create a window | Split course data | ## Browser Automation<br>Launches Airtop session, opens Udemy search page, and scrapes course data using AI extraction. |
| Split course data | Split Out | Convert products array to items | Scrape course | Check offer available or not | ## Data Processing<br>Splits scraped courses into individual items and filters for those with active discount offers. |
| Check offer available or not | IF | Filter: must have original price | Split course data | Loop Over Items (true), No Operation, do nothing (false) | ## Data Processing<br>Splits scraped courses into individual items and filters for those with active discount offers. |
| No Operation, do nothing | NoOp | Discard non-offer items | Check offer available or not (false) | — | ## Data Processing<br>Splits scraped courses into individual items and filters for those with active discount offers. |
| Loop Over Items | Split In Batches | Iteration controller | Check offer available or not (true); Send notify course deal; Check discount percentage 50%> (false) | Close a window; Find discount |  |
| Find discount | Code | Compute discount percentage | Loop Over Items | Check discount percentage 50%> | ## Discount Detection<br>Calculates discount percentage and saves courses with 50%+ discounts to Google Sheets, then sends Telegram alerts. |
| Check discount percentage 50%> | IF | Filter: discount ≥ 50 | Find discount | Append 50% up disc data (true); Loop Over Items (false) | ## Discount Detection<br>Calculates discount percentage and saves courses with 50%+ discounts to Google Sheets, then sends Telegram alerts. |
| Append 50% up disc data | Google Sheets | Persist deal row | Check discount percentage 50%> (true) | Send notify course deal | ## Discount Detection<br>Calculates discount percentage and saves courses with 50%+ discounts to Google Sheets, then sends Telegram alerts. |
| Send notify course deal | Telegram | Send deal alert | Append 50% up disc data | Loop Over Items | ## Discount Detection<br>Calculates discount percentage and saves courses with 50%+ discounts to Google Sheets, then sends Telegram alerts. |
| Close a window | Airtop | Close browser window | Loop Over Items | Terminate a session | ## Cleanup<br>Closes browser window and terminates Airtop session after processing all courses. |
| Terminate a session | Airtop | End Airtop session | Close a window | — | ## Cleanup<br>Closes browser window and terminates Airtop session after processing all courses. |
| Overview | Sticky Note | Documentation / setup guidance | — | — | ## How it works<br>This workflow monitors Udemy courses for deep discounts (50%+) and alerts you via Telegram.<br><br>Every day, it launches a browser session using Airtop, scrapes course listings from Udemy search results, and extracts pricing data. Each course is checked for active offers—if the original price exists, the workflow calculates the discount percentage. Courses with 50%+ discounts are logged to Google Sheets and sent as formatted Telegram notifications with instructor, rating, and time-limited offer details.<br><br>## Setup steps<br>1. Connect your Airtop account (for browser automation)<br>2. Update the search URL in "Create a window" to match your course interests<br>3. Connect Google Sheets and select your tracking spreadsheet<br>4. Add your Telegram bot token and chat ID<br>5. Adjust the schedule trigger frequency (default: daily at midnight)<br><br>### Sample sheet:<br>https://docs.google.com/spreadsheets/d/1JQE5aW11UriY-jtybq9LTXqu-Un3khZuNp7SzSjXnVU/edit?gid=0#gid=0 |
| Browser Automation | Sticky Note | Section label | — | — |  |
| Data Processing | Sticky Note | Section label | — | — |  |
| Discount Detection | Sticky Note | Section label | — | — |  |
| Cleanup | Sticky Note | Section label | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Schedule Trigger” (Schedule Trigger node)**
   - Set it to your desired frequency (commonly daily).
   - Ensure timezone matches your expectations.

2) **Create “Create a session” (Airtop node)**
   - Operation: *Create Session* (session creation)
   - Profile name: `udemy` (or your Airtop profile)
   - Timeout: 30 minutes
   - Connect: **Schedule Trigger → Create a session**
   - **Credentials:** add Airtop credentials in n8n.

3) **Create “Create a window” (Airtop node)**
   - Resource: Window
   - Operation: Create/Open window
   - URL: set to your Udemy search, e.g.  
     `https://www.udemy.com/courses/search/?q=n8n&src=sac`
   - Enable Live View (optional but helpful).
   - Connect: **Create a session → Create a window**

4) **Create “Scrape course” (Airtop node)**
   - Resource: Extraction
   - Operation: Query
   - Prompt: ask for `title`, `current_price`, `original_price` (blank if missing), `url`, `instructors_name`, `rating`, `offerLeftTime`.
   - Enable JSON parsing of output.
   - Provide an output schema containing:
     - root object with `products: array of objects`
     - required fields matching the prompt
   - Connect: **Create a window → Scrape course**

5) **Create “Split course data” (Split Out node)**
   - Field to split out: `output.products`
   - Connect: **Scrape course → Split course data**

6) **Create “Check offer available or not” (IF node)**
   - Condition: String → `original_price` → “is not empty”
   - Expression: `{{ $json.original_price }}`
   - Connect: **Split course data → Check offer available or not**

7) **Create “No Operation, do nothing” (NoOp node)**
   - Connect: **Check offer available or not (false) → No Operation, do nothing**

8) **Create “Loop Over Items” (Split In Batches node)**
   - Keep default batch size (or set to 1 explicitly if desired).
   - Connect: **Check offer available or not (true) → Loop Over Items**

9) **Create “Find discount” (Code node)**
   - Paste logic to:
     - parse `current_price`, `original_price`
     - compute `discount_percentage`
     - output normalized prices + discount
   - Connect: **Loop Over Items → Find discount**

10) **Create “Check discount percentage 50%>” (IF node)**
   - Condition: Number → `{{ $json.discount_percentage }}` → `>= 50`
   - Loose type validation: enabled (to allow string numeric).
   - Connect: **Find discount → Check discount percentage 50%>**
   - Connect false path to loop continuation: **Check discount… (false) → Loop Over Items**

11) **Create “Append 50% up disc data” (Google Sheets node)**
   - Operation: Append
   - Select Spreadsheet and Sheet (create a sheet with headers such as Url, Title, Rating, Discount, Offer left, Current price, Original price, Instructor Name).
   - Map fields:
     - Use loop item for scraped fields, e.g. `{{ $('Loop Over Items').item.json.title }}`
     - Use code output for computed fields, e.g. `{{ $json.discount_percentage }}% OFF`
   - Connect: **Check discount… (true) → Append 50% up disc data**
   - **Credentials:** connect Google Sheets OAuth2.

12) **Create “Send notify course deal” (Telegram node)**
   - Operation: Send Message
   - Set `chatId` to your target chat ID.
   - Provide message template using sheet output fields (the appended row output) and/or loop item fields.
   - Connect: **Append 50% up disc data → Send notify course deal**
   - Loop continuation: **Send notify course deal → Loop Over Items**
   - **Credentials:** Telegram bot token.

13) **Create cleanup nodes**
   - **“Close a window” (Airtop node)**
     - Resource: Window
     - Operation: Close
     - `windowId`: `{{ $('Create a window').item.json.data.windowId }}`
     - `sessionId`: `{{ $('Create a session').item.json.sessionId }}`
   - **“Terminate a session” (Airtop node)**
     - Operation: Terminate
     - `sessionId`: `{{ $('Create a session').item.json.sessionId }}`
     - Connect: **Close a window → Terminate a session**
   - Connect cleanup from looping mechanism as in the JSON (**Loop Over Items → Close a window**).  
     If cleanup happens too early in your tests, adjust wiring so cleanup runs only after all batches are processed (implementation depends on your SplitInBatches behavior/version).

14) **Add Sticky Notes (optional)**
   - Add notes for Overview, Browser Automation, Data Processing, Discount Detection, Cleanup.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sample sheet: https://docs.google.com/spreadsheets/d/1JQE5aW11UriY-jtybq9LTXqu-Un3khZuNp7SzSjXnVU/edit?gid=0#gid=0 | Provided in the workflow’s “Overview” sticky note |
| Setup essentials: Airtop credentials + correct profile; update Udemy search URL; Google Sheets OAuth; Telegram bot token + chat ID; adjust schedule | From the “Overview” sticky note |
| Caution: Udemy pages may show consent/CAPTCHA or lazy-load results; extraction reliability depends on page state | Operational consideration for Airtop scraping |
| Cleanup wiring may terminate the session early depending on SplitInBatches behavior | Review loop/cleanup flow if you see premature window closure |