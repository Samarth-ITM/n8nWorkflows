LinkedIn Person ICP Scoring Automation with Airtop & Google Sheets

https://n8nworkflows.xyz/workflows/linkedin-person-icp-scoring-automation-with-airtop---google-sheets-3476


# LinkedIn Person ICP Scoring Automation with Airtop & Google Sheets

### 1. Workflow Overview

This workflow automates the process of scoring individual LinkedIn profiles against an Ideal Customer Profile (ICP) for sales qualification purposes. It targets sales and marketing teams who want to prioritize leads efficiently by leveraging real-time LinkedIn data enriched through Airtop’s AI-powered extraction. The workflow consists of four main logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow and retrieval of lead data from a Google Sheet.
- **1.2 LinkedIn Profile Data Extraction:** Use Airtop to extract detailed LinkedIn profile information and calculate an ICP score based on AI interest, technical depth, and seniority.
- **1.3 Data Formatting:** Process and format the extracted data to prepare it for updating the Google Sheet.
- **1.4 Google Sheet Update:** Update the original Google Sheet row with the calculated ICP score and relevant LinkedIn URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and retrieves the list of individuals (leads) from a specified Google Sheet to be processed.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Get person (Google Sheets)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command for testing or manual execution.  
    - Configuration: No parameters; triggers workflow execution.  
    - Inputs: None  
    - Outputs: Triggers the next node "Get person".  
    - Edge cases: None typical; manual trigger requires user interaction.  

  - **Get person**  
    - Type: Google Sheets  
    - Role: Reads rows from a Google Sheet containing LinkedIn URLs and possibly other lead data.  
    - Configuration:  
      - Document ID: Points to a Google Sheet named "ICP Score for Template".  
      - Sheet Name: "Person" (gid=0).  
      - Reads all rows without filters.  
    - Key expressions: None; outputs all rows as JSON items.  
    - Inputs: Trigger from manual node.  
    - Outputs: Passes each row to the next node for processing.  
    - Edge cases:  
      - Google Sheets API errors (auth, quota).  
      - Empty or malformed rows.  
      - Missing required columns like Linkedin_URL_Person.

#### 2.2 LinkedIn Profile Data Extraction and ICP Scoring

- **Overview:**  
  This block uses Airtop’s AI-powered extraction to analyze each LinkedIn profile URL, extract detailed profile information, and compute an ICP score based on predefined criteria.

- **Nodes Involved:**  
  - Calculate ICP PersonScoring (Airtop)

- **Node Details:**

  - **Calculate ICP PersonScoring**  
    - Type: Airtop (AI extraction node)  
    - Role: Queries Airtop API to extract structured data from LinkedIn profiles and calculate ICP score.  
    - Configuration:  
      - URL: Dynamically set from the input JSON field `Linkedin_URL_Person`.  
      - Prompt: Detailed extraction instructions requesting full name, job title, employer, location, connections, followers, about section, AI interest level, seniority level, technical depth, and ICP score calculation.  
      - Resource: "extraction"  
      - Operation: "query"  
      - Session Mode: New session per request.  
      - Output Schema: JSON schema defining expected fields and types, ensuring data consistency.  
    - Key expressions: `={{ $json['Linkedin_URL_Person'] }}` for URL input.  
    - Inputs: Each row from Google Sheets with LinkedIn URL.  
    - Outputs: JSON object with extracted profile data and calculated ICP score.  
    - Version-specific: Requires Airtop API key credential configured in n8n.  
    - Edge cases:  
      - Invalid or inaccessible LinkedIn URLs.  
      - API rate limits or authentication failures.  
      - Parsing errors if LinkedIn page format changes.  
      - Incorrect or incomplete extraction leading to missing fields.  

#### 2.3 Data Formatting

- **Overview:**  
  This block formats the raw response from Airtop to extract only the necessary fields for updating the Google Sheet, specifically the row number, LinkedIn URL, and ICP score.

- **Nodes Involved:**  
  - Format response (Code node)

- **Node Details:**

  - **Format response**  
    - Type: Code (JavaScript)  
    - Role: Processes each item to extract and structure the output data for the Google Sheets update.  
    - Configuration:  
      - Runs once per item (`runOnceForEachItem` mode).  
      - JavaScript code extracts:  
        - `row_number` from the "Get person" node input.  
        - `Linkedin_URL_Person` from the same input.  
        - `ICP_Score_Person` parsed from Airtop’s JSON response (`data.modelResponse`).  
    - Key expressions: Uses `$()` syntax to access data from previous nodes.  
    - Inputs: Output from Airtop node.  
    - Outputs: JSON with keys `row_number`, `Linkedin_URL_Person`, and `ICP_Score_Person`.  
    - Edge cases:  
      - Parsing errors if Airtop response is malformed or missing `icp_score`.  
      - Missing `row_number` or LinkedIn URL in input data.  

#### 2.4 Google Sheet Update

- **Overview:**  
  This block updates the original Google Sheet row with the newly calculated ICP score, ensuring the lead data is enriched and ready for sales prioritization.

- **Nodes Involved:**  
  - Update row (Google Sheets)

