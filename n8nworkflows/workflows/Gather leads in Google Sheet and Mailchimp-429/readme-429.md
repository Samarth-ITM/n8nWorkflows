Gather leads in Google Sheet and Mailchimp

https://n8nworkflows.xyz/workflows/gather-leads-in-google-sheet-and-mailchimp-429


# Gather leads in Google Sheet and Mailchimp

### 1. Workflow Overview

This workflow automates the process of gathering leads from a Google Sheet and adding them to a Mailchimp audience list. It is designed to streamline marketing and sales efforts by automatically syncing new or updated lead data into Mailchimp for email marketing campaigns.

The workflow consists of three main logical blocks:

- **1.1 Trigger and Scheduling:** Initiates the workflow either manually or on a timed interval.
- **1.2 Data Retrieval:** Fetches lead data (email and other columns) from a specified Google Sheet.
- **1.3 Lead Sync to Mailchimp:** Subscribes each lead email to a specific Mailchimp mailing list with “subscribed” status.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Scheduling

**Overview:**  
This block controls when the workflow runs. It includes a manual trigger node for on-demand execution and an interval node to automatically trigger the workflow every 2 minutes.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Interval

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Allows manual execution by a user from the n8n editor or UI.  
  - Configuration: No parameters; simply triggers the workflow when clicked.  
  - Inputs: None  
  - Outputs: Triggers downstream nodes (though here it has no direct output connection)  
  - Edge cases: None specific; manual action required.  
  - Notes: This node is useful for testing or manual runs but does not initiate data flow in this configuration.

- **Interval**  
  - Type: Interval Trigger  
  - Role: Automatically triggers the workflow periodically.  
  - Configuration: Interval set to 2 minutes (every 2 minutes the node triggers).  
  - Inputs: None  
  - Outputs: Connected to Google Sheets node to start data fetching.  
  - Edge cases: Potential timing overlap if workflow executions take longer than 2 minutes. No concurrency control visible.  
  - Notes: Enables automated, continuous syncing of leads without manual intervention.

#### 1.2 Data Retrieval

**Overview:**  
This block reads lead data from a Google Sheet. It extracts email addresses and any other columns specified within the range.

**Nodes Involved:**  
- Google Sheets

**Node Details:**

- **Google Sheets**  
  - Type: Google Sheets node  
  - Role: Reads data from a specified sheet and range.  
  - Configuration:  
    - Spreadsheet ID: `1jwEoPPrkQ2qYMYLZ_I0hlME_Ya_p2YZvaxG10Nf_R20`  
    - Range: `sheetone!A:C` (columns A to C on sheet named "sheetone")  
    - Options: None specified.  
  - Credentials: Uses Google API OAuth2 credentials named "Google mailchimp"  
  - Key expressions: Pulls the `email` field from each row to pass to the next node.  
  - Inputs: Triggered by Interval node.  
  - Outputs: Passes each row as JSON to the Mailchimp node.  
  - Edge cases:  
    - Empty or malformed rows could cause missing emails.  
    - API rate limits or auth expiry could cause failures.  
    - Incorrect spreadsheet ID or range will fail to fetch data.  
  - Notes: Proper credential setup with access to the Google Sheet is mandatory.

#### 1.3 Lead Sync to Mailchimp

**Overview:**  
This block subscribes each lead from the Google Sheet to a Mailchimp mailing list, setting their subscription status to “subscribed.”

**Nodes Involved:**  
- Mailchimp

**Node Details:**

- **Mailchimp**  
  - Type: Mailchimp node  
  - Role: Adds or updates a subscriber in a Mailchimp list.  
  - Configuration:  
    - List ID: `90d12734de` (target Mailchimp audience list)  
    - Email: Extracted dynamically using expression `={{$node["Google Sheets"].json["email"]}}`  
    - Status: `subscribed` (marks the contact as actively subscribed)  
    - Options: Default, no additional parameters set.  
  - Credentials: Uses Mailchimp API credentials named "Google mailchimp"  
  - Inputs: Receives lead data from Google Sheets node.  
  - Outputs: None connected further; end node in the chain.  
  - Edge cases:  
    - Missing or invalid email addresses will cause errors.  
    - API authentication failures or rate limits may block subscription.  
    - Duplicate emails in the Mailchimp list are handled by Mailchimp API but may cause warnings.  
  - Notes: Reliable Mailchimp credentials with appropriate permissions required.

---

### 3. Summary Table

| Node Name                | Node Type             | Functional Role                | Input Node(s)         | Output Node(s)      | Sticky Note                                   |
|--------------------------|-----------------------|-------------------------------|-----------------------|---------------------|-----------------------------------------------|
| On clicking 'execute'     | Manual Trigger        | Manual execution trigger       | None                  | None (no connections) |                                               |
| Interval                 | Interval Trigger      | Automated periodic trigger     | None                  | Google Sheets        |                                               |
| Google Sheets            | Google Sheets         | Fetch leads data from sheet   | Interval              | Mailchimp            |                                               |
| Mailchimp                | Mailchimp             | Subscribe leads to mailing list | Google Sheets         | None                |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name it "On clicking 'execute'."  
   - No parameters needed. This allows manual execution.

3. **Add an Interval Trigger node:**  
   - Name it "Interval."  
   - Set interval to 2 minutes (set `interval` parameter to 2).  
   - No credentials needed.

4. **Add a Google Sheets node:**  
   - Name it "Google Sheets."  
   - Set operation to "Read" (default for fetching data).  
   - Enter the Spreadsheet ID: `1jwEoPPrkQ2qYMYLZ_I0hlME_Ya_p2YZvaxG10Nf_R20`.  
   - Set the range to `sheetone!A:C` to read columns A to C.  
   - Configure credentials: Create or select Google API OAuth2 credentials with access to the target Google Sheet.

5. **Add a Mailchimp node:**  
   - Name it "Mailchimp."  
   - Operation: "Add subscriber" or equivalent.  
   - Set List ID to `90d12734de`.  
   - For the Email field, use an expression to pull from Google Sheets node data: `{{$node["Google Sheets"].json["email"]}}`.  
   - Set subscriber status to `subscribed`.  
   - Configure credentials: Create or select Mailchimp API credentials with permissions to add subscribers to the specified list.

6. **Connect nodes:**  
   - Connect Interval node's output to Google Sheets node input.  
   - Connect Google Sheets node's output to Mailchimp node input.  
   - The Manual Trigger node is standalone and can be triggered manually but is not connected downstream.

7. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                   |
|-----------------------------------------------------------------------------------------------|---------------------------------|
| For Mailchimp API authentication setup, refer to https://mailchimp.com/developer/             | Mailchimp API documentation     |
| Google Sheets API credentials require enabling Google Sheets API in Google Cloud Console       | Google API setup instructions   |
| Be cautious about API rate limits for both Google Sheets and Mailchimp when scheduling frequent interval triggers | Potential integration limits    |
| The workflow screenshot is available for visual reference in the provided file attachment     | Workflow screenshot (fileId:46) |

---

This documentation provides a detailed and structured reference for understanding, reproducing, and maintaining the "Google Sheet to Mailchimp" workflow, ensuring effective lead management automation.