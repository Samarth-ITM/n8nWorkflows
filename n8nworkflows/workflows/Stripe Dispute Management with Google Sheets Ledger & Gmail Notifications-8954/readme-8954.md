Stripe Dispute Management with Google Sheets Ledger & Gmail Notifications

https://n8nworkflows.xyz/workflows/stripe-dispute-management-with-google-sheets-ledger---gmail-notifications-8954


# Stripe Dispute Management with Google Sheets Ledger & Gmail Notifications

---

### 1. Workflow Overview

This workflow automates the management of Stripe payment disputes by integrating Stripe API data with Google Sheets for ledger keeping and Gmail for customer notifications. Its main purpose is to fetch the latest dispute information from Stripe, log disputes in a dedicated Google Sheet, update the payment ledger with dispute details, and notify customers about the dispute via email.

The workflow is structured into the following logical blocks:

- **1.1 Input Trigger:** Manual initiation of the workflow.
- **1.2 Fetching and Formatting Dispute Data:** Retrieval of dispute data from Stripe and normalization of the response.
- **1.3 Logging Disputes:** Appending new dispute records to a historical Google Sheet.
- **1.4 Payment Ledger Lookup and Update:** Searching for the payment in the master ledger and updating it if found.
- **1.5 Customer Notification:** Sending a personalized email to the customer regarding the dispute.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block provides the manual trigger to start the workflow execution on demand.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Configuration: Default manual trigger with no parameters required.  
    - Inputs: None  
    - Outputs: Triggers the next node to fetch disputes.  
    - Edge Cases: None. Manual trigger means no automatic scheduling.  
    - Version Requirements: Compatible with n8n v1+.

#### 1.2 Fetching and Formatting Dispute Data

- **Overview:**  
  This block fetches the latest disputes from Stripe API and formats the data into a normalized structure suitable for further processing.

- **Nodes Involved:**  
  - Fetch Latest Disputes from Stripe (HTTP Request)  
  - Format Stripe Dispute Data (Code)

- **Node Details:**

  - **Fetch Latest Disputes from Stripe**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET  
      - URL: Configured with Stripe API endpoint for disputes (placeholder `{{YOUR/STRIPE/URL}}`)  
      - Headers: Authorization with Stripe Secret Key (`Bearer {{YOUR/STRIPE/ SECRETEKEY}}`)  
      - Sends headers as specified  
    - Inputs: Trigger from manual node  
    - Outputs: Raw JSON response containing dispute data array  
    - Edge Cases:  
      - Authentication failure if secret key invalid  
      - API rate limiting or network errors  
      - Empty dispute data response  
    - Version Requirements: HTTP Request v4.2 or higher

  - **Format Stripe Dispute Data**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Parses the raw Stripe response, extracts only the first (latest) dispute  
      - Extracts fields: dispute_id, charge_id, payment_intent, amount (converted to decimal), currency (uppercase), reason, status, timestamps, customer email/name, dispute fee and currency  
      - Returns array with single normalized dispute object  
    - Inputs: Raw response JSON from HTTP Request node  
    - Outputs: Structured JSON ready for Google Sheets and email nodes  
    - Edge Cases:  
      - No disputes found returns empty array halting downstream functions  
      - Missing optional fields (customer email/name) handled with empty strings  
      - Potential errors if data structure changes in Stripe API  
    - Version Requirements: Code node v2

#### 1.3 Logging Disputes

- **Overview:**  
  This block appends every fetched dispute as a new row in a dedicated Google Sheet for audit and historical record.

- **Nodes Involved:**  
  - Log Dispute in Disputes Sheet

