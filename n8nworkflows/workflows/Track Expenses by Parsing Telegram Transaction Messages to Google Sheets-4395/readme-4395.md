Track Expenses by Parsing Telegram Transaction Messages to Google Sheets

https://n8nworkflows.xyz/workflows/track-expenses-by-parsing-telegram-transaction-messages-to-google-sheets-4395


# Track Expenses by Parsing Telegram Transaction Messages to Google Sheets

### 1. Workflow Overview

This workflow automates tracking of expense transactions by parsing transactional messages received via Telegram and appending the extracted data into a Google Sheets spreadsheet. It is designed for personal or organizational finance management where transaction notifications are received as text messages on Telegram.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Captures incoming Telegram messages containing transaction details.
- **1.2 Transaction Validation:** Filters messages to process only those containing relevant transaction data.
- **1.3 Transaction Parsing and Data Storage:** Extracts structured data from diverse transaction message formats using custom JavaScript code and appends the parsed data to a predefined Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for new messages arriving in a specific Telegram account and triggers the workflow execution.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* `telegramTrigger`  
    - *Role:* Entry point that listens for incoming Telegram updates, specifically messages.  
    - *Configuration:* Listens for the "message" update type; no additional filters set.  
    - *Key Expressions:* N/A (receives raw Telegram message JSON)  
    - *Input Connections:* None (trigger node)  
    - *Output Connections:* Connected to `If` node for filtering  
    - *Version Requirements:* Version 1.2 or later recommended for webhook stability  
    - *Potential Failures:* Telegram API connectivity issues, webhook misconfiguration, credential expiration or revocation  
    - *Credentials:* Uses configured Telegram API credentials for authentication

#### 1.2 Transaction Validation

- **Overview:**  
  Filters incoming Telegram messages to proceed only if the message text contains an equal sign `"="`, which is indicative of a transaction message format.

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - *Type:* `if` node  
    - *Role:* Conditional gate to verify message relevance before parsing.  
    - *Configuration:* Checks if the incoming message text (`{{$json.message.text}}`) contains the character `"="`.  
    - *Key Expressions:* `{{$json.message.text}}` contains `"="`  
    - *Input Connections:* From `Telegram Trigger`  
    - *Output Connections:* Main output to `transactions` node (code parser) if condition is true; no action if false  
    - *Version Requirements:* Version 2.2 used, which supports advanced string contains operator  
    - *Potential Failures:* Expression evaluation errors if message text is missing or malformed; edge case messages without "=" but still valid transactions will be excluded  
    - *Notes:* The use of `"="` as a filter is a heuristic and may miss some transaction messages or include irrelevant ones.

#### 1.3 Transaction Parsing and Data Storage

- **Overview:**  
  Parses Telegram message text using multiple regex patterns tailored for different transaction message formats to extract structured fields such as Amount, Merchant, Card Ending, Date, Time, and Available Limit. It then appends this structured data as a new row into a Google Sheets spreadsheet.

- **Nodes Involved:**  
  - transactions (Code)  
  - Google Sheets

- **Node Details:**

  - **transactions (Code)**  
    - *Type:* `code` (JavaScript)  
    - *Role:* Extracts structured transaction data from raw text using regex matching.  
    - *Configuration:*  
      - Three regex patterns identify distinct transaction message formats common in Telegram notifications.  
      - The code attempts these regexes sequentially and outputs the first successful match fields.  
      - If no regex matches, outputs an error field with the original text for diagnostics.  
    - *Key Expressions / Variables:*  
      - Input: `message.text` from Telegram data  
      - Variables: `format1`, `format2`, `format3` regexes for parsing various formats  
      - Output: JSON object with extracted fields or error  
    - *Input Connections:* From `If` node (filtered Telegram messages)  
    - *Output Connections:* To `Google Sheets` node for data appending  
    - *Version Requirements:* Version 2 or higher recommended for code node stability  
    - *Potential Failures:*  
      - Regex mismatch leading to error output  
      - Unexpected message formats not covered by regex  
      - Code runtime errors if input data is undefined or malformed  
    - *Notes:* This node is key to adapting the workflow to different transaction message formats by updating regexes.

  - **Google Sheets**  
    - *Type:* `googleSheets`  
    - *Role:* Appends parsed transaction data as a new row in a specific Google Sheets spreadsheet and sheet.  
    - *Configuration:*  
      - Operation: `append`  
      - Document ID: "1tl5OVxvN-eRIPwauSwV4i7mhQrp67tlcWR8Lc6BGQz0"  
      - Sheet Name: "gid=0" (Sheet1)  
      - Columns mapped explicitly: Date, Time, Amount, Merchant, Card Ending, Available Limit  
      - Data types: Stored as strings without conversion attempts  
    - *Key Expressions:* References JSON fields output from `transactions` node  
    - *Input Connections:* From `transactions` node  
    - *Output Connections:* None (end of workflow)  
    - *Version Requirements:* Version 4.6 to support latest Google Sheets API features  
    - *Potential Failures:*  
      - Credential expiration or permission issues  
      - Quota exceeded on Google Sheets API  
      - Schema mismatch if new fields are added without updating mapping  
      - Network or API errors  
    - *Credentials:* Uses OAuth2 Google Sheets API credentials with edit access

