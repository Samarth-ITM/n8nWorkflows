Competitor Price Monitoring with Web Scraping, Google Sheets & Discord Alerts

https://n8nworkflows.xyz/workflows/competitor-price-monitoring-with-web-scraping--google-sheets---discord-alerts-6179


# Competitor Price Monitoring with Web Scraping, Google Sheets & Discord Alerts

### 1. Workflow Overview

This n8n workflow automates competitor price monitoring by scraping competitor product pages, comparing prices against internal records stored in Google Sheets, and issuing alerts on Discord when competitor prices are lower. It is designed for daily scheduled checks to identify pricing changes that may impact competitive positioning.

**Target Use Cases:**  
- E-commerce teams tracking competitor prices for dynamic pricing strategies.  
- Retailers wanting automated alerts on competitor price drops.  
- Businesses integrating Google Sheets as data storage with Discord notifications.

**Logical Blocks:**  
- **1.1 Scheduled Trigger:** Initiates the workflow on a daily schedule.  
- **1.2 Data Retrieval:** Fetches unchecked product rows from Google Sheets (rows where competitor prices have not been checked).  
- **1.3 Price Scraping & Parsing:** Requests competitor product pages, extracts price HTML, converts price strings to numeric values.  
- **1.4 Price Comparison & Alerts:** Compares competitor prices with internal prices, sends Discord alerts if competitor price is lower.  
- **1.5 Status Update:** Updates Google Sheets to mark rows as checked or resets checked status for the next cycle.  
- **1.6 Workflow Completion Notification:** Sends a completion message to Discord when all checks are done and resets check flags for the next run.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** Starts the workflow automatically on a daily interval to trigger price checks.  
- **Nodes Involved:**  
  - Schedule Trigger  
  - Sticky Note2 (comment)  
- **Node Details:**  
  - **Schedule Trigger:**  
    - Type: Schedule Trigger  
    - Configuration: Interval set to daily (default empty interval object implies 24-hour repeat)  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "get unchecked row" node  
    - Failure Modes: Misconfiguration of schedule; no output if disabled  
  - **Sticky Note2:**  
    - Purpose: Annotates that the trigger should start at the desired schedule, daily in this example.

#### 2.2 Data Retrieval (Fetching Unchecked Rows)

- **Overview:** Queries Google Sheets to retrieve the first product row where the 'checked' column is "0", indicating the competitor price has not been checked yet.  
- **Nodes Involved:**  
  - get unchecked row (Google Sheets)  
  - row exist? (If)  
  - Sticky Note4  
  - Sticky Note7  
- **Node Details:**  
  - **get unchecked row:**  
    - Type: Google Sheets  
    - Configuration: Reads from a specific Google Sheet (ID: 16-hEaIl8Tng5SB5jbpu26kT7G-g1cFM5_w2AILwT3Pc, Sheet1)  
    - Filters: "checked" column equals "0"  
    - Returns only the first matching row  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: To "row exist?"  
    - Credentials: Google Sheets OAuth2 account required  
    - Failure Modes: API failures, no matching rows (empty output)  
  - **row exist?:**  
    - Type: If  
    - Condition: Checks if input JSON data is not empty (row exists)  
    - True output: Proceeds with HTTP request to product page  
    - False output: Sends "Price checks complete" Discord message and ends cycle  
  - **Sticky Notes:**  
    - Note4 highlights that this node retrieves unchecked rows with product name, URL, and our price.  
    - Note7 explains the logic branch: if unchecked row exists, extract data; else send completion message and reset flags.

#### 2.3 Price Scraping & Parsing

- **Overview:** Requests the competitor product page URL, extracts the price from HTML using CSS selectors, and converts the price string into a numeric value for comparison.  
- **Nodes Involved:**  
  - HTTP Request to product page  
  - price extract from page (HTML Extract)  
  - Code to convert price into number (Code)  
  - Sticky Note3  