- **Node Details:**

  - **Log Dispute in Disputes Sheet**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Append row  
      - Target Sheet: Disputes sheet URL and Spreadsheet ID (placeholders `{{YOUR/SHEET/URL}}` and `{{YOUR/SPREADSHEET/URL}}`)  
      - Columns: Defined below (mapping fields from dispute data)  
    - Inputs: Structured dispute data from Code node  
    - Outputs: Passes data downstream for notification and ledger update  
    - Credentials: Google Sheets OAuth2  
    - Edge Cases:  
      - Credential expiration or access issues  
      - Rate limits on Google Sheets API  
      - Missing or malformed spreadsheet URL or sheet name  
    - Version Requirements: Google Sheets v4.6+

#### 1.4 Payment Ledger Lookup and Update

- **Overview:**  
  This block looks up the disputed charge in the master Payments ledger Google Sheet and updates the existing row with dispute information if the payment is found.

- **Nodes Involved:**  
  - Find Payment in Ledger (Google Sheets)  
  - Check if Payment Exists (If node)  
  - Update Payment Record with Dispute Info (Google Sheets)

- **Node Details:**

  - **Find Payment in Ledger**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Search for rows matching `charge_id` in Payments sheet  
      - Spreadsheet and sheet URLs as placeholders  
    - Inputs: Formatted dispute data  
    - Outputs: Rows matching the disputed charge  
    - Credentials: Google Sheets OAuth2  
    - Edge Cases:  
      - No matching charge found (empty result)  
      - Access or rate limiting errors  
    - Version Requirements: Google Sheets v4.6+

  - **Check if Payment Exists**  
    - Type: If node  
    - Configuration:  
      - Checks if `charge_id` value exists in the output from the previous node  
      - Condition: `{{$json.charge_id}}` exists (string exists operator)  
    - Inputs: Output from Find Payment in Ledger  
    - Outputs:  
      - True branch: Payment found → proceed to update  
      - False branch: Payment not found → skip update  
    - Edge Cases: Misconfiguration can lead to false negatives or false positives  
    - Version Requirements: If node v2.2+

  - **Update Payment Record with Dispute Info**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Update row in Payments sheet with dispute-related fields  
      - Requires mapping dispute fields to columns in the same row as matched payment  
      - Spreadsheet and sheet URLs as placeholders  
    - Inputs: True branch from If node, dispute data, and matched row info  
    - Credentials: Google Sheets OAuth2  
    - Edge Cases:  
      - Update fails if row index or ID is incorrect  
      - Credential or API errors  
      - Partial updates if mapping incomplete  
    - Version Requirements: Google Sheets v4.6+

#### 1.5 Customer Notification

- **Overview:**  
  This block sends a personalized email to the customer informing them about the dispute, including dispute details and next steps.

- **Nodes Involved:**  
  - Send Customer Dispute Notification Email (Gmail)

- **Node Details:**

  - **Send Customer Dispute Notification Email**  
    - Type: Gmail  
    - Configuration:  
      - Recipient: Customer email extracted from dispute data (`{{$json['customer_email ']}}`)  
      - Subject: Contains amount, currency, and respond-by date dynamically inserted  
      - Message body: Personalized text with dispute details (ID, amount, reason, status, respond-by date), reassurance about resolution time, and support contact prompt  
      - Email type: Plain text  
    - Inputs: Output from Log Dispute in Disputes Sheet node  
    - Credentials: Gmail OAuth2 (configured with user account)  
    - Edge Cases:  
      - Missing or invalid customer email address leads to email failures  
      - Gmail quota or authentication errors  
      - Formatting errors if expressions are incorrect (note trailing spaces in keys)  
    - Version Requirements: Gmail node v2.1

---

### 3. Summary Table

