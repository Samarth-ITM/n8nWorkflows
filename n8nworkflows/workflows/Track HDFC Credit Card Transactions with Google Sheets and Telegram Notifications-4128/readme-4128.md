Track HDFC Credit Card Transactions with Google Sheets and Telegram Notifications

https://n8nworkflows.xyz/workflows/track-hdfc-credit-card-transactions-with-google-sheets-and-telegram-notifications-4128


# Track HDFC Credit Card Transactions with Google Sheets and Telegram Notifications

---
### 1. Workflow Overview

This workflow automates the tracking of HDFC Credit Card transactions by extracting transaction details from incoming Gmail alerts and updating a Google Sheets document. It also sends Telegram notifications to two recipients to alert them about new transactions. The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Monitor Gmail inbox for new transaction alert emails from HDFC Bank.
- **1.2 Data Extraction & Filtering:** Parse transaction details from email content and filter out already processed alerts using stored Gmail message IDs.
- **1.3 Data Storage & Deduplication:** Append new transaction data to a Google Sheet and maintain processing state.
- **1.4 Notification Dispatch:** Send formatted Telegram messages to two designated users notifying them about the transaction.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block listens for new transaction alert emails from HDFC Bank using Gmail’s API and triggers the workflow every 5 minutes.
- **Nodes Involved:**  
  - Gmail Trigger  
  - Loop Over Items

- **Node Details:**

  - **Gmail Trigger**
    - Type: Gmail Trigger
    - Role: Watches Gmail inbox for new emails matching filter criteria.
    - Configuration:
      - Filter query: Emails from "HDFC Bank InstaAlerts <alerts@hdfcbank.net>" in the "CATEGORY_UPDATES" label.
      - Poll interval: Every 5 minutes.
      - Credential: Gmail OAuth2.
    - Input: External trigger (email arrival).
    - Output: List of new Gmail messages matching filter.
    - Edge Cases:  
      - Gmail API rate limits or auth token expiry.
      - No new emails triggering no output.
    - Version: 1.2

  - **Loop Over Items**
    - Type: SplitInBatches
    - Role: Processes each email message individually to handle them one-by-one.
    - Configuration: Default batch size (likely 1).
    - Input: List of Gmail messages.
    - Output: Single message item per execution.
    - Edge Cases:  
      - Large batch sizes can overwhelm downstream nodes.
    - Version: 3

#### 1.2 Data Extraction & Filtering

- **Overview:** Extract transaction details from the email snippet and filter out already processed emails using stored Gmail IDs.
- **Nodes Involved:**  
  - Google Sheets (read existing entries)  
  - map used articls ids (Code)  
  - filter only unused Ids  
  - Extract the required data from mail (Code)

- **Node Details:**

  - **Google Sheets (Read)**
    - Type: Google Sheets
    - Role: Reads the existing transaction records to get processed Gmail message IDs.
    - Configuration:
      - Spreadsheet ID: "1fWshqrsS8A0ykEPasbvdH_U5KYqkfMvROXXnsQghHso"
      - Sheet: "Sheet1" (gid=0)
      - Operation: Read all rows (default).
      - Credential: Google Sheets OAuth2.
    - Input: Individual Gmail message from Loop Over Items.
    - Output: All rows in the sheet with transaction data including Gmail IDs.
    - Edge Cases:
      - API quota limits or auth errors.
      - Empty sheet returns no IDs.
    - Version: 4.5

  - **map used articls ids (Code)**
    - Type: Code (JavaScript)
    - Role: Extracts all stored Gmail message IDs from the Google Sheets data.
    - Configuration:
      - Maps all input items to an array of Gmail IDs.
      - Outputs one item containing array of all Gmail IDs.
    - Key expressions:
      - `let values = $input.all().map(item => item.json.gmailId);`
    - Input: Rows from Google Sheets.
    - Output: Object with `values` array of Gmail IDs.
    - Edge Cases:
      - No rows results in empty array.
    - Version: 2

  - **filter only unused Ids**
    - Type: Filter
    - Role: Filters out emails whose Gmail IDs are already present in the sheet.
    - Configuration:
      - Condition: The Gmail ID of the current email (`$('Loop Over Items').item.json.id`) is NOT contained in the array of stored IDs (`$('map used articls ids').item.json.values`).
    - Input: Current Gmail email message.
    - Output: Only messages with new Gmail IDs pass.
    - Edge Cases:
      - Expression failure if dependencies are missing.
      - All emails filtered out results in no further processing.
    - Version: 2.2

  - **Extract the required data from mail (Code)**
    - Type: Code (JavaScript)
    - Role: Parses transaction details such as amount, recipient, date, and UPI reference from the email snippet.
    - Configuration:
      - Uses regex patterns matching the email snippet content.
      - Extracts:
        - Amount (e.g., Rs.1260.00)
        - Recipient (e.g., "redbus3.payu@hdfcbank REDBUS IN")
        - Date (format dd-mm-yy)
        - UPI Reference Number
    - Key expressions:
      - Regex matches like `/Rs\.([\d.,]+)/`
      - Returns JSON with extracted fields.
    - Input: Single Gmail email message.
    - Output: JSON with transaction details.
    - Edge Cases:
      - Missing or malformed fields result in null values.
      - Email format changes may break regex.
    - Version: 2