- **Sticky Note:**  
  A large sticky note titled "Expense Tracker Automation" is present but contains only a heading text and no operational information.

---

### 3. Summary Table

| Node Name        | Node Type             | Functional Role                      | Input Node(s)  | Output Node(s) | Sticky Note             |
|------------------|-----------------------|------------------------------------|----------------|----------------|-------------------------|
| Telegram Trigger  | telegramTrigger       | Listens for incoming Telegram messages | None           | If             | # Expense Tracker Automation |
| If               | if                    | Filters messages containing "="    | Telegram Trigger | transactions   | # Expense Tracker Automation |
| transactions     | code                  | Parses transaction messages using regex | If             | Google Sheets  | # Expense Tracker Automation |
| Google Sheets    | googleSheets          | Appends parsed data to Google Sheets | transactions   | None           | # Expense Tracker Automation |
| Sticky Note      | stickyNote            | Visual note with workflow title     | None           | None           | # Expense Tracker Automation |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: `telegramTrigger`  
   - Configure to listen for "message" update type only.  
   - Set up webhook with Telegram API credentials (OAuth or Bot Token).  
   - No additional filters.  
   - Position it as the workflow entry point.

2. **Create If Node for Filtering**  
   - Type: `if` node (version 2.2 or later)  
   - Condition: Check if `{{$json.message.text}}` contains the character `"="`.  
   - Use strict type validation and case sensitivity enabled.  
   - Connect main output from Telegram Trigger to this If node.

3. **Create Code Node for Parsing Transactions**  
   - Type: `code` node (JavaScript)  
   - Paste the following logic (adapted as needed):  
     - Define three regex patterns matching known transaction message formats.  
     - Attempt matching in order; if matched, extract relevant fields: Amount, Merchant, Card Ending, Date, Time, Available Limit.  
     - If no match, output error with original text.  
   - Input: Connect main output from If node (true branch) to this node.

4. **Create Google Sheets Node**  
   - Type: `googleSheets` node (version 4.6+)  
   - Operation: Append new row  
   - Connect credentials for Google Sheets OAuth2 with write access.  
   - Document ID: Set to the target Google Sheets document ID (e.g., `1tl5OVxvN-eRIPwauSwV4i7mhQrp67tlcWR8Lc6BGQz0`).  
   - Sheet Name: Use sheet with `gid=0` or specify "Sheet1".  
   - Map columns explicitly:  
     - Date → `{{$json.Date}}`  
     - Time → `{{$json.Time}}`  
     - Amount → `{{$json.Amount}}`  
     - Merchant → `{{$json.Merchant}}`  
     - Card Ending → `{{$json['Card Ending']}}`  
     - Available Limit → `{{$json['Available Limit']}}`  
   - Connect main output from Code node to Google Sheets node.

5. **Optional: Add Sticky Note**  
   - Add a sticky note named "Expense Tracker Automation" for documentation or visual aid.

6. **Activate Workflow**  
   - Ensure all credentials are valid and tested.  
   - Activate the workflow to start listening for Telegram transaction messages.

---

### 5. General Notes & Resources

| Note Content                                      | Context or Link                                                                                  |
|-------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses regexes tuned for three common transaction message formats typical in UAE banking SMS/Telegram alerts. | These regexes can be updated to support other message patterns or regional formats.             |
| Google Sheets document must have columns matching the mapped fields for seamless appending. | https://docs.google.com/spreadsheets/d/1tl5OVxvN-eRIPwauSwV4i7mhQrp67tlcWR8Lc6BGQz0/edit        |
| Telegram Bot must have proper permissions and webhook set for real-time message capture. | https://core.telegram.org/bots/api#making-requests                                              |
| The "=" character filter is a heuristic and may need adjustment depending on message styles. | Adjust in the If node for better coverage or to avoid false positives.                           |
| Monitor Google Sheets API quotas to avoid rate limiting. | https://developers.google.com/sheets/api/limits                                                 |

---

This documentation enables detailed understanding, reproduction, and modification of the “Track Expenses by Parsing Telegram Transaction Messages to Google Sheets” workflow while highlighting potential error points and integration specifics.