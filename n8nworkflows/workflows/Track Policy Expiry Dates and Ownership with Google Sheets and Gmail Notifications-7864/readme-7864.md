Track Policy Expiry Dates and Ownership with Google Sheets and Gmail Notifications

https://n8nworkflows.xyz/workflows/track-policy-expiry-dates-and-ownership-with-google-sheets-and-gmail-notifications-7864


# Track Policy Expiry Dates and Ownership with Google Sheets and Gmail Notifications

### 1. Workflow Overview

This workflow automates the monitoring of policy expiry dates stored in a Google Sheets document and manages notification processes related to policy ownership. Its main purpose is to track policies approaching expiration and ensure that each policy has a designated owner. It sends email notifications if ownership information is missing for policies that are nearing expiry.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger:** Manual initiation of the workflow.
- **1.2 Data Retrieval:** Fetching policy data from Google Sheets.
- **1.3 Policy Expiry Check:** Determining if any policies are expiring soon.
- **1.4 Ownership Verification:** Checking if an owner is assigned for each expiring policy.
- **1.5 Notification Dispatch:** Sending email alerts when owner information is missing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview:**  
  This block provides a manual trigger to start the workflow execution on demand.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type and Technical Role:* Manual Trigger node; initiates workflow execution manually.  
    - *Configuration Choices:* No additional parameters; default manual trigger setup.  
    - *Key Expressions/Variables:* None.  
    - *Input and Output Connections:* No input; outputs to "Fetch Policy Data".  
    - *Version Requirements:* Compatible with n8n version supporting manual triggers (standard).  
    - *Edge Cases / Failure Modes:* None expected; manual trigger reliable.  
    - *Sub-workflow:* None.

#### 2.2 Data Retrieval

- **Overview:**  
  Retrieves policy details, including expiry dates and ownership, from a configured Google Sheets spreadsheet.

- **Nodes Involved:**  
  - Fetch Policy Data

- **Node Details:**

  - **Fetch Policy Data**  
    - *Type and Technical Role:* Google Sheets node; reads data from a specified sheet.  
    - *Configuration Choices:*  
      - Likely configured to read a range or entire sheet containing policy records.  
      - Requires Google Sheets credentials with read access.  
    - *Key Expressions/Variables:* None visible, but probably outputs structured rows with policy details including expiry date and owner fields.  
    - *Input and Output Connections:* Input from manual trigger; output to "Is Policy Expiring Soon?".  
    - *Version Requirements:* Requires n8n version supporting Google Sheets node v4.6 or later for best compatibility.  
    - *Edge Cases / Failure Modes:*  
      - Google Sheets API authentication failure.  
      - Empty or malformed data rows.  
      - API rate limits or connectivity issues.  
    - *Sub-workflow:* None.

#### 2.3 Policy Expiry Check

- **Overview:**  
  Evaluates each policy’s expiry date to determine if it is within a threshold that qualifies as "expiring soon".

- **Nodes Involved:**  
  - Is Policy Expiring Soon?

- **Node Details:**

  - **Is Policy Expiring Soon?**  
    - *Type and Technical Role:* If node; conditional branching based on expiry date criteria.  
    - *Configuration Choices:*  
      - Condition likely compares policy expiry date field to current date plus a set number of days (e.g., 30 days).  
      - Only policies meeting the "expiring soon" condition pass to the next node.  
    - *Key Expressions/Variables:* Expression to calculate date difference or threshold.  
    - *Input and Output Connections:* Input from "Fetch Policy Data"; output to "Is Owner Missing?".  
    - *Version Requirements:* Requires n8n If node v2.2 or later.  
    - *Edge Cases / Failure Modes:*  
      - Invalid or missing expiry dates causing expression errors.  
      - Timezone discrepancies affecting date comparison.  
    - *Sub-workflow:* None.

#### 2.4 Ownership Verification

- **Overview:**  
  Checks if the "owner" field for each expiring policy is populated; routes to notification if missing.

- **Nodes Involved:**  
  - Is Owner Missing?

- **Node Details:**

  - **Is Owner Missing?**  
    - *Type and Technical Role:* If node; evaluates presence of owner information.  
    - *Configuration Choices:*  
      - Condition tests if owner field is empty, null, or otherwise unassigned.  
    - *Key Expressions/Variables:* Expression checking owner field presence, e.g., `{{$json["owner"] === ""}}`.  
    - *Input and Output Connections:* Input from "Is Policy Expiring Soon?"; output to "Notify Missing Owner" on true condition.  
    - *Version Requirements:* Requires If node v2.2 or later.  
    - *Edge Cases / Failure Modes:*  
      - Owner field missing from data structure causing expression failure.  
      - Variations in owner data format leading to false negatives.  
    - *Sub-workflow:* None.

