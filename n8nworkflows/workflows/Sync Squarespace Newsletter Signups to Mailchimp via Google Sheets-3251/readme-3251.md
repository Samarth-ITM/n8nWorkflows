Sync Squarespace Newsletter Signups to Mailchimp via Google Sheets

https://n8nworkflows.xyz/workflows/sync-squarespace-newsletter-signups-to-mailchimp-via-google-sheets-3251


# Sync Squarespace Newsletter Signups to Mailchimp via Google Sheets

### 1. Workflow Overview

This workflow automates the synchronization of newsletter signups collected via a Squarespace form into a Mailchimp audience list, using Google Sheets as an intermediary data store. It addresses the limitation in Squarespace’s native Mailchimp integration, which only supports new, empty audiences, by capturing form submissions in a Google Sheet and then adding those contacts to Mailchimp.

The workflow supports two trigger methods: manual execution and scheduled runs for continuous synchronization.

Logical blocks:

- **1.1 Trigger Reception:** Handles manual and scheduled triggers to start the workflow.
- **1.2 Data Retrieval from Google Sheets:** Fetches newsletter signup data stored in a Google Sheet.
- **1.3 Iteration over Signups:** Processes each signup entry individually.
- **1.4 Mailchimp Contact Creation:** Adds each signup as a new subscriber to the specified Mailchimp audience.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Reception

**Overview:**  
This block initiates the workflow either manually or on a schedule, allowing flexible execution modes.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows user to manually start the workflow for testing or on-demand sync.  
  - Configuration: No parameters; triggers workflow immediately when clicked.  
  - Inputs: None  
  - Outputs: Connects to "Squarespace newsletter submissions" node.  
  - Edge Cases: None specific; manual trigger depends on user action.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow at regular intervals for continuous sync.  
  - Configuration: Interval set to run once every day (default daily interval).  
  - Inputs: None  
  - Outputs: Connects to "Squarespace newsletter submissions" node.  
  - Edge Cases: Scheduling misconfiguration could cause missed or overlapping runs.

---

#### 1.2 Data Retrieval from Google Sheets

**Overview:**  
This block reads the newsletter signup data from a specified Google Sheet, which acts as the data source capturing Squarespace form submissions.

**Nodes Involved:**  
- Squarespace newsletter submissions (Google Sheets node)

**Node Details:**

- **Squarespace newsletter submissions**  
  - Type: Google Sheets  
  - Role: Retrieves rows from the configured Google Sheet containing newsletter signups.  
  - Configuration:  
    - Document ID set to the Google Sheet storing form submissions.  
    - Sheet name set to the first sheet (gid=0).  
    - No filters or ranges specified, so it fetches all rows.  
  - Credentials: Uses Google Sheets OAuth2 credentials for API access.  
  - Inputs: Trigger nodes ("When clicking ‘Test workflow’" and "Schedule Trigger").  
  - Outputs: Connects to "Loop Over each item" node.  
  - Edge Cases:  
    - API authentication errors if credentials expire or are revoked.  
    - Empty or malformed sheets could cause no data or errors.  
    - Large sheets may cause performance issues or timeouts.

---

#### 1.3 Iteration over Signups

**Overview:**  
This block splits the retrieved data into individual items to process each newsletter signup separately.

**Nodes Involved:**  
- Loop Over each item (SplitInBatches node)

**Node Details:**

- **Loop Over each item**  
  - Type: SplitInBatches  
  - Role: Processes each row from the Google Sheet one by one, enabling individual handling of each signup.  
  - Configuration: Default batch size (processes one item at a time).  
  - Inputs: Receives array of rows from "Squarespace newsletter submissions".  
  - Outputs: On main output, sends one item at a time to "Add new member to Mailchimp".  
  - Edge Cases:  
    - If input data is empty, no iterations occur.  
    - Large datasets may slow down processing; batch size can be adjusted if needed.

---

#### 1.4 Mailchimp Contact Creation

**Overview:**  
This block creates or updates contacts in Mailchimp audiences based on the newsletter signup data.

**Nodes Involved:**  
- Add new member to Mailchimp (Mailchimp node)

**Node Details:**

- **Add new member to Mailchimp**  
  - Type: Mailchimp  
  - Role: Adds each newsletter signup as a subscribed member to the specified Mailchimp audience.  
  - Configuration:  
    - Email: Extracted from the "Email Address" column of the Google Sheet row, with an appended row number (likely a minor error or for uniqueness).  
    - Status: Set to "subscribed".  
    - Timestamp Signup: Set from the "Submitted On" column.  
    - Merge Fields: Sets "FNAME" to the "Name" column value.  
    - On error: Continues workflow on error (e.g., if contact already exists).  
  - Credentials: Uses Mailchimp API key credentials.  
  - Inputs: Receives single signup item from "Loop Over each item".  
  - Outputs: Connects back to "Loop Over each item" to process next item.  
  - Edge Cases:  
    - Duplicate emails may cause errors or be ignored due to "continueErrorOutput".  
    - Invalid email formats or missing data may cause API errors.  
    - API rate limits or authentication failures could interrupt processing.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                          | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                               |