- **Node Details:**

  - **Update row**  
    - Type: Google Sheets  
    - Role: Updates a specific row in the Google Sheet with the ICP score and LinkedIn URL.  
    - Configuration:  
      - Document ID: Same as "Get person" node.  
      - Sheet Name: "Person" (gid=0).  
      - Operation: Update row by matching on `row_number`.  
      - Columns mapped automatically from input JSON keys: `Linkedin_URL_Person`, `ICP_Score_Person`, `row_number`.  
      - Matching column: `row_number` ensures correct row update.  
    - Inputs: Formatted JSON from "Format response" node.  
    - Outputs: Confirmation of update operation.  
    - Edge cases:  
      - Google Sheets API errors (auth, quota, row not found).  
      - Mismatched or missing `row_number` leading to failed updates.  
      - Concurrent edits causing conflicts.  

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                        | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                               |
|----------------------------|--------------------|-------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Starts the workflow manually         | None                        | Get person                 |                                                                                                          |
| Get person                 | Google Sheets      | Reads lead data from Google Sheet    | When clicking ‘Test workflow’ | Calculate ICP PersonScoring |                                                                                                          |
| Calculate ICP PersonScoring | Airtop             | Extracts LinkedIn profile data and calculates ICP score | Get person                  | Format response            |                                                                                                          |
| Format response            | Code               | Formats Airtop response for sheet update | Calculate ICP PersonScoring | Update row                 |                                                                                                          |
| Update row                 | Google Sheets      | Updates Google Sheet row with ICP score | Format response             | None                       |                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named "When clicking ‘Test workflow’".  
   - No parameters needed. This node will start the workflow manually.

2. **Add Google Sheets Node to Read Leads**  
   - Add a "Google Sheets" node named "Get person".  
   - Set operation to "Read Rows".  
   - Configure credentials for your Google account with access to the target sheet.  
   - Set Document ID to your Google Sheet containing lead data (e.g., "ICP Score for Template").  
   - Set Sheet Name to "Person" (gid=0).  
   - Leave filters empty to read all rows.  
   - Connect "When clicking ‘Test workflow’" output to this node.

3. **Add Airtop Node for LinkedIn Profile Extraction and ICP Scoring**  
   - Add an "Airtop" node named "Calculate ICP PersonScoring".  
   - Configure Airtop credentials with your API key.  
   - Set Resource to "extraction" and Operation to "query".  
   - Set Session Mode to "new".  
   - In the URL field, use expression: `={{ $json["Linkedin_URL_Person"] }}` to dynamically pass the LinkedIn URL from the sheet.  
   - Paste the provided detailed prompt instructing Airtop to extract profile data and calculate ICP score.  
   - Paste the provided JSON output schema to enforce response structure.  
   - Connect "Get person" output to this node.

4. **Add Code Node to Format Airtop Response**  
   - Add a "Code" node named "Format response".  
   - Set mode to "Run Once For Each Item".  
   - Paste the JavaScript code:  
     ```javascript
     const row_number = $('Get person').item.json.row_number;
     const Linkedin_URL_Person = $('Get person').item.json.Linkedin_URL_Person;
     const ICP_Score_Person = JSON.parse($input.item.json.data.modelResponse).icp_score;

     return { json: { row_number, Linkedin_URL_Person, ICP_Score_Person } };
     ```  
   - Connect "Calculate ICP PersonScoring" output to this node.

5. **Add Google Sheets Node to Update Row**  
   - Add a "Google Sheets" node named "Update row".  
   - Set operation to "Update Row".  
   - Use the same Google Sheets credentials as before.  
   - Set Document ID and Sheet Name to the same as "Get person".  
   - Configure columns mapping to automatically map `Linkedin_URL_Person`, `ICP_Score_Person`, and `row_number` from input JSON.  
   - Set matching column to `row_number` to update the correct row.  
   - Connect "Format response" output to this node.

6. **Verify and Test**  
   - Save the workflow.  
   - Run the manual trigger node to start the process.  
   - Monitor execution for errors and validate that the Google Sheet updates with ICP scores.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| A free Airtop API key is required to use the Airtop node for LinkedIn profile extraction. Obtain it from the Airtop dashboard.                                                                                               | https://portal.airtop.ai/?utm_campaign=n8n                                                              |
| The Google Sheet must contain at least the columns `Linkedin_URL_Person` and `ICP_Score_Person` for input and output respectively.                                                                                          | https://docs.google.com/spreadsheets/d/1O69nQkKr4fyWl5AQUrX7y-nwPCMDeFwp-2swG0YW6Cg/edit?usp=sharing      |
| The ICP scoring criteria are customizable and based on AI interest, technical depth, and seniority with specific point allocations.                                                                                           | See workflow prompt details                                                                               |
| Consider adding notification triggers or CRM integrations for enhanced automation and sales follow-up.                                                                                                                        | Customization suggestions in the workflow description                                                    |
| Regularly review and refine ICP criteria to maintain lead qualification accuracy as market conditions evolve.                                                                                                                 | Best practices section                                                                                     |
| This workflow uses Airtop’s AI extraction capabilities, which depend on LinkedIn’s publicly available data and may be affected by LinkedIn’s UI changes or access restrictions.                                                | General caution about third-party data extraction reliability                                            |

---

This document fully describes the workflow structure, node configurations, and logic to enable reproduction, modification, and troubleshooting by advanced users or AI agents.