🔔 Meta Ads Low Balance Alert – Auto Notification via WhatsApp or Email

https://n8nworkflows.xyz/workflows/---meta-ads-low-balance-alert---auto-notification-via-whatsapp-or-email-3695


# 🔔 Meta Ads Low Balance Alert – Auto Notification via WhatsApp or Email

### 1. Workflow Overview

This workflow automates the monitoring of Meta Ads (Facebook) ad account balances and sends alerts when balances fall below predefined thresholds. It targets marketing professionals and agencies managing multiple Meta Ads accounts who want to avoid campaign interruptions due to low balances. The workflow integrates with Google Sheets for account data and thresholds, Meta Ads API for balance retrieval, Evolution API for WhatsApp alerts, and Gmail for email notifications.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Initialization:** Starts the workflow either on a scheduled basis or manually, then maps and loads client account data from Google Sheets.
- **1.2 Account Loop & Meta Ads API Query:** Iterates over each client account, queries the Meta Ads API for the current balance.
- **1.3 Payment Type Decision & Threshold Check:** Determines if the account is prepaid or not, then compares the balance against the minimum threshold.
- **1.4 Alert Dispatch:** Sends alerts via WhatsApp (Evolution API) or Gmail depending on payment type and balance status.
- **1.5 Google Sheets Update & Loop Continuation:** Updates the Google Sheet with the latest balance and status, then waits before processing the next account.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Initialization

**Overview:**  
This block initiates the workflow either by a scheduled trigger (daily or custom interval) or manual trigger. It then maps input data and loads the client account list from a Google Sheet.

**Nodes Involved:**  
- Start by Period (Schedule Trigger)  
- Start by Click (Manual Trigger)  
- Mapping (Set)  
- Customer Base (Google Sheets)

**Node Details:**

- **Start by Period**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow on a defined schedule (e.g., daily).  
  - Configuration: Default scheduling parameters (can be customized).  
  - Inputs: None (trigger node).  
  - Outputs: Triggers Mapping node.  
  - Edge Cases: Scheduling misconfiguration or time zone issues could delay or skip runs.

- **Start by Click**  
  - Type: Manual Trigger  
  - Role: Allows manual execution of the workflow for on-demand checks.  
  - Configuration: Default manual trigger.  
  - Inputs: None.  
  - Outputs: Triggers Mapping node.  
  - Edge Cases: None significant.

- **Mapping**  
  - Type: Set  
  - Role: Prepares or adjusts data before fetching client accounts.  
  - Configuration: Likely sets parameters or variables for downstream nodes.  
  - Inputs: From trigger nodes.  
  - Outputs: Customer Base node.  
  - Edge Cases: Misconfigured expressions could cause data errors.

- **Customer Base**  
  - Type: Google Sheets  
  - Role: Reads the client accounts and their minimum balance thresholds from a Google Sheet.  
  - Configuration: Reads specific sheet and range containing account info (client IDs, thresholds, payment types).  
  - Inputs: From Mapping node.  
  - Outputs: Loop Over Items node.  
  - Edge Cases: Google API auth errors, sheet range misconfiguration, empty or malformed data.

---

#### 2.2 Account Loop & Meta Ads API Query

**Overview:**  
Iterates over each client account from the sheet, querying the Meta Ads API to retrieve the current ad account balance.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Meta Ads (HTTP Request)

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes accounts one at a time or in small batches to avoid API rate limits.  
  - Configuration: Default batch size (likely 1).  
  - Inputs: Customer Base node output.  
  - Outputs: Meta Ads node (main output), or loop continuation.  
  - Edge Cases: Large datasets could slow processing; batch size misconfiguration.

- **Meta Ads**  
  - Type: HTTP Request  
  - Role: Calls Facebook Graph API to fetch the ad account balance.  
  - Configuration:  
    - HTTP Method: GET  
    - URL: Facebook Graph API endpoint for ad account balance  
    - Authentication: OAuth2 or Access Token credential for Meta Ads API  
    - Query Parameters: Account ID from current loop item  
  - Inputs: Loop Over Items node.  
  - Outputs: Is it Prepaid? node.  
  - Edge Cases: API rate limits, expired tokens, network errors, invalid account IDs.

---