- **Node Details:**  
  - **HTTP Request to product page:**  
    - Type: HTTP Request  
    - URL: Taken dynamically from the "Competitor URL" field of the unchecked row  
    - Inputs: From "row exist?" true branch  
    - Outputs: HTML content to "price extract from page"  
    - Failure Modes: HTTP errors, timeouts, invalid URLs, connection issues  
  - **price extract from page:**  
    - Type: HTML Extract  
    - Operation: Extract HTML content using CSS selector `.price_color` to fetch the competitor price string  
    - Inputs: HTML response from HTTP Request  
    - Outputs: JSON with 'price' string to Code node  
    - Failure Modes: Selector mismatch, empty extraction if page structure changes  
  - **Code to convert price into number:**  
    - Type: Code  
    - JavaScript logic: Removes currency symbol (£), trims string, parses to float  
    - Outputs: JSON with numeric 'price' field  
    - Failure Modes: Parsing errors if input format unexpected  
  - **Sticky Note3:**  
    - Describes that this block obtains competitor price from product page and converts it into a number.

#### 2.4 Price Comparison & Alerts

- **Overview:** Compares the competitor price with the internal price stored in Google Sheets. If the competitor price is lower, sends an alert to Discord and updates the status in Google Sheets. Otherwise, only updates status without alert.  
- **Nodes Involved:**  
  - price greater than ours? (If)  
  - Discord price alert  
  - Update status to 1 (Google Sheets)  
  - Sticky Note5  
  - Sticky Note6  
- **Node Details:**  
  - **price greater than ours?:**  
    - Type: If  
    - Condition: Checks if competitor price < internal price (our price)  
    - True: Sends Discord price alert  
    - False: Proceeds to update status to 1 without alert  
    - Inputs: From Code node output  
    - Outputs: Two branches (alert or no alert)  
  - **Discord price alert:**  
    - Type: Discord webhook  
    - Configuration: Sends message with product name, our price, competitor price  
    - Inputs: True branch of If node  
    - Failure Modes: Webhook errors, authentication issues  
    - Credentials: Discord Webhook API credentials required  
  - **Update status to 1:**  
    - Type: Google Sheets  
    - Operation: Update row for the product, setting 'checked' to "1" and updating 'competitor price' with scraped value  
    - Inputs: From Discord alert or price comparison false branch (to ensure status updates regardless of alert)  
    - Failure Modes: API update errors, concurrency conflicts  
  - **Sticky Notes:**  
    - Note5 explains the Discord alert logic for when our price is greater than competition.  
    - Note6 highlights the setting of 'checked' to 1 after processing.

#### 2.5 Status Reset & Workflow Completion Notification

- **Overview:** After all unchecked rows are processed, this block resets the 'checked' column for the next day's checks and sends a completion notification to Discord.  
- **Nodes Involved:**  
  - Get status row (Google Sheets)  
  - row returned? (If)  
  - Update status back to 0 (Google Sheets)  
  - Discord1 (Discord webhook)  
  - Sticky Note1  
  - Sticky Note  
