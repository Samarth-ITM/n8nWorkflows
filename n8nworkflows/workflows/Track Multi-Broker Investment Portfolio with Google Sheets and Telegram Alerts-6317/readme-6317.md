Track Multi-Broker Investment Portfolio with Google Sheets and Telegram Alerts

https://n8nworkflows.xyz/workflows/track-multi-broker-investment-portfolio-with-google-sheets-and-telegram-alerts-6317


# Track Multi-Broker Investment Portfolio with Google Sheets and Telegram Alerts

### 1. Workflow Overview

This n8n workflow is designed to track investment portfolios across multiple brokers or platforms by aggregating data stored in a Google Sheet and delivering formatted Profit & Loss (P&L) reports via Telegram. It supports both on-demand queries through Telegram commands and scheduled automatic updates, enabling investors and portfolio managers to monitor their investments in real-time or at preset intervals.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Authentication:** Receives Telegram messages and verifies the authorized user.
- **1.2 Command Routing:** Determines whether the request is for a specific broker/platform or the total portfolio.
- **1.3 Data Retrieval:** Queries Google Sheets for portfolio data—either the entire dashboard or filtered by platform.
- **1.4 Data Aggregation and Formatting:** Processes raw data, aggregates totals when necessary, and formats messages for Telegram.
- **1.5 Output Dispatch:** Sends the formatted P&L summary back to the user's Telegram chat.
- **1.6 Scheduled Automatic Updates:** Triggers automatic portfolio summaries at configured times (10 AM and 4 PM).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Authentication

- **Overview:**  
  This block listens for incoming Telegram messages and filters them to ensure only authorized users receive portfolio updates.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Edit Fields  
  - If

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger node for Telegram messages  
    - Configuration: Watches for new messages (`updates`: "message"), restricted to a specific Telegram chat ID (`chatIds`).  
    - Inputs: External Telegram messages  
    - Outputs: Telegram message JSON payload  
    - Edge Cases: Unauthorized users sending commands; malformed messages  
    - Notes: Requires Telegram API credentials

  - **Edit Fields**  
    - Type: Set node  
    - Configuration: Sets a static variable `myId` with the authorized Telegram chat ID for comparison.  
    - Inputs: Telegram Trigger output  
    - Outputs: Passes original data plus `myId` field  
    - Edge Cases: Incorrect chat ID setting can block legit users

  - **If**  
    - Type: Conditional node  
    - Configuration: Checks if incoming message's chat ID equals `myId`. Only passes true branch if authorized.  
    - Inputs: Output from Edit Fields  
    - Outputs: True branch to proceed, false branch stops workflow  
    - Edge Cases: Case sensitivity or type mismatch in comparison could cause false negatives

---

#### 2.2 Command Routing

- **Overview:**  
  Routes the execution depending on the Telegram command text: either a specific platform/broker update or a total portfolio summary.

- **Nodes Involved:**  
  - Switch

- **Node Details:**

  - **Switch**  
    - Type: Router node  
    - Configuration: Two outputs:  
      - "Specific Platform": When the message text is anything except `/total`  
      - "Total": When the message text equals `/total` exactly  
    - Inputs: True branch output from If node  
    - Outputs:  
      - "Specific Platform" → Fetch platform-specific data  
      - "Total" → Fetch total portfolio data  
    - Edge Cases: Commands must match exactly; case sensitivity enforced; unexpected commands are routed to platform path

---

#### 2.3 Data Retrieval

- **Overview:**  
  Fetches portfolio data from Google Sheets—either filtered by platform or entire dashboard depending on the command.

- **Nodes Involved:**  
  - Get row(s) in sheet2 (platform-specific)  
  - Get row(s) in sheet (total portfolio)

- **Node Details:**

  - **Get row(s) in sheet2**  
    - Type: Google Sheets node (read rows)  
    - Configuration: Filters rows where the "Platform" column matches the Telegram message text after removing slashes (e.g., `/Robinhood` → `Robinhood`).  
    - Inputs: Switch node "Specific Platform" output  
    - Credentials: Google Sheets OAuth2 (user must configure)  
    - Outputs: Rows filtered by platform for further processing  
    - Edge Cases: Platform name mismatches, empty results, Google Sheets API errors

  - **Get row(s) in sheet**  
    - Type: Google Sheets node (read all rows)  
    - Configuration: Retrieves all rows from the sheet named "Dashboard" to aggregate total portfolio data.  
    - Inputs: Switch node "Total" output and scheduled trigger (see below)  
    - Credentials: Google Sheets OAuth2  
    - Outputs: All rows for aggregation  
    - Edge Cases: Large datasets causing timeouts, sheet access errors

---

#### 2.4 Data Aggregation and Formatting

- **Overview:**  
  Processes the fetched data, formats it into an informative P&L report string with currency and percentage formatting for Telegram.

- **Nodes Involved:**  
  - Aggregate  
  - Code (total portfolio formatting)  
  - Code1 (platform-specific formatting)

