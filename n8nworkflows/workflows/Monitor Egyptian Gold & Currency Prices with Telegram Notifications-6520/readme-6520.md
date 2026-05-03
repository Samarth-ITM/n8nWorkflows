Monitor Egyptian Gold & Currency Prices with Telegram Notifications

https://n8nworkflows.xyz/workflows/monitor-egyptian-gold---currency-prices-with-telegram-notifications-6520


# Monitor Egyptian Gold & Currency Prices with Telegram Notifications

### 1. Workflow Overview

This workflow monitors Egyptian gold and currency prices by scraping data from a financial website and sends formatted updates as Telegram messages. It is designed for users interested in real-time financial price tracking in Egypt, providing price details with emojis and localized formatting. The workflow includes the following logical blocks:

- **1.1 Scheduled Trigger & Time Filtering:** Initiates the workflow every hour and restricts execution to daytime hours (10:00–22:00 Cairo time).
- **1.2 Data Collection via HTTP Request:** Fetches the raw HTML content from the target website.
- **1.3 Data Extraction (HTML Parsing):** Parses specific HTML tables containing gold prices and currency rates.
- **1.4 Data Processing & Formatting (Code Node):** Cleans, processes, and formats the extracted data into a Telegram-ready message with emojis and timestamps.
- **1.5 Notification Dispatch (Telegram):** Sends the formatted message to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Time Filtering

**Overview:**  
This block triggers the workflow automatically on a schedule (hourly) and ensures execution only occurs during the active monitoring window between 10 AM and 10 PM Cairo time.

**Nodes Involved:**  
- Schedule Trigger  
- If

**Node Details:**

- **Schedule Trigger**  
  - Type & Role: `Schedule Trigger` — initiates workflow at specified intervals.  
  - Configuration: Set to trigger every hour (`interval: hours`).  
  - Inputs: None (start node).  
  - Outputs: Connects to the If node.  
  - Edge Cases: Risk of triggering outside target time window without the If node filtering.  
  - Notes: No special version requirements.

- **If**  
  - Type & Role: `If` — conditional gate based on current hour.  
  - Configuration: Checks if current hour (system time) is ≥ 10 and ≤ 22, using expression `={{ $now.hour }}`.  
  - Inputs: From Schedule Trigger.  
  - Outputs: Passes to HTTP Request node if condition true; no downstream if false (stops execution).  
  - Edge Cases: Timezone assumption depends on n8n server time; may cause mismatch if server is not in Cairo timezone.  
  - Recommendations: Confirm server time or explicitly convert to Cairo timezone if needed.

---

#### 1.2 Data Collection via HTTP Request

**Overview:**  
This block performs an HTTP GET request to the target website to retrieve the full HTML page containing gold and currency prices.

**Nodes Involved:**  
- HTTP Request

**Node Details:**

- **HTTP Request**  
  - Type & Role: `HTTP Request` — fetches the webpage content.  
  - Configuration:  
    - URL: `https://edahabapp.com/`  
    - Method: GET (default)  
    - Retry on Fail: Enabled for robustness.  
  - Inputs: From If node (conditional pass).  
  - Outputs: Raw HTML to HTML node.  
  - Edge Cases: Network errors, site downtime, or changes in page structure may cause failure or invalid responses.  
  - Recommendations: Monitor failure rates; consider adding error handling or alerting on repeated failures.

---

#### 1.3 Data Extraction (HTML Parsing)

**Overview:**  
This block extracts specific HTML sections (tables) for gold prices and currency rates using CSS selectors, enabling further structured processing.

**Nodes Involved:**  
- HTML

**Node Details:**

- **HTML**  
  - Type & Role: `HTML Extract` — parses HTML content and extracts data using CSS selectors.  
  - Configuration:  
    - Operation: `extractHtmlContent`  
    - Extraction Values:  
      - `gold_prices`: Selects the first section’s first grid div’s table (CSS: `section:first-of-type .grid div:first-child table`)  
      - `currency_rates`: Selects the first section’s second grid div’s table (CSS: `section:first-of-type .grid div:nth-child(2) table`)  
  - Inputs: Raw HTML from HTTP Request node.  
  - Outputs: Extracted HTML snippets for gold and currency tables to Code node.  
  - Edge Cases: Changes in website HTML structure or CSS classes may break extraction.  
  - Recommendations: Periodically verify selectors; implement fallback or error detection if extraction returns empty.

---

#### 1.4 Data Processing & Formatting (Code Node)

**Overview:**  
Processes extracted HTML tables to produce a clean, readable Telegram message with emojis, Arabic localization, and timestamp.

**Nodes Involved:**  
- Code

**Node Details:**

- **Code**  
  - Type & Role: `Code` (JavaScript) — transforms raw HTML into formatted text message.  
  - Configuration:  
    - Uses RegEx to parse table rows and cells.  
    - Defines helper functions:  
      - `getCurrencyEmoji`: Returns appropriate emoji based on currency name (e.g., USD = 🇺🇸).  
      - `cleanText`: Strips HTML tags from strings.  
    - Processes gold prices: Adds emojis for gold types, formats buy/sell prices with localized currency units.  
    - Processes currency rates: Adds currency-specific emojis, formats prices with currency unit (جنيه).  
    - Adds timestamp in Cairo local time, formatted in Arabic.  
    - Constructs final multi-line Telegram markdown message.  
  - Inputs: Extracted HTML snippets from HTML node.  
  - Outputs: JSON with key `telegram_message` containing the formatted message string.  
  - Edge Cases:  
    - HTML parsing via RegEx may fail if input HTML format changes.  
    - Missing or malformed data leads to empty or incomplete messages.  
    - Timezone assumptions rely on server locale or Node.js environment.  
  - Recommendations: Add error handling for unexpected HTML or empty data; consider using a robust HTML parser if complexity grows.

