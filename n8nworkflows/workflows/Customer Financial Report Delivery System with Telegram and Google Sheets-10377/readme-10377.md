Customer Financial Report Delivery System with Telegram and Google Sheets

https://n8nworkflows.xyz/workflows/customer-financial-report-delivery-system-with-telegram-and-google-sheets-10377


# Customer Financial Report Delivery System with Telegram and Google Sheets

---

### 1. Workflow Overview

This workflow automates the delivery of customer financial reports via Telegram, using Google Sheets as the data source. It is designed to handle user interactions on Telegram, validate user access rights, retrieve customer financial data from Google Sheets, generate summarized financial reports, and send them back to the user through Telegram messages.

The workflow is logically divided into three main blocks:

- **1.1 User Input & Data Fetch Process**  
  Captures and interprets Telegram user input, determines user intent (welcome, error, or customer search), and fetches initial customer data from Google Sheets.

- **1.2 Access Validation Process**  
  Validates if the Telegram user is authorized to access the requested customer data by cross-checking chat IDs against an Access sheet in Google Sheets.

- **1.3 Report Generation & Delivery**  
  Aggregates financial data, formats a detailed report, and sends the report or appropriate messages (access denied or error) back to the user via Telegram.

---

### 2. Block-by-Block Analysis

---

#### 2.1 User Input & Data Fetch Process

**Overview:**  
This block starts with receiving Telegram messages, interprets user commands or customer names, and queries Google Sheets to fetch customer-related data for further processing.

**Nodes Involved:**  
- Input user2 (Telegram Trigger)  
- Code (JavaScript)  
- If4 (Conditional Branch)  
- Wellcome1 (Telegram Send)  
- Get row(s) in sheet4 (Google Sheets)  
- Code2 (JavaScript)  
- Merge4 (Merge Node)

**Node Details:**

- **Input user2**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point capturing Telegram messages  
  - *Config:* Listens to "message" updates from Telegram Bot API using provided credentials  
  - *Inputs:* Telegram user messages  
  - *Outputs:* Message JSON passed downstream  
  - *Failures:* Telegram API auth errors, webhook setup issues  

- **Code**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Parses Telegram message text to determine action:
    - If message is "/start" or "hi" → action = "welcome"  
    - If empty message → action = "error"  
    - Else → action = "searchCustomer" with customerName extracted  
  - *Key expressions:* `message.text.trim()`, action assignment  
  - *Inputs:* Telegram message JSON  
  - *Outputs:* JSON with action, chatId, extracted customerName, username  
  - *Edge cases:* Empty or malformed messages; missing chat IDs  
  - *Version:* v2 of Code Node used  

- **If4**  
  - *Type:* If Node  
  - *Role:* Branches workflow based on `action` field  
  - *Condition:* Checks if action equals "welcome" (strict string match)  
  - *Inputs:* Output of Code node  
  - *Outputs:* Yes branch → Wellcome1 node; No branch → Google Sheets lookup and merge  
  - *Failures:* Expression evaluation errors if action missing  

- **Wellcome1**  
  - *Type:* Telegram Send  
  - *Role:* Sends welcome message to user  
  - *Config:* Uses Telegram API credentials; sends predefined "Your Custom Message" text; uses chatId from JSON  
  - *Inputs:* JSON with chatId and reply  
  - *Outputs:* Sends message, no further output  
  - *Failures:* Telegram API errors, invalid chatId  

- **Get row(s) in sheet4**  
  - *Type:* Google Sheets  
  - *Role:* Fetches customer details from "Sheet1" tab filtering by `customerName`  
  - *Config:* Uses Google Sheets OAuth2 credentials; filters rows where "Customer name" column equals incoming customerName; always outputs data  
  - *Inputs:* JSON with customerName  
  - *Outputs:* Rows matching customerName or empty if none found  
  - *Failures:* Google Sheets API quota issues, incorrect column names, permission errors  

- **Code2**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Adds a boolean `found` flag to indicate if customer data was found; merges chatId for downstream use  
  - *Key logic:* Checks if data object has keys; sets found true/false accordingly  
  - *Inputs:* Google Sheets data + chatId  
  - *Outputs:* JSON enriched with found flag and chatId  
  - *Failures:* Empty data handling  

