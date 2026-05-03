Expense Logging with Telegram and Google Sheets: Weekly Summaries and Budget Alerts

https://n8nworkflows.xyz/workflows/expense-logging-with-telegram-and-google-sheets--weekly-summaries-and-budget-alerts-8268


# Expense Logging with Telegram and Google Sheets: Weekly Summaries and Budget Alerts

### 1. Workflow Overview

This workflow enables users to log daily expenses via Telegram messages, store them in Google Sheets, receive a weekly expense summary, and get budget alert notifications when spending exceeds a predefined threshold.

It is structured into the following logical blocks:

- **1.1 Telegram Input Reception**: Captures expense entries sent as Telegram messages in a specific command format.
- **1.2 Expense Parsing and Logging**: Parses the Telegram message to extract expense details and logs them into a Google Sheets document.
- **1.3 Budget Monitoring and Alerts**: Retrieves real-time expense data from Google Sheets, calculates total weekly spending, and sends alerts via Telegram if the budget limit is exceeded.
- **1.4 Weekly Summary Generation**: On a scheduled weekly trigger, fetches weekly expenses, calculates the total, sends a summary message to Telegram, and optionally cleans up old data.
- **1.5 Setup and Documentation**: Contains sticky notes with overview and setup instructions for easier understanding and configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Reception

- **Overview:**  
  Listens for Telegram messages and triggers the workflow when a message with an expense command is received.

- **Nodes Involved:**  
  - Telegram - Get Expense Command

- **Node Details:**

  - **Telegram - Get Expense Command**  
    - Type: Telegram Trigger  
    - Role: Listens for incoming Telegram messages to capture expense commands.  
    - Configuration: Monitors "message" updates; uses Telegram API credentials.  
    - Input: Telegram chat messages.  
    - Output: Raw Telegram message JSON including text, chat ID, and metadata.  
    - Edge Cases:  
      - Message format errors (unexpected command structure).  
      - Telegram API connectivity or authentication issues.  
      - Missing or invalid chat ID.  
    - Sub-workflow: None.

#### 2.2 Expense Parsing and Logging

- **Overview:**  
  Parses the incoming Telegram message to extract amount and description, formats the date, and appends the expense data to a Google Sheets spreadsheet.

- **Nodes Involved:**  
  - Parse Telegram Message  
  - Google Sheets - Log Expense  
  - Google Sheets - Get Expenses (Realtime)  
  - Check Weekly Budget  
  - Telegram - Send Budget Alert

- **Node Details:**

  - **Parse Telegram Message**  
    - Type: Code (JavaScript)  
    - Role: Parses the message text, e.g., `/spent 5 coffee` into structured JSON with amount, description, and current date.  
    - Configuration: Splits the text on spaces, converts amount to float, joins remaining parts as description, and sets date to ISO format.  
    - Key Expressions: `parseFloat(parts[1])`, `parts.slice(2).join(" ")`, `new Date().toISOString().split("T")[0]`.  
    - Input: Telegram message JSON from previous node.  
    - Output: JSON with fields: amount, description, date.  
    - Edge Cases:  
      - Incorrect message format leading to NaN amount.  
      - Missing description (valid but empty).  
      - Date generation errors unlikely but possible if system time is incorrect.  
    - Sub-workflow: None.

  - **Google Sheets - Log Expense**  
    - Type: Google Sheets (Append)  
    - Role: Appends parsed expense data as a new row in Sheet1 of the connected Google Sheets document.  
    - Configuration: Maps amount, description, and date to respective columns; uses OAuth2 credentials.  
    - Input: Parsed expense JSON.  
    - Output: Confirmation of row append operation.  
    - Edge Cases:  
      - Google Sheets API quota or permission errors.  
      - Invalid or missing document ID or sheet name.  
      - Data type mismatches (though conversion is disabled).  
    - Sub-workflow: None.

  - **Google Sheets - Get Expenses (Realtime)**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves all logged expenses from Sheet1 for budget checking.  
    - Configuration: Reads all rows without filters; uses same OAuth2 credentials.  
    - Input: Output of previous Google Sheets append operation.  
    - Output: Array of expense rows as JSON.  
    - Edge Cases: Same as above; also empty sheet returns empty array.  
    - Sub-workflow: None.

  - **Check Weekly Budget**  
    - Type: Code (JavaScript)  
    - Role: Sums all retrieved expenses to check if total exceeds the threshold (€100).  
    - Configuration: Parses amount fields safely, sums up values, compares to 100. Returns alert JSON if threshold exceeded.  
    - Key Expressions: `parseFloat(amountStr)`, conditional return of alert message.  
    - Input: Rows from Google Sheets get operation.  
    - Output: Alert JSON if budget exceeded, empty otherwise.  
    - Edge Cases:  
      - Parsing errors on amount fields.  
      - No expenses logged (sum zero).  
    - Sub-workflow: None.

  - **Telegram - Send Budget Alert**  
    - Type: Telegram  
    - Role: Sends budget alert message to Telegram chat if budget exceeded.  
    - Configuration: Uses alert message from previous node, sends to configured chat ID, disables attribution.  
    - Input: Alert JSON from budget check node.  
    - Output: Confirmation of message sent.  
    - Edge Cases:  
      - Missing or incorrect chat ID.  
      - Telegram API errors.  
      - Empty input (no message sent).  
    - Sub-workflow: None.

