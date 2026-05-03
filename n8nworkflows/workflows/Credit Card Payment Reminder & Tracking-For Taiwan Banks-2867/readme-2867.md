Credit Card Payment Reminder & Tracking-For Taiwan Banks

https://n8nworkflows.xyz/workflows/credit-card-payment-reminder---tracking-for-taiwan-banks-2867


# Credit Card Payment Reminder & Tracking-For Taiwan Banks

### 1. Workflow Overview

This workflow automates the management of credit card payment reminders for eight Taiwanese banks by processing incoming credit card statement emails. It extracts key payment details such as due dates and amounts from email bodies or attached PDFs, logs this data into Google Sheets for record-keeping, and creates corresponding reminders in Google Calendar. Additionally, it provides a webhook interface to update payment statuses, reflecting changes in both Google Calendar events and Google Sheets records.

The workflow is logically divided into the following blocks:

- **1.1 Email Retrieval:** Listens for incoming credit card statement emails from eight specific Taiwanese banks using Gmail triggers.
- **1.2 Payment Information Extraction:** Parses email content and attached PDFs to extract payment due dates, total amounts, and minimum payments.
- **1.3 Data Consolidation and Storage:** Standardizes extracted data fields and stores them in a Google Sheets document.
- **1.4 Google Calendar Integration:** Creates calendar events as payment reminders and updates event statuses upon payment.
- **1.5 Webhook for Payment Status Updates:** Provides an endpoint for users to mark payments as paid, triggering updates in Google Calendar and Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Retrieval

- **Overview:**  
  This block listens for new credit card statement emails from eight Taiwanese banks using Gmail triggers. Each bank has a dedicated Gmail trigger node configured with specific email filters to capture relevant emails.

- **Nodes Involved:**  
  - Gmail-SINOPAC  
  - Gmail-Cathay  
  - Gmail-CTBC  
  - Gmail-Fubon  
  - Gmail-E.SUN  
  - Gmail-DBS  
  - Gmail-Union  
  - Gmail-Taishin

- **Node Details:**

  - **Gmail-SINOPAC**  
    - *Type:* Gmail Trigger  
    - *Role:* Listens for emails from SinoPac Bank with filter: `from:(newebill.banksinopac.com.tw) SinoPac Bank Credit Card E-Statement Notification`  
    - *Input:* Incoming emails matching filter  
    - *Output:* Email data forwarded to PDF extraction node  
    - *Failure Modes:* Authentication errors, Gmail API rate limits, email filter mismatches  
    - *Version:* 1.2

  - **Gmail-Cathay**  
    - *Type:* Gmail Trigger  
    - *Role:* Listens for emails from Cathay United Bank with filter: `from:(service@pxbillrc01.cathaybk.com.tw) Cathay United Bank Monthly E-Statement`  
    - *Input:* Incoming emails matching filter  
    - *Output:* Email data forwarded to Cathay-specific set node  
    - *Failure Modes:* Same as above  
    - *Version:* 1.2

  - **Gmail-CTBC**  
    - *Type:* Gmail Trigger  
    - *Role:* Listens for emails from CTBC Bank with filter: `from:(ebill@estats.ctbcbank.com)`  
    - *Input:* Incoming emails matching filter  
    - *Output:* Email data forwarded to CTBC-specific set node  
    - *Failure Modes:* Same as above  
    - *Version:* 1.2

  - **Gmail-Fubon**  
    - *Type:* Gmail Trigger  
    - *Role:* Listens for emails from Taipei Fubon Bank with filter: `from:(rs@cf.taipeifubon.com.tw)`  
    - *Input:* Incoming emails matching filter  
    - *Output:* Email data forwarded to PDF extraction node  
    - *Failure Modes:* Same as above  
    - *Version:* 1.2

  - **Gmail-E.SUN**  
    - *Type:* Gmail Trigger  
    - *Role:* Listens for emails from E.SUN Commercial Bank with filter: `from:(estatement@esunbank.com)`  
    - *Input:* Incoming emails matching filter  
    - *Output:* Email data forwarded to PDF extraction node  
    - *Failure Modes:* Same as above  
    - *Version:* 1.2

  - **Gmail-DBS**  
    - *Type:* Gmail Trigger  
    - *Role:* Listens for emails from DBS Bank with filter: `from:(eservicetw@dbs.com)`  
    - *Input:* Incoming emails matching filter  
    - *Output:* Email data forwarded to PDF extraction node  
    - *Failure Modes:* Same as above  
    - *Version:* 1.2

  - **Gmail-Union**  
    - *Type:* Gmail Trigger  
    - *Role:* Listens for emails from Union Bank of Taiwan with filter: `from:(聯邦銀行信用卡)`  
    - *Input:* Incoming emails matching filter  
    - *Output:* Email data forwarded to PDF extraction node  
    - *Failure Modes:* Same as above  
    - *Version:* 1.2

  - **Gmail-Taishin**  
    - *Type:* Gmail Trigger  
    - *Role:* Listens for emails from Taishin International Bank with filter: `from:(webmaster@bhurecv.taishinbank.com.tw)`  
    - *Input:* Incoming emails matching filter  
    - *Output:* Email data forwarded to PDF extraction node  
    - *Failure Modes:* Same as above  
    - *Version:* 1.2