#### 2.5 Notification Dispatch

- **Overview:**  
  Sends an email notification alerting responsible parties that a policy is expiring soon but lacks a designated owner.

- **Nodes Involved:**  
  - Notify Missing Owner

- **Node Details:**

  - **Notify Missing Owner**  
    - *Type and Technical Role:* Gmail node; sends email notifications.  
    - *Configuration Choices:*  
      - Configured with OAuth2 Gmail credentials for sending email.  
      - Email content likely includes policy details and a call to action for assigning ownership.  
      - Recipient addresses presumably fixed or dynamic, based on workflow design.  
    - *Key Expressions/Variables:* Email body probably uses expressions to insert policy data such as policy name and expiry date.  
    - *Input and Output Connections:* Input from "Is Owner Missing?"; outputs none specified.  
    - *Version Requirements:* Gmail node v2.1 or later recommended for OAuth2 support.  
    - *Edge Cases / Failure Modes:*  
      - Authentication or authorization errors with Gmail API.  
      - Email delivery failures or invalid recipient addresses.  
      - Rate limits on sending emails.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                        | Input Node(s)             | Output Node(s)             | Sticky Note                    |
|-------------------------|----------------------|-------------------------------------|---------------------------|----------------------------|-------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger       | Start workflow execution manually    | —                         | Fetch Policy Data           |                               |
| Fetch Policy Data       | Google Sheets        | Retrieve policy data from spreadsheet| When clicking ‘Execute workflow’ | Is Policy Expiring Soon?     |                               |
| Is Policy Expiring Soon?| If                   | Check if policy expiry date is near  | Fetch Policy Data          | Is Owner Missing?           |                               |
| Is Owner Missing?       | If                   | Check if policy owner is assigned    | Is Policy Expiring Soon?   | Notify Missing Owner        |                               |
| Notify Missing Owner    | Gmail                | Send email notification for missing owner | Is Owner Missing?          | —                          |                               |
| Sticky Note             | Sticky Note          | (Empty content)                      | —                         | —                          |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To manually start the workflow.

2. **Set up Google Sheets node to fetch policy data**  
   - Name: `Fetch Policy Data`  
   - Type: Google Sheets (read operation)  
   - Configure credentials for Google Sheets API with read access.  
   - Set the spreadsheet ID and worksheet name containing policy data.  
   - Define the data range (e.g., entire sheet or specific columns with expiry dates and owners).  
   - Connect output of manual trigger node to this node.

3. **Add an If node to check if policies are expiring soon**  
   - Name: `Is Policy Expiring Soon?`  
   - Configure condition to compare the policy expiry date field with the current date plus a threshold (e.g., `expiryDate <= now + 30 days`).  
   - Use appropriate date expressions to compute this comparison.  
   - Connect input from `Fetch Policy Data`.  
   - Connect true branch to next node.

4. **Add an If node to verify if the owner field is missing**  
   - Name: `Is Owner Missing?`  
   - Configure condition to check if the owner field is empty, null, or missing.  
   - Expression example: check if `owner` attribute is an empty string or undefined.  
   - Connect input from `Is Policy Expiring Soon?`.  
   - Connect true branch to notification node.

5. **Add Gmail node to send email notifications**  
   - Name: `Notify Missing Owner`  
   - Configure OAuth2 credentials for Gmail.  
   - Set recipient email(s), subject, and message body incorporating policy details via expressions.  
   - Connect input from `Is Owner Missing?` node.

6. **Optionally add Sticky Note node**  
   - For documentation or reminders within the workflow editor.

7. **Validate the entire workflow connections:**  
   - Manual Trigger → Fetch Policy Data → Is Policy Expiring Soon? → Is Owner Missing? → Notify Missing Owner.

8. **Test the workflow manually** using the trigger, check logs, and verify email notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                         |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| This workflow requires Google Sheets API access with proper permissions set.                     | Google Sheets API documentation: https://developers.google.com/sheets/api |
| Gmail node requires OAuth2 authentication set up with appropriate scope for sending emails.     | Gmail API OAuth2 setup guide: https://developers.google.com/gmail/api/auth/about-auth |
| Date expressions must be carefully configured considering timezone differences and date formats.| n8n Expressions docs: https://docs.n8n.io/nodes/expressions/             |
| Manual Trigger node is used here for testing; for automation, a schedule trigger could replace it.| n8n Scheduling docs: https://docs.n8n.io/nodes/nodes-triggers/schedule/   |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.