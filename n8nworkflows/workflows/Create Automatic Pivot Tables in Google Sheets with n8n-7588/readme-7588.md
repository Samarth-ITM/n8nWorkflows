Create Automatic Pivot Tables in Google Sheets with n8n

https://n8nworkflows.xyz/workflows/create-automatic-pivot-tables-in-google-sheets-with-n8n-7588


# Create Automatic Pivot Tables in Google Sheets with n8n

### 1. Workflow Overview

This workflow automates the creation of two pivot tables in a Google Sheets document based on campaign data. It is designed for marketing or data teams that regularly update campaign metrics and want to generate summarized views by Campaign and by Channel automatically.

The workflow is organized into these logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Retrieval:** Fetch raw campaign data from Google Sheets.
- **1.3 Data Cleanup:** Clear existing pivot table sheets to prepare for fresh data.
- **1.4 Data Aggregation:** Summarize campaign data grouped by Campaign and Channel.
- **1.5 Pivot Table Creation:** Append or update summarized data into respective pivot table sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block initiates the workflow manually.
- **Nodes Involved:** `Start Workflow`
- **Node Details:**
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)
  - **Role:** Allows user to trigger the workflow on demand.
  - **Configuration:** No parameters needed.
  - **Connections:** Outputs to `Get Data From Google`.
  - **Edge Cases:** None typical; workflow will only run when manually triggered.
  - **Version:** v1

#### 1.2 Data Retrieval

- **Overview:** This block fetches the raw campaign data from a Google Sheets document.
- **Nodes Involved:** `Get Data From Google`
- **Node Details:**
  - **Type:** Google Sheets (n8n-nodes-base.googleSheets)
  - **Role:** Reads the "Data" tab containing raw campaign metrics.
  - **Configuration:** 
    - Operation: Read/select rows.
    - Document ID: Points to the marketing data Google Sheets file.
    - Sheet Name: The "Data" tab by its GID.
  - **Credentials:** Uses Google Sheets OAuth2 credentials named "Google Sheets account 3."
  - **Connections:** Outputs data to `Sum Campaigns1` and `Sum Channels1`.
  - **Edge Cases:** 
    - Google API quota limits.
    - Authentication errors if credentials expire.
    - Empty or malformed data in the sheet.
  - **Version:** v4.7

#### 1.3 Data Cleanup

- **Overview:** Clears the content of existing pivot table sheets to avoid data duplication.
- **Nodes Involved:** `Clear Campaign Sheet1`, `Clear Channel Sheet`
- **Node Details:**

  - **Clear Campaign Sheet1:**
    - **Type:** Google Sheets (n8n-nodes-base.googleSheets)
    - **Role:** Clears the "Campaign Pivot" sheet.
    - **Configuration:**
      - Operation: Clear.
      - Document ID: Target Google Sheets document.
      - Sheet Name: "Campaign Pivot" tab by GID.
    - **Credentials:** Same Google Sheets OAuth2 credentials.
    - **Connections:** None downstream; runs in parallel after `Get Data From Google`.
    - **Edge Cases:** Sheet might be protected or inaccessible, causing errors.
    - **Version:** v4.7

  - **Clear Channel Sheet:**
    - Same as above but targets the "Channel Pivot" tab.
    - Positioned to run simultaneously with campaign sheet clear.

#### 1.4 Data Aggregation

- **Overview:** Summarizes raw data to create pivot tables grouped by Campaign and Channel.
- **Nodes Involved:** `Sum Campaigns1`, `Sum Channels1`
- **Node Details:**

  - **Sum Campaigns1:**
    - **Type:** Summarize (n8n-nodes-base.summarize)
    - **Role:** Aggregates campaign data grouped by the "Campaign" field.
    - **Configuration:**
      - Group by: Campaign.
      - Summarize fields: Sum of "Spend ($)", "Clicks", and "Conversions".
    - **Connections:** Outputs summarized data to `Create Campaign Pivot Table`.
    - **Edge Cases:** Data type mismatches, missing fields.
    - **Version:** v1.1

  - **Sum Channels1:**
    - Same as above but groups by the "Channel" field.
    - Outputs to `Create Channel Pivot Table`.