---

#### 2.2 Payment Information Extraction

- **Overview:**  
  This block extracts payment due dates, total amounts, and minimum payments from the email content or attached PDFs. For banks providing PDFs, the workflow extracts text from attachments before parsing. For others, it uses regex parsing on email bodies.

- **Nodes Involved:**  
  - SINOPAC_PDF  
  - SINOPAC_set field  
  - Cathay_set field  
  - CTBC _set field  
  - Fubon_PDF  
  - Fubon_set field  
  - E.SUN_PDF  
  - E.SUN_set field  
  - DBS_PDF  
  - DBS_set field  
  - Union_PDF  
  - Union_set fields  
  - Taishin_PDF  
  - Taishin_set fields

- **Node Details:**

  - **PDF Extraction Nodes (e.g., SINOPAC_PDF, Fubon_PDF, E.SUN_PDF, DBS_PDF, Union_PDF, Taishin_PDF)**  
    - *Type:* Extract From File  
    - *Role:* Extracts text content from PDF attachments in emails  
    - *Input:* Email attachments (PDF files)  
    - *Output:* Extracted text forwarded to corresponding set node  
    - *Failure Modes:* Missing or corrupted PDF attachments, extraction errors, unsupported PDF formats  
    - *Version:* 1

  - **Set Field Nodes (e.g., SINOPAC_set field, Cathay_set field, CTBC _set field, Fubon_set field, E.SUN_set field, DBS_set field, Union_set fields, Taishin_set fields)**  
    - *Type:* Set  
    - *Role:* Parses extracted text or email body using regex or string operations to extract:  
      - `payment_due_date`  
      - `payment_amount`  
      - `minimum_payment`  
      - `email_id` (unique identifier)  
      - `bank` (bank name)  
      - `email_subject`  
    - *Input:* Extracted text or email content  
    - *Output:* Structured payment data forwarded to data consolidation node  
    - *Key Expressions:* Regex patterns tailored per bank to parse dates and amounts  
    - *Failure Modes:* Regex mismatches due to email format changes, missing fields, parsing errors  
    - *Version:* 3.4

---

#### 2.3 Data Consolidation and Storage

- **Overview:**  
  This block standardizes the extracted payment data into a uniform format and stores it in a Google Sheets document for tracking.

- **Nodes Involved:**  
  - Organize Data  
  - Google Sheets

- **Node Details:**

  - **Organize Data**  
    - *Type:* Set  
    - *Role:* Consolidates and formats fields from various bank-specific set nodes into a standardized schema with fields:  
      - `payment_due_date`  
      - `payment_amount`  
      - `minimum_payment`  
      - `email_id`  
      - `bank`  
      - `email_subject`  
    - *Input:* Output from bank-specific set nodes  
    - *Output:* Data forwarded to Google Calendar event creation and Google Sheets  
    - *Failure Modes:* Missing or inconsistent data fields, data type mismatches  
    - *Version:* 3.4

  - **Google Sheets**  
    - *Type:* Google Sheets  
    - *Role:* Inserts or updates rows in the sheet named `n8n-Credit Card Payment Reminder` with columns:  
      - `calendar_id`  
      - `Paid`  
      - `Billing Period`  
      - `Amount`  
      - `Minimum Payment`  
      - `Bank`  
      - `email_id`  
    - *Input:* Consolidated payment data  
    - *Output:* Triggers update of Google Calendar event status  
    - *Credential:* Requires Google Sheets OAuth2 credentials with write access  
    - *Failure Modes:* API quota limits, authentication errors, sheet access issues  
    - *Version:* 4.5

---

#### 2.4 Google Calendar Integration

