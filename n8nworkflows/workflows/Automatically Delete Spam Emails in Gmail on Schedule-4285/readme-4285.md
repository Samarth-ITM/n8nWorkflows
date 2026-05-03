Automatically Delete Spam Emails in Gmail on Schedule

https://n8nworkflows.xyz/workflows/automatically-delete-spam-emails-in-gmail-on-schedule-4285


# Automatically Delete Spam Emails in Gmail on Schedule

### 1. Workflow Overview

This workflow is designed to automatically delete all spam emails from a Gmail account on a scheduled basis. It targets users who want to maintain a clean inbox by regularly purging their spam folder without manual intervention. The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Defines when the workflow runs automatically (e.g., weekly on Sunday at midnight).
- **1.2 Spam Email Retrieval:** Fetches all emails labeled as SPAM from the Gmail account.
- **1.3 Spam Email Deletion:** Iterates over the retrieved spam emails and deletes each one.

Additionally, there are two sticky notes providing usage instructions and reminders.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow execution at a defined recurring interval, specifically set to trigger weekly.

- **Nodes Involved:**  
  - Trigger at Sunday midnight

- **Node Details:**  
  - **Node Name:** Trigger at Sunday midnight  
  - **Type:** Schedule Trigger  
  - **Technical Role:** Starts the workflow automatically based on a time schedule.  
  - **Configuration:**  
    - Set to trigger on a weekly interval.  
    - The exact day/time is implied by the node name but configured as a generic weekly interval; specific day/time setting should be confirmed or customized by the user.  
  - **Key Expressions/Variables:** None.  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Outputs to "Get all SPAM emails" node.  
  - **Version-specific Requirements:** Requires n8n version supporting Schedule Trigger v1.2 or above.  
  - **Edge Cases / Potential Failures:**  
    - Misconfiguration of interval may cause unexpected trigger times.  
    - If the n8n instance time zone differs from user expectations, triggers may run at unintended hours.  
    - No authentication issues as it is a trigger node.  
  - **Sub-workflow Reference:** None.

#### 2.2 Spam Email Retrieval

- **Overview:**  
  Retrieves all emails from the Gmail SPAM label, returning the full list to be processed for deletion.

- **Nodes Involved:**  
  - Get all SPAM emails

- **Node Details:**  
  - **Node Name:** Get all SPAM emails  
  - **Type:** Gmail node (operation: getAll)  
  - **Technical Role:** Queries Gmail API to list all emails tagged with the SPAM label.  
  - **Configuration:**  
    - Operation set to "getAll" to fetch all emails without pagination limits.  
    - Filter applied with labelIds set to ["SPAM"] to restrict retrieval to spam folder.  
    - Return all items enabled (true).  
  - **Key Expressions/Variables:** None explicitly; static filter configuration.  
  - **Input Connections:** Receives trigger from "Trigger at Sunday midnight".  
  - **Output Connections:** Passes each email item to "Delete SPAM emails".  
  - **Version-specific Requirements:** Requires Gmail node version 2.1 or higher for label filter support.  
  - **Edge Cases / Potential Failures:**  
    - Gmail API quota limits may be reached if many emails are retrieved frequently.  
    - Authentication errors if Gmail OAuth2 credentials expire or are invalid.  
    - If no spam emails exist, output will be empty, resulting in no deletion attempts.  
    - Rate limiting by Gmail API could cause temporary failures.  
  - **Sub-workflow Reference:** None.

#### 2.3 Spam Email Deletion

- **Overview:**  
  Deletes each spam email identified by the prior node.

- **Nodes Involved:**  
  - Delete SPAM emails

- **Node Details:**  
  - **Node Name:** Delete SPAM emails  
  - **Type:** Gmail node (operation: delete)  
  - **Technical Role:** Deletes individual email messages by ID.  
  - **Configuration:**  
    - Operation set to "delete".  
    - Message ID dynamically set using expression: `{{$json.id}}` (fetches the ID of each email item passed from previous node).  
  - **Key Expressions/Variables:** Message ID expression to target correct email.  
  - **Input Connections:** Receives emails from "Get all SPAM emails".  
  - **Output Connections:** None (end of chain).  
  - **Version-specific Requirements:** Gmail node v2.1+ to support delete operation with dynamic input.  
  - **Edge Cases / Potential Failures:**  
    - Deletion may fail for messages already deleted or moved.  
    - Authentication errors if OAuth token is invalid or expired.  
    - API rate limits may cause temporary deletion failures.  
    - Large volumes of emails may cause execution timeouts or require chunking.  
  - **Sub-workflow Reference:** None.

