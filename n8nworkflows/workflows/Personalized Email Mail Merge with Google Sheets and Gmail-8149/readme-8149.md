Personalized Email Mail Merge with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/personalized-email-mail-merge-with-google-sheets-and-gmail-8149


# Personalized Email Mail Merge with Google Sheets and Gmail

### 1. Workflow Overview

This workflow automates sending personalized emails directly from data stored in Google Sheets using Gmail. It supports two primary use cases:

- **Bulk Mail Merge:** Sending individualized emails to multiple recipients based on rows in a Google Sheet.
- **Triggered Emails:** Automatically dispatching emails when certain conditions in the sheet data are met (e.g., a status column value like "Scheduled for send").

The workflow comprises these logical blocks:

- **1.1 Trigger and Data Retrieval:** Periodically initiates the workflow and reads data from a specified Google Sheet.
- **1.2 Filtering Rows for Sending:** Filters rows that are flagged as ready for sending based on a keyword in the sheet.
- **1.3 Email Sending:** Sends personalized emails to recipients using Gmail.
- **1.4 Data Merging and Status Update:** Combines email sending results with the original data and updates the Google Sheet to reflect email sending status.
- **1.5 Workflow Documentation:** Provides user instructions and setup notes via a sticky note.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Data Retrieval

- **Overview:**  
  This block schedules the workflow to run at regular intervals and fetches the entire dataset from a specified Google Sheet for processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Read Google Sheets data

- **Node Details:**

  - **Schedule Trigger**  
    - **Type:** Schedule Trigger  
    - **Role:** Initiates workflow execution every hour.  
    - **Configuration:** Interval set to trigger every 1 hour.  
    - **Input:** None (trigger node).  
    - **Output:** Triggers the next node to fetch Google Sheets data.  
    - **Edge Cases:** Trigger misfires if system time changes or if n8n is offline during trigger time.  
    - **Version:** 1.2

  - **Read Google Sheets data**  
    - **Type:** Google Sheets node  
    - **Role:** Reads data from a specified Google Sheets document and sheet.  
    - **Configuration:**  
      - `documentId` and `sheetName` parameters dynamically set via expressions or list mode (user must specify).  
      - Uses OAuth2 credentials for Google Sheets access.  
    - **Input:** Trigger from Schedule Trigger.  
    - **Output:** Outputs rows of data from the sheet.  
    - **Potential Failures:** Authentication errors, API quota exceeded, invalid document or sheet name, network issues.  
    - **Version:** 4.7  
    - **Credentials:** Google Sheets OAuth2 API

#### 2.2 Filtering Rows for Sending

- **Overview:**  
  This block filters the read rows to identify which entries are marked as "Scheduled for send" in a specific column, preparing them for email dispatch.

- **Nodes Involved:**  
  - Pass on "Scheduled for send" to Gmail (Filter node)

- **Node Details:**

  - **Pass on "Scheduled for send" to Gmail**  
    - **Type:** Filter node  
    - **Role:** Passes only rows where a specific field contains the string "Scheduled for send" and where several other fields exist (non-empty).  
    - **Configuration:**  
      - Condition checks that the targeted field `contains` "Scheduled for send".  
      - Additional conditions require existence of multiple fields (likely email, name, etc.), ensuring sufficient data for sending email.  
    - **Input:** Output from Google Sheets data node.  
    - **Output:** Two outputs:  
      - Main output (index 0): rows that match the filter, sent to email sending and merging nodes.  
      - Second output (index 1): filtered-out rows (not used here).  
    - **Edge Cases:** Filter mismatch due to case sensitivity or unexpected data formats might cause rows to be skipped.  
    - **Version:** 2.2

#### 2.3 Email Sending

- **Overview:**  
  This block sends personalized plain-text emails using Gmail to recipients specified in filtered sheet data.

- **Nodes Involved:**  
  - Send an email

- **Node Details:**

  - **Send an email**  
    - **Type:** Gmail node  
    - **Role:** Sends emails individually for each input record.  
    - **Configuration:**  
      - Email type set to "text" (no HTML).  
      - Subject and body text expected to be customized by the user, potentially with variables from the sheet data (not explicitly shown in JSON).  
      - Uses OAuth2 credentials for Gmail access.  
    - **Input:** Filtered rows from Pass on "Scheduled for send" node.  
    - **Output:** Email sending results.  
    - **Edge Cases:**  
      - Authentication failures or token expiration.  
      - Gmail API rate limits or sending limits.  
      - Invalid email addresses causing send failures.  
      - Network timeouts.  
    - **Version:** 2.1  
    - **Webhook ID:** Present, possibly for asynchronous responses or tracking.  
    - **Credentials:** Gmail OAuth2

