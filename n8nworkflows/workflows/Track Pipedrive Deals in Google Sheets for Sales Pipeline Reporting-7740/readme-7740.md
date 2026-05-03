Track Pipedrive Deals in Google Sheets for Sales Pipeline Reporting

https://n8nworkflows.xyz/workflows/track-pipedrive-deals-in-google-sheets-for-sales-pipeline-reporting-7740


# Track Pipedrive Deals in Google Sheets for Sales Pipeline Reporting

### 1. Workflow Overview

This workflow automates the extraction of deal data from Pipedrive, categorizes each deal by its sales stage, and appends the enriched data into a Google Sheet for sales pipeline reporting. It is designed for sales teams and analysts who want to track deal progress over time using a spreadsheet, enabling clear pipeline visibility and reporting.

**Logical Blocks:**

- **1.1 Input Trigger and Data Retrieval:** Initiates the workflow manually and fetches all deals from Pipedrive.
- **1.2 Data Categorization:** Transforms numerical stage IDs into readable stage names.
- **1.3 Date Generation:** Produces the current date to timestamp the data entries.
- **1.4 Data Merging and Preparation:** Combines deal data and the current date, sets fields for Google Sheets insertion.
- **1.5 Data Storage:** Appends the processed data as new rows into a specified Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger and Data Retrieval

- **Overview:**  
  This block starts the workflow manually and retrieves all deals from the connected Pipedrive account without filters.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get many deals1 (Pipedrive)

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution manually by user command.  
    - Configuration: No parameters.  
    - Inputs: None  
    - Outputs: Triggers "Get many deals1".  
    - Failures: None expected except user not executing.

  - **Get many deals1**  
    - Type: Pipedrive node  
    - Role: Retrieves all deals data from Pipedrive via API.  
    - Configuration:  
      - Operation: `getAll`  
      - Return all records: `true`  
      - Filters: None (fetches all deals)  
      - Credential: Pipedrive API (configured with company domain and API token)  
    - Inputs: Trigger from manual node  
    - Outputs: JSON array of deal objects with all available fields  
    - Failures:  
      - Authentication errors (invalid token/domain)  
      - API rate limits or downtime  
      - Network issues  

---

#### 2.2 Data Categorization

- **Overview:**  
  Converts numerical `stage_id` values from Pipedrive deals into human-readable stage names for clarity in reporting.

- **Nodes Involved:**  
  - Categorize stages (Code)

- **Node Details:**  
  - **Categorize stages**  
    - Type: Code (JavaScript)  
    - Role: Maps numeric stage IDs to descriptive names (e.g., 1 → "Prospecting")  
    - Configuration:  
      - Custom JS code that creates a mapping object and adds `stage_name` field to each item.  
      - Handles unknown stage IDs by labeling them as "Unknown (ID)".  
    - Inputs: Deals array from "Get many deals1"  
    - Outputs: Deals array with added `stage_name` property  
    - Failures:  
      - Unexpected null or missing `stage_id`  
      - Runtime JS errors if data shape changes  
    - Version-specific: Uses Code node v2 syntax.  

---

#### 2.3 Date Generation

- **Overview:**  
  Generates today’s date in ISO format (YYYY-MM-DD) to timestamp each row added to the Google Sheet.

- **Nodes Involved:**  
  - Today's Date (Code)

- **Node Details:**  
  - **Today's Date**  
    - Type: Code (JavaScript)  
    - Role: Returns current date as ISO string split to date only  
    - Configuration: Returns JSON with `badges.today` containing the date string  
    - Inputs: None (standalone)  
    - Outputs: Single item with today's date  
    - Failures: Minimal, only runtime exceptions if system clock unavailable  

---

#### 2.4 Data Merging and Preparation

- **Overview:**  
  Combines the deal data with the current date, then prepares each row’s fields specifically for Google Sheets input.

- **Nodes Involved:**  
  - Merge  
  - Set Fields

