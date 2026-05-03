Send School Fee Reminders with Payment Links via Email & WhatsApp

https://n8nworkflows.xyz/workflows/send-school-fee-reminders-with-payment-links-via-email---whatsapp-7039


# Send School Fee Reminders with Payment Links via Email & WhatsApp

### 1. Workflow Overview

This workflow is designed to automate the process of sending school fee reminders to parents via both email and WhatsApp, including secure payment links. It targets educational institutions needing to systematically notify parents of upcoming fee due dates, encouraging timely payments and reducing manual follow-up effort.

The workflow is logically divided into these key blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at 8 AM to check for pending fees.
- **1.2 Data Retrieval:** Reads fee records with a "Pending" status from a Microsoft Excel workbook.
- **1.3 Fee Processing & Filtering:** Filters fees due within the next three days and generates payment links.
- **1.4 Reminder Preparation:** Creates personalized reminder messages tailored for email and WhatsApp channels.
- **1.5 Controlled Delivery:** Uses wait nodes to manage asynchronous sending of reminders.
- **1.6 Notification Sending:** Sends emails and WhatsApp messages to parents.
- **1.7 Status Update:** Updates the Excel worksheet to reflect that reminders have been sent.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Triggers the workflow automatically every day at 8 AM to start the fee reminder process.
- **Nodes Involved:**  
  - *Daily Fee Check - 8 AM*

- **Node Details:**

  - **Daily Fee Check - 8 AM**  
    - *Type & Role:* Schedule Trigger; initiates workflow execution daily at a fixed hour.  
    - *Configuration:* Trigger set to activate at 8 AM every day.  
    - *Expressions/Variables:* Fixed schedule, no dynamic expressions.  
    - *Connections:* Outputs to *Read Pending Fees*.  
    - *Edge Cases:* Workflow will not start if n8n instance is offline at trigger time; time zone considerations may affect exact trigger time.  
    - *Version:* 1.2

---

#### 1.2 Data Retrieval

- **Overview:** Reads pending fee records from an Excel worksheet to identify which students have outstanding payments.
- **Nodes Involved:**  
  - *Read Pending Fees*

- **Node Details:**

  - **Read Pending Fees**  
    - *Type & Role:* Microsoft Excel node to read data filtered by status.  
    - *Configuration:* Reads from a workbook identified by ID (parameterized as "=fee-records-workbook"). Filters rows where the "Status" column equals "Pending".  
    - *Credentials:* Uses Microsoft Excel OAuth2 credentials ("Microsoft Excel account - test").  
    - *Connections:* Receives trigger from *Daily Fee Check - 8 AM*; outputs to *Process Fee Reminders*.  
    - *Edge Cases:* Authentication failure with Microsoft Excel API, workbook or worksheet ID errors, empty data if no pending fees found.  
    - *Version:* 2

---

#### 1.3 Fee Processing & Filtering

- **Overview:** Filters fees to those due within the next 3 days and generates payment links for each applicable fee.
- **Nodes Involved:**  
  - *Process Fee Reminders*

- **Node Details:**

  - **Process Fee Reminders**  
    - *Type & Role:* Code node running JavaScript to filter and enrich fee records.  
    - *Configuration:*  
      - Calculates the current date and a "reminder date" three days ahead.  
      - Iterates over all input fee records.  
      - For fees with due dates between now and three days ahead, constructs a payment URL embedding student ID, amount, and fee ID.  
      - Prepares an array of reminders with relevant details, including days remaining until due.  
    - *Expressions:* Uses standard JavaScript date manipulation and array mapping.  
    - *Connections:* Inputs from *Read Pending Fees*; outputs to both *Prepare Email Reminder* and *Prepare WhatsApp Reminder*.  
    - *Edge Cases:*  
      - Date parsing errors if dueDate is malformed.  
      - Empty input array leads to no reminders generated (workflow quietly ends).  
      - Potential timezone mismatches affecting date calculations.  
    - *Version:* 2

---

#### 1.4 Reminder Preparation

- **Overview:** Prepares personalized email and WhatsApp messages for each fee reminder.
- **Nodes Involved:**  
  - *Prepare Email Reminder*  
  - *Prepare WhatsApp Reminder*

- **Node Details:**

  - **Prepare Email Reminder**  
    - *Type & Role:* Code node to build an email subject and body with fee details and payment link.  
    - *Configuration:*  
      - Uses first input item’s JSON data.  
      - Constructs subject mentioning fee type and days remaining.  
      - Creates text email with detailed fee info and payment link.  
    - *Expressions:* String templates embedding dynamic fee data.  
    - *Connections:* Input from *Process Fee Reminders*; outputs to *Wait For Prepare Reminder*.  
    - *Edge Cases:* If input data missing fields (e.g. parentEmail), email will fail to send later.  
    - *Version:* 2

  - **Prepare WhatsApp Reminder**  
    - *Type & Role:* Code node to build a WhatsApp-formatted message string.  
    - *Configuration:*  
      - Uses first input JSON.  
      - Message includes fee type, amount, due date, days remaining, and payment link with basic formatting (bold, emojis).  
    - *Expressions:* String template with newline escapes and Markdown-like formatting.  
    - *Connections:* Input from *Process Fee Reminders*; outputs to *Wait For Prepare Reminder for WhatsApp*.  
    - *Edge Cases:* Missing or invalid phone number may cause WhatsApp sending failure.  
    - *Version:* 2