- **Merge4**  
  - *Type:* Merge Node  
  - *Role:* Combines outputs from Get row(s) in sheet4 and Code2 nodes to unify data for next block  
  - *Config:* Combine mode, combining all inputs  
  - *Inputs:* Data from Google Sheets and Code2  
  - *Outputs:* Merged JSON  
  - *Failures:* Merge conflicts unlikely here since data is combined  

---

#### 2.2 Access Validation Process

**Overview:**  
This block checks if the requesting Telegram user is authorized to access the requested customer's financial data by matching the user's chat ID against authorized chat IDs stored in the "Access" Google Sheet.

**Nodes Involved:**  
- If5 (Conditional Branch)  
- Get row(s) in sheet (Access)2 (Google Sheets)  
- Merge5 (Merge Node)  
- Check Match2 (Code Node)  
- Enter Correct name (Telegram Send)

**Node Details:**

- **If5**  
  - *Type:* If Node  
  - *Role:* Checks if customer data was found (`found` boolean)  
  - *Condition:* found == true (boolean strict match)  
  - *Inputs:* Merged data from Merge4  
  - *Outputs:* Yes branch → Fetch Access sheet rows; No branch → Sends error message to user  
  - *Failures:* Expression errors if `found` field missing  

- **Get row(s) in sheet (Access)2**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves access control entries from "Access" sheet tab filtering by "Customers Group" matching the group name from input  
  - *Config:* Uses OAuth2 credentials; filters rows on "Customers Group" column with value from JSON  
  - *Inputs:* JSON with group name  
  - *Outputs:* Access rows with chat IDs for validation  
  - *Failures:* Incorrect group names, API errors, missing data  

- **Merge5**  
  - *Type:* Merge Node  
  - *Role:* Combines Access sheet data with previous data for validation  
  - *Config:* Combine mode, combine all  
  - *Inputs:* Outputs from Get row(s) in sheet (Access)2 and If5 yes branch  
  - *Outputs:* Combined JSON for access check  
  - *Failures:* Merge conflicts unlikely  

- **Check Match2**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Compares Telegram user chatId with authorized chat IDs (ChatID1, ChatID2) from Access sheet rows  
  - *Key logic:*  
    - Normalizes keys ignoring spaces and case  
    - Checks if user chatId matches either access chat IDs  
    - Returns `access` field: "granted" or "denied"  
  - *Inputs:* Merged access data  
  - *Outputs:* JSON with access status, groupName, customerName, chatId  
  - *Failures:* Missing keys, case sensitivity, empty chat IDs  

- **Enter Correct name**  
  - *Type:* Telegram Send  
  - *Role:* Sends a prompt asking user to enter a correct customer name when no customer data found or no access  
  - *Config:* Uses Telegram API credentials; sends predefined "Your custom Message" text; targets user chatId  
  - *Inputs:* JSON with chatId  
  - *Outputs:* Sends Telegram message  
  - *Failures:* Telegram API errors, invalid chatId  

---

#### 2.3 Report Generation & Delivery

**Overview:**  
If access is validated, this block retrieves detailed customer financial data from Google Sheets, computes aggregates (debit, credit, balance), formats a textual report, and delivers it back to the user on Telegram. If access is denied, it sends a denial message.

**Nodes Involved:**  
- If3 (Conditional Branch)  
- Get row(s) in sheet5 (Google Sheets)  
- Aggregate Summary2 (Code Node)  
- Format Details2 (Code Node)  
- Combine Summary + Details2 (Code Node)  
- Send Report2 (Telegram Send)  
- No permission (Telegram Send)

**Node Details:**

- **If3**  
  - *Type:* If Node  
  - *Role:* Checks if `access` is "granted"  
  - *Condition:* access == "granted" (string strict match)  
  - *Inputs:* Output from Check Match2  
  - *Outputs:* Yes branch → Fetch customer financial data; No branch → Send no permission message  
  - *Failures:* Expression errors if `access` missing or malformed  

- **Get row(s) in sheet5**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves detailed financial data for the customer by filtering "customerNam" column with customerName  
  - *Config:* OAuth2 credentials; sheet tab "Sheet1" (gid=0); always output data  
  - *Inputs:* JSON with customerName  
  - *Outputs:* Rows of financial data (debits, credits, weights)  
  - *Failures:* Lookup column mismatch, API quota, permission issues  