#### 1.5 Pivot Table Creation

- **Overview:** Writes the aggregated summaries into their respective Google Sheets tabs.
- **Nodes Involved:** `Create Campaign Pivot Table`, `Create Channel Pivot Table`
- **Node Details:**

  - **Create Campaign Pivot Table:**
    - **Type:** Google Sheets (n8n-nodes-base.googleSheets)
    - **Role:** Appends or updates summarized campaign data into "Campaign Pivot" tab.
    - **Configuration:**
      - Operation: Append or update rows.
      - Mapping: Automatically maps summarized fields such as sum_Spend_($), sum_Clicks, sum_Conversions, Campaign.
      - Document ID and Sheet Name: Points to "Campaign Pivot" tab.
    - **Credentials:** Google Sheets OAuth2.
    - **Edge Cases:** 
      - Duplicate data if mapping fails.
      - Schema mismatches.
    - **Version:** v4.7

  - **Create Channel Pivot Table:**
    - Same as above but writes to the "Channel Pivot" tab.
    - Input: Summarized data grouped by Channel.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                  | Input Node(s)       | Output Node(s)              | Sticky Note                                                                                       |
|-------------------------|--------------------|---------------------------------|---------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| Start Workflow          | Manual Trigger     | Initiates workflow manually     | —                   | Get Data From Google        |                                                                                                 |
| Get Data From Google     | Google Sheets      | Reads raw campaign data         | Start Workflow      | Sum Campaigns1, Sum Channels1 |                                                                                                 |
| Clear Campaign Sheet1    | Google Sheets      | Clears Campaign Pivot sheet     | Start Workflow      | —                          |                                                                                                 |
| Clear Channel Sheet      | Google Sheets      | Clears Channel Pivot sheet      | Start Workflow      | —                          |                                                                                                 |
| Sum Campaigns1           | Summarize          | Aggregate data by Campaign      | Get Data From Google | Create Campaign Pivot Table |                                                                                                 |
| Sum Channels1            | Summarize          | Aggregate data by Channel       | Get Data From Google | Create Channel Pivot Table  |                                                                                                 |
| Create Campaign Pivot Table | Google Sheets    | Writes campaign summary to sheet | Sum Campaigns1       | —                          |                                                                                                 |
| Create Channel Pivot Table | Google Sheets     | Writes channel summary to sheet | Sum Channels1        | —                          |                                                                                                 |
| Sticky Note3             | Sticky Note        | Instructions and setup guide    | —                   | —                          | This n8n workflow pulls campaign data from Google Sheets and creates two pivot tables automatically each time it runs. [Campaign Data Sheet](https://docs.google.com/spreadsheets/d/1lUEY6kPQbXizbmszLLNUJ_pBfGIKd75hu4uHj0vGRZQ/edit?usp=sharing) |
| Sticky Note4             | Sticky Note        | Label: Aggregate and Combine Data | —                   | —                          | ### Aggregate and Combine Data                                                                  |
| Sticky Note6             | Sticky Note        | Workflow title                  | —                   | —                          | ## 📊 Create Automatic Pivot Tables in Google Sheets with n8n                                   |
| Sticky Note11            | Sticky Note        | Setup instructions and sheet requirements | —                   | —                          | Must include Data tab and pivot view tabs, connect via Google Sheets OAuth2. [Campaign Data Sheet](https://docs.google.com/spreadsheets/d/1lUEY6kPQbXizbmszLLNUJ_pBfGIKd75hu4uHj0vGRZQ/edit?gid=365710158) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: `Start Workflow`
   - Type: Manual Trigger
   - No parameters.
   - Position: Optional.

2. **Create Google Sheets Node to Read Raw Data**
   - Name: `Get Data From Google`
   - Type: Google Sheets
   - Operation: Read rows.
   - Document ID: Use your Google Sheets document ID containing campaign data.
   - Sheet Name: Select the "Data" tab by its GID.
   - Credentials: Create Google Sheets OAuth2 credentials.
   - Connect output to next nodes.

3. **Create Two Google Sheets Nodes to Clear Pivot Sheets**
   - Name: `Clear Campaign Sheet1`
     - Type: Google Sheets
     - Operation: Clear
     - Document ID: Same as above
     - Sheet Name: "Campaign Pivot" tab by GID
     - Credentials: Same OAuth2
   - Name: `Clear Channel Sheet`
     - Same configuration but for "Channel Pivot" tab.
   - Connect outputs of `Get Data From Google` to both clear nodes (parallel execution).

4. **Create Summarize Nodes to Aggregate Data**
   - Name: `Sum Campaigns1`
     - Type: Summarize
     - Group by: Campaign
     - Summarize fields: Sum of "Spend ($)", "Clicks", "Conversions"
     - Connect from `Get Data From Google`.
   - Name: `Sum Channels1`
     - Same as above but group by Channel.
     - Connect from `Get Data From Google`.

5. **Create Google Sheets Nodes to Append/Update Pivot Data**
   - Name: `Create Campaign Pivot Table`
     - Type: Google Sheets
     - Operation: Append or Update
     - Document ID: Same document as above
     - Sheet Name: "Campaign Pivot"
     - Map columns automatically: sum_Spend_($), sum_Clicks, sum_Conversions, Campaign
     - Connect input from `Sum Campaigns1`.
     - Credentials: Same OAuth2.
   - Name: `Create Channel Pivot Table`
     - Same as above but for "Channel Pivot" tab and columns: sum_Spend_($), sum_Clicks, sum_Conversions, Channel
     - Connect input from `Sum Channels1`.

6. **Connect Workflow**
   - Connect `Start Workflow` → `Get Data From Google`
   - Connect `Get Data From Google` → `Clear Campaign Sheet1`
   - Connect `Get Data From Google` → `Clear Channel Sheet`
   - Connect `Get Data From Google` → `Sum Campaigns1`
   - Connect `Get Data From Google` → `Sum Channels1`
   - Connect `Sum Campaigns1` → `Create Campaign Pivot Table`
   - Connect `Sum Channels1` → `Create Channel Pivot Table`

7. **Credentials Setup**
   - In n8n, add new credentials of type "Google Sheets OAuth2 API."
   - Authenticate with your Google account.
   - Ensure access to target spreadsheet.

8. **Validation**
   - Test manual trigger.
   - Verify data retrieval.
   - Confirm sheets are cleared.
   - Check summarized data appears correctly in pivot tabs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This n8n workflow pulls campaign data from Google Sheets and creates two pivot tables automatically each time it runs.                                           | Sticky Note with setup instructions inside workflow                                                     |
| Use this Google Sheet as example or template: [Campaign Data Sheet](https://docs.google.com/spreadsheets/d/1lUEY6kPQbXizbmszLLNUJ_pBfGIKd75hu4uHj0vGRZQ/edit) | Provided in Sticky Note3 and Sticky Note11                                                              |
| For OAuth2 Google Sheets credentials setup: go to Credentials in n8n, create new Google Sheets OAuth2 API credential, login, and authorize.                      | Sticky Note3                                                                                             |
| You can also use Airtable, Notion, or a database as an alternative data source if desired, but the workflow is configured for Google Sheets.                     | Sticky Note11                                                                                            |
| For help or questions, contact robert@ynteractive.com or visit [LinkedIn](https://www.linkedin.com/in/robert-breen-29429625/)                                   | Sticky Note3                                                                                             |

---

*Disclaimer: This document is based solely on an automated n8n workflow and complies with all content policies. No illegal or protected data is included.*