#### 2.4 Informational Notes

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  
  - These provide user guidance and reminders on usage and configuration.  
  - Content highlights the workflow’s purpose and advises to update the trigger schedule as needed.

---

### 3. Summary Table

| Node Name             | Node Type        | Functional Role          | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                      |
|-----------------------|------------------|-------------------------|------------------------|------------------------|-------------------------------------------------------------------------------------------------|
| Trigger at Sunday midnight | Schedule Trigger | Initiates workflow on schedule | None                   | Get all SPAM emails     | ## :point_up: UPDATE ME Select when to trigger the workflow                                    |
| Get all SPAM emails   | Gmail            | Retrieves all SPAM emails | Trigger at Sunday midnight | Delete SPAM emails      |                                                                                                 |
| Delete SPAM emails    | Gmail            | Deletes each spam email  | Get all SPAM emails     | None                   |                                                                                                 |
| Sticky Note           | Sticky Note      | Informational note       | None                   | None                   | ## Automatically delete your SPAM emails on frequent basis This automation will run repeatedly at a particular time to delete all your SPAM emails |
| Sticky Note1          | Sticky Note      | Informational note       | None                   | None                   | ## :point_up: UPDATE ME Select when to trigger the workflow                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Node:**  
   - Add a **Schedule Trigger** node.  
   - Set the interval to "weeks" to run once per week.  
   - Configure the exact day/time to Sunday midnight (adjust based on time zone).  
   - Name it "Trigger at Sunday midnight".

2. **Create the Gmail Node to Get Spam Emails:**  
   - Add a **Gmail** node.  
   - Set operation to **getAll**.  
   - Under filters, set `labelIds` to ["SPAM"] to target spam folder emails.  
   - Enable **Return All** to get all spam emails without limit.  
   - Connect the output of the Schedule Trigger to this node.  
   - Attach proper **Gmail OAuth2 Credentials** with read access to Gmail.  
   - Name it "Get all SPAM emails".

3. **Create the Gmail Node to Delete Emails:**  
   - Add another **Gmail** node.  
   - Set operation to **delete**.  
   - For **Message ID**, set expression to `{{$json.id}}` to dynamically delete each email by its ID.  
   - Connect the output of "Get all SPAM emails" to this node.  
   - Use the same **Gmail OAuth2 Credentials** with delete permissions.  
   - Name it "Delete SPAM emails".

4. **Add Informational Sticky Notes (Optional):**  
   - Add a **Sticky Note** with content explaining the workflow deletes spam emails regularly.  
   - Add another **Sticky Note** reminding to update the trigger schedule as needed.

5. **Verify Credentials:**  
   - Ensure Gmail OAuth2 credentials are valid and have the necessary scopes (read/write access).  
   - Confirm OAuth tokens are refreshed properly to avoid authentication errors.

6. **Activate the Workflow:**  
   - Save and activate the workflow.  
   - Monitor the initial runs to verify emails are deleted as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                              |
|-----------------------------------------------------------------------------------------------|----------------------------------------------|
| This workflow automates spam email deletion using Gmail’s API and n8n’s scheduling capabilities. | Workflow purpose summary                      |
| Make sure the Gmail OAuth2 credentials have sufficient permissions: Gmail read & modify scopes. | Credential setup reminder                      |
| Adjust the schedule trigger timing to your preferred deletion frequency (e.g., daily, weekly). | Usage customization note                       |
| Gmail API quota limits may impact workflows with very high email volumes.                     | Gmail API limits documentation: https://developers.google.com/gmail/api/guides/quotas |
| To avoid time zone issues, verify n8n server time zone matches expected trigger time.         | n8n time zone configuration docs              |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.