- **Node Details:**

  - **Aggregate**  
    - Type: Aggregate node  
    - Configuration: Aggregates all item data into a single array for processing.  
    - Inputs: Output from "Get row(s) in sheet" (total portfolio)  
    - Outputs: Aggregated data array for code formatting  
    - Edge Cases: Empty data sets, aggregation failures

  - **Code**  
    - Type: Function (JavaScript) node  
    - Configuration:  
      - Iterates over each row in total portfolio data except "Total" row initially.  
      - Formats invested amount, P&L, P&L change, and current value with Indian Rupee symbol and two decimals.  
      - Appends a summary for the "Total" row with overall portfolio statistics.  
      - Returns a JSON object with `chat_id`, formatted `text` (Markdown), and `parse_mode`.  
    - Inputs: Aggregate node output  
    - Outputs: Formatted message JSON for total portfolio update  
    - Edge Cases: Data missing expected columns, numeric conversion errors

  - **Code1**  
    - Type: Function (JavaScript) node  
    - Configuration:  
      - Processes single platform row fetched from sheet2.  
      - Formats invested amount, P&L, daily change, and current value with currency and percentage.  
      - Returns JSON message for Telegram.  
    - Inputs: Output from "Get row(s) in sheet2"  
    - Outputs: Formatted message JSON for platform-specific update  
    - Edge Cases: Empty or missing platform data rows

---

#### 2.5 Output Dispatch

- **Overview:**  
  Sends the formatted P&L messages to the authorized Telegram chat.

- **Nodes Involved:**  
  - Broker PNL Update  
  - Total PNL Update

- **Node Details:**

  - **Broker PNL Update**  
    - Type: Telegram node (message sending)  
    - Configuration: Sends the message text generated by Code1 to the chat ID also from Code1 output.  
    - Inputs: Code1 output  
    - Credentials: Telegram API OAuth2  
    - Edge Cases: Telegram API rate limits, invalid chat IDs

  - **Total PNL Update**  
    - Type: Telegram node (message sending)  
    - Configuration: Sends message text generated by Code to the chat ID from Code output.  
    - Inputs: Code output  
    - Credentials: Telegram API  
    - Edge Cases: Same as Broker PNL Update

---

#### 2.6 Scheduled Automatic Updates

- **Overview:**  
  Automatically triggers a total portfolio summary update every day at 10 AM and 4 PM (server time).

- **Nodes Involved:**  
  - Auto Update at 10AM and 4PM  
  - Get row(s) in sheet  
  - Aggregate  
  - Code  
  - Total PNL Update

- **Node Details:**

  - **Auto Update at 10AM and 4PM**  
    - Type: Schedule Trigger  
    - Configuration: Triggers workflow execution at 10:00 and 16:00 hours daily.  
    - Outputs: Starts data retrieval for full portfolio update  
    - Edge Cases: Server timezone differences affecting trigger time

  - The chain continues as in blocks 2.3 to 2.5 for data retrieval, aggregation, formatting, and dispatch.

---

### 3. Summary Table