#### 2.4 Data Merging and Status Update

- **Overview:**  
  This block combines the email sending results with the original data and updates the Google Sheet to reflect the email sending status for each row.

- **Nodes Involved:**  
  - Combine both datasets  
  - Pass data from Gmail  
  - Update Status email column

- **Node Details:**

  - **Combine both datasets**  
    - **Type:** Merge node  
    - **Role:** Joins the data streams from Google Sheets and Gmail sends, combining data for further processing.  
    - **Configuration:** Mode set to "combine" to join datasets without matching keys.  
    - **Input:**  
      - From Pass on "Scheduled for send" (filtered sheet data)  
      - From Send an email (email sending results)  
    - **Output:** Combined dataset.  
    - **Version:** 3.2

  - **Pass data from Gmail**  
    - **Type:** Set node  
    - **Role:** Processes and prepares data fields from combined results for updating the sheet.  
    - **Configuration:**  
      - Custom assignments defined (empty strings in JSON but intended for mapping email status or timestamps).  
      - Includes all other incoming fields.  
    - **Input:** Combined dataset.  
    - **Output:** Prepared data for Google Sheets update.  
    - **Edge Cases:** Incorrect field mappings or missing data could cause update failures.  
    - **Version:** 3.4

  - **Update Status email column**  
    - **Type:** Google Sheets node  
    - **Role:** Updates specific rows in the Google Sheet to reflect the email sending result/status.  
    - **Configuration:**  
      - Uses update operation on specified sheet and document ID.  
      - Requires correct row identification to update corresponding entries.  
    - **Input:** Processed data from Pass data from Gmail node.  
    - **Output:** Updated Google Sheets data confirmation.  
    - **Potential Failures:** Authentication errors, row mismatch, API limits, network issues.  
    - **Version:** 4.7  
    - **Credentials:** Google Sheets OAuth2 API

#### 2.5 Workflow Documentation and User Instructions

- **Overview:**  
  Provides embedded instructions and resources for users to understand, set up, and customize the workflow.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - **Type:** Sticky Note node  
    - **Role:** Contains detailed user instructions, workflow purpose explanation, setup steps, and helpful links.  
    - **Content Highlights:**  
      - Explains mail merge and triggered email functionalities.  
      - Provides links to a Google Sheets template and a YouTube video guide.  
    - **Position:** Separate from main workflow flow, purely informational.  
    - **Version:** 1

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                      | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                                  |
|--------------------------------|-----------------------|------------------------------------|-----------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger      | Initiates workflow periodically    | None                        | Read Google Sheets data            |                                                                                                              |
| Read Google Sheets data        | Google Sheets         | Reads data from Google Sheets      | Schedule Trigger            | Pass on "Scheduled for send"       |                                                                                                              |
| Pass on "Scheduled for send" to Gmail | Filter                | Filters rows marked for sending    | Read Google Sheets data     | Send an email; Combine both datasets |                                                                                                              |
| Send an email                  | Gmail                 | Sends personalized emails          | Pass on "Scheduled for send" | Combine both datasets              |                                                                                                              |
| Combine both datasets          | Merge                 | Combines sheet data and email results | Pass on "Scheduled for send", Send an email | Pass data from Gmail                |                                                                                                              |
| Pass data from Gmail           | Set                   | Prepares data for sheet update     | Combine both datasets       | Update Status email column          |                                                                                                              |
| Update Status email column     | Google Sheets         | Updates sheet with email status    | Pass data from Gmail        | None                              |                                                                                                              |
| Sticky Note                   | Sticky Note           | Provides usage instructions        | None                        | None                              | ## Mail merge or send email from Google Sheets Send personalized emails & mail merge from Google Sheets using only n8n The workflow connects Google Sheets with Gmail to let you send emails in either of two ways: 1. Bulk emails (mail merge): Use data from your sheet to send an email to multiple email addresses, one by one. 2. Triggered emails: Automatically send an email whenever specific values or conditions in your sheet are met. No need to manually copy, paste, or switch to Gmail, because the process is fully automated. ## How to set it up 1. Copy [this template](https://docs.google.com/spreadsheets/d/1fWg_GOU0m_2cQpah7foDiz1WqTRKjCbJJCLBGCvJlXc/edit?usp=sharing) into your personal n8n workspace. 2. Customize the email nodes with your subject line, body text, and variables (e.g., names or links from your sheet). 3. For a step-by-step walkthrough, check out [this video guide on YouTube](https://www.youtube.com/watch?v=XJQ0W3yWR-0) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node:**  
   - Type: Schedule Trigger  
   - Configuration: Set to trigger every 1 hour (interval: hours, value: 1).  
   - No credentials needed.  