- **Node Details:**  
  - **Merge**  
    - Type: Merge node  
    - Role: Combines two streams of data: deals with stage names + today's date, using "combine all" mode  
    - Configuration: Mode "combine", combineBy "combineAll" (Cartesian product)  
    - Inputs:  
      - Input 1: Categorized deals  
      - Input 2: Today's Date  
    - Outputs: Combined items with both deal info and date  
    - Failures: Possible data mismatch if inputs have incompatible lengths or structures  

  - **Set Fields**  
    - Type: Set node  
    - Role: Defines the exact fields and values to write to Google Sheets  
    - Configuration:  
      - Sets `Date` to the current date from `badges.today`  
      - Sets `stage_name` to the stage name from categorized data  
      - Sets `Deal` to the deal title from the original Pipedrive data  
    - Inputs: Merged data  
    - Outputs: Structured data ready for Google Sheets append operation  
    - Failures: Expression errors if referenced data is missing or incorrectly mapped  

---

#### 2.5 Data Storage

- **Overview:**  
  Appends each prepared deal record as a new row in a specified Google Sheet for persistent sales pipeline tracking.

- **Nodes Involved:**  
  - Store in google (Google Sheets)

- **Node Details:**  
  - **Store in google**  
    - Type: Google Sheets node  
    - Role: Appends rows to a Google Sheet  
    - Configuration:  
      - Operation: Append  
      - Document ID: ID of target Google Spreadsheet  
      - Sheet Name: Sheet1 (gid=0)  
      - Columns defined: Date, Deal, stage_name  
      - Uses OAuth2 credentials for Google Sheets API  
    - Inputs: Items with fields `Date`, `Deal`, `stage_name`  
    - Outputs: Operation result of append action  
    - Failures:  
      - OAuth token expiration or invalid credentials  
      - Sheet access permissions errors  
      - API quota limits or Google service outages  

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                               | Input Node(s)                | Output Node(s)          | Sticky Note                                                                                                   |
|-------------------------|-----------------------|-----------------------------------------------|-----------------------------|------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger        | Starts the workflow manually                   | None                        | Get many deals1         |                                                                                                               |
| Get many deals1         | Pipedrive             | Fetches all deals from Pipedrive               | When clicking ‘Execute workflow’ | Categorize stages       | See sticky note for Pipedrive API setup instructions                                                         |
| Categorize stages       | Code (JavaScript)     | Maps stage_id numbers to readable stage names | Get many deals1             | Merge                  |                                                                                                               |
| Today's Date            | Code (JavaScript)     | Produces today’s date in ISO format             | None                        | Merge                  |                                                                                                               |
| Merge                   | Merge                 | Combines deal data and current date             | Categorize stages, Today's Date | Set Fields             |                                                                                                               |
| Set Fields              | Set                   | Prepares fields (Date, Deal, stage_name) for Google Sheets | Merge                      | Store in google         |                                                                                                               |
| Store in google         | Google Sheets         | Appends data to Google Sheet                     | Set Fields                  | None                   | See sticky note for Google Sheets setup instructions and sample sheet link                                   |
| Sticky Note58           | Sticky Note           | Describes workflow purpose                       | None                        | None                   | # 📋 Pipedrive → Google Sheets: This workflow pulls deals from Pipedrive, categorizes them, and logs them into a Google Sheet for reporting and tracking. |
| Sticky Note15           | Sticky Note           | Setup instructions for Pipedrive and Google Sheets | None                        | None                   | See detailed setup instructions and contact info for customization support.                                   |
| Sticky Note12           | Sticky Note           | Pipedrive API connection steps                   | None                        | None                   | 2️⃣ Connect Pipedrive instructions: copy API token, create credentials, select in node                        |
| Sticky Note13           | Sticky Note           | Google Sheets preparation steps                   | None                        | None                   | 2️⃣ Prepare Your Google Sheet: sample sheet, column headers, OAuth2 credential setup                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No additional parameters.