---

#### 1.5 Controlled Delivery

- **Overview:** Inserts wait nodes between preparation and sending steps to control timing or enable asynchronous processing.
- **Nodes Involved:**  
  - *Wait For Prepare Reminder*  
  - *Wait For Prepare Reminder for WhatsApp*

- **Node Details:**

  - **Wait For Prepare Reminder**  
    - *Type & Role:* Wait node delaying execution; acts as a buffer before sending email.  
    - *Configuration:* No explicit wait time configured; likely waiting for webhook trigger or external signal.  
    - *Connections:* Input from *Prepare Email Reminder*; outputs to *Send Email Reminder*.  
    - *Edge Cases:* If webhook trigger never received, workflow pause indefinitely.  
    - *Version:* 1.1

  - **Wait For Prepare Reminder for WhatsApp**  
    - *Type & Role:* Wait node delaying execution before WhatsApp sending.  
    - *Configuration:* Same as above, no explicit wait time.  
    - *Connections:* Input from *Prepare WhatsApp Reminder*; outputs to *Send message*.  
    - *Edge Cases:* Same as above.  
    - *Version:* 1.1

---

#### 1.6 Notification Sending

- **Overview:** Sends the prepared reminders via email and WhatsApp channels.
- **Nodes Involved:**  
  - *Send Email Reminder*  
  - *Send message* (WhatsApp)

- **Node Details:**

  - **Send Email Reminder**  
    - *Type & Role:* Email Send node to dispatch reminders.  
    - *Configuration:*  
      - Sends plain text emails.  
      - Subject, body, and recipient email taken from incoming JSON fields.  
      - From address fixed as finance@school.edu.  
    - *Credentials:* SMTP credentials named "SMTP -test".  
    - *Connections:* Input from *Wait For Prepare Reminder*; outputs to *Update Reminder Status*.  
    - *Edge Cases:* SMTP authentication failures, invalid recipient email, network issues.  
    - *Version:* 2.1

  - **Send message** (WhatsApp)  
    - *Type & Role:* WhatsApp node to send messages via WhatsApp API.  
    - *Configuration:*  
      - Sends message text from JSON body field.  
      - Recipient phone number from JSON "toforwp" field (note: field name may be incorrectly referenced in JSON).  
      - Phone number ID configured as a fixed string "+919988776655" (may represent sender or WhatsApp API configuration).  
    - *Credentials:* WhatsApp API credentials "WhatsApp-test".  
    - *Connections:* Input from *Wait For Prepare Reminder for WhatsApp*; outputs to *Update Reminder Status*.  
    - *Edge Cases:* Invalid phone numbers, API quota limits, authentication failures.  
    - *Version:* 1

---

#### 1.7 Status Update

- **Overview:** Updates the Excel worksheet to mark reminders as sent or update their status accordingly.
- **Nodes Involved:**  
  - *Update Reminder Status*

- **Node Details:**

  - **Update Reminder Status**  
    - *Type & Role:* Microsoft Excel node to update worksheet rows.  
    - *Configuration:*  
      - Uses "autoMap" data mode to map incoming data fields to worksheet columns automatically.  
      - Matches rows on the "name" column (expression "=name").  
      - Workbook and worksheet IDs parameterized.  
    - *Credentials:* Same Microsoft Excel OAuth2 account as reading node.  
    - *Connections:* Inputs from both *Send Email Reminder* and *Send message* nodes (merging outputs).  
    - *Edge Cases:* Possible mismatches if "name" field does not uniquely identify rows, Excel API update failures, concurrency issues if simultaneous updates occur.  
    - *Version:* 2

---

### 3. Summary Table