2. **Create "Read Google Sheets data" node:**  
   - Type: Google Sheets  
   - Operation: Read sheet data (default read operation).  
   - Parameters:  
     - Set `documentId` to your Google Sheets document ID.  
     - Set `sheetName` to the specific sheet name containing your data.  
   - Credentials: Use Google Sheets OAuth2 credentials (connect your Google account).  
   - Connect "Schedule Trigger" output to this node's input.  

3. **Create "Pass on 'Scheduled for send' to Gmail" node:**  
   - Type: Filter  
   - Parameters:  
     - Set conditions to filter rows where the target column contains "Scheduled for send".  
     - Add conditions to check that required fields (e.g., email, name) exist and are not empty.  
   - Connect "Read Google Sheets data" main output to this filter node.  

4. **Create "Send an email" node:**  
   - Type: Gmail  
   - Parameters:  
     - Email type: Text (or HTML if desired).  
     - Customize subject and body using expressions to insert sheet data fields (e.g., recipient name, personalized content).  
   - Credentials: Connect your Gmail account via OAuth2.  
   - Connect filter node main output to this node's input.  

5. **Create "Combine both datasets" node:**  
   - Type: Merge  
   - Parameters:  
     - Mode: Combine  
   - Connect filter node second output (or main output if sending) and "Send an email" node output to this node.  
   - This node joins the filtered data and email send results.  

6. **Create "Pass data from Gmail" node:**  
   - Type: Set  
   - Parameters:  
     - Map or assign values to fields that will be used for updating the sheet (e.g., email status).  
     - Include all other incoming fields to preserve data.  
   - Connect "Combine both datasets" output to this node.  

7. **Create "Update Status email column" node:**  
   - Type: Google Sheets  
   - Operation: Update rows in the sheet.  
   - Parameters:  
     - Set the same `documentId` and `sheetName` as in the read node.  
     - Configure to update the row corresponding to each data record with email status or timestamp.  
   - Credentials: Use Google Sheets OAuth2 credentials.  
   - Connect "Pass data from Gmail" node output to this node.  

8. **Add a "Sticky Note" node for documentation:**  
   - Type: Sticky Note  
   - Content: Paste the workflow description, usage instructions, links to the Google Sheets template and video guide.  
   - Position separately for clarity.  

9. **Review all connections:**  
   - Schedule Trigger → Read Google Sheets data → Filter → Send an email & Combine both datasets → Pass data from Gmail → Update Status email column  

10. **Activate the workflow:**  
    - Ensure all credentials are valid and test with sample data.  
    - Monitor for errors and adjust filters or email node content as needed.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Copy [this Google Sheets template](https://docs.google.com/spreadsheets/d/1fWg_GOU0m_2cQpah7foDiz1WqTRKjCbJJCLBGCvJlXc/edit?usp=sharing) to start. | Official sheet template to structure your data properly for this workflow.                                                         |
| Watch [this video guide on YouTube](https://www.youtube.com/watch?v=XJQ0W3yWR-0) for a detailed walkthrough.                                       | Step-by-step visual guide for setup and customization.                                                                               |
| Ensure Gmail API quota limits and Google Sheets API limits are considered when sending bulk emails to avoid throttling.                             | Gmail sending limits and API quotas may affect workflow performance during high-volume runs.                                         |
| Customize email body and subject in the Gmail node using expressions to personalize emails with data from Google Sheets (e.g., {{ $json["Name"] }}). | Personalization is key for mail merges; use n8n expressions referencing sheet columns to customize emails.                          |

---

This completes the comprehensive, structured documentation for the "Personalized Email Mail Merge with Google Sheets and Gmail" workflow.