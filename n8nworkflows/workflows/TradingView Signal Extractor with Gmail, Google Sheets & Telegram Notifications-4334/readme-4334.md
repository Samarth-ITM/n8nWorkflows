TradingView Signal Extractor with Gmail, Google Sheets & Telegram Notifications

https://n8nworkflows.xyz/workflows/tradingview-signal-extractor-with-gmail--google-sheets---telegram-notifications-4334


# TradingView Signal Extractor with Gmail, Google Sheets & Telegram Notifications

### 1. Workflow Overview

This workflow automates the process of extracting trading signals from emails sent by TradingView, logs the relevant data into a Google Sheet, and sends notifications via Telegram. It is designed for users who receive automated trading alerts by email and want to track and act on these signals efficiently.

The workflow can be logically divided into the following blocks:

- **1.1 Input Reception:** Watches for incoming emails in Gmail matching TradingView alerts.
- **1.2 Email Processing & Verification:** Cleans sender email, verifies the source, fetches full email content.
- **1.3 Signal Extraction:** Parses the email subject to extract the stock symbol or trading signal.
- **1.4 Date & Time Management:** Captures and formats the current date and time for record keeping.
- **1.5 Data Logging and Notification:** Appends extracted data to a Google Sheet and sends a Telegram message as notification.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block continuously monitors the configured Gmail inbox for new incoming emails, triggering the workflow when an email arrives.

- **Nodes Involved:**  
  - Email Received

- **Node Details:**

  - **Email Received**  
    - Type: Gmail Trigger  
    - Role: Entry point of the workflow, triggers on new emails in Gmail.  
    - Configuration: Polls every minute for new emails without specific filters, authenticated with Gmail OAuth2 credentials.  
    - Inputs: None (trigger node)  
    - Outputs: Passes email metadata (including message ID) downstream.  
    - Edge Cases: Gmail API rate limits or OAuth token expiry can cause failures.  
    - Version: 1.2

  - **Gmail**  
    - Type: Gmail (operation node)  
    - Role: Marks the triggered email as read after workflow start.  
    - Configuration: Uses the message ID from the trigger; operation is "markAsRead".  
    - Inputs: Email Received node  
    - Outputs: Passes the email data forward.  
    - Edge Cases: Authentication errors, Gmail API quota limits.  
    - Version: 2.1

---

#### 1.2 Email Processing & Verification

- **Overview:**  
  This block cleans the sender’s email address from the raw "From" field, verifies that the email is from the expected TradingView address, and fetches the full email content.

- **Nodes Involved:**  
  - Clean Email  
  - Verify Mail  
  - Get Email

- **Node Details:**

  - **Clean Email**  
    - Type: Code (JavaScript)  
    - Role: Extracts and cleans the sender’s email address from the raw "From" string in the email metadata.  
    - Configuration: Uses regex to extract email within angle brackets or trims the string if no brackets found. Defaults to "[Missing From field]" if invalid.  
    - Input: Gmail node output (email metadata)  
    - Output: JSON with `cleanEmail` field.  
    - Edge Cases: Missing or malformed "From" field could cause empty or incorrect extraction.  
    - Version: 2

  - **Verify Mail**  
    - Type: If  
    - Role: Checks if the cleaned email exactly matches "noreply@tradingview.com" to ensure signal authenticity.  
    - Configuration: String equality check, case-sensitive, strict validation.  
    - Input: Clean Email node  
    - Output: On true, proceeds to Get Email node; on false, halts flow.  
    - Edge Cases: Emails from other senders will block processing, preventing false positives.  
    - Version: 2.2

  - **Get Email**  
    - Type: Gmail  
    - Role: Retrieves the full email content (including subject and snippet) using message ID.  
    - Configuration: Uses message ID from "Email Received" node; operation "get".  
    - Input: Verify Mail node  
    - Output: Full email JSON including subject and snippet.  
    - Edge Cases: Gmail API failures, message ID mismatch.  
    - Version: 2.1

---

#### 1.3 Signal Extraction

- **Overview:**  
  Extracts trading symbol or company name from the email subject line, which typically includes the stock symbol following the phrase "for".

- **Nodes Involved:**  
  - Extract Company Name

- **Node Details:**

  - **Extract Company Name**  
    - Type: Code (JavaScript)  
    - Role: Parses the subject string to isolate the stock symbol after the word "for".  
    - Configuration: Splits subject on "for ", trims result; also returns original subject for reference.  
    - Input: Get Email node output (full email)  
    - Output: JSON with `symbol` and `originalSubject`.  
    - Edge Cases: Subject not containing "for" results in empty symbol string; malformed subjects may cause unexpected output.  
    - Version: 2