| Node Name                             | Node Type           | Functional Role                         | Input Node(s)                   | Output Node(s)                         | Sticky Note                                                                                                  |
|-------------------------------------|---------------------|---------------------------------------|--------------------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’     | Manual Trigger      | Workflow start trigger                 | None                           | Fetch Latest Disputes from Stripe    |                                                                                                              |
| Fetch Latest Disputes from Stripe    | HTTP Request        | Retrieve disputes from Stripe API     | When clicking ‘Execute workflow’ | Format Stripe Dispute Data            | Action: Sends a GET request to the Stripe API endpoint, using your Stripe secret key. Description: This node retrieves the raw list of recent disputes from Stripe. The API returns all the dispute details such as dispute ID, charge ID, amount, currency, reason, customer info, and deadlines. At this stage, the data is unstructured and includes extra fields you don’t need, which is why the Code node comes next. |
| Format Stripe Dispute Data           | Code                | Normalize and extract key dispute data| Fetch Latest Disputes from Stripe| Log Dispute in Disputes Sheet, Find Payment in Ledger | Action: Uses JavaScript to clean up the Stripe response and prepare only the most relevant fields. Description: This node extracts essential details like dispute_id, charge_id, payment_intent, amount, currency, reason, status, created_at, respond_by, customer_email, customer_name, dispute_fee. By normalizing and restructuring the data, this step ensures the information can flow seamlessly into Google Sheets and Gmail nodes without breaking or needing manual adjustments later. |
| Log Dispute in Disputes Sheet       | Google Sheets       | Append dispute record to audit log    | Format Stripe Dispute Data      | Send Customer Dispute Notification Email | Action: Appends a new row into the Disputes sheet every time a new dispute is fetched. Description: This step creates a historical audit log of all disputes ever raised. Every run adds a new entry, which allows the finance or support team to see a chronological record of disputes over time. This sheet acts as your permanent reference to cross-check dispute activity. |
| Send Customer Dispute Notification Email | Gmail           | Notify customer of dispute             | Log Dispute in Disputes Sheet  | None                                 | Action: Sends an email to the customer using their email address from Stripe (customer_email). Description: This node generates a personalized email notifying the customer that a dispute has been raised. It includes key details like dispute ID, amount, currency, reason, status, and respond-by date. It also reassures the customer by stating that once resolved, it may take 2–3 business days for the payment or refund to reflect in their account. This step ensures clear customer communication and reduces confusion, while also keeping your support workload lighter. |
| Find Payment in Ledger              | Google Sheets       | Search for payment in Payments ledger | Format Stripe Dispute Data      | Check if Payment Exists               | Action: Searches for a row in the Payments sheet where the charge_id matches the one in the current dispute. Description: The Payments sheet acts as your master ledger of all payments. By looking up the dispute’s charge_id in this sheet, the workflow can check if the payment already exists in your records. This step is crucial for linking disputes back to the original transaction, so that one row in the ledger always reflects the current status of that payment. |
| Check if Payment Exists             | If                  | Conditional check if payment record found | Find Payment in Ledger          | Update Payment Record with Dispute Info | Action: Evaluates the output of the Get Row(s) node. Description: If the payment exists in the ledger (i.e., a row is found for that charge_id), the workflow continues to the Update Row node. If no row is found, the branch is skipped. This logic prevents errors and ensures that only existing payments get updated, while still allowing the workflow to continue for logging and email notification. |
| Update Payment Record with Dispute Info | Google Sheets   | Update payment row with dispute info  | Check if Payment Exists         | None                                 | Action: Updates the existing row in the Payments sheet with dispute-specific fields. Description: This step enriches your Payments ledger with new details about the dispute. By keeping these values tied directly to the original payment row, your Payments sheet becomes the single source of truth, showing not only payment details but also its refund/dispute lifecycle. Finance and support teams can instantly see the latest state of any transaction without cross-checking multiple sheets. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger (n8n-nodes-base.manualTrigger)  
   - No additional configuration needed. This node starts the workflow on demand.

2. **Create HTTP Request Node to Fetch Disputes from Stripe**  
   - Type: HTTP Request  
   - Set Method to GET  
   - Set URL to Stripe disputes endpoint (e.g., `https://api.stripe.com/v1/disputes`)  
   - Add Header: Authorization with value `Bearer YOUR_STRIPE_SECRET_KEY` (replace with actual secret key)  
   - Enable 'Send Headers'  
   - Connect Manual Trigger node output to this node input.

