Automated Invoice Follow-ups with Smart Response Tones via Google Sheets & Email

https://n8nworkflows.xyz/workflows/automated-invoice-follow-ups-with-smart-response-tones-via-google-sheets---email-6192


# Automated Invoice Follow-ups with Smart Response Tones via Google Sheets & Email

### 1. Workflow Overview

This workflow automates invoice follow-ups by integrating Google Sheets and email communication, designed primarily for freelancers, agencies, and finance teams who manage unpaid invoices. It runs daily to scan and process invoice data, calculate how overdue invoices are, generate personalized reminder messages with tones varying according to overdue duration, and send these reminders via email.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow once daily at a configured time.
- **1.2 Invoice Data Loading:** Retrieves invoice data from a Google Sheet.
- **1.3 Overdue Invoice Filtering:** Selects only unpaid invoices that are past their due date.
- **1.4 Days Past Due (DPD) Calculation:** Computes how many days each invoice is overdue.
- **1.5 Personalized Message Generation:** Creates email messages with tone varying by DPD.
- **1.6 Email Dispatch:** Sends the generated reminders via email.
- **1.7 Documentation and Notes:** Provides contextual information and setup instructions through sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow automatically once per day to ensure regular invoice follow-up.

- **Nodes Involved:**  
  - *Daily Trigger*

- **Node Details:**

  - **Daily Trigger**  
    - *Type:* Cron Trigger  
    - *Role:* Initiates workflow execution daily at 6:00 AM.  
    - *Configuration:* Set to trigger every day at hour 6 (6 AM).  
    - *Input:* None (trigger node).  
    - *Output:* Starts the flow to the next node.  
    - *Edge Cases:*  
      - Workflow depends on server time; misconfigured timezone may cause unexpected trigger time.  
      - Cron node failure or misconfiguration would halt the workflow initiation.

#### 1.2 Invoice Data Loading

- **Overview:**  
  Fetches invoice data from a specified range in a Google Sheets document using OAuth2 authentication.

- **Nodes Involved:**  
  - *Load Invoices*

- **Node Details:**

  - **Load Invoices**  
    - *Type:* Google Sheets (Lookup operation)  
    - *Role:* Reads invoice records from the "Invoices" sheet, range A1:G1000.  
    - *Configuration:*  
      - Uses OAuth2 credentials for Google Sheets API access.  
      - Range is fixed covering columns A to G, rows 1 to 1000.  
      - Operation set to "lookup" to retrieve data.  
    - *Input:* Trigger from Daily Trigger node.  
    - *Output:* Passes all invoice data downstream.  
    - *Edge Cases:*  
      - Authentication failure or expired token causes data retrieval failure.  
      - Sheet ID must be replaced with the user’s actual Google Sheet ID; incorrect ID results in errors.  
      - Range must include all relevant invoice columns (`client_name`, `email`, `due_date`, `invoice_number`, `status`).  
      - Large datasets may cause performance lag or API rate limits.

#### 1.3 Overdue Invoice Filtering

- **Overview:**  
  Filters invoice records to keep only those that are unpaid and past their due date.

- **Nodes Involved:**  
  - *Filter Overdue Invoices*

- **Node Details:**

  - **Filter Overdue Invoices**  
    - *Type:* Function Node  
    - *Role:* Applies JavaScript filtering to omit paid or not-yet-due invoices.  
    - *Configuration:*  
      - Filters items where `status` equals "unpaid" (case-insensitive) and `due_date` is before the current date.  
      - Assumes `due_date` is in a format parseable by JavaScript Date.  
    - *Input:* Invoice data from Load Invoices.  
    - *Output:* Only overdue unpaid invoices passed forward.  
    - *Edge Cases:*  
      - If `due_date` is missing or malformed, Date parsing may fail.  
      - Case sensitivity of `status` handled by `.toLowerCase()`.  
      - Empty results mean no overdue invoices; downstream nodes must handle empty input gracefully.

#### 1.4 Days Past Due (DPD) Calculation

- **Overview:**  
  Calculates how many days each invoice is overdue to customize the follow-up tone.

- **Nodes Involved:**  
  - *Calculate DPD*