---

#### 1.4 Date & Time Management

- **Overview:**  
  Captures current date and time and formats it for insertion into the Google Sheet.

- **Nodes Involved:**  
  - Current Date & Time  
  - Formatted Date & Time

- **Node Details:**

  - **Current Date & Time**  
    - Type: DateTime  
    - Role: Captures the current timestamp in the timezone Asia/Kolkata.  
    - Configuration: Timezone set to Asia/Kolkata.  
    - Input: Extract Company Name output  
    - Output: JSON with `currentDate` field.  
    - Edge Cases: Timezone misconfiguration.  
    - Version: 2

  - **Formatted Date & Time**  
    - Type: DateTime  
    - Role: Formats the captured date to a custom format "DD" (day of month).  
    - Configuration: Uses the `currentDate` from previous node; format: custom "DD".  
    - Input: Current Date & Time node  
    - Output: JSON with `formattedDate` field.  
    - Edge Cases: Invalid date input would cause formatting errors.  
    - Version: 2

---

#### 1.5 Data Logging and Notification

- **Overview:**  
  Appends the extracted stock symbol and current date to a Google Sheet and sends a Telegram message notification with the email snippet.

- **Nodes Involved:**  
  - Google Sheets  
  - Send Message

- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends a new row containing the date and stock symbol to a specific Google Sheet.  
    - Configuration:  
      - Operation: Append  
      - Sheet: "Sheet1" (gid=0)  
      - Document ID: Google Sheet ID 1PBvePZN2yvVg23hDIDB62e91ogTgbuNXVduZTai-bZc  
      - Columns mapped:  
        - Date: from `formattedDate`  
        - Stock: from `symbol` extracted earlier  
    - Input: Formatted Date & Time node  
    - Output: Passes to Send Message node  
    - Edge Cases: Google API quota limits, authentication errors, sheet name or ID errors.  
    - Version: 4.5

  - **Send Message**  
    - Type: Telegram  
    - Role: Sends a Telegram message with a snippet of the email as notification to a specific chat.  
    - Configuration:  
      - Text: Prefixed with "KITE 11: " plus the email snippet from Get Email node.  
      - Chat ID: 1520681602  
    - Input: Google Sheets node  
    - Output: None (end node)  
    - Edge Cases: Telegram API quota, invalid chat ID, token expiry.  
    - Version: 1.2

---

### 3. Summary Table

| Node Name             | Node Type        | Functional Role                          | Input Node(s)         | Output Node(s)          | Sticky Note                                               |
|-----------------------|------------------|----------------------------------------|-----------------------|-------------------------|-----------------------------------------------------------|
| Email Received        | Gmail Trigger    | Triggers workflow on new email         | -                     | Gmail                   | ## Read Emails                                            |
| Gmail                 | Gmail            | Marks email as read                     | Email Received        | Clean Email              | ## Read Emails                                            |
| Clean Email           | Code             | Cleans sender email address             | Gmail                  | Verify Mail              | ## Identify Email                                         |
| Verify Mail           | If               | Verifies email is from TradingView     | Clean Email           | Get Email                | ## Identify Email                                         |
| Get Email             | Gmail            | Retrieves full email content            | Verify Mail           | Extract Company Name     | ## Get Email Data - Extract Buy or Sell Signal            |
| Extract Company Name  | Code             | Extracts stock symbol from subject      | Get Email             | Current Date & Time      | ## Get Email Data - Extract Buy or Sell Signal            |
| Current Date & Time   | DateTime         | Gets current date/time in Asia/Kolkata | Extract Company Name  | Formatted Date & Time    | ## Get Current Date & Time                                |
| Formatted Date & Time | DateTime         | Formats date to "DD"                     | Current Date & Time   | Google Sheets            | ## Get Current Date & Time                                |
| Google Sheets         | Google Sheets    | Appends data row to Google Sheet        | Formatted Date & Time | Send Message             | ## Update Data On Sheet & Notify                          |
| Send Message          | Telegram         | Sends notification message via Telegram | Google Sheets         | -                       | ## Update Data On Sheet & Notify                          |
| Sticky Note           | Sticky Note      | Visual block label                      | -                     | -                       | ## Read Emails                                            |
| Sticky Note1          | Sticky Note      | Visual block label                      | -                     | -                       | ## Identify Email                                         |
| Sticky Note2          | Sticky Note      | Visual block label                      | -                     | -                       | ## Get Email Data - Extract Buy or Sell Signal            |
| Sticky Note3          | Sticky Note      | Visual block label                      | -                     | -                       | ## Get Current Date & Time                                |
| Sticky Note4          | Sticky Note      | Visual block label                      | -                     | -                       | ## Update Data On Sheet & Notify                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node ("Email Received")**  
   - Type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail  
   - Set Polling to every minute, no filters  
   - Position: Input node to start workflow