|------------------------------|---------------------|----------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Manual start of workflow                | None                          | Squarespace newsletter submissions |                                                                                                          |
| Schedule Trigger              | Schedule Trigger    | Scheduled start of workflow             | None                          | Squarespace newsletter submissions |                                                                                                          |
| Squarespace newsletter submissions | Google Sheets       | Retrieve newsletter signups from sheet | When clicking ‘Test workflow’, Schedule Trigger | Loop Over each item            |                                                                                                          |
| Loop Over each item           | SplitInBatches      | Process each signup individually        | Squarespace newsletter submissions | Add new member to Mailchimp    |                                                                                                          |
| Add new member to Mailchimp   | Mailchimp           | Add signup as Mailchimp contact         | Loop Over each item            | Loop Over each item            |                                                                                                          |
| Sticky Note1                 | Sticky Note         | Workflow description and instructions   | None                          | None                          | ## Create Mailchimp contact based on Squarespace newsletter<br>This workflow will get Squarespace newsletter signups and create new Mailchimp contact in the given Audience on Mailchimp<br><br>This overcome the limitation between Squarespace forms and Mailchimp connection where only new, empty audience can be used<br><br>You can run the workflow on demand or by schedule<br><br>## Spreadsheet template<br><br>The sheet columns are inspire from Squarespace newsletter block connection, but you can change the node to adapt new columns format<br><br>Clone the [sample sheet here](https://docs.google.com/spreadsheets/d/1wi2Ucb4b35e0-fuf-96sMnyzTft0ADz3MwdE_cG_WnQ/edit?usp=sharing)<br>- Submitted On<br>- Email Address<br>- Name |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".  
   - No parameters needed.

2. **Create Schedule Trigger Node**  
   - Add a **Schedule Trigger** node named "Schedule Trigger".  
   - Set the interval to daily (or desired frequency).

3. **Create Google Sheets Node**  
   - Add a **Google Sheets** node named "Squarespace newsletter submissions".  
   - Set **Operation** to "Read Rows".  
   - Set **Document ID** to your Google Sheet ID containing Squarespace newsletter signups.  
   - Set **Sheet Name** to the appropriate sheet (e.g., "Sheet1" or gid=0).  
   - Use **Google Sheets OAuth2** credentials configured with access to the spreadsheet.

4. **Create SplitInBatches Node**  
   - Add a **SplitInBatches** node named "Loop Over each item".  
   - Default batch size is 1 (process one row at a time).  
   - Connect the output of "Squarespace newsletter submissions" to this node.

5. **Create Mailchimp Node**  
   - Add a **Mailchimp** node named "Add new member to Mailchimp".  
   - Set **Operation** to "Add or Update Member".  
   - Set **List Name or ID** to your target Mailchimp audience ID or name.  
   - Set **Email** field to expression: `{{$json["Email Address"]}}` (remove the appended row number unless needed).  
   - Set **Status** to "subscribed".  
   - Under **Options**, set **Timestamp Signup** to `{{$json["Submitted On"]}}`.  
   - Under **Merge Fields**, add a field:  
     - Name: `FNAME`  
     - Value: `{{$json["Name"]}}`  
   - Set **On Error** to "Continue" to skip errors and continue processing.  
   - Use **Mailchimp API Key** credentials.

6. **Connect Nodes**  
   - Connect both "When clicking ‘Test workflow’" and "Schedule Trigger" nodes to "Squarespace newsletter submissions".  
   - Connect "Squarespace newsletter submissions" to "Loop Over each item".  
   - Connect "Loop Over each item" to "Add new member to Mailchimp".  
   - Connect the main output of "Add new member to Mailchimp" back to the second output of "Loop Over each item" to continue processing batches.

7. **Credential Setup**  
   - Configure Google Sheets OAuth2 credentials with access to the target spreadsheet.  
   - Configure Mailchimp API credentials with a valid API key and access to the target audience.

8. **Optional: Add Sticky Note**  
   - Add a sticky note with workflow description and instructions, including the link to the sample Google Sheet template.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow overcomes the limitation in Squarespace’s native Mailchimp integration which only supports new, empty audiences.                                                                                              | Workflow purpose                                                                                 |
| Clone the sample Google Sheets template used for storing newsletter signups: [Google Sheets Template](https://docs.google.com/spreadsheets/d/1wi2Ucb4b35e0-fuf-96sMnyzTft0ADz3MwdE_cG_WnQ/edit?usp=sharing)                      | Spreadsheet template                                                                             |
| Mailchimp API Authentication Guide: [Mailchimp API Guide](https://mailchimp.com/developer/marketing/guides/quick-start/)                                                                                                    | For setting up Mailchimp API credentials                                                        |
| Explore more n8n templates by the creator: [n8n Templates by bangank36](https://n8n.io/creators/bangank36/)                                                                                                                  | Additional resources                                                                            |
| The appended row number in the email field expression (`{{$json['Email Address']}}{{$json.row_number}}`) seems unintended and may cause invalid emails; consider removing it for correct email formatting.                      | Potential issue in Mailchimp node configuration                                                 |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the "Sync Squarespace Newsletter Signups to Mailchimp via Google Sheets" workflow.