#### 1.3 Data Storage & Deduplication

- **Overview:** Append newly extracted transaction data to Google Sheets and update state for deduplication.
- **Nodes Involved:**  
  - sent notification to telegram to pavi  
  - update the usage in sheet row

- **Node Details:**

  - **sent notification to telegram to pavi**
    - Type: Telegram
    - Role: Sends a notification about the new transaction to the Telegram user "pavi".
    - Configuration:
      - Message: Formatted with transaction date, amount, recipient, and UPI reference.
      - Chat ID: (Configured via credentials or set dynamically).
      - Credential: Telegram Bot API.
      - Attribution disabled.
    - Input: Extracted transaction data.
    - Output: Triggers next node to update sheet.
    - Edge Cases:
      - Telegram API limits or connectivity issues.
      - Missing chat ID results in failed send.
    - Version: 1.2

  - **update the usage in sheet row**
    - Type: Google Sheets
    - Role: Appends the new transaction data row into the sheet.
    - Configuration:
      - Spreadsheet ID and sheet same as read node.
      - Operation: Append row.
      - Columns mapped from extracted data and Gmail ID.
      - Matching columns: "row_number" (used internally).
      - Credential: Google Sheets OAuth2.
    - Input: Transaction data from previous node.
    - Output: Confirmation of append operation.
    - Edge Cases:
      - API quota or auth errors.
      - Data type mismatches or empty fields.
    - Version: 4.5

#### 1.4 Notification Dispatch

- **Overview:** Sends a Telegram notification about the transaction to a second recipient named "krishna".
- **Nodes Involved:**  
  - sent notification to telegram to krishna

- **Node Details:**

  - **sent notification to telegram to krishna**
    - Type: Telegram
    - Role: Sends a formatted transaction alert to "krishna".
    - Configuration:
      - Same message format as the pavi notification.
      - Credential: Telegram Bot API (same as pavi).
      - Attribution disabled.
    - Input: Transaction data from extraction node.
    - Output: Terminal node (no further connections).
    - Edge Cases:
      - Telegram API errors, missing chat ID.
    - Version: 1.2

---

### 3. Summary Table

| Node Name                    | Node Type                | Functional Role                       | Input Node(s)           | Output Node(s)                        | Sticky Note                     |
|------------------------------|--------------------------|-------------------------------------|------------------------|-------------------------------------|--------------------------------|
| Gmail Trigger                | Gmail Trigger            | Listen for new HDFC credit card mails | -                      | Loop Over Items                     |                                |
| Loop Over Items              | SplitInBatches           | Process each email individually     | Gmail Trigger          | Google Sheets (read), Google Sheets (write) (via chain) |                                |
| Google Sheets (Read)         | Google Sheets            | Read existing processed transaction IDs | Loop Over Items        | map used articls ids                |                                |
| map used articls ids         | Code                     | Extract array of stored Gmail IDs   | Google Sheets (Read)   | filter only unused Ids              |                                |
| filter only unused Ids       | Filter                   | Filter out already processed emails | map used articls ids   | Extract the required data from mail |                                |
| Extract the required data from mail | Code               | Parse transaction details from mail | filter only unused Ids | sent notification to telegram to krishna, sent notification to telegram to pavi |                                |
| sent notification to telegram to krishna | Telegram        | Notify Krishna of transaction       | Extract the required data from mail | -                                 |                                |
| sent notification to telegram to pavi | Telegram            | Notify Pavi of transaction           | Extract the required data from mail | update the usage in sheet row     |                                |
| update the usage in sheet row | Google Sheets           | Append new transaction data to sheet | sent notification to telegram to pavi | Loop Over Items                  |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**
   - Type: Gmail Trigger
   - Set filter query: `from:HDFC Bank InstaAlerts <alerts@hdfcbank.net>`
   - Label IDs: `CATEGORY_UPDATES`
   - Poll every 5 minutes
   - Set Gmail OAuth2 credentials (create or select)
   - Connect output to next node