- **Overview:**  
  This block creates calendar events as payment reminders and updates event statuses when payments are marked as paid.

- **Nodes Involved:**  
  - Create Google Calendar Event  
  - Update Google Calendar - change status  
  - Get Google Calendar Event by id  
  - Update Google Calendar event staus

- **Node Details:**

  - **Create Google Calendar Event**  
    - *Type:* Google Calendar  
    - *Role:* Creates an event titled `Credit Card Payment - {{ bank }}` on the `payment_due_date` with reminders set at 30 minutes, 60 minutes, and 1 day before the event  
    - *Input:* Consolidated payment data from Organize Data node  
    - *Output:* Forwards event details to Google Sheets for record linkage  
    - *Credential:* Requires Google Calendar OAuth2 credentials with event creation permissions  
    - *Failure Modes:* API limits, authentication errors, invalid date formats  
    - *Version:* 1.3

  - **Update Google Calendar - change status**  
    - *Type:* Google Calendar  
    - *Role:* Updates the calendar event to reflect payment status changes (e.g., marking as paid) by modifying event title and description  
    - *Input:* Data from Google Sheets node after payment status update  
    - *Output:* None (final step in update chain)  
    - *Credential:* Same as above  
    - *Failure Modes:* Event not found, permission errors, API rate limits  
    - *Version:* 1.3

  - **Get Google Calendar Event by id**  
    - *Type:* Google Calendar  
    - *Role:* Retrieves a calendar event by its ID to prepare for status update  
    - *Input:* Triggered by webhook input containing event ID  
    - *Output:* Passes event data to Update Google Calendar event staus node  
    - *Credential:* Same as above  
    - *Failure Modes:* Invalid event ID, event not found, permission errors  
    - *Version:* 1.3

  - **Update Google Calendar event staus**  
    - *Type:* Google Calendar  
    - *Role:* Updates event details after retrieval to reflect payment status changes  
    - *Input:* Event data from Get Google Calendar Event by id node  
    - *Output:* Triggers Google Sheets update for payment status  
    - *Credential:* Same as above  
    - *Failure Modes:* Same as above  
    - *Version:* 1.3

---

#### 2.5 Webhook for Payment Status Updates

- **Overview:**  
  This block exposes a webhook endpoint allowing users to mark payments as paid. Upon receiving a request, it retrieves the corresponding Google Calendar event, updates its status, and synchronizes the payment status in Google Sheets.

- **Nodes Involved:**  
  - Webhook  
  - Get Google Calendar Event by id  
  - Update Google Calendar event staus  
  - Update Google Sheets pay status

