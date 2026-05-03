Sync Leads from Google Sheets to Instantly Email Campaigns with Data Tables

https://n8nworkflows.xyz/workflows/sync-leads-from-google-sheets-to-instantly-email-campaigns-with-data-tables-9743


# Sync Leads from Google Sheets to Instantly Email Campaigns with Data Tables

### 1. Workflow Overview

This workflow automates the synchronization of lead data from a Google Sheets spreadsheet into Instantly email campaigns, while maintaining lead tracking via n8n Data Tables. It is designed to handle bulk lead processing efficiently, avoiding API rate limits by processing leads in batches and ensuring no duplicate leads are sent to the campaign.

The workflow consists of two main logical blocks:

- **1.1 Data Transfer Flow:** This block fetches lead data from Google Sheets, processes it in batches, and updates the n8n Data Table with additional information (focus area). It ensures all leads from the sheet are mirrored into the Data Table with the necessary transformation.

- **1.2 Instantly Campaign Sync:** This block selects leads marked as ready for campaign (`campaign = "start"`), sends them one by one to an Instantly email campaign, then updates their status in the Data Table to prevent duplicate processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Transfer Flow

**Overview:**  
This block transfers leads from Google Sheets to the n8n Data Table, handling batches of 30 leads at a time to prevent API overloads. It enriches lead data by mapping the "Title" from the sheet into a "focusarea" field in the Data Table.

**Nodes Involved:**  
- When clicking 'Execute workflow' (Manual Trigger)  
- Get row(s) in sheet (Google Sheets)  
- Loop Over Items (SplitInBatches)  
- Update row(s)1 (Data Table)  

**Node Details:**

- **When clicking 'Execute workflow'**  
  - Type: Manual Trigger  
  - Role: Entry point to run the workflow manually  
  - Configuration: No parameters  
  - Inputs: None  
  - Outputs: Connects to "Get row(s) in sheet"  
  - Failure Modes: None specific; manual trigger may not start if UI issues  

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Fetches all rows from the configured sheet named "apollo-contacts-export 9" within the spreadsheet "instantly leads"  
  - Configuration: Uses OAuth2 credentials for Google Sheets; no filter applied (fetches all rows)  
  - Inputs: From manual trigger  
  - Outputs: Leads data to batch processing  
  - Failure Modes: Authentication errors with Google; API rate limits; empty sheet returns no data  
  - Version: 4.7  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes leads in batches of 30 items to avoid API rate limits and timeouts  
  - Configuration: Batch size set to 30  
  - Inputs: Leads array from Google Sheets  
  - Outputs: Batches of leads passed to Data Table update  
  - Failure Modes: Batch processing may fail if batch size too large; empty input results in no processing  
  - Version: 3  

- **Update row(s)1**  
  - Type: Data Table  
  - Role: Updates the "Leads" Data Table by setting the "focusarea" field to the "Title" value from the Google Sheet lead  
  - Configuration: Filters rows by matching lead email; updates focusarea column; no type conversion  
  - Inputs: Each lead batch item from Loop Over Items  
  - Outputs: Connects back to Loop Over Items for continuous batch processing  
  - Failure Modes: Filtering failure if email missing; permission issues with Data Table; data type mismatches  
  - Version: 1  

---

#### 1.2 Instantly Campaign Sync

**Overview:**  
This block handles syncing of leads marked with `campaign = "start"` in the Data Table to the Instantly email campaign "Launchday 1". It processes leads one by one, creates corresponding leads in Instantly, then updates their campaign status to prevent duplicates.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) (Data Table)  
- Loop Over Items1 (SplitInBatches)  
- Create a lead (Instantly)  
- Update row(s) (Data Table)  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically starts workflow on a schedule (interval unspecified but one firing per execution assumed)  
  - Configuration: Default interval setting (run periodically)  
  - Inputs: None  
  - Outputs: Starts "Get row(s)" node  
  - Failure Modes: Scheduling misconfiguration; server downtime  