#### 2.3 Weekly Summary Generation

- **Overview:**  
  Runs on a scheduled weekly trigger, fetches all weekly expenses from Google Sheets, calculates the total, sends a summary message on Telegram, and optionally deletes old data.

- **Nodes Involved:**  
  - Weekly Summary Trigger  
  - Google Sheets - Get Weekly Expenses  
  - Calculate Weekly Total  
  - Telegram - Send Weekly Summary  
  - Google Sheets - Clean Up (Optional)

- **Node Details:**

  - **Weekly Summary Trigger**  
    - Type: Schedule Trigger  
    - Role: Fires the workflow every Sunday at 11:00 to generate weekly summaries.  
    - Configuration: Weekly interval, trigger hour set to 11.  
    - Input: Time trigger.  
    - Output: Trigger event to downstream nodes.  
    - Edge Cases:  
      - Scheduler downtime or misconfiguration.  
    - Sub-workflow: None.

  - **Google Sheets - Get Weekly Expenses**  
    - Type: Google Sheets (Read)  
    - Role: Reads all logged expenses from Sheet1 to compute the weekly total.  
    - Configuration: Same as realtime get expenses node.  
    - Input: Trigger from schedule node.  
    - Output: Array of all expense rows.  
    - Edge Cases: Same as above.  
    - Sub-workflow: None.

  - **Calculate Weekly Total**  
    - Type: Code (JavaScript)  
    - Role: Iterates over all rows, filters for expenses within the last 7 days, sums amounts, and returns a formatted summary string.  
    - Configuration: Uses date parsing, comparison with week-ago date, and amount summation. Returns JSON with summary text.  
    - Key Expressions: Date filtering on `d >= weekAgo && d <= now`, amount summation, template literal for summary.  
    - Input: Expense rows from Google Sheets.  
    - Output: Summary JSON.  
    - Edge Cases:  
      - Invalid date strings.  
      - Empty or no expenses in last week.  
    - Sub-workflow: None.

  - **Telegram - Send Weekly Summary**  
    - Type: Telegram  
    - Role: Sends the calculated summary as a Telegram message to the configured chat.  
    - Configuration: Uses summary text from previous node, disables attribution, targets configured chat ID.  
    - Input: Summary JSON.  
    - Output: Confirmation of message sent.  
    - Edge Cases: Same as other Telegram nodes.  
    - Sub-workflow: None.

  - **Google Sheets - Clean Up (Optional)**  
    - Type: Google Sheets (Delete)  
    - Role: Deletes the last 30 rows from Sheet1 to keep the sheet clean after summary.  
    - Configuration: Deletes 30 rows from the bottom, uses same OAuth2 credentials.  
    - Input: Confirmation of Telegram send operation.  
    - Output: Confirmation of deletion.  
    - Edge Cases:  
      - Sheet may have fewer than 30 rows (operation may fail or delete fewer rows).  
      - Permission errors for deletion.  
    - Sub-workflow: None.

#### 2.4 Setup and Documentation

- **Overview:**  
  Contains sticky notes providing workflow overview, daily and weekly expense logging explanations, and detailed setup instructions.

- **Nodes Involved:**  
  - Sticky Note - Overview  
  - Sticky Note - Daily Log  
  - Sticky Note - Weekly Total  
  - Sticky Note - Setup Guide