- **Node Details:**

  - **Webhook**  
    - *Type:* Webhook  
    - *Role:* Listens on path `darrell_demo_creditcard_pay_update_path` for incoming HTTP requests to update payment status  
    - *Input:* HTTP POST/GET requests containing payment and event identifiers  
    - *Output:* Triggers retrieval of Google Calendar event by ID  
    - *Failure Modes:* Invalid or missing parameters, unauthorized access (if not secured), timeout  
    - *Version:* 2

  - **Update Google Sheets pay status**  
    - *Type:* Google Sheets  
    - *Role:* Updates the `Paid` status in the Google Sheets record corresponding to the payment  
    - *Input:* Data from Update Google Calendar event staus node  
    - *Credential:* Same as Google Sheets node above  
    - *Failure Modes:* Sheet access errors, row not found, API limits  
    - *Version:* 4.5

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                          | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                      |
|--------------------------------|-----------------------|----------------------------------------|--------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking 'Test workflow'   | Manual Trigger        | Manual trigger for testing workflow    | -                              | -                                 |                                                                                                 |
| Gmail-SINOPAC                  | Gmail Trigger         | Retrieve SinoPac Bank emails           | -                              | SINOPAC_PDF                       | Email coming from sinopac creditcard email                                                      |
| SINOPAC_PDF                   | Extract From File     | Extract text from SinoPac PDF attachment| Gmail-SINOPAC                  | SINOPAC_set field                 |                                                                                                 |
| SINOPAC_set field             | Set                   | Parse SinoPac payment info             | SINOPAC_PDF                    | Organize Data                    |                                                                                                 |
| Gmail-Cathay                  | Gmail Trigger         | Retrieve Cathay United Bank emails     | -                              | Cathay_set field                 |                                                                                                 |
| Cathay_set field              | Set                   | Parse Cathay payment info              | Gmail-Cathay                   | Organize Data                    |                                                                                                 |
| Gmail-CTBC                   | Gmail Trigger         | Retrieve CTBC Bank emails              | -                              | CTBC _set field                 |                                                                                                 |
| CTBC _set field              | Set                   | Parse CTBC payment info                 | Gmail-CTBC                    | Organize Data                    |                                                                                                 |
| Gmail-Fubon                  | Gmail Trigger         | Retrieve Taipei Fubon Bank emails      | -                              | Fubon_PDF                       |                                                                                                 |
| Fubon_PDF                   | Extract From File     | Extract text from Fubon PDF attachment | Gmail-Fubon                   | Fubon_set field                 |                                                                                                 |
| Fubon_set field             | Set                   | Parse Fubon payment info               | Fubon_PDF                    | Organize Data                    |                                                                                                 |
| Gmail-E.SUN                 | Gmail Trigger         | Retrieve E.SUN Bank emails             | -                              | E.SUN_PDF                      |                                                                                                 |
| E.SUN_PDF                  | Extract From File     | Extract text from E.SUN PDF attachment | Gmail-E.SUN                   | E.SUN_set field                |                                                                                                 |
| E.SUN_set field            | Set                   | Parse E.SUN payment info               | E.SUN_PDF                    | Organize Data                    |                                                                                                 |
| Gmail-DBS                   | Gmail Trigger         | Retrieve DBS Bank emails               | -                              | DBS_PDF                       |                                                                                                 |
| DBS_PDF                    | Extract From File     | Extract text from DBS PDF attachment   | Gmail-DBS                    | DBS_set field                 |                                                                                                 |
| DBS_set field              | Set                   | Parse DBS payment info                 | DBS_PDF                     | Organize Data                    |                                                                                                 |
| Gmail-Union                 | Gmail Trigger         | Retrieve Union Bank emails             | -                              | Union_PDF                     |                                                                                                 |
| Union_PDF                  | Extract From File     | Extract text from Union PDF attachment | Gmail-Union                  | Union_set fields              |                                                                                                 |
| Union_set fields           | Set                   | Parse Union payment info               | Union_PDF                   | Organize Data                    |                                                                                                 |
| Gmail-Taishin               | Gmail Trigger         | Retrieve Taishin Bank emails           | -                              | Taishin_PDF                   |                                                                                                 |
| Taishin_PDF                | Extract From File     | Extract text from Taishin PDF attachment| Gmail-Taishin                | Taishin_set fields            |                                                                                                 |
| Taishin_set fields         | Set                   | Parse Taishin payment info             | Taishin_PDF                 | Organize Data                    |                                                                                                 |
| Organize Data              | Set                   | Consolidate and standardize payment data| Multiple set nodes           | Create Google Calendar Event    |                                                                                                 |
| Create Google Calendar Event | Google Calendar       | Create payment reminder event          | Organize Data                | Google Sheets                  |                                                                                                 |
| Google Sheets              | Google Sheets         | Store payment data                      | Create Google Calendar Event | Update Google Calendar - change status |                                                                                                 |
| Update Google Calendar - change status | Google Calendar       | Update calendar event after payment    | Google Sheets                | -                             |                                                                                                 |
| Webhook                    | Webhook               | Receive payment status update requests | -                           | Get Google Calendar Event by id |                                                                                                 |
| Get Google Calendar Event by id | Google Calendar       | Retrieve calendar event by ID           | Webhook                     | Update Google Calendar event staus |                                                                                                 |
| Update Google Calendar event staus | Google Calendar       | Update calendar event status            | Get Google Calendar Event by id | Update Google Sheets pay status |                                                                                                 |
| Update Google Sheets pay status | Google Sheets         | Update payment status in Google Sheets  | Update Google Calendar event staus | -                             |                                                                                                 |
| When clicking 'Test workflow' | Manual Trigger        | Manual trigger for testing workflow    | -                              | -                                 |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Nodes for Each Bank**  
   - Create eight Gmail Trigger nodes named after each bank (e.g., Gmail-SINOPAC).  
   - Configure each with the respective email filter:  
     - SinoPac: `from:(newebill.banksinopac.com.tw) SinoPac Bank Credit Card E-Statement Notification`  
     - Cathay: `from:(service@pxbillrc01.cathaybk.com.tw) Cathay United Bank Monthly E-Statement`  
     - CTBC: `from:(ebill@estats.ctbcbank.com)`  
     - Taipei Fubon: `from:(rs@cf.taipeifubon.com.tw)`  
     - E.SUN: `from:(estatement@esunbank.com)`  
     - DBS: `from:(eservicetw@dbs.com)`  
     - Union Bank: `from:(聯邦銀行信用卡)`  
     - Taishin: `from:(webmaster@bhurecv.taishinbank.com.tw)`  
   - Set node version to 1.2.