| Node Name                | Node Type                 | Functional Role                         | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                  |
|--------------------------|---------------------------|---------------------------------------|-------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Telegram Trigger          | telegramTrigger           | Receives Telegram commands             | (External)              | Edit Fields                | Secure, only authorized chat ID allowed                                                      |
| Edit Fields              | set                       | Stores authorized Telegram chat ID     | Telegram Trigger        | If                         | Configure your Telegram Chat ID here                                                         |
| If                       | if                        | Filters messages by authorized user    | Edit Fields             | Switch                     |                                                                                              |
| Switch                   | switch                    | Routes commands: platform or total     | If                      | Get row(s) in sheet2, Get row(s) in sheet |                                                                                              |
| Get row(s) in sheet2     | googleSheets              | Fetch platform-specific portfolio data | Switch                  | Code1                      |                                                                                              |
| Get row(s) in sheet      | googleSheets              | Fetch full portfolio data               | Switch, Auto Update at 10AM and 4PM | Aggregate                  | [Google Sheet Link](https://docs.google.com/spreadsheets/d/1dakq9EhU8GrDgBsk82KvAen0N1P3FySAwNHFtG2lsLI/edit?usp=sharing) |
| Aggregate                | aggregate                 | Aggregate full portfolio data           | Get row(s) in sheet     | Code                       |                                                                                              |
| Code                     | code                      | Format total portfolio message          | Aggregate               | Total PNL Update            |                                                                                              |
| Code1                    | code                      | Format platform-specific message        | Get row(s) in sheet2    | Broker PNL Update           |                                                                                              |
| Broker PNL Update        | telegram                  | Send platform-specific P&L report       | Code1                   | (end)                      |                                                                                              |
| Total PNL Update         | telegram                  | Send total portfolio P&L report          | Code                    | (end)                      |                                                                                              |
| Auto Update at 10AM and 4PM | scheduleTrigger          | Schedule daily automatic portfolio update | (timer)                 | Get row(s) in sheet        |                                                                                              |
| Sticky Note              | stickyNote                | Documentation and overview notes        | (none)                  | (none)                     | **All-in-One Portfolio Tracker & Telegram Finance Updates Workflow for n8n: Multi-Broker, Real-Time, Global 🚀** |
| Sticky Note1             | stickyNote                | Easy setup instructions                  | (none)                  | (none)                     | Easy Setup Steps including Google Sheet config and Telegram chat ID setup                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Telegram Trigger node**  
   - Set to listen for "message" updates.  
   - Restrict to your Telegram chat ID in `chatIds`.  
   - Connect your Telegram API credentials.

2. **Add a Set node named "Edit Fields"**  
   - Create a string field `myId` with your authorized Telegram chat ID.  
   - Connect Telegram Trigger's output to this node.

3. **Add an If node named "If"**  
   - Condition: Check if incoming message chat ID equals `myId`.  
   - Input: Output of Edit Fields node.  
   - Output True branch connects to Switch node.

4. **Add a Switch node named "Switch"**  
   - Add two outputs:  
     - "Specific Platform": When Telegram message text ≠ `/total`.  
     - "Total": When Telegram message text = `/total`.  
   - Input: True branch of If node.

5. **Add a Google Sheets node named "Get row(s) in sheet2"**  
   - Set to read rows from your Google Sheet (document & sheet ID).  
   - Add filter: Column "Platform" equals Telegram message text stripped of slashes.  
   - Input: "Specific Platform" output of Switch node.  
   - Connect Google Sheets OAuth2 credentials.

6. **Add a Google Sheets node named "Get row(s) in sheet"**  
   - Set to read all rows from your Google Sheet "Dashboard" tab.  
   - Input: "Total" output from Switch node and the scheduled trigger (to be added).  
   - Connect Google Sheets OAuth2 credentials.

7. **Add an Aggregate node**  
   - Configure to aggregate all item data into one.  
   - Input: Output of "Get row(s) in sheet".

8. **Add two Code nodes:**

   - **Code1 (Platform formatting):**  
     - Input: Output of "Get row(s) in sheet2".  
     - JavaScript code to format platform-specific P&L message with currency ₹ and percentages.  
     - Include `chat_id` from Edit Fields node and output text in Markdown.

   - **Code (Total formatting):**  
     - Input: Output of Aggregate node.  
     - JavaScript code to iterate over all platforms except "Total" row, format each, then add the "Total" portfolio summary.  
     - Output `chat_id` and Markdown-formatted text.

9. **Add two Telegram nodes to send messages:**

   - **Broker PNL Update:**  
     - Input: Output of Code1 node.  
     - Send message text to Telegram chat ID in JSON.  
     - Connect Telegram API credentials.

   - **Total PNL Update:**  
     - Input: Output of Code node.  
     - Send message text to Telegram chat ID in JSON.  
     - Connect Telegram API credentials.

10. **Add a Schedule Trigger node named "Auto Update at 10AM and 4PM"**  
    - Configure to trigger at 10:00 and 16:00 hours daily.  
    - Connect output to "Get row(s) in sheet" node for automatic total portfolio updates.

11. **Connect all nodes as follows:**  
    - Telegram Trigger → Edit Fields → If → Switch  
    - Switch →  
      - "Specific Platform" → Get row(s) in sheet2 → Code1 → Broker PNL Update  
      - "Total" → Get row(s) in sheet → Aggregate → Code → Total PNL Update  
    - Schedule Trigger → Get row(s) in sheet → Aggregate → Code → Total PNL Update

12. **Credentials Setup:**  
    - Google Sheets OAuth2 credentials with access to your portfolio sheet.  
    - Telegram API credentials linked to your bot.

13. **Default values and constraints:**  
    - Ensure Telegram chat ID in Edit Fields matches your authorized ID.  
    - Google Sheet must have a "Dashboard" sheet with columns: Platform, Invested Amount, PNL, PNL %, PNL Change, PNL Change %.  
    - Platform names in the sheet must match Telegram command inputs exactly (case-sensitive).  
    - Text messages are formatted in Markdown for Telegram.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| All-in-One Portfolio Tracker & Telegram Finance Updates Workflow for n8n: Multi-Broker, Real-Time, Global 🚀<br>Tracks investments across multiple brokers with scheduled and on-demand Telegram updates. Supports customization for any country or market.<br><br>Google Sheet Example: https://docs.google.com/spreadsheets/d/1dakq9EhU8GrDgBsk82KvAen0N1P3FySAwNHFtG2lsLI/edit?usp=sharing | Workflow overview and design rationale included in Sticky Note node                                                    |
| Easy Setup Steps:<br>1. Copy the workflow JSON to n8n.<br>2. Configure Google Sheet with broker/platform rows.<br>3. Set your Telegram chat ID.<br>4. Adjust schedule times as needed.<br>5. Use Telegram commands `/total` or `/BrokerName` for updates.                                                                                                                                 | Setup instructions from Sticky Note1 node                                                                              |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.