- **Get row(s)**  
  - Type: Data Table  
  - Role: Fetches leads from the "Leads" Data Table where `campaign = "start"` (i.e., leads ready to be sent to Instantly)  
  - Configuration: Filter condition where "campaign" equals "start"  
  - Inputs: From schedule trigger  
  - Outputs: Leads to batch processing  
  - Failure Modes: Empty or no matching rows; Data Table access issues  
  - Version: 1  

- **Loop Over Items1**  
  - Type: SplitInBatches  
  - Role: Splits leads into individual items (default batch size) for sequential processing to respect API rate limits  
  - Configuration: Default batch size (1 implied by empty options)  
  - Inputs: Leads from Data Table filter  
  - Outputs: Individual lead items to Instantly create lead node  
  - Failure Modes: Empty input; batch size misconfiguration  

- **Create a lead**  
  - Type: Instantly (custom n8n node)  
  - Role: Creates a lead in Instantly under the campaign "Launchday 1"  
  - Configuration: Maps lead fields (Email, Firstname, Company, Website) from JSON; campaign ID statically set; last_name empty; personalization blank  
  - Inputs: Single lead JSON from Loop Over Items1  
  - Outputs: Connects to update row status  
  - Failure Modes: API key authentication failure; invalid campaign ID; network timeouts; missing email or required fields  
  - Version: 1  

- **Update row(s)**  
  - Type: Data Table  
  - Role: Updates the lead's campaign field in the Data Table to "added to instantly" to mark as processed  
  - Configuration: Filters by lead email; updates campaign column value  
  - Inputs: From Instantly node  
  - Outputs: Connects back to Loop Over Items1 for continuous processing  
  - Failure Modes: Filtering failure if email missing; update failure due to permission or data errors  
  - Version: 1  

---

### 3. Summary Table

| Node Name              | Node Type               | Functional Role                                    | Input Node(s)          | Output Node(s)             | Sticky Note                                                      |
|------------------------|-------------------------|---------------------------------------------------|-----------------------|----------------------------|-----------------------------------------------------------------|
| When clicking 'Execute workflow' | Manual Trigger          | Manual start of data transfer flow                 | None                  | Get row(s) in sheet        | Automated lead management overview with setup instructions      |
| Get row(s) in sheet     | Google Sheets           | Fetch all leads from Google Sheets                  | When clicking 'Execute workflow' | Loop Over Items             | Batch size note: processes 30 leads at a time                   |
| Loop Over Items         | SplitInBatches          | Batch processing of leads from Google Sheets       | Get row(s) in sheet   | Update row(s)1; (no output main branch) | Batch size note                                                   |
| Update row(s)1          | Data Table              | Update Data Table focusarea field from Title       | Loop Over Items       | Loop Over Items             | Data Table schema note                                           |
| Schedule Trigger        | Schedule Trigger        | Automatically trigger Instantly sync flow           | None                  | Get row(s)                 | Configuration note including campaign and rate limit settings   |
| Get row(s)              | Data Table              | Fetch leads with campaign = "start" for Instantly  | Schedule Trigger      | Loop Over Items1            | Lead filter note: fetch only campaign = "start" leads           |
| Loop Over Items1        | SplitInBatches          | Process qualified leads one by one                   | Get row(s)            | Create a lead              | Individual loop note: ensures proper API rate limiting          |
| Create a lead           | Instantly               | Create lead in Instantly campaign                    | Loop Over Items1      | Update row(s)              | Instantly sync note: campaign "Launchday 1" used                |
| Update row(s)           | Data Table              | Mark lead as added to Instantly                       | Create a lead         | Loop Over Items1            | Status update note: prevent duplicate sends                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add "Manual Trigger" node named "When clicking 'Execute workflow'"  
   - No configuration needed  

2. **Create Google Sheets Node**  
   - Add "Google Sheets" node named "Get row(s) in sheet"  
   - Operation: Read Rows  
   - Document ID: Use your Google Sheet ID (e.g., "1idx3Q3Lr_qbF8trCUKVY3uw52SF513_oZ6oFeuc73Zk")  
   - Sheet Name: "apollo-contacts-export 9" or your target sheet  
   - Credentials: Setup Google Sheets OAuth2 credentials and authorize access  
   - Connect output of Manual Trigger to this node  