---

#### 1.5 Notification Dispatch (Telegram)

**Overview:**  
Sends the formatted message to a Telegram chat using the Telegram API.

**Nodes Involved:**  
- Send a text message (Telegram)

**Node Details:**

- **Send a text message**  
  - Type & Role: `Telegram` — sends text messages via Telegram Bot API.  
  - Configuration:  
    - Text: Set via expression `={{ $json.telegram_message }}` from Code node output.  
    - Chat ID: Environment variable `telegram_chat_id` (secured outside workflow).  
    - Additional Fields: `appendAttribution` disabled (message sent as-is).  
  - Credentials: Uses preconfigured Telegram API credential.  
  - Inputs: Message JSON from Code node.  
  - Outputs: None (end node).  
  - Edge Cases: Telegram API errors such as invalid token, chat ID, or network issues.  
  - Recommendations: Monitor Telegram API rate limits and errors; secure credentials and environment variables.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                      | Input Node(s)       | Output Node(s)   | Sticky Note                                           |
|--------------------|---------------------|------------------------------------|---------------------|------------------|-------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger    | Initiates workflow hourly           | -                   | If               |                                                       |
| If                 | If                  | Time filter: run only 10:00-22:00  | Schedule Trigger    | HTTP Request     |                                                       |
| HTTP Request       | HTTP Request        | Fetches gold & currency prices HTML| If                  | HTML             | Retry enabled for resilience                           |
| HTML               | HTML Extract        | Extracts gold and currency tables  | HTTP Request        | Code             |                                                       |
| Code               | Code (JavaScript)   | Parses tables, formats Telegram msg| HTML                 | Send a text message| Uses RegEx parsing; adds emojis and Arabic formatting |
| Send a text message | Telegram            | Sends Telegram message              | Code                 | -                | Uses Telegram credential; chat ID from environment     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Schedule Trigger` node:**  
   - Set to trigger every 1 hour (`interval: hours`).  
   - Position as start node.

3. **Add an `If` node:**  
   - Connect input from `Schedule Trigger`.  
   - Configure conditions:  
     - Left value: expression `={{ $now.hour }}`  
     - Condition 1: `>= 10`  
     - Condition 2: `<= 22`  
     - Combine conditions with AND.  
   - This node filters execution to daytime Cairo hours.

4. **Add an `HTTP Request` node:**  
   - Connect input from `If` node’s “true” output.  
   - Set URL to `https://edahabapp.com/`.  
   - Method: GET (default).  
   - Enable “Retry On Fail” for robustness.

5. **Add an `HTML` node:**  
   - Connect input from `HTTP Request`.  
   - Set operation to `extractHtmlContent`.  
   - Add two extraction values:  
     - `gold_prices`: CSS selector `section:first-of-type .grid div:first-child table`, return HTML.  
     - `currency_rates`: CSS selector `section:first-of-type .grid div:nth-child(2) table`, return HTML.

6. **Add a `Code` node:**  
   - Connect input from `HTML`.  
   - Paste JavaScript code that:  
     - Defines helper functions for emoji assignment and HTML tag removal.  
     - Parses gold and currency HTML tables using RegEx.  
     - Formats prices with emojis, Arabic text, and markdown for Telegram.  
     - Adds a Cairo-localized timestamp.  
     - Returns final message as `telegram_message` in JSON.

7. **Add a `Telegram` node named “Send a text message”:**  
   - Connect input from `Code`.  
   - Set text parameter to expression `={{ $json.telegram_message }}`.  
   - Set chat ID parameter to expression `={{ $env.telegram_chat_id }}`.  
   - In Additional Fields, disable “append attribution.”  
   - Configure credentials by creating or selecting a Telegram API credential linked to your bot token.

8. **Set node connections:**  
   - Schedule Trigger → If (main)  
   - If (true output) → HTTP Request  
   - HTTP Request → HTML  
   - HTML → Code  
   - Code → Send a text message

9. **Set environment variable:**  
   - Define `telegram_chat_id` in n8n environment to target the correct Telegram chat.

10. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow runs hourly but only sends updates during 10:00-22:00 Cairo time to avoid spam.        | Execution time filtering in If node.                                                                |
| Telegram chat ID is stored securely as an environment variable (`telegram_chat_id`).                | Secure handling of chat identification.                                                            |
| The website structure is critical; changes to CSS selectors or HTML tables may break data extraction.| Regular maintenance recommended.                                                                   |
| The code node uses RegEx for HTML parsing which is efficient but sensitive to markup changes.       | Consider more robust HTML parsers if complexity increases.                                         |
| Telegram API credentials must be configured with a valid bot token and correct chat permissions.    | See https://core.telegram.org/bots/api for Telegram Bot API details.                               |
| For timezone-dependent logic, ensure n8n server timezone matches Cairo or adjust code accordingly. | Timezone awareness for accurate timing and message timestamps.                                     |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly conforms to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.