- **Aggregate Summary2**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Aggregates financial columns:
    - Converts string numbers with commas, spaces, slashes to numbers  
    - Sums total debit and credit values  
    - Computes balance as debit minus credit  
  - *Key function:* `toNumber` to parse numeric strings robustly  
  - *Inputs:* Array of financial rows  
  - *Outputs:* JSON with `totalDebitMali`, `totalCreditMali`, `balanceMali`, and `accountName`  
  - *Failures:* Parsing errors on malformed numbers, empty data arrays  

- **Format Details2**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Formats a user-friendly report text with emojis and labels for debit, credit, balance  
  - *Inputs:* Aggregated summary JSON  
  - *Outputs:* JSON with formatted `text` field  
  - *Failures:* Missing input fields, malformed text variables  

- **Combine Summary + Details2**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Combines summary and detailed report text into a final summary string  
  - *Inputs:* Outputs from Format Details2 and Aggregate Summary2  
  - *Outputs:* JSON with combined `summary` text  
  - *Failures:* Reference errors on keys, string concatenation issues  

- **Send Report2**  
  - *Type:* Telegram Send  
  - *Role:* Sends the final combined report text to the requesting user on Telegram  
  - *Config:* Uses Telegram API credentials; sends `summary` text; targets chatId from Input user2 node  
  - *Inputs:* JSON with summary and chatId  
  - *Outputs:* Sends message, no further output  
  - *Failures:* Telegram API errors, invalid chatId  

- **No permission**  
  - *Type:* Telegram Send  
  - *Role:* Sends an access denial message when user is unauthorized  
  - *Config:* Uses Telegram API credentials; sends predefined "Your Custom Message"; targets user chatId  
  - *Inputs:* JSON with chatId  
  - *Outputs:* Sends message  
  - *Failures:* Telegram API errors, invalid chatId  

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                                  | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                         |
|-------------------------|--------------------|-------------------------------------------------|-----------------------------|--------------------------------|---------------------------------------------------------------------------------------------------|
| Input user2             | Telegram Trigger   | Entry point to receive Telegram messages         | -                           | Code                           |                                                                                                |
| Code                    | Code (JS)          | Parses message, identifies action or customer     | Input user2                 | If4                           |                                                                                                |
| If4                     | If Node            | Branch on action = "welcome"                      | Code                        | Wellcome1 / Get row(s) in sheet4 + Merge4 | Sticky Note2 describes STEP 1: USER INPUT & DATA FETCH PROCESS                                    |
| Wellcome1               | Telegram Send      | Sends welcome message                             | If4 (yes branch)            | -                              |                                                                                                |
| Get row(s) in sheet4    | Google Sheets      | Fetch customer data by name                        | If4 (no branch)             | Code2                         |                                                                                                |
| Code2                   | Code (JS)          | Adds found flag and merges chatId                 | Get row(s) in sheet4        | Merge4                        |                                                                                                |
| Merge4                  | Merge Node         | Combines customer data and user input             | Get row(s) in sheet4, Code2 | If5                           |                                                                                                |
| If5                     | If Node            | Check if customer data found (found == true)      | Merge4                      | Get row(s) in sheet (Access)2 / Enter Correct name | Sticky Note5 describes STEP 2: ACCESS VALIDATION PROCESS                                         |
| Get row(s) in sheet (Access)2 | Google Sheets | Fetch access info by group                         | If5 (yes branch)            | Merge5                        |                                                                                                |
| Merge5                  | Merge Node         | Combines access data with previous data            | Get row(s) in sheet (Access)2, If5 | Check Match2                 |                                                                                                |
| Check Match2             | Code (JS)          | Validates user chatId against access chatIds       | Merge5                      | If3                           |                                                                                                |
| Enter Correct name       | Telegram Send      | Sends message to enter correct customer name       | If5 (no branch)             | -                              |                                                                                                |
| If3                     | If Node            | Checks if access is granted                         | Check Match2                | Get row(s) in sheet5 / No permission | Sticky Note6 describes STEP 3: REPORT GENERATION & DELIVERY                                      |
| Get row(s) in sheet5    | Google Sheets      | Fetch detailed financial data for customer          | If3 (yes branch)            | Aggregate Summary2            |                                                                                                |
| Aggregate Summary2       | Code (JS)          | Aggregates debit, credit, and balance amounts       | Get row(s) in sheet5        | Format Details2               |                                                                                                |
| Format Details2          | Code (JS)          | Formats detailed report text with emojis            | Aggregate Summary2          | Combine Summary + Details2    |                                                                                                |
| Combine Summary + Details2 | Code (JS)        | Combines summary and details into final report text | Format Details2, Aggregate Summary2 | Send Report2            |                                                                                                |
| Send Report2             | Telegram Send      | Sends final report to Telegram user                  | Combine Summary + Details2  | -                              |                                                                                                |
| No permission            | Telegram Send      | Sends access denied message                           | If3 (no branch)             | -                              |                                                                                                |
| Sticky Note2             | Sticky Note        | Describes STEP 1: USER INPUT & DATA FETCH PROCESS   | -                           | -                              | Covers Input user2 to Merge4 nodes                                                               |
| Sticky Note5             | Sticky Note        | Describes STEP 2: ACCESS VALIDATION PROCESS          | -                           | -                              | Covers If5 to Enter Correct name nodes                                                           |
| Sticky Note6             | Sticky Note        | Describes STEP 3: REPORT GENERATION & DELIVERY       | -                           | -                              | Covers If3 to No permission nodes                                                                |
| Sticky Note3             | Sticky Note        | High-level workflow explanation and setup notes     | -                           | -                              | General overview of entire workflow process                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node ("Input user2")**  
   - Select "Telegram Trigger" node type  
   - Configure with your Telegram Bot API credentials  
   - Set to listen for "message" updates  
   - Position it as the workflow entry point