2. **Add Pipedrive Node**  
   - Name: `Get many deals1`  
   - Type: Pipedrive  
   - Operation: `getAll`  
   - Return All: `true`  
   - Filters: None (optional: add filters if desired)  
   - Credentials: Create Pipedrive API credential with your company domain and API token.  
   - Connect trigger output to this node’s input.

3. **Add Code Node for Categorizing Stages**  
   - Name: `Categorize stages`  
   - Type: Code (JavaScript, version 2)  
   - Paste the following code:
     ```javascript
     const stageMap = {
       1: "Prospecting",
       2: "Qualified",
       3: "Proposal Sent",
       4: "Negotiation",
       5: "Closed Won"
     };
     return items.map(item => {
       const stageId = item.json.stage_id;
       item.json.stage_name = stageMap[stageId] || `Unknown (${stageId})`;
       return item;
     });
     ```
   - Connect output of `Get many deals1` to this node.

4. **Add Code Node for Today’s Date**  
   - Name: `Today's Date`  
   - Type: Code (JavaScript, version 2)  
   - Code:
     ```javascript
     return [
       {
         json: {
           badges: {
             today: new Date().toISOString().split('T')[0]
           }
         }
       }
     ];
     ```
   - No input connections (standalone).

5. **Add Merge Node**  
   - Name: `Merge`  
   - Type: Merge  
   - Mode: Combine  
   - Combine By: Combine All  
   - Connect `Categorize stages` output to first input (Input 1).  
   - Connect `Today's Date` output to second input (Input 2).

6. **Add Set Node to Prepare Fields**  
   - Name: `Set Fields`  
   - Type: Set  
   - Assignments:  
     - Date → Expression: `{{ $json.badges.today }}`  
     - stage_name → Expression: `{{ $json.stage_name }}`  
     - Deal → Expression: `{{ $('Get many deals1').item.json.title }}`  
   - Connect output of `Merge` to this node.

7. **Add Google Sheets Node**  
   - Name: `Store in google`  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Use your Google Sheet ID (e.g., `1u0i-sfPxmfmm5YMU3ekEdQgdOHA6OgnbI-VwuRMDq4Q`)  
   - Sheet Name: `Sheet1` (or `gid=0`)  
   - Columns mapping:  
     - Date: `={{ $json.Date }}`  
     - Deal: `={{ $json.Deal }}`  
     - stage_name: `={{ $json.stage_name }}`  
   - Credentials: Set up Google Sheets OAuth2 credentials for the Google account with access to the target spreadsheet.  
   - Connect output of `Set Fields` to this node.

8. **Verify and Save**  
   - Ensure all credentials are valid and saved.  
   - Test by clicking “Execute workflow” manually to confirm data flows and appends correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                          | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Sample Google Sheet format for the workflow: https://docs.google.com/spreadsheets/d/1u0i-sfPxmfmm5YMU3ekEdQgdOHA6OgnbI-VwuRMDq4Q/edit?gid=0#gid=0                                                                                                                     | Used as template for sheet column headers and formatting                                             |
| Pipedrive API token retrieval and credential setup instructions: `https://{your-company}.pipedrive.com/settings/personal/api`                                                                                                                                        | Required for Pipedrive node authentication                                                          |
| Google Sheets OAuth2 credential creation requires logging in with a Google account having edit permissions on the target spreadsheet.                                                                                                                                  | n8n credential setup guide                                                                            |
| Contact for customization help: robert@ynteractive.com | LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ | Website: https://ynteractive.com                                                                 |  
| Workflow purpose: Pull deals, categorize by stage, and store in Google Sheets for sales pipeline reporting and tracking.                                                                                                                                               | Sticky Notes in workflow provide setup and usage instructions                                        |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow, respecting all applicable content policies. It contains no illegal, offensive, or protected content. All processed data is legal and public.