3. **Create SplitInBatches Node**  
   - Add "SplitInBatches" node named "Loop Over Items"  
   - Batch Size: 30  
   - Connect output of Google Sheets node to this node  

4. **Create Data Table Update Node**  
   - Add "Data Table" node named "Update row(s)1"  
   - Operation: Update  
   - Data Table: Select or create "Leads" Data Table with columns: Firstname, Lastname, email, website, company, title, campaign, focusarea  
   - Filter: Match rows where email equals `={{ $json.Email }}`  
   - Columns to update: focusarea = `={{ $json.Title }}`  
   - Connect output of SplitInBatches node to this node  
   - Connect output of this node back to SplitInBatches node (to loop until all batches processed)  

5. **Create Schedule Trigger Node**  
   - Add "Schedule Trigger" node named "Schedule Trigger"  
   - Configure interval for automatic runs (e.g., every hour or daily)  

6. **Create Data Table Get Rows Node for Instantly Sync**  
   - Add "Data Table" node named "Get row(s)"  
   - Operation: Get  
   - Data Table: "Leads"  
   - Filter: campaign = "start"  
   - Connect output of Schedule Trigger to this node  

7. **Create SplitInBatches Node for Instantly Sync**  
   - Add "SplitInBatches" node named "Loop Over Items1"  
   - Batch Size: default (process one at a time)  
   - Connect output of Data Table Get Rows to this node  

8. **Create Instantly Node for Creating Leads**  
   - Add "Instantly" node named "Create a lead"  
   - Operation: Create lead resource  
   - Map fields: email = `={{ $json.Email }}`, first_name = `={{ $json.Firstname }}`, company_name = `={{ $json.company }}`, website = `={{ $json.Website }}`  
   - Set campaign ID to your Instantly campaign (e.g., "100fa5a2-3ed0-4f12-967c-b2cc4a07c3e8")  
   - Credentials: Setup Instantly API key credentials  
   - Connect output of SplitInBatches node to this node  

9. **Create Data Table Update Node for Status Update**  
   - Add "Data Table" node named "Update row(s)"  
   - Operation: Update  
   - Data Table: "Leads"  
   - Filter: email equals `={{ $json.email }}` (note lowercase consistent with previous nodes)  
   - Columns to update: campaign = "added to instantly"  
   - Connect output of Instantly node to this node  
   - Connect output of this node back to SplitInBatches node to continue looping  

10. **Verify and Save Workflow**  
    - Ensure all connections are correct  
    - Test with small batches first  
    - Adjust batch sizes or trigger intervals if needed  
    - Confirm credentials are valid and have required permissions  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow automates lead sync from Google Sheets to Instantly with tracking via n8n Data Tables                | Workflow overview (Sticky Note - Main Overview)                                                   |
| Google Sheets template with required columns: Firstname, Email, Website, Company, Title                       | https://docs.google.com/spreadsheets/d/1eFXld6aiZvnQXg1lgt1COFyUh-zcy9VGx2GmFXwmYzM/edit?usp=sharing |
| Instantly campaign "Launchday 1" campaign ID: 100fa5a2-3ed0-4f12-967c-b2cc4a07c3e8 (replace with your own)    | Configuration note                                                                                 |
| Batch size configured as 30 to avoid API rate limits and timeouts                                            | Sticky Note - Batch Size                                                                           |
| Only leads with campaign = "start" are sent to Instantly to prevent duplicate sends                           | Sticky Note - Filter                                                                              |
| Processing leads one by one to ensure API rate limiting and accurate status updates                           | Sticky Note - Loop                                                                                |
| After sending leads to Instantly, campaign field updated to "added to instantly" to prevent duplicates       | Sticky Note - Update                                                                              |
| Data Tables schema defined for "Leads" table including fields: Firstname, Lastname, email, website, company, title, campaign, focusarea | Sticky Note - Schema                                                                              |
| Video tutorial available for detailed setup and explanation                                                  | https://www.youtube.com/watch?v=c8iv1u_jxDY                                                      |
| Contact for customization or help: david@daexai.com                                                          | Provided in main overview sticky note                                                             |

---

**Disclaimer:** The content above is generated from an automated n8n workflow export and adheres strictly to current content policies. All data processed is legal and public.