- **Node Details:**  
  - **Get status row:**  
    - Type: Google Sheets  
    - Operation: Reads the first row where 'checked' equals "1" (processed rows)  
    - Inputs: From the Update status to 1 node (looped back) and from Discord1  
    - Outputs: To "row returned?"  
  - **row returned?:**  
    - Type: If  
    - Condition: Checks if any row with 'checked' = 1 exists  
    - True: Resets 'checked' to "0" for that row (Update status back to 0)  
    - False: Ends workflow (no rows to reset)  
  - **Update status back to 0:**  
    - Type: Google Sheets  
    - Operation: Updates row to set 'checked' column back to "0" to enable next day's checks  
    - Inputs: From row returned? true branch  
  - **Discord1:**  
    - Type: Discord webhook  
    - Sends a message "Price checks complete." indicating workflow completion  
    - Inputs: From row exist? false branch (no unchecked rows) and from Update status back to 0 node  
  - **Sticky Notes:**  
    - Note1 refers to the daily completion message sent to Discord.  
    - Note (unnumbered) describes setting checked column back to "0" for next day checks.

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                        | Input Node(s)              | Output Node(s)                      | Sticky Note                                  |
|-----------------------------|--------------------|-------------------------------------|----------------------------|-----------------------------------|----------------------------------------------|
| Schedule Trigger             | Schedule Trigger   | Starts workflow daily                | None                       | get unchecked row                 | start at desired schedule; daily in example  |
| get unchecked row           | Google Sheets      | Fetch first unchecked product row   | Schedule Trigger            | row exist?                       | get un checked row: extract name, url, price |
| row exist?                  | If                 | Checks if unchecked row exists      | get unchecked row           | HTTP Request to product page / Discord1 | if unchecked row exists, process; else finish |
| HTTP Request to product page| HTTP Request       | Fetch competitor product page HTML  | row exist? true branch      | price extract from page           | get price from competitor page and convert   |
| price extract from page     | HTML Extract       | Extract price string from HTML       | HTTP Request to product page| Code to convert price into number | get price from competitor page and convert   |
| Code to convert price into number | Code          | Convert price string to number       | price extract from page     | price greater than ours?           | get price from competitor page and convert   |
| price greater than ours?    | If                 | Compare competitor price to ours     | Code to convert price       | Discord price alert / Update status to 1 | discord alert when competitor price lower    |
| Discord price alert         | Discord Webhook    | Send alert if competitor price lower| price greater than ours?    | Update status to 1                | discord alert: our price greater than competition |
| Update status to 1          | Google Sheets      | Mark row as checked, update competitor price | Discord price alert / price greater than ours? false branch | get unchecked row | set checked to 1                              |
| Get status row              | Google Sheets      | Get rows with checked=1              | Update status to 1          | row returned?                    |                                              |
| row returned?               | If                 | Check if any rows are checked=1     | Get status row              | Update status back to 0           |                                              |
| Update status back to 0     | Google Sheets      | Reset checked column to 0            | row returned? true branch   | Get status row / Discord1         | setting checked column to "0" for next day   |
| Discord1                   | Discord Webhook    | Send "Price checks complete" message| row exist? false branch / Update status back to 0 | Get status row | price checks completed message to discord   |
| Sticky Note                 | Sticky Note        | Notes and comments                   | None                       | None                            | setting checked column to "0" for next day checks |
| Sticky Note1                | Sticky Note        | Notes and comments                   | None                       | None                            | price checks completed message to discord     |
| Sticky Note2                | Sticky Note        | Notes and comments                   | None                       | None                            | start at desired schedule; daily in example  |
| Sticky Note3                | Sticky Note        | Notes and comments                   | None                       | None                            | get price from competitor's product page and convert into number |
| Sticky Note4                | Sticky Note        | Notes and comments                   | None                       | None                            | get un checked roe - extract name, page-url, our price |
| Sticky Note5                | Sticky Note        | Notes and comments                   | None                       | None                            | discord alert - our price is greater than competition |
| Sticky Note6                | Sticky Note        | Notes and comments                   | None                       | None                            | set checked to 1                              |
| Sticky Note7                | Sticky Note        | Notes and comments                   | None                       | None                            | if unchecked row exist? yes? (then extracts data) else sends message checks are done and set checks to 0 for next day checks |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set interval to daily (every 24 hours)  
   - Connect output to "get unchecked row" node

2. **Create Google Sheets Node "get unchecked row":**  
   - Operation: Read rows  
   - Google Sheets document ID: `16-hEaIl8Tng5SB5jbpu26kT7G-g1cFM5_w2AILwT3Pc`  
   - Sheet: `Sheet1` (gid=0)  
   - Filter: `checked` column equals `"0"`  
   - Return first match only (limit to 1 row)  
   - Connect input from Schedule Trigger  
   - Output to "row exist?" node  
   - Use Google Sheets OAuth2 credentials

3. **Create If Node "row exist?":**  
   - Condition: Check if input JSON is not empty  
   - True output: Connect to "HTTP Request to product page"  
   - False output: Connect to Discord node that sends "Price checks complete" message

4. **Create HTTP Request Node "HTTP Request to product page":**  
   - URL: Use expression `{{$json["Competitor URL"]}}` to dynamically fetch URL from the unchecked row  
   - Method: GET (default)  
   - Connect output to "price extract from page" node

5. **Create HTML Extract Node "price extract from page":**  
   - Operation: Extract HTML content  
   - CSS Selector: `.price_color` to extract competitor price string  
   - Connect output to "Code to convert price into number"