#### 2.3 Payment Type Decision & Threshold Check

**Overview:**  
Determines if the account is prepaid or not and compares the current balance against the minimum threshold defined in the Google Sheet. Branches logic accordingly.

**Nodes Involved:**  
- Is it Prepaid? (If)  
- Base Value (If)  
- Base Value1 (If)  
- Google Sheets  
- Google Sheets1

**Node Details:**

- **Is it Prepaid?**  
  - Type: If  
  - Role: Checks if the payment method for the account is prepaid (e.g., Boleto or PIX).  
  - Configuration: Expression evaluating payment type field from account data.  
  - Inputs: Meta Ads node output.  
  - Outputs:  
    - True branch: Base Value node (for prepaid accounts)  
    - False branch: Base Value1 node (for credit card accounts)  
  - Edge Cases: Missing or malformed payment type data.

- **Base Value**  
  - Type: If  
  - Role: Compares current balance to minimum threshold for prepaid accounts.  
  - Configuration: Expression comparing Meta Ads balance to "Base Value" from Google Sheet.  
  - Inputs: Is it Prepaid? true branch.  
  - Outputs: Google Sheets (update node) if balance is low; else no alert.  
  - Edge Cases: Missing threshold values, expression errors.

- **Base Value1**  
  - Type: If  
  - Role: Compares current balance to minimum threshold for credit card accounts.  
  - Configuration: Expression similar to Base Value but for non-prepaid accounts.  
  - Inputs: Is it Prepaid? false branch.  
  - Outputs: Google Sheets1 (update node) if balance is low; else no alert.  
  - Edge Cases: Same as Base Value.

- **Google Sheets**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet with new balance, check date, and status for prepaid accounts.  
  - Configuration: Writes to specific sheet and range, updating relevant columns.  
  - Inputs: Base Value node (true branch).  
  - Outputs: Evolution node (WhatsApp alert).  
  - Edge Cases: API auth errors, write conflicts.

- **Google Sheets1**  
  - Type: Google Sheets  
  - Role: Updates Google Sheet similarly for credit card accounts.  
  - Configuration: Similar to Google Sheets node but for different sheet or range.  
  - Inputs: Base Value1 node (true branch).  
  - Outputs: Evolution1 node (WhatsApp alert).  
  - Edge Cases: Same as Google Sheets.

---

#### 2.4 Alert Dispatch

**Overview:**  
Sends low balance alerts via WhatsApp using the Evolution API or via Gmail email notifications, customized based on payment type.

**Nodes Involved:**  
- Evolution (HTTP Request)  
- Evolution1 (HTTP Request)  
- Gmail (Gmail) [Disabled]  
- Gmail1 (Gmail) [Disabled]

**Node Details:**

- **Evolution**  
  - Type: HTTP Request  
  - Role: Sends WhatsApp alert via Evolution API for prepaid accounts.  
  - Configuration:  
    - HTTP Method: POST  
    - URL: Evolution API endpoint  
    - Authentication: API key or token stored in credentials  
    - Payload: Customized message including account info and alert details  
  - Inputs: Google Sheets node output.  
  - Outputs: Gmail node (disabled by default).  
  - Edge Cases: API downtime, auth errors, message formatting issues.

- **Evolution1**  
  - Type: HTTP Request  
  - Role: Sends WhatsApp alert for credit card accounts via Evolution API.  
  - Configuration: Similar to Evolution node but with different message template.  
  - Inputs: Google Sheets1 node output.  
  - Outputs: Gmail1 node (disabled by default).  
  - Edge Cases: Same as Evolution.

- **Gmail** (Disabled)  
  - Type: Gmail  
  - Role: Sends email alert for prepaid accounts if enabled.  
  - Configuration: Uses Gmail OAuth2 credentials, email template with alert details.  
  - Inputs: Evolution node output.  
  - Outputs: Replace Me node.  
  - Edge Cases: Gmail API rate limits, auth errors.

- **Gmail1** (Disabled)  
  - Type: Gmail  
  - Role: Sends email alert for credit card accounts if enabled.  
  - Configuration: Similar to Gmail node.  
  - Inputs: Evolution1 node output.  
  - Outputs: Replace Me node.  
  - Edge Cases: Same as Gmail.

---