2. **Create SplitInBatches node ("Loop Over Items")**
   - Type: SplitInBatches
   - Connect Gmail Trigger output to this node
   - Default batch size (1)
   - Connect output to Google Sheets read node

3. **Create Google Sheets node (Read)**
   - Type: Google Sheets
   - Operation: Read rows
   - Document ID: `1fWshqrsS8A0ykEPasbvdH_U5KYqkfMvROXXnsQghHso`
   - Sheet: `Sheet1` (gid=0)
   - Use Google Sheets OAuth2 credentials
   - Connect output to Code node "map used articls ids"

4. **Create Code node ("map used articls ids")**
   - Type: Code
   - JavaScript code:
     ```javascript
     let values = $input.all().map(item => item.json.gmailId);
     return [{ json: { values } }];
     ```
   - Connect output to Filter node "filter only unused Ids"

5. **Create Filter node ("filter only unused Ids")**
   - Add condition:
     - Check that current Gmail ID (from Loop Over Items) is NOT in array from map used articls ids.
     - Expression example for left: `={{ $('map used articls ids').item.json.values }}`
     - Right: `={{ $('Loop Over Items').item.json.id }}`
     - Operator: Array "does not contain"
   - Connect output to next Code node "Extract the required data from mail"

6. **Create Code node ("Extract the required data from mail")**
   - JavaScript code to parse transaction details from email snippet:
     ```javascript
     const message = $('Loop Over Items').first().json.snippet;

     const amountMatch = message.match(/Rs\.([\d.,]+)/);
     const recipientMatch = message.match(/to (.+?) on/);
     const dateMatch = message.match(/on (\d{2}-\d{2}-\d{2})/);
     const referenceMatch = message.match(/UPI transaction reference number is (\d+)/);

     return [{
       json: {
         Amount: amountMatch ? amountMatch[1] : null,
         Recipient: recipientMatch ? recipientMatch[1].trim() : null,
         Date: dateMatch ? dateMatch[1] : null,
         UpiReference: referenceMatch ? referenceMatch[1] : null
       }
     }];
     ```
   - Connect output to two Telegram notification nodes

7. **Create Telegram node ("sent notification to telegram to krishna")**
   - Type: Telegram
   - Message text:
     ```
     💳 *Transaction Alert*

     📅 *Date:* `{{ $json.Date }}`
     💰 *Amount:* `₹{{ $json.Amount }}`
     🏷️ *Recipient:* `{{ $json.Recipient }}`
     🔢 *UPI Reference No:* `{{ $json.UpiReference }}`
     ```
   - Set Chat ID for Krishna (must be configured or set dynamically)
   - Disable attribution
   - Use Telegram Bot API credentials (Reminder Bot API)
   - No further output connections

8. **Create Telegram node ("sent notification to telegram to pavi")**
   - Same configuration as above, but with Pavi’s chat ID
   - Connect output to Google Sheets append node

9. **Create Google Sheets node ("update the usage in sheet row")**
   - Type: Google Sheets
   - Operation: Append row
   - Document ID and Sheet same as read node
   - Map columns:
     - date: `={{ $('Extract the required data from mail').first().json.Date }}`
     - amount: `={{ $('Extract the required data from mail').first().json.Amount }}`
     - gmailId: `={{ $('Loop Over Items').first().json.id }}`
     - Recipient: `={{ $('Extract the required data from mail').first().json.Recipient }}`
     - upiReference: `={{ $('Extract the required data from mail').first().json.UpiReference }}`
   - Use Google Sheets OAuth2 credentials
   - Connect output back to "Loop Over Items" to continue batch processing

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Telegram message formatting uses Markdown for bold and code styles.                                      | Telegram API message formatting documentation: https://core.telegram.org/bots/api#formatting-options |
| Gmail filter query uses standard Gmail search operators to target specific sender and label.             | Gmail search operators documentation: https://support.google.com/mail/answer/7190?hl=en             |
| Google Sheets node requires OAuth2 credentials with read/write access to the specific spreadsheet used.  | Google Sheets API documentation: https://developers.google.com/sheets/api                            |
| Regex extraction depends on the email format from HDFC Bank and may require updates if email format changes. | Monitor email content changes periodically to adjust regex patterns accordingly.                    |
| Telegram Bot API quota and chat IDs must be correctly configured to ensure notifications are delivered.  | Telegram Bot documentation: https://core.telegram.org/bots/api                                     |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created using n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.