- **Node Details:**

  - **Sticky Note - Overview**  
    - Type: Sticky Note  
    - Role: High-level description of the workflow purpose and features, including command usage and schedule.  
    - Content includes: command format `/spent 5 coffee`, summary schedule, budget alert threshold, and a note to read the setup sticky note.  
    - Position: Top-left area for easy visibility.  
    - No inputs or outputs.

  - **Sticky Note - Daily Log**  
    - Type: Sticky Note  
    - Role: Brief note indicating daily expenses are logged into Google Sheets.  
    - Position: Near daily expense logging nodes.  
    - No inputs or outputs.

  - **Sticky Note - Weekly Total**  
    - Type: Sticky Note  
    - Role: Indicates the weekly total expense calculation and summary process.  
    - Position: Near weekly summary nodes.  
    - No inputs or outputs.

  - **Sticky Note - Setup Guide**  
    - Type: Sticky Note  
    - Role: Step-by-step setup instructions including:  
      - Telegram bot creation and API key retrieval  
      - How to find Telegram chat ID  
      - Link to Google Sheets template copy  
      - Credential setup instructions  
    - Contains author credit and external links for setup.  
    - No inputs or outputs.

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                      | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                  |
|-------------------------------|-------------------------|------------------------------------|----------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note - Overview         | Sticky Note             | Workflow overview and summary      | -                          | -                                 | # 💸 Telegram Expense Tracker with Google Sheets ... Read the setup sticky note for config.  |
| Sticky Note - Daily Log        | Sticky Note             | Notes daily logged expenses        | -                          | -                                 | ## Daily logged expenses in sheets                                                         |
| Sticky Note - Weekly Total     | Sticky Note             | Notes weekly total summary          | -                          | -                                 | ## Weekly Total                                                                             |
| Sticky Note - Setup Guide      | Sticky Note             | Setup instructions and credits     | -                          | -                                 | # 🛠️ Setup Guide ... Links to Telegram bot and Google Sheets template                        |
| Telegram - Get Expense Command | Telegram Trigger        | Receives expense commands           | -                          | Parse Telegram Message             |                                                                                              |
| Parse Telegram Message         | Code                    | Parses Telegram command text        | Telegram - Get Expense Command | Google Sheets - Log Expense     |                                                                                              |
| Google Sheets - Log Expense    | Google Sheets (Append)  | Logs parsed expense to sheet        | Parse Telegram Message      | Google Sheets - Get Expenses (Realtime) |                                                                                              |
| Google Sheets - Get Expenses (Realtime) | Google Sheets (Read)     | Retrieves all expenses for budget check | Google Sheets - Log Expense | Check Weekly Budget               |                                                                                              |
| Check Weekly Budget            | Code                    | Checks if weekly budget exceeded    | Google Sheets - Get Expenses (Realtime) | Telegram - Send Budget Alert |                                                                                              |
| Telegram - Send Budget Alert   | Telegram                | Sends budget alert message          | Check Weekly Budget         | -                                 |                                                                                              |
| Weekly Summary Trigger         | Schedule Trigger        | Triggers weekly summary process     | -                          | Google Sheets - Get Weekly Expenses |                                                                                              |
| Google Sheets - Get Weekly Expenses | Google Sheets (Read)     | Retrieves weekly expenses           | Weekly Summary Trigger      | Calculate Weekly Total            |                                                                                              |
| Calculate Weekly Total         | Code                    | Calculates weekly total expenses    | Google Sheets - Get Weekly Expenses | Telegram - Send Weekly Summary |                                                                                              |
| Telegram - Send Weekly Summary | Telegram                | Sends weekly summary to Telegram    | Calculate Weekly Total      | Google Sheets - Clean Up (Optional) |                                                                                              |
| Google Sheets - Clean Up (Optional) | Google Sheets (Delete)    | Deletes old rows from sheet         | Telegram - Send Weekly Summary | -                               |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation**  
   - Create four sticky notes with the following contents and positions:  
     - Overview (position: approx [-96,112], width 640, height 308) with content describing the workflow and usage.  
     - Daily Log note near daily logging nodes (position approx [576,400], color 4).  
     - Weekly Total note near weekly summary nodes (position approx [576,704], color 5).  
     - Setup Guide with detailed instructions (position approx [16,416], color 6).

2. **Set up Telegram Trigger Node**  
   - Create a Telegram Trigger node named "Telegram - Get Expense Command".  
   - Configure to listen for "message" updates only.  
   - Set up Telegram API credentials (bot token).  
   - Position around [624,496].

3. **Add Code Node to Parse Telegram Messages**  
   - Create a Code node named "Parse Telegram Message" after the Telegram trigger.  
   - Use JavaScript code to split incoming message text by spaces, extract the amount as float, description as remaining text, and current date in YYYY-MM-DD format.  
   - Connect Telegram trigger output to this node.  
   - Position around [816,496].

4. **Add Google Sheets Append Node to Log Expense**  
   - Create a Google Sheets node named "Google Sheets - Log Expense".  
   - Operation: Append row.  
   - Map columns: Date, Amount, Description from parsed code output.  
   - Use OAuth2 Google Sheets credentials.  
   - Specify Sheet1 and your Google Sheets document ID (replace `<YOUR_SHEET_ID>`).  
   - Connect Parse Telegram Message node output to this node.  
   - Position around [1008,496].