2. **Add PDF Extraction Nodes for Banks Providing PDFs**  
   - For SinoPac, Fubon, E.SUN, DBS, Union, and Taishin, add "Extract From File" nodes connected to their Gmail triggers.  
   - Configure to extract text from PDF attachments.  
   - Version 1.

3. **Add Set Nodes to Parse Payment Information**  
   - For each bank, add a Set node connected to the PDF extraction node or Gmail trigger (for Cathay and CTBC which parse email body).  
   - Configure regex or string parsing expressions to extract:  
     - `payment_due_date`  
     - `payment_amount`  
     - `minimum_payment`  
     - `email_id` (unique email identifier)  
     - `bank` (bank name)  
     - `email_subject`  
   - Version 3.4.

4. **Add a Consolidation Set Node ("Organize Data")**  
   - Connect all bank-specific Set nodes to this node.  
   - Configure to standardize and unify the data fields into a consistent format.  
   - Version 3.4.

5. **Create Google Calendar Event Node**  
   - Connect "Organize Data" to a Google Calendar node.  
   - Configure event creation with:  
     - Title: `Credit Card Payment - {{ bank }}`  
     - Date: `payment_due_date`  
     - Reminders: 30 minutes, 60 minutes, and 1 day before event  
   - Requires Google Calendar OAuth2 credentials.  
   - Version 1.3.

6. **Create Google Sheets Node to Store Data**  
   - Connect Google Calendar event node to Google Sheets node.  
   - Configure to insert/update rows in sheet `n8n-Credit Card Payment Reminder` with columns:  
     - `calendar_id` (from created event)  
     - `Paid` (default to unpaid)  
     - `Billing Period` (if applicable)  
     - `Amount`  
     - `Minimum Payment`  
     - `Bank`  
     - `email_id`  
   - Requires Google Sheets OAuth2 credentials.  
   - Version 4.5.

7. **Add Google Calendar Update Node for Payment Status Changes**  
   - Connect Google Sheets node to a Google Calendar node configured to update event title/description to reflect payment status.  
   - Version 1.3.

8. **Create Webhook Node for Payment Status Updates**  
   - Add a Webhook node with path `darrell_demo_creditcard_pay_update_path`.  
   - Configure to accept HTTP requests with payment and event identifiers.  
   - Version 2.

9. **Add Google Calendar Get Event Node**  
   - Connect Webhook node to a Google Calendar node to retrieve event by ID.  
   - Version 1.3.

10. **Add Google Calendar Update Event Node**  
    - Connect Get Event node to Google Calendar node to update event status.  
    - Version 1.3.

11. **Add Google Sheets Update Node for Payment Status**  
    - Connect Google Calendar update node to Google Sheets node to update the `Paid` column.  
    - Version 4.5.

12. **Optional: Add Manual Trigger Node for Testing**  
    - Add a Manual Trigger node to test workflow manually.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow automates credit card payment reminders for eight Taiwanese banks using Gmail, Google Sheets, and Google Calendar integrations. | Workflow description provided by author.                                                          |
| Webhook path for payment status updates: `darrell_demo_creditcard_pay_update_path`                   | Used to mark payments as paid and update calendar and sheets accordingly.                          |
| Google Sheets document name: `n8n-Credit Card Payment Reminder`                                      | Stores all extracted payment data and payment statuses.                                           |
| Google Calendar events include reminders at 30 minutes, 60 minutes, and 1 day before due date.       | Ensures timely notifications for payments.                                                        |
| Gmail filters are bank-specific to ensure only relevant emails trigger the workflow.                 | Critical for filtering out unrelated emails and avoiding false triggers.                           |
| Potential failure points include Gmail API limits, PDF extraction errors, regex parsing mismatches, and Google API authentication errors. | Users should monitor API quotas and update regex patterns if email formats change.                 |
| For detailed regex patterns and bank-specific parsing logic, refer to the Set nodes configuration.   | Parsing logic is customized per bank due to varying email and PDF formats.                         |

---

This document provides a comprehensive understanding of the workflow structure, node configurations, and reproduction steps to facilitate maintenance, customization, or automation by advanced users and AI agents.