#### 2.5 Google Sheets Update & Loop Continuation

**Overview:**  
After sending alerts, the workflow updates the Google Sheet with the latest balance and status, then waits briefly before processing the next account to avoid API rate limits.

**Nodes Involved:**  
- Replace Me (NoOp)  
- Wait (Wait)  
- Loop Over Items (SplitInBatches)

**Node Details:**

- **Replace Me**  
  - Type: NoOp  
  - Role: Placeholder node to maintain workflow structure; used after Gmail nodes.  
  - Configuration: No parameters.  
  - Inputs: Gmail or Gmail1 nodes.  
  - Outputs: Wait node.  
  - Edge Cases: None.

- **Wait**  
  - Type: Wait  
  - Role: Pauses execution for a configured time to avoid hitting API rate limits.  
  - Configuration: Default or custom wait time (e.g., seconds).  
  - Inputs: Replace Me node.  
  - Outputs: Loop Over Items node (to continue processing next batch).  
  - Edge Cases: Excessive wait times could slow workflow; zero wait risks rate limits.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Receives control to process next batch or ends loop if done.  
  - Inputs: Wait node output.  
  - Outputs: Meta Ads node or ends workflow.  
  - Edge Cases: Infinite loops if batch size or loop termination misconfigured.

---

### 3. Summary Table

| Node Name        | Node Type           | Functional Role                         | Input Node(s)          | Output Node(s)         | Sticky Note                         |
|------------------|---------------------|---------------------------------------|-----------------------|-----------------------|-----------------------------------|
| Start by Period  | Schedule Trigger    | Scheduled workflow start               | None                  | Mapping               |                                   |
| Start by Click   | Manual Trigger      | Manual workflow start                  | None                  | Mapping               |                                   |
| Mapping          | Set                 | Data preparation                      | Start by Period, Start by Click | Customer Base         |                                   |
| Customer Base    | Google Sheets       | Load client accounts and thresholds   | Mapping                | Loop Over Items        |                                   |
| Loop Over Items  | SplitInBatches      | Iterate over accounts                  | Customer Base          | Meta Ads (main), Loop Over Items (second output) |                                   |
| Meta Ads         | HTTP Request        | Query Meta Ads API for balance        | Loop Over Items        | Is it Prepaid?         |                                   |
| Is it Prepaid?   | If                  | Check payment type                     | Meta Ads               | Base Value (true), Base Value1 (false) |                                   |
| Base Value       | If                  | Compare balance vs threshold (prepaid)| Is it Prepaid? (true)  | Google Sheets          |                                   |
| Base Value1      | If                  | Compare balance vs threshold (credit card) | Is it Prepaid? (false) | Google Sheets1         |                                   |
| Google Sheets    | Google Sheets       | Update sheet for prepaid accounts     | Base Value              | Evolution              |                                   |
| Google Sheets1   | Google Sheets       | Update sheet for credit card accounts | Base Value1             | Evolution1             |                                   |
| Evolution        | HTTP Request        | Send WhatsApp alert (prepaid)         | Google Sheets           | Gmail (disabled)       |                                   |
| Evolution1       | HTTP Request        | Send WhatsApp alert (credit card)     | Google Sheets1          | Gmail1 (disabled)      |                                   |
| Gmail            | Gmail               | Send email alert (prepaid) [disabled] | Evolution               | Replace Me             |                                   |
| Gmail1           | Gmail               | Send email alert (credit card) [disabled] | Evolution1              | Replace Me             |                                   |
| Replace Me       | NoOp                 | Placeholder after email sending       | Gmail, Gmail1           | Wait                   |                                   |
| Wait             | Wait                 | Pause between iterations               | Replace Me              | Loop Over Items         |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node named "Start by Period" and configure it to run daily or as desired.  
   - Add a **Manual Trigger** node named "Start by Click" for manual execution.

2. **Add Data Preparation Node:**  
   - Add a **Set** node named "Mapping" connected to both triggers. Configure it to set any necessary variables or parameters (e.g., date/time or flags).

3. **Load Client Accounts:**  
   - Add a **Google Sheets** node named "Customer Base" connected to "Mapping".  
   - Configure it to read from the Google Sheet containing client accounts, thresholds, and payment types. Specify the sheet name and range.  
   - Set Google Sheets credentials with appropriate access.