- **Node Details:**

  - **Calculate DPD**  
    - *Type:* Function Node  
    - *Role:* Computes integer DPD by subtracting due date from current date.  
    - *Configuration:*  
      - Uses JavaScript to calculate difference in milliseconds, converts to days.  
      - Adds a new `dpd` field to each item’s JSON.  
    - *Input:* Filtered overdue invoices.  
    - *Output:* Items enriched with `dpd` value.  
    - *Edge Cases:*  
      - Invalid or missing `due_date` will cause incorrect or NaN `dpd`.  
      - Timezone inconsistencies could slightly affect day difference calculation.

#### 1.5 Personalized Message Generation

- **Overview:**  
  Generates personalized reminder messages with tone tailored to the days overdue.

- **Nodes Involved:**  
  - *Generate Message*

- **Node Details:**

  - **Generate Message**  
    - *Type:* Function Node  
    - *Role:* Constructs email message content conditionally by `dpd` ranges:  
      - <=7 days: friendly reminder  
      - 8–14 days: firmer request  
      - >14 days: final warning  
    - *Configuration:*  
      - Uses template literals and variables: `client_name`, `invoice_number`, `due_date`, and computed `dpd`.  
      - Stores generated text in `message` field in each JSON item.  
    - *Input:* Items with `dpd`.  
    - *Output:* Items with `message` ready to send.  
    - *Edge Cases:*  
      - Missing client details (`client_name`, `invoice_number`) will result in incomplete messages.  
      - Non-string or malformed fields might disrupt message formatting.

#### 1.6 Email Dispatch

- **Overview:**  
  Sends the generated invoice reminders via email to clients.

- **Nodes Involved:**  
  - *Send Email*

- **Node Details:**

  - **Send Email**  
    - *Type:* Email Send Node  
    - *Role:* Sends an email using the message content as the email body.  
    - *Configuration:*  
      - Subject fixed as "Invoice Reminder".  
      - Email body populated dynamically from the `message` field of incoming JSON.  
      - Requires configured SMTP or Gmail credentials for sending emails.  
    - *Input:* Items with generated messages.  
    - *Output:* Email sent confirmation or error.  
    - *Edge Cases:*  
      - Missing or invalid recipient email addresses in item JSON (assumed present but not explicitly shown) may cause send failures.  
      - Email server authentication errors or quota limits.  
      - Network issues causing timeouts.  
      - If multiple items processed, ensure batching or concurrency settings to prevent server overload.

#### 1.7 Documentation and Notes

- **Overview:**  
  Provides user guidance, problem context, and setup instructions embedded inside sticky notes visible in the workflow editor.

- **Nodes Involved:**  
  - *Sticky Note*  
  - *Sticky Note1*

- **Node Details:**

  - **Sticky Note**  
    - *Type:* Sticky Note  
    - *Role:* Placeholder (empty content with a title "## Flow").  
    - *Input/Output:* None.

  - **Sticky Note1**  
    - *Type:* Sticky Note  
    - *Role:* Detailed description of the workflow purpose, problem, solution, audience, scope, and setup instructions.  
    - *Content Highlights:*  
      - Problem statement about late payments.  
      - Solution overview with daily scan, filtering, calculation, personalized messages, email sending.  
      - Target users: freelancers, agencies, finance teams, etc.  
      - Setup steps with placeholders to replace sheet ID and configure email credentials.  
      - Suggestions for expansion (WhatsApp, Slack, penalty logic).  
    - *Input/Output:* None.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                      | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                                      |
|-----------------------|---------------------|------------------------------------|------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Daily Trigger         | Cron Trigger        | Starts workflow daily at 6 AM      | None                   | Load Invoices             |                                                                                                                                 |
| Load Invoices         | Google Sheets       | Loads invoices from Google Sheet   | Daily Trigger          | Filter Overdue Invoices   |                                                                                                                                 |
| Filter Overdue Invoices| Function            | Filters unpaid and overdue invoices| Load Invoices          | Calculate DPD             |                                                                                                                                 |
| Calculate DPD         | Function            | Calculates days past due (DPD)     | Filter Overdue Invoices| Generate Message          |                                                                                                                                 |
| Generate Message      | Function            | Generates email message per DPD    | Calculate DPD          | Send Email                |                                                                                                                                 |
| Send Email            | Email Send          | Sends reminder emails               | Generate Message       | None                     |                                                                                                                                 |
| Sticky Note           | Sticky Note         | Workflow placeholder note          | None                   | None                     | "## Flow"                                                                                                                       |
| Sticky Note1          | Sticky Note         | Detailed workflow description      | None                   | None                     | "# AI Invoice & Payment Follow-Up Agent ... Setup instructions and use cases"                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**
   - Type: Cron Trigger  
   - Name: Daily Trigger  
   - Set trigger time to daily at 6:00 AM (hour: 6).  
   - No credentials needed.