| Node Name                     | Node Type                   | Functional Role                  | Input Node(s)               | Output Node(s)                          | Sticky Note                                                                                         |
|-------------------------------|-----------------------------|---------------------------------|-----------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------|
| Daily Fee Check - 8 AM         | Schedule Trigger            | Workflow start trigger at 8 AM  |                             | Read Pending Fees                      |                                                                                                   |
| Read Pending Fees              | Microsoft Excel             | Reads pending fee records       | Daily Fee Check - 8 AM       | Process Fee Reminders                   |                                                                                                   |
| Process Fee Reminders          | Code                        | Filters fees due soon & generates payment links | Read Pending Fees            | Prepare Email Reminder, Prepare WhatsApp Reminder |                                                                                                   |
| Prepare Email Reminder         | Code                        | Prepares email message content  | Process Fee Reminders        | Wait For Prepare Reminder               |                                                                                                   |
| Prepare WhatsApp Reminder      | Code                        | Prepares WhatsApp message       | Process Fee Reminders        | Wait For Prepare Reminder for WhatsApp |                                                                                                   |
| Wait For Prepare Reminder      | Wait                        | Controls flow before email send | Prepare Email Reminder       | Send Email Reminder                    |                                                                                                   |
| Wait For Prepare Reminder for WhatsApp | Wait               | Controls flow before WhatsApp send | Prepare WhatsApp Reminder    | Send message                          |                                                                                                   |
| Send Email Reminder            | Email Send                  | Sends fee reminder emails       | Wait For Prepare Reminder    | Update Reminder Status                 |                                                                                                   |
| Send message                  | WhatsApp                    | Sends fee reminder WhatsApp messages | Wait For Prepare Reminder for WhatsApp | Update Reminder Status                 |                                                                                                   |
| Update Reminder Status         | Microsoft Excel             | Updates status in Excel         | Send Email Reminder, Send message |                                      |                                                                                                   |
| Workflow Info                 | Sticky Note                 | Documentation & process overview |                             |                                        | ### **Fee Reminder Workflow**  ⏰ Daily Check: Runs at 8 AM  🔍 Process: 1. Read pending fees from Excel 2. Filter fees due in 3 days 3. Generate payment links 4. Send email & WhatsApp reminders 5. Update reminder status  📧 Channels: - Email with payment link - WhatsApp with formatted message  🔗 Payment Integration: Automatically generates secure payment links for each fee reminder. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Name: "Daily Fee Check - 8 AM"  
   - Set to trigger daily at 8:00 AM (hour 8).  

2. **Add a Microsoft Excel node:**  
   - Name: "Read Pending Fees"  
   - Operation: Read worksheet rows.  
   - Configure to read from the school’s fee records workbook (use appropriate workbook ID).  
   - Apply filter: "Status" equals "Pending".  
   - Attach Microsoft Excel OAuth2 credentials.  
   - Connect output from "Daily Fee Check - 8 AM".  

3. **Add a Code node:**  
   - Name: "Process Fee Reminders"  
   - Paste JavaScript code that:  
     - Gets current date, calculates reminder date +3 days.  
     - Iterates over fee records and filters those with due dates ≤ reminder date and > current date.  
     - Generates a payment link embedding studentId, amount, and feeId.  
     - Outputs array of reminders with student and fee data.  
   - Connect input from "Read Pending Fees".  

4. **Add two Code nodes in parallel:**  
   - "Prepare Email Reminder":  
     - Builds email subject and body with fee details and payment link using template strings.  
   - "Prepare WhatsApp Reminder":  
     - Builds a WhatsApp formatted message with emojis, fee info, and payment link.  
   - Connect both from "Process Fee Reminders".  

5. **Add two Wait nodes:**  
   - "Wait For Prepare Reminder" connected after "Prepare Email Reminder".  
   - "Wait For Prepare Reminder for WhatsApp" connected after "Prepare WhatsApp Reminder".  
   - No explicit wait time set; these can be configured for throttling or manual control if desired.  

6. **Add notification sending nodes:**  
   - Email Send node "Send Email Reminder":  
     - From: finance@school.edu  
     - Subject, To, Text body mapped from previous node's JSON fields.  
     - Attach SMTP credentials.  
     - Connect from "Wait For Prepare Reminder".  

   - WhatsApp node "Send message":  
     - Text body mapped from JSON message field.  
     - Recipient phone number mapped from JSON phone field (ensure correct field naming).  
     - Configure WhatsApp API credentials.  
     - Connect from "Wait For Prepare Reminder for WhatsApp".  

7. **Add a Microsoft Excel node for updating:**  
   - Name: "Update Reminder Status"  
   - Operation: Update worksheet rows using auto-mapping.  
   - Match rows on a unique identifier column (e.g., "name").  
   - Connect inputs from both "Send Email Reminder" and "Send message" nodes.  
   - Use same Microsoft Excel OAuth2 credentials as reading node.  

8. **Add a Sticky Note node (optional):**  
   - Name: "Workflow Info"  
   - Content describing the workflow process, schedule, and channels for documentation purposes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------|
| Workflow runs daily at a fixed time (8 AM) to maintain timely reminders.                                                                                                       | Scheduling best practice            |
| Payment links dynamically generated embedding student ID, amount, and fee reference for secure and direct payment access.                                                     | Payment integration details        |
| Email and WhatsApp messages include clear fee details and payment links, formatted for clarity and engagement.                                                                | Communication best practice        |
| Excel workbook must be accessible with correct permissions and up-to-date data for accurate reminders.                                                                         | Data source requirements           |
| Wait nodes can be configured to handle rate limiting or manual approval if needed.                                                                                              | Flow control and throttling        |
| Credentials for Microsoft Excel, SMTP, and WhatsApp API must be properly configured and tested before workflow activation.                                                    | Credential management              |
| Phone number field mapping for WhatsApp sending must be validated to avoid message delivery failures.                                                                           | Data validation                    |
| Sticky note contains a concise summary of the workflow process for quick reference.                                                                                            | Workflow documentation             |

---

*Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.*