5. **Add Google Sheets Read Node for Realtime Expense Retrieval**  
   - Create a Google Sheets node named "Google Sheets - Get Expenses (Realtime)".  
   - Operation: Read all rows.  
   - Use same sheet and document ID.  
   - Connect output of "Google Sheets - Log Expense" to this node.  
   - Position around [1232,496].

6. **Add Code Node to Check Weekly Budget**  
   - Create a Code node named "Check Weekly Budget".  
   - JavaScript code sums all expense amounts.  
   - If total > 100€, return alert JSON with warning message.  
   - Else return empty array.  
   - Connect output of "Google Sheets - Get Expenses (Realtime)" to this node.  
   - Position around [1440,496].

7. **Add Telegram Node to Send Budget Alert**  
   - Create Telegram node named "Telegram - Send Budget Alert".  
   - Use alert message JSON from previous node as text.  
   - Set chat ID to your Telegram chat ID (replace `<YOUR_CHAT_ID>`).  
   - Disable attribution.  
   - Connect "Check Weekly Budget" output to this node.  
   - Position around [1632,496].

8. **Add Schedule Trigger for Weekly Summary**  
   - Create a Schedule Trigger node named "Weekly Summary Trigger".  
   - Configure to trigger weekly on Sundays at 11:00.  
   - Position around [640,800].

9. **Add Google Sheets Read Node to Get Weekly Expenses**  
   - Create a Google Sheets node named "Google Sheets - Get Weekly Expenses".  
   - Operation: Read all rows from Sheet1.  
   - Use same document ID and credentials.  
   - Connect Schedule Trigger output to this node.  
   - Position around [848,800].

10. **Add Code Node to Calculate Weekly Total**  
    - Create a Code node named "Calculate Weekly Total".  
    - JavaScript code filters expenses from last 7 days, sums amounts, and returns formatted summary string.  
    - Connect output of "Google Sheets - Get Weekly Expenses" to this node.  
    - Position around [1056,800].

11. **Add Telegram Node to Send Weekly Summary**  
    - Create Telegram node named "Telegram - Send Weekly Summary".  
    - Uses summary string from previous node as message text.  
    - Set chat ID to your Telegram chat ID.  
    - Disable attribution.  
    - Connect output of "Calculate Weekly Total" to this node.  
    - Position around [1264,800].

12. **Add Optional Google Sheets Delete Node to Clean Up Old Data**  
    - Create a Google Sheets node named "Google Sheets - Clean Up (Optional)".  
    - Operation: Delete rows.  
    - Configure to delete 30 rows from Sheet1.  
    - Use same document ID and credentials.  
    - Connect output of "Telegram - Send Weekly Summary" to this node.  
    - Position around [1472,800].

13. **Credentials Setup**  
    - Configure Telegram credentials with your bot token under Telegram API.  
    - Configure Google OAuth2 credentials to have access to your Google Sheets document.

14. **Replace Placeholder Values**  
    - Replace `<YOUR_SHEET_ID>` in all Google Sheets nodes with your actual spreadsheet ID.  
    - Replace `<YOUR_CHAT_ID>` in all Telegram nodes with your Telegram chat ID.

15. **Test the Workflow**  
    - Send a message in Telegram in the format `/spent 5 coffee`.  
    - Verify the expense logs into Google Sheets.  
    - Wait for weekly trigger or manually trigger schedule to test summary.  
    - Test budget alert by logging expenses exceeding €100.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Author: Giorgos Tarasidis                                                                                                                                         | Setup sticky note credits                                                                                         |
| Telegram bot creation instructions: create bot via BotFather, obtain API token, find chat ID via Telegram API                                                     | https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates                                                          |
| Google Sheets Expense Bot Template (copy before use)                                                                                                            | https://docs.google.com/spreadsheets/d/1uyQpX9_ZZUhLtBhyxfjdU80D6Q6cqbMV286MnhF5QBY/edit?usp=sharing               |
| Budget alert threshold is hardcoded in "Check Weekly Budget" node (default €100) - adjust code to change                                                        | Code node "Check Weekly Budget"                                                                                   |
| Message format to log expense: `/spent <amount> <description>`                                                                                                   | Overview sticky note                                                                                              |
| Weekly summary schedule: Sundays at 11:00                                                                                                                       | Schedule Trigger node configuration                                                                               |

---

*Disclaimer:* The content provided is extracted exclusively from an n8n automated workflow. It fully complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.