2. **Create Google Sheets Node**
   - Type: Google Sheets  
   - Name: Load Invoices  
   - Operation: Lookup  
   - Set Sheet ID to your Google Sheet containing invoices.  
   - Range: `Invoices!A1:G1000` (adjust to your data range).  
   - Authentication: Use OAuth2 credentials for Google Sheets API.  
   - Connect output of Daily Trigger to input of Load Invoices.

3. **Create Function Node for Filtering Overdue**
   - Type: Function  
   - Name: Filter Overdue Invoices  
   - Paste the following JavaScript code:

     ```javascript
     return items.filter(item => {
       const dueDate = new Date(item.json.due_date);
       const today = new Date();
       return item.json.status.toLowerCase() === "unpaid" && dueDate < today;
     });
     ```

   - Connect Load Invoices → Filter Overdue Invoices.

4. **Create Function Node for Calculating DPD**
   - Type: Function  
   - Name: Calculate DPD  
   - Paste code:

     ```javascript
     return items.map(item => {
       const dueDate = new Date(item.json.due_date);
       const today = new Date();
       const dpd = Math.floor((today - dueDate) / (1000 * 60 * 60 * 24));
       item.json.dpd = dpd;
       return item;
     });
     ```

   - Connect Filter Overdue Invoices → Calculate DPD.

5. **Create Function Node for Message Generation**
   - Type: Function  
   - Name: Generate Message  
   - Paste code:

     ```javascript
     return items.map(item => {
       const dpd = item.json.dpd;
       let tone = "";
       if (dpd <= 7) {
         tone = `Hi ${item.json.client_name}, just a quick reminder about your invoice #${item.json.invoice_number} due on ${item.json.due_date}. Let us know if you need anything!`;
       } else if (dpd <= 14) {
         tone = `Hi ${item.json.client_name}, your invoice #${item.json.invoice_number} is now ${dpd} days overdue. Please arrange the payment soon.`;
       } else {
         tone = `Final Reminder: Invoice #${item.json.invoice_number} is ${dpd} days overdue. Immediate payment is required to avoid further action.`;
       }
       item.json.message = tone;
       return item;
     });
     ```

   - Connect Calculate DPD → Generate Message.

6. **Create Email Send Node**
   - Type: Email Send  
   - Name: Send Email  
   - Subject: `Invoice Reminder`  
   - Text: Use expression `{{$json.message}}` to dynamically insert generated message.  
   - Configure SMTP or Gmail credentials for sending emails.  
   - Connect Generate Message → Send Email.

7. **Add Sticky Notes (Optional) for Documentation**
   - Add two sticky notes with the same content as original for context and instructions, positioned as desired.

8. **Activate the Workflow**
   - Ensure credentials are valid and OAuth2 tokens refreshed.  
   - Activate the Cron Trigger node to start daily execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Replace `YOUR_SHEET_ID` in the Google Sheets node with the actual ID of your Google Sheet containing invoice data.                                                                                                            | Google Sheets setup                               |
| Customize the reminder messages in the “Generate Message” node to fit your preferred tone or language.                                                                                                                         | Message personalization                            |
| Set up SMTP or Gmail credentials in the Email Send node to enable sending emails.                                                                                                                                                | Email credentials setup                           |
| Optional expansions include adding multi-channel notifications (WhatsApp, Slack), penalty logic, automatic PDF resend, or Google Calendar integration for finance tracking.                                                     | Workflow extensibility suggestions                 |
| Workflow designed specifically for freelancers, agencies, finance teams managing invoice collections, but adaptable for broader billing automation use cases.                                                                  | Target user groups                                |
| Official n8n documentation for Google Sheets OAuth2 setup and Email node configuration is recommended for credential management.                                                                                              | https://docs.n8n.io/integrations/builtin/google-sheets, https://docs.n8n.io/integrations/builtin/email/ |

---

**Disclaimer:** The provided text is extracted exclusively from a workflow automated with n8n, a tool for integration and automation. This processing strictly respects content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.