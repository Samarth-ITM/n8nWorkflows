Automated Multi-Platform Game Deals Tracker with Deku Deals & Gmail Alerts

https://n8nworkflows.xyz/workflows/automated-multi-platform-game-deals-tracker-with-deku-deals---gmail-alerts-6666


# Automated Multi-Platform Game Deals Tracker with Deku Deals & Gmail Alerts

### 1. Workflow Overview

This workflow automates the daily tracking of popular game deals from the Deku Deals website and sends email notifications for new deals that have not been previously notified. It is designed to run every day at 8:00 AM, scrape the latest game deal listings, extract structured deal information, check if the deals have already been seen, store new deals in a local SQLite database, and notify the user via Gmail of any new deals.

Logical blocks include:

- **1.1 Scheduled Trigger:** Automatically triggers the workflow daily.
- **1.2 Data Retrieval & Extraction:** Downloads the Deku Deals webpage and extracts individual deal cards.
- **1.3 Parsing & Structuring Data:** Parses HTML of each deal card to extract detailed information.
- **1.4 Local Persistence & Deduplication:** Uses SQLite to store and check notified deals, preventing duplicate alerts.
- **1.5 Decision & Notification:** Determines if there are new deals and sends formatted email notifications accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the entire workflow automatically every day at 8:00 AM, enabling frequent deal checks without manual intervention.

- **Nodes Involved:**  
  - Daily Check (8 AM)

- **Node Details:**

  - **Daily Check (8 AM)**  
    - Type: Cron Trigger  
    - Role: Schedule the workflow to run daily at a specific time.  
    - Configuration:  
      - Mode set to "everyDay"  
      - Hour: 8, Minute: 0  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "Fetch Deku Deals Page" node.  
    - Edge Cases / Failures:  
      - Time zone discrepancies if n8n server timezone differs from user expectation.  
      - Cron misconfiguration can lead to missed or too frequent triggers.  
    - Notes: Schedule can be changed by adjusting hour/minute fields.

---

#### 1.2 Data Retrieval & Extraction

- **Overview:**  
  Downloads the HTML content of the Deku Deals 'Most Popular' page and extracts each individual game deal card's HTML, preparing for detailed data parsing.

- **Nodes Involved:**  
  - Fetch Deku Deals Page  
  - Extract Each Deal Card

- **Node Details:**

  - **Fetch Deku Deals Page**  
    - Type: HTTP Request  
    - Role: Retrieve raw HTML content of the target deals webpage.  
    - Configuration:  
      - URL: https://www.dekudeals.com/most-popular  
      - Response Format: String (raw HTML)  
    - Inputs: Trigger from Cron node  
    - Outputs: HTML content passed to "Extract Each Deal Card" node  
    - Edge Cases / Failures:  
      - Network errors or site downtime causing failed requests.  
      - Website layout or URL changes may require URL and subsequent parsing updates.  
      - Potential rate limiting or anti-scraping protections.  

  - **Extract Each Deal Card**  
    - Type: HTML Extract  
    - Role: Extract each deal card's full HTML block from the entire page HTML.  
    - Configuration:  
      - HTML source: Output of "Fetch Deku Deals Page" node  
      - Selector: `div.game-card` (targets each deal container)  
      - Attribute: `html` (extract entire HTML snippet of each card)  
    - Inputs: HTML string from previous node  
    - Outputs: List of deal card HTML snippets (one per deal)  
    - Edge Cases / Failures:  
      - Changes in class names or DOM structure breaking the selector.  
      - Empty results if page content changes or blocks scraping.

---

#### 1.3 Parsing & Structuring Data

- **Overview:**  
  Parses each extracted deal card HTML snippet to extract structured data fields like title, prices, discount, platforms, and generates a unique ID for tracking.

- **Nodes Involved:**  
  - Parse Deal Details (Function Code)

- **Node Details:**

  - **Parse Deal Details (Function Code)**  
    - Type: Function (JavaScript)  
    - Role: Use `jsdom` to parse HTML and extract specific details for each deal card.  
    - Configuration:  
      - Uses selectors such as `div.name > a` for title/link, `div.price.current` for current price, `div.price.original` for original price, and `div.price.discount` for discount.  
      - Aggregates platform names from `div.platforms > span.platform`.  
      - Constructs a unique deal ID combining title, platforms, and current price for deduplication.  
    - Inputs: Items each containing one deal card HTML snippet.  
    - Outputs: Items with JSON containing parsed deal fields:  
      - dealUniqueId, gameTitle, gameLink, platforms, currentPrice, originalPrice, discount  
    - Edge Cases / Failures:  
      - DOM selector changes can break parsing logic.  
      - Missing elements default to 'N/A' to avoid breaking workflow.  
      - JavaScript runtime errors if HTML is malformed.  
    - Version: Uses built-in n8n `jsdom` library, no external dependencies needed.  