2. **Create Gmail Node ("Gmail")**  
   - Type: Gmail  
   - Operation: markAsRead  
   - Message ID: Use expression `={{ $json.id }}` from "Email Received" node  
   - Connect "Email Received" output to this node input

3. **Create Code Node ("Clean Email")**  
   - Type: Code (JavaScript)  
   - Script: Use regex to extract email from "From" field or fallback to trimmed string or "[Missing From field]"  
   - Input: Connect from "Gmail" node  
   - Output will contain `cleanEmail` field

4. **Create If Node ("Verify Mail")**  
   - Type: If  
   - Condition: Check if `cleanEmail` equals exactly "noreply@tradingview.com" (case-sensitive, strict)  
   - Connect from "Clean Email" node  
   - On True output: proceed to next node ("Get Email")  
   - On False output: do nothing (stop workflow for unmatched senders)

5. **Create Gmail Node ("Get Email")**  
   - Type: Gmail  
   - Operation: get  
   - Message ID: Use expression `={{ $('Email Received').item.json.id }}`  
   - Connect True output of "Verify Mail" node to this node

6. **Create Code Node ("Extract Company Name")**  
   - Type: Code (JavaScript)  
   - Script: Parse `Subject` field from "Get Email" node, extract substring after "for " as `symbol`, keep original subject  
   - Connect from "Get Email" node

7. **Create DateTime Node ("Current Date & Time")**  
   - Type: DateTime  
   - Operation: Current Date & Time  
   - Timezone: Asia/Kolkata  
   - Connect from "Extract Company Name" node

8. **Create DateTime Node ("Formatted Date & Time")**  
   - Type: DateTime  
   - Operation: Format Date  
   - Date: Use `currentDate` from previous node  
   - Format: Custom `"DD"` (day of month)  
   - Connect from "Current Date & Time" node

9. **Create Google Sheets Node ("Google Sheets")**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: `1PBvePZN2yvVg23hDIDB62e91ogTgbuNXVduZTai-bZc` (replace with your own if needed)  
   - Sheet Name: `gid=0` (Sheet1)  
   - Map columns:  
     - Date: `={{ $json.formattedDate }}`  
     - Stock: `={{ $('Extract Company Name').item.json.symbol }}`  
   - Connect from "Formatted Date & Time" node  
   - Configure Google Sheets OAuth2 credentials

10. **Create Telegram Node ("Send Message")**  
    - Type: Telegram  
    - Text: `"KITE 11: {{ $('Get Email').item.json.snippet }}"`  
    - Chat ID: `1520681602` (replace with your own chat ID)  
    - Connect from "Google Sheets" node  
    - Configure Telegram API credentials

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow is designed to track TradingView alert emails and log them into a Google Sheet, with notifications sent via Telegram for real-time alerting. | Core functionality description                                                                     |
| Telegram chat ID and Google Sheet ID should be replaced with user-specific values for deployment. | Credentials and IDs are environment-specific                                                        |
| Gmail OAuth2 credentials need proper Google Cloud Console setup with Gmail API enabled.          | Gmail API setup required for trigger and operations                                                |
| Google Sheets OAuth2 credentials require Google Sheets API enabled in Google Cloud Console.      | Google Sheets API setup required                                                                   |
| Telegram Bot token and chat ID must be configured correctly to send messages.                    | Telegram Bot creation and permission setup                                                        |
| Regex in "Clean Email" node extracts emails within angle brackets, common in email 'From' fields. | Robustness depends on well-formed "From" header                                                    |
| Date formatting uses day of month only ("DD") – adjust formatting as needed for more detail.     | DateTime node custom formats                                                                        |
| The workflow assumes TradingView emails are sent from "noreply@tradingview.com" to filter signals.| Email verification to avoid processing irrelevant emails                                           |

---

*Disclaimer:* The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.