4. **Loop Over Accounts:**  
   - Add a **SplitInBatches** node named "Loop Over Items" connected to "Customer Base".  
   - Set batch size to 1 to process accounts individually.

5. **Query Meta Ads API:**  
   - Add an **HTTP Request** node named "Meta Ads" connected to the main output of "Loop Over Items".  
   - Configure it to call the Facebook Graph API endpoint for ad account balance, using the current account ID from the batch.  
   - Use OAuth2 or access token credentials for Meta Ads API authentication.

6. **Payment Type Decision:**  
   - Add an **If** node named "Is it Prepaid?" connected to "Meta Ads".  
   - Configure the condition to check the payment type field from the account data (e.g., if payment method equals "Boleto" or "PIX").

7. **Balance Threshold Checks:**  
   - Add two **If** nodes named "Base Value" (for prepaid) and "Base Value1" (for credit card), connected to the true and false outputs of "Is it Prepaid?" respectively.  
   - Configure each to compare the current balance from Meta Ads to the minimum threshold value from the Google Sheet.

8. **Update Google Sheets:**  
   - Add two **Google Sheets** nodes named "Google Sheets" (prepaid) and "Google Sheets1" (credit card), connected to the true outputs of "Base Value" and "Base Value1".  
   - Configure each to update the respective row with the new balance, check date, and status. Use the same Google Sheets credentials.

9. **Send WhatsApp Alerts:**  
   - Add two **HTTP Request** nodes named "Evolution" and "Evolution1" connected to "Google Sheets" and "Google Sheets1" respectively.  
   - Configure each to send a POST request to the Evolution API endpoint with appropriate authentication and a customized message payload based on payment type.

10. **Send Email Alerts (Optional):**  
    - Add two **Gmail** nodes named "Gmail" and "Gmail1" connected to "Evolution" and "Evolution1".  
    - Configure Gmail OAuth2 credentials.  
    - Set up email templates for alerts.  
    - These nodes are disabled by default; enable if email alerts are desired.

11. **Add Placeholder Node:**  
    - Add a **NoOp** node named "Replace Me" connected to the Gmail nodes.  
    - This node acts as a placeholder to maintain workflow structure.

12. **Add Wait Node:**  
    - Add a **Wait** node named "Wait" connected to "Replace Me".  
    - Configure a short wait time (e.g., a few seconds) to avoid hitting API rate limits.

13. **Loop Continuation:**  
    - Connect the output of "Wait" back to the second output of "Loop Over Items" to continue processing the next account.

14. **Credentials Setup:**  
    - Configure Google Sheets credentials with access to the client accounts spreadsheet.  
    - Configure Meta Ads API credentials (OAuth2 or access token).  
    - Configure Evolution API credentials (API key or token).  
    - Configure Gmail OAuth2 credentials if email alerts are enabled.

15. **Testing & Validation:**  
    - Test the workflow manually using "Start by Click".  
    - Verify data retrieval, balance checks, alert sending, and sheet updates.  
    - Adjust batch size or wait times if rate limits are encountered.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Use this Google Sheets template to set up client accounts and thresholds: [Google Sheet Example](https://docs.google.com/spreadsheets/d/1wwjHif98Jyc8QUGZI15YI-Z68QXznIuMcK951AuUYdY/edit?usp=sharing) | Required for client data input and threshold management.                                                |
| Workflow supports both WhatsApp alerts via Evolution API and email alerts via Gmail; email nodes are disabled by default. | Allows flexible alerting channels depending on user preference.                                         |
| Secure credentials are managed within n8n; no tokens are exposed in the workflow JSON.                          | Ensures security best practices for API integrations.                                                   |
| The workflow is compatible with both n8n Cloud and self-hosted installations.                                   | Broad deployment options for users.                                                                     |
| For workflow purchase and additional templates, visit: https://iloveflows.gumroad.com                          | Source of the original workflow and related resources.                                                  |
| Try n8n Cloud with partner link: [https://n8n.partnerlinks.io/amanda](https://n8n.partnerlinks.io/amanda)       | Recommended hosting platform for ease of use.                                                           |

---

This documentation provides a comprehensive understanding of the Meta Ads Low Balance Alert workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents alike.