---

#### 1.4 Local Persistence & Deduplication

- **Overview:**  
  Ensures a local SQLite database table exists to track notified deals, checks if each parsed deal has been previously notified, and splits new vs already notified deals.

- **Nodes Involved:**  
  - SQLite: Ensure Table Exists  
  - SQLite: Check if Notified  
  - Split into Notified/New

- **Node Details:**

  - **SQLite: Ensure Table Exists**  
    - Type: SQLite  
    - Role: Creates the `notified_deals` table if it does not already exist.  
    - Configuration:  
      - Database: `dekudeals` (local file `dekudeals.db`)  
      - Query: `CREATE TABLE IF NOT EXISTS notified_deals (deal_id TEXT PRIMARY KEY, game_title TEXT, platforms TEXT, current_price TEXT, original_price TEXT, discount TEXT, deal_link TEXT, notified_date TEXT)`  
    - Inputs: Triggered after parsing deals  
    - Outputs: Passes to "SQLite: Check if Notified"  
    - Edge Cases / Failures:  
      - File permission issues on database file.  
      - Query syntax errors if modified improperly.  

  - **SQLite: Check if Notified**  
    - Type: SQLite  
    - Role: Checks if each deal's unique ID is already in the database.  
    - Configuration:  
      - Database: `dekudeals`  
      - Query: `SELECT deal_id FROM notified_deals WHERE deal_id = '{{ $json.dealUniqueId }}'`  
    - Inputs: Items from parsing node  
    - Outputs: Items with matches sent forward, no item if not found  
    - Edge Cases / Failures:  
      - Query failures if invalid IDs or database issues occur.  
      - Empty results need proper handling downstream.  

  - **Split into Notified/New**  
    - Type: Item Lists  
    - Role: Splits the deals into two outputs: new deals (no DB record) and already notified deals (DB record found).  
    - Configuration:  
      - Mode: Split in batches based on presence of deal ID in DB query result  
      - Property: `dealUniqueId`  
    - Inputs: Two inputs: original deals and DB query results  
    - Outputs:  
      - Output 1: New deals (no matching DB record)  
      - Output 2: Already notified deals  
    - Edge Cases / Failures:  
      - Misalignment between inputs could cause incorrect splitting.  

---

#### 1.5 Decision & Notification

- **Overview:**  
  Checks if there are new deals and if so, inserts them into the database, formats a notification message, and sends an email to the configured recipient.

- **Nodes Involved:**  
  - If (New Deals Found)  
  - SQLite: Insert New Deals  
  - Format Notification Message  
  - Send Email Notification