2. **Add Code Node ("Code")**  
   - Connect from "Input user2"  
   - Paste the JavaScript to parse message text:  
     - Detects "/start" or "hi" → action = "welcome"  
     - Empty text → action = "error"  
     - Otherwise → action = "searchCustomer" with extracted customerName  
   - Outputs JSON with `action`, `chatId`, `customerName`, `username`

3. **Add If Node ("If4")**  
   - Connect from "Code"  
   - Configure condition: `action == "welcome"` (strict string equality)  
   - Yes branch → "Wellcome1" node  
   - No branch → "Get row(s) in sheet4" and "Merge4" nodes

4. **Add Telegram Send Node ("Wellcome1")**  
   - Connect from If4 "Yes" branch  
   - Configure with Telegram API credentials  
   - Send a welcome text (replace placeholder "Your Custom Message")  
   - Use chatId from JSON input

5. **Add Google Sheets Node ("Get row(s) in sheet4")**  
   - Connect from If4 "No" branch  
   - Set credentials for Google Sheets OAuth2  
   - DocumentId: your Google Sheet ID  
   - SheetName: e.g., "Sheet1" (customer data sheet)  
   - Filter rows where "Customer name" column equals `{{$json.customerName}}`  
   - Set "Always Output Data" to true

6. **Add Code Node ("Code2")**  
   - Connect from "Get row(s) in sheet4"  
   - Add JavaScript to:  
     - Check if returned data has keys → `found` boolean  
     - Merge chatId to output JSON

7. **Add Merge Node ("Merge4")**  
   - Connect "Code2" and "Get row(s) in sheet4" outputs as inputs  
   - Set mode to "Combine All"

8. **Add If Node ("If5")**  
   - Connect from "Merge4"  
   - Configure to check: `found == true` (boolean equality)  
   - Yes branch → "Get row(s) in sheet (Access)2" and "Merge5"  
   - No branch → "Enter Correct name"

9. **Add Google Sheets Node ("Get row(s) in sheet (Access)2")**  
   - Connect from "If5" Yes branch  
   - Use OAuth2 credentials  
   - DocumentId: same Google Sheet ID  
   - SheetName: "Access" (access control sheet)  
   - Filter rows where "Customers Group" equals `{{$json.Groups}}`  
   - Always output data true

10. **Add Merge Node ("Merge5")**  
    - Combine outputs of "Get row(s) in sheet (Access)2" and "If5" Yes branch

11. **Add Code Node ("Check Match2")**  
    - Connect from "Merge5"  
    - Add JavaScript code to:  
      - Normalize keys (chatId1, chatId2)  
      - Compare user chatId with access chatIds  
      - Output `access` as "granted" or "denied"