6. **Create Code Node "Code to convert price into number":**  
   - Language: JavaScript  
   - Code snippet:  
     ```javascript
     const priceStr = $input.first().json.price;
     const priceNumber = parseFloat(priceStr.replace("£", "").trim());
     return [{ json: { price: priceNumber } }];
     ```  
   - Connect output to "price greater than ours?" node

7. **Create If Node "price greater than ours?":**  
   - Condition: Check if competitor price `<` our price  
   - Left value: `{{$json.price}}` (from code node)  
   - Right value: `{{$node["get unchecked row"].json["my Price"]}}`  
   - True output: Connect to "Discord price alert"  
   - False output: Connect directly to "Update status to 1"

8. **Create Discord Node "Discord price alert":**  
   - Use Discord Webhook credentials  
   - Content:  
     ```
     competitor has decreased price on product: {{$node["get unchecked row"].json["Product Name"]}}

     our price: {{$node["get unchecked row"].json["my Price"]}}
     competitor's price: {{$node["Code to convert price into number"].json.price}}
     ```  
   - Connect output to "Update status to 1"

9. **Create Google Sheets Node "Update status to 1":**  
   - Operation: Update row  
   - Document & Sheet: Same as above  
   - Update 'checked' field to `"1"`  
   - Update 'competitor price' with the new price from competitor  
   - Matching column: "Product Name"  
   - Connect output back to "get unchecked row" to continue loop until no unchecked rows  
   - Use Google Sheets OAuth2 credentials

10. **Create Google Sheets Node "Get status row":**  
    - Operation: Read rows  
    - Filter: `checked` equals `"1"`  
    - Return first match only  
    - Connect input from "Update status to 1" output and from Discord completion node output (see below)  
    - Output to "row returned?"

11. **Create If Node "row returned?":**  
    - Condition: Check if input JSON is not empty  
    - True output: Connect to "Update status back to 0"  
    - False output: End workflow or no further action

12. **Create Google Sheets Node "Update status back to 0":**  
    - Operation: Update row  
    - Set 'checked' to `"0"` for the row with 'Product Name' equal to the returned row  
    - Matching column: "Product Name"  
    - Connect output to "Get status row" (to iterate if multiple rows) and to "Discord1"

13. **Create Discord Node "Discord1":**  
    - Use Discord Webhook credentials  
    - Content: `"Price checks complete."`  
    - Connect input from "row exist?" false branch (no unchecked rows) and from "Update status back to 0"

14. **Create If Node "row exist?" false branch to Discord1:**  
    - To send completion message immediately if no unchecked rows found

15. **Add Sticky Notes:**  
    - Around schedule trigger to explain daily execution  
    - Near "get unchecked row" to describe the purpose of fetching unchecked data  
    - Near price extraction nodes to describe scraping and parsing  
    - Near price comparison to explain alert logic  
    - Near status update nodes to explain marking rows checked/un-checked  
    - Near Discord nodes to clarify notification purpose

**Credentials Required:**  
- Google Sheets OAuth2 (with read/write access to the target spreadsheet)  
- Discord Webhook API (for sending alerts and completion messages)

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow relies on consistent sheet structure: columns 'Product Name', 'my Price', 'Competitor URL', 'checked', and 'competitor price'. | Google Sheets setup                                                                              |
| Discord Webhook URLs must be created in the Discord server and configured with appropriate permissions to send messages.                     | Discord Webhook setup                                                                           |
| CSS Selector `.price_color` is critical and depends on competitor website HTML structure; update if site changes.                            | Web Scraping selector                                                                            |
| Schedule Trigger interval can be adjusted for more frequent or less frequent price checks as needed.                                           | Scheduling flexibility                                                                          |
| Possible edge cases include empty competitor URLs, HTTP request failures, Google Sheets API rate limits, and Discord webhook downtime.          | Error handling considerations                                                                  |
| Using ‘returnFirstMatch’ for Google Sheets nodes ensures the workflow processes one product at a time, enabling controlled looping.             | Workflow design detail                                                                          |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.