- **Node Details:**

  - **If (New Deals Found)**  
    - Type: If  
    - Role: Conditional check if the new deals list is non-empty.  
    - Configuration:  
      - Condition: `$json.length != 0`  
    - Inputs: New deals from "Split into Notified/New" output 1  
    - Outputs:  
      - True: Proceed with insertion and notification.  
      - False: Workflow ends here.  
    - Edge Cases / Failures:  
      - Incorrect length evaluation could trigger false positives or negatives.  

  - **SQLite: Insert New Deals**  
    - Type: SQLite  
    - Role: Inserts new deals into the notification tracking database to avoid repeated alerts.  
    - Configuration:  
      - Database: `dekudeals`  
      - Query:  
        ```sql
        INSERT INTO notified_deals (deal_id, game_title, platforms, current_price, original_price, discount, deal_link, notified_date) 
        VALUES ('{{ $json.dealUniqueId }}', '{{ $json.gameTitle }}', '{{ $json.platforms }}', '{{ $json.currentPrice }}', '{{ $json.originalPrice }}', '{{ $json.discount }}', '{{ $json.gameLink }}', '{{ new Date().toISOString() }}')
        ```  
    - Inputs: New deals from "If (New Deals Found)" True branch  
    - Outputs: Passes inserted deals to formatting node  
    - Edge Cases / Failures:  
      - Duplicate key errors if the same deal is inserted twice in quick succession.  
      - Database write failures.  

  - **Format Notification Message**  
    - Type: Function  
    - Role: Formats a user-friendly notification message listing all new deals with relevant details.  
    - Configuration:  
      - Prepends a fixed header with emojis.  
      - For each deal, shows title, platforms, current price, optionally original price and discount, and the deal URL.  
      - Ends with a link to the full deal page.  
    - Inputs: Newly inserted deals  
    - Outputs: Single JSON with `notificationMessage` string  
    - Edge Cases / Failures:  
      - If any deal field is missing or has unexpected content, message formatting might be affected.  

  - **Send Email Notification**  
    - Type: Gmail  
    - Role: Sends the formatted notification via Gmail API.  
    - Configuration:  
      - Gmail API credential required  
      - From Email: Gmail account email address (must match authenticated user)  
      - To Email: User’s recipient email (must be updated manually)  
      - Subject: "🎮 New Game Deals Alert! (Deku Deals)"  
      - Text: Populated from the formatted message node’s output  
    - Inputs: Formatted notification message  
    - Outputs: Workflow ends after sending email  
    - Edge Cases / Failures:  
      - Authentication failures due to missing or invalid Gmail credentials.  
      - Email sending limits or quota exceeded.  
      - Incorrect recipient email configuration causing delivery failures.  
    - Notes: Can be replaced with other notification nodes (Telegram, Slack, Discord) by mapping `notificationMessage` accordingly.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                                     | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                                                |
|----------------------------|---------------------|----------------------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Daily Check (8 AM)          | Cron                | Scheduled daily trigger at 8:00 AM                  | None                        | Fetch Deku Deals Page           | Triggers workflow every day at 8:00 AM. Adjust hour/minute to change schedule.                                              |
| Fetch Deku Deals Page       | HTTP Request        | Downloads HTML of Deku Deals 'Most Popular' page    | Daily Check (8 AM)           | Extract Each Deal Card          | Downloads HTML content; update URL or selectors if site layout changes.                                                    |
| Extract Each Deal Card      | HTML Extract        | Extracts each individual deal card HTML             | Fetch Deku Deals Page        | Parse Deal Details (Function Code) | Extracts each 'game-card' div; update selector if layout changes.                                                          |
| Parse Deal Details (Function Code) | Function (JS)      | Parses deal card HTML to extract structured details | Extract Each Deal Card       | SQLite: Check if Notified       | Uses jsdom to parse details; selectors must be updated if site changes.                                                     |
| SQLite: Ensure Table Exists | SQLite              | Creates 'notified_deals' table if not existing      | Parse Deal Details           | SQLite: Check if Notified       | Ensures database table exists for tracking notified deals.                                                                  |
| SQLite: Check if Notified   | SQLite              | Checks if dealUniqueId already notified              | SQLite: Ensure Table Exists, Parse Deal Details | Split into Notified/New         | Checks database for existing deal IDs to avoid duplicate notifications.                                                     |
| Split into Notified/New     | Item Lists          | Splits deals into new and already notified           | SQLite: Check if Notified    | If (New Deals Found)             | Automatically splits items into new vs notified based on DB query results.                                                  |
| If (New Deals Found)        | If                  | Checks if there are any new deals to notify          | Split into Notified/New      | SQLite: Insert New Deals (True), Workflow end (False) | Proceeds only if new deals exist.                                                                                             |
| SQLite: Insert New Deals    | SQLite              | Inserts new deals into the notified_deals database   | If (New Deals Found) True    | Format Notification Message      | Stores new deal details to prevent duplicate alerts.                                                                         |
| Format Notification Message | Function            | Formats deals into readable notification text        | SQLite: Insert New Deals     | Send Email Notification          | Creates an email-friendly message listing all new deals.                                                                     |
| Send Email Notification     | Gmail               | Sends the notification email                          | Format Notification Message  | None                           | Sends email via Gmail API; update recipient email and credentials accordingly.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Node:**  
   - Add a Cron node named “Daily Check (8 AM)”.  
   - Set mode to “Every Day”, Hour to 8, Minute to 0.

2. **Create HTTP Request Node:**  
   - Add “Fetch Deku Deals Page” node.  
   - Connect from “Daily Check (8 AM)”.  
   - Set URL to `https://www.dekudeals.com/most-popular`.  
   - Set Response Format to `String`.

3. **Create HTML Extract Node:**  
   - Add “Extract Each Deal Card” node.  
   - Connect from “Fetch Deku Deals Page”.  
   - Set HTML input to `{{ $node["Fetch Deku Deals Page"].json.data }}`.  
   - Add extract operation: Selector `div.game-card`, Attribute `html`, Property Name `dealHtml`.

4. **Create Function Node for Parsing:**  
   - Add “Parse Deal Details (Function Code)” node.  
   - Connect from “Extract Each Deal Card”.  
   - Paste the provided JavaScript code that uses `jsdom` to extract: `gameTitle`, `gameLink`, `platforms`, `currentPrice`, `originalPrice`, `discount`, and `dealUniqueId`.

5. **Create SQLite Node to Ensure Table:**  
   - Add “SQLite: Ensure Table Exists” node.  
   - Connect from “Parse Deal Details (Function Code)”.  
   - Database name: `dekudeals` (creates `dekudeals.db` locally).  
   - Query:  
     ```sql
     CREATE TABLE IF NOT EXISTS notified_deals (
       deal_id TEXT PRIMARY KEY,
       game_title TEXT,
       platforms TEXT,
       current_price TEXT,
       original_price TEXT,
       discount TEXT,
       deal_link TEXT,
       notified_date TEXT
     )
     ```

6. **Create SQLite Node to Check if Notified:**  
   - Add “SQLite: Check if Notified” node.  
   - Connect from “SQLite: Ensure Table Exists”.  
   - Database: `dekudeals`.  
   - Query:  
     ```sql
     SELECT deal_id FROM notified_deals WHERE deal_id = '{{ $json.dealUniqueId }}'
     ```

7. **Create Item Lists Node:**  
   - Add “Split into Notified/New” node.  
   - Connect main input 1 from “Parse Deal Details (Function Code)” (original parsed deals).  
   - Connect main input 2 from “SQLite: Check if Notified” (query result).  
   - Set mode to split based on presence in database.

8. **Create If Node to Check for New Deals:**  
   - Add “If (New Deals Found)” node.  
   - Connect Output 1 (new deals) from “Split into Notified/New” to this node.  
   - Condition: Check if number of items (`$json.length`) is not zero.

9. **Create SQLite Node to Insert New Deals:**  
   - Add “SQLite: Insert New Deals” node.  
   - Connect from “If (New Deals Found)” True branch.  
   - Database: `dekudeals`.  
   - Query:  
     ```sql
     INSERT INTO notified_deals (deal_id, game_title, platforms, current_price, original_price, discount, deal_link, notified_date)
     VALUES ('{{ $json.dealUniqueId }}', '{{ $json.gameTitle }}', '{{ $json.platforms }}', '{{ $json.currentPrice }}', '{{ $json.originalPrice }}', '{{ $json.discount }}', '{{ $json.gameLink }}', '{{ new Date().toISOString() }}')
     ```

10. **Create Function Node to Format Notification Message:**  
    - Add “Format Notification Message” node.  
    - Connect from “SQLite: Insert New Deals”.  
    - Paste the provided JavaScript code that builds a message string listing each deal with emojis and formatting.

11. **Create Gmail Node to Send Notification:**  
    - Add “Send Email Notification” node.  
    - Connect from “Format Notification Message”.  
    - Select Gmail API credentials (OAuth2).  
    - Set From Email to your Gmail address.  
    - Set To Email to your recipient email (replace placeholder).  
    - Set Subject to `🎮 New Game Deals Alert! (Deku Deals)`.  
    - Map Text field to `{{ $json.notificationMessage }}`.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Deku Deals website structure may change; update selectors in HTML Extract and Function nodes accordingly.    | https://www.dekudeals.com                                                                        |
| To change notification channel, replace Gmail node with Telegram, Slack, or Discord node and configure creds. | n8n documentation on messaging integrations                                                     |
| SQLite database file `dekudeals.db` is created in n8n data directory; ensure file system permissions allow write/read. | SQLite local persistence notes                                                                  |
| Gmail API credential setup requires OAuth2 with appropriate scopes for sending emails.                        | https://developers.google.com/gmail/api/quickstart/js                                            |
| Cron node schedule depends on n8n server timezone; confirm timezone settings to match expected trigger time.  | n8n Cron node documentation                                                                     |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to prevailing content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.