12. **Add Telegram Send Node ("Enter Correct name")**  
    - Connect from "If5" No branch  
    - Use Telegram credentials  
    - Send prompt message "Your custom Message" to chatId

13. **Add If Node ("If3")**  
    - Connect from "Check Match2"  
    - Check if `access == "granted"` (string equality)  
    - Yes branch → "Get row(s) in sheet5" and subsequent nodes  
    - No branch → "No permission"

14. **Add Google Sheets Node ("Get row(s) in sheet5")**  
    - Connect from "If3" Yes branch  
    - OAuth2 credentials  
    - DocumentId: same Google Sheet ID  
    - SheetName: e.g., "Sheet1" (or tab with financial data)  
    - Filter on "customerNam" column equals `{{$json.customerName}}`  
    - Always output data true

15. **Add Code Node ("Aggregate Summary2")**  
    - Connect from "Get row(s) in sheet5"  
    - Paste JavaScript that:  
      - Defines `toNumber` function to clean and parse numbers  
      - Iterates rows summing debit and credit fields  
      - Calculates balance = totalDebit - totalCredit  
      - Outputs totals and balance

16. **Add Code Node ("Format Details2")**  
    - Connect from "Aggregate Summary2"  
    - Formats a text report with emojis and labels: debit, credit, balance  
    - Outputs JSON with a `text` field

17. **Add Code Node ("Combine Summary + Details2")**  
    - Connect from "Format Details2" and "Aggregate Summary2" (multi-input)  
    - Combines summary and detailed text into a single `summary` string

18. **Add Telegram Send Node ("Send Report2")**  
    - Connect from "Combine Summary + Details2"  
    - Use Telegram API credentials  
    - Send `summary` text to chatId from "Input user2"

19. **Add Telegram Send Node ("No permission")**  
    - Connect from "If3" No branch  
    - Use Telegram credentials  
    - Send "Your Custom Message" as access denied notification

20. **Configure all Telegram nodes with your Telegram Bot API credentials**  
    - Ensure credentials reuse and correct token setup

21. **Configure all Google Sheets nodes with OAuth2 credentials**  
    - Ensure Sheets API enabled in Google Cloud, proper scopes granted

22. **Replace all placeholder texts like "Your Custom Message", "Your Custom Name", and column names in filters with actual values matching your data**

23. **Test the workflow by messaging your Telegram bot with /start and valid customer names**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| This workflow uses emojis in report formatting to improve readability (e.g., 💰 for debit, 💸 for credit, ⚖️ for balance). Ensure your Telegram client supports emoji rendering.                                                                                                                                                                                       | Formatting in Format Details2 node                                  |
| Google Sheets document and sheet names (e.g., "Access", "Sheet1") and column names (e.g., "Customer name", "Customers Group") must be consistent with your actual spreadsheet setup to ensure filtering works correctly.                                                                                                                                                 | Google Sheets node configurations                                  |
| Telegram API credentials must be generated via BotFather, and corresponding webhook URLs configured in n8n for Telegram Trigger nodes to work properly.                                                                                                                                                                                                               | Telegram integration setup                                         |
| The JavaScript code nodes rely on specific JSON keys being present; any mismatch in incoming data structure can cause expression errors. Carefully map your Telegram message structure and Google Sheets columns to the expected keys in the code.                                                                                                                     | Code nodes in all blocks                                            |
| For improved security, consider restricting access to the webhook URL and using environment variables for sensitive credentials.                                                                                                                                                                                                                                     | Security best practices                                            |
| The workflow expects the "Access" sheet to have at least two chat ID columns ("ChatID1" and "ChatID2") for validating user permissions. Modify the code if your access control uses a different schema.                                                                                                                                                               | Access validation logic (Check Match2 node)                        |
| The workflow includes sticky notes within n8n for inline documentation and mapping guidance. Use them to keep track of field mappings and custom messages for easier maintenance.                                                                                                                                                                                    | Sticky Note nodes in workflow                                      |
| Expected setup time is approximately 10–15 minutes, assuming prior familiarity with n8n, Telegram bots, and Google Sheets API.                                                                                                                                                                                                                                       | User expectations                                                  |
| For further customization, you can extend the code nodes to handle additional financial metrics or multiple customer groups.                                                                                                                                                                                                                                        | Workflow extensibility                                            |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---