3. **Create Code Node to Format Stripe Dispute Data**  
   - Type: Code (JavaScript)  
   - Paste JavaScript code to:  
     - Extract the first dispute object from the `data` array  
     - Normalize fields: dispute_id, charge_id, payment_intent, amount (divide by 100), currency (uppercase), reason, status, timestamps converted to ISO strings, customer email/name with fallback, dispute fee & currency  
   - Connect HTTP Request node output to this Code node.

4. **Create Google Sheets Node to Append Dispute to Disputes Sheet**  
   - Type: Google Sheets  
   - Operation: Append Row  
   - Configure Document ID and Sheet Name with your Disputes audit sheet URLs  
   - Map columns from Code node output fields accordingly  
   - Connect Code node output to this node.

5. **Create Google Sheets Node to Find Payment in Ledger**  
   - Type: Google Sheets  
   - Operation: Lookup or Read Rows filtering by `charge_id` matching the dispute's charge_id  
   - Use your Payments master ledger sheet URL and sheet name  
   - Connect Code node output to this node.

6. **Create If Node to Check if Payment Exists**  
   - Type: If  
   - Condition: Check if `charge_id` field exists in the previous node's output (non-empty)  
   - Connect Find Payment in Ledger output to this node.

7. **Create Google Sheets Node to Update Payment Record with Dispute Info**  
   - Type: Google Sheets  
   - Operation: Update Row (use row ID or index from Find Payment in Ledger node)  
   - Map dispute-related fields (status, reason, respond-by, etc.) to columns in the Payments sheet  
   - Connect If node "true" branch to this node.

8. **Create Gmail Node to Send Customer Notification Email**  
   - Type: Gmail  
   - Set recipient to customer email from dispute data (`{{$json['customer_email']}}`)  
   - Compose subject with dynamic fields (amount, currency, respond-by date)  
   - Compose plain text email body with personalized dispute details and support info  
   - Connect Log Dispute in Disputes Sheet node output to Gmail node.

9. **Configure Credentials:**  
   - Google Sheets nodes require OAuth2 credentials with access to your spreadsheet.  
   - Gmail node requires OAuth2 credentials authorized to send emails from your account.  
   - HTTP Request node uses the Stripe Secret Key in header for authentication.

10. **Test Workflow:**  
    - Execute manual trigger  
    - Verify dispute data fetched, logged, ledger updated, and customer email sent.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| The Payments sheet is the master ledger that reflects payment and dispute lifecycle details in a single row, enabling finance/support teams to track status without cross-referencing multiple sheets.                                                                                                         | Workflow description                                                                                               |
| The Disputes sheet serves as a permanent, chronological audit log of all disputes ever raised for historical reference and reconciliation.                                                                                                                                                                   | Workflow description                                                                                               |
| Customer email notifications include reassurance about the dispute resolution timeline to reduce confusion and support workload.                                                                                                                                                                            | Sticky Note on Gmail node                                                                                           |
| Stripe API keys and Google Sheet URLs must be replaced with actual credentials and URLs before running the workflow.                                                                                                                                                                                         | Configuration notes                                                                                                |
| Gmail node uses OAuth2 and may require prior setup in Google Cloud Console to enable API and consent screen.                                                                                                                                                                                                   | Gmail OAuth2 credentials setup                                                                                      |
| This workflow is inactive by default and requires activation for scheduled or automated triggers beyond manual execution.                                                                                                                                                                                  | Workflow settings                                                                                                  |
| For more details on Stripe disputes API, refer to Stripe’s official documentation: https://stripe.com/docs/api/disputes                                                                                                                                                                                     | Stripe API Documentation                                                                                           |
| For Google Sheets API usage in n8n, see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/                                                                                                                                                                                   | n8n Google Sheets documentation                                                                                   |
| Gmail node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/                                                                                                                                                                                                           | n8n Gmail node documentation                                                                                       |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with content policies and does not contain any illegal, offensive, or protected elements. All manipulated data is legal and public.

---