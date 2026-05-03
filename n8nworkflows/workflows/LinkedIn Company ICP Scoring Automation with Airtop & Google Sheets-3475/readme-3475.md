LinkedIn Company ICP Scoring Automation with Airtop & Google Sheets

https://n8nworkflows.xyz/workflows/linkedin-company-icp-scoring-automation-with-airtop---google-sheets-3475


# LinkedIn Company ICP Scoring Automation with Airtop & Google Sheets

### 1. Workflow Overview

This workflow automates the process of scoring and prioritizing company leads based on their LinkedIn profiles to identify Ideal Customer Profiles (ICP). It targets sales and marketing teams who want to efficiently qualify leads by leveraging real-time data extraction and AI-based analysis. The workflow integrates Google Sheets (for input/output data management) and Airtop AI (for LinkedIn profile enrichment and scoring).

**Logical blocks:**

- **1.1 Input Reception:** Manual trigger initiating the workflow.
- **1.2 Data Retrieval:** Fetch company LinkedIn URLs and related data from Google Sheets.
- **1.3 AI Profile Analysis & ICP Scoring:** Use Airtop AI to analyze LinkedIn company profiles, extracting detailed information and calculating an ICP score based on defined criteria.
- **1.4 Data Formatting:** Prepare the AI response data into a simplified format for updating.
- **1.5 Data Output:** Update the original Google Sheet rows with the calculated ICP scores.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow manually, allowing users to start the scoring process on demand.

**Nodes Involved:**  
- When clicking ‘Test workflow’

**Node Details:**

- **Node:** When clicking ‘Test workflow’  
  - Type: Manual Trigger  
  - Configuration: No parameters; triggers execution when manually activated in n8n.  
  - Inputs: None  
  - Outputs: Triggers the next node ("Get companies")  
  - Edge Cases: None typical; potential user error if triggered unintentionally.  

---

#### 2.2 Data Retrieval

**Overview:**  
Retrieves company data from a specified Google Sheet, particularly the LinkedIn URLs needed for profile analysis.

**Nodes Involved:**  
- Get companies

**Node Details:**

- **Node:** Get companies  
  - Type: Google Sheets (Read)  
  - Configuration:  
    - Operation: Read rows from the “Company” sheet (sheet ID 1729280298) in the "ICP Score for Template" Google Spreadsheet.  
    - Document ID: 1WC_awgb-Ohtb0f4o_OJgRcvunTLuS8kFQgk6l8fkR2Q  
    - Credentials: Google Drive OAuth2 connection configured.  
  - Key Variables: Retrieves rows with at least the column `Linkedin_URL_Company`.  
  - Inputs: Trigger input from manual trigger node.  
  - Outputs: Emits an array of rows, each containing LinkedIn URLs and metadata such as row number.  
  - Edge Cases / Failure Types:  
    - Authentication errors with Google OAuth.  
    - Sheet or document not found.  
    - Empty or malformed rows without `Linkedin_URL_Company`.  
    - API rate limiting or connectivity issues.  

---

#### 2.3 AI Profile Analysis & ICP Scoring

**Overview:**  
Uses Airtop AI to extract detailed company profile data from LinkedIn URLs and calculate an ICP score based on multiple weighted criteria.

**Nodes Involved:**  
- Calculate ICP Scoring

**Node Details:**

- **Node:** Calculate ICP Scoring  
  - Type: Airtop (AI Extraction & Query)  
  - Configuration:  
    - URL: Dynamically set per company row (`{{$json["Linkedin_URL_Company"]}}`).  
    - Prompt: Detailed JSON extraction prompt requesting company identity, scale, business classification, technical sophistication, investment profile, and a composite ICP score based on specified scoring rules.  
    - Operation: Query extraction with a new session mode.  
  - Key Expressions:  
    - Uses dynamic expression for URL input.  
    - The prompt includes detailed instructions for AI to return structured JSON with required fields.  
  - Inputs: Rows from Google Sheets with LinkedIn URLs.  
  - Outputs: JSON object with enriched company data and scoring results.  
  - Version Requirements: Airtop API key credential required and valid.  
  - Edge Cases / Potential Failures:  
    - Invalid or inaccessible LinkedIn URLs.  
    - Airtop API authentication failure.  
    - API rate limiting or timeout.  
    - AI response parsing errors if the returned JSON is malformed or incomplete.  
    - Handling companies with missing or private data.  
  - Notes: This node performs both data extraction and scoring in a single AI query, streamlining the process.

---

#### 2.4 Data Formatting

**Overview:**  
Extracts and formats the relevant scoring data from the AI response to prepare it for updating the Google Sheet.

**Nodes Involved:**  
- Format response

**Node Details:**

- **Node:** Format response  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Mode: Run once for each item (row).  
    - Code Logic:  
      - Extracts `row_number` and `Linkedin_URL_Company` from the original Google Sheets row.  
      - Parses AI response JSON (`data.modelResponse`) to access `icp_scoring.total_score`.  
      - Returns a simplified JSON object with:  
        - `row_number` (for matching row update)  
        - `Linkedin_URL_Company` (for reference)  
        - `ICP_Score_Company` (the calculated ICP score)  
  - Inputs: AI response data per company row.  
  - Outputs: Formatted JSON for Google Sheets update.  
  - Edge Cases:  
    - JSON parsing errors if AI response is malformed.  
    - Missing expected fields in AI response (e.g., `icp_scoring.total_score`).  
  - Version-specific: Uses n8n Code node v2 syntax.  

---

#### 2.5 Data Output

**Overview:**  
Updates the original Google Sheet with the new ICP scores for each company.

**Nodes Involved:**  
- Update row

**Node Details:**

- **Node:** Update row  
  - Type: Google Sheets (Update)  
  - Configuration:  
    - Operation: Update row in the “Company” sheet (sheet ID 1729280298) within the "ICP Score for Template" Google Spreadsheet.  
    - Matching Column: `row_number` used to identify the correct row to update.  
    - Columns Mapped: At minimum, updates `ICP_Score_Company`, preserves `Linkedin_URL_Company`, and includes metadata fields (`meta`, `data`, `errors`, `warnings`, `parsed`), though some may be blank if not used.  
    - Credentials: Google Drive OAuth2 connection configured.  
  - Inputs: Formatted JSON with row number and ICP score.  
  - Outputs: Confirmation of updated rows (optional).  
  - Edge Cases:  
    - Authentication issues with Google Sheets API.  
    - Row number mismatch or missing row number.  
    - Rate limits or API unavailability.  
  - Notes: Auto-maps incoming data to sheet columns, ensuring scores are written to the correct rows.

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                  | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                                   |
|-------------------------|-------------------------|---------------------------------|---------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Initiates the workflow manually | None                      | Get companies            |                                                                                                                               |
| Get companies           | Google Sheets (Read)     | Retrieves company LinkedIn URLs | When clicking ‘Test workflow’ | Calculate ICP Scoring    |                                                                                                                               |
| Calculate ICP Scoring   | Airtop (AI Extraction)   | Extracts LinkedIn data and computes ICP score | Get companies             | Format response          | Uses Airtop to extract detailed company profile data and calculate ICP score based on multiple weighted criteria.             |
| Format response         | Code (JavaScript)        | Formats AI response for update  | Calculate ICP Scoring      | Update row               | Parses AI JSON output and prepares minimal data payload for Google Sheets update.                                              |
| Update row              | Google Sheets (Update)   | Updates Google Sheet rows with ICP scores | Format response           | None                     | Updates ICP_Score_Company in the Google Sheet matching by row number.                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To start the workflow manually.

2. **Add a Google Sheets Node (Read Operation):**  
   - Name: `Get companies`  
   - Operation: Read rows from Google Sheets.  
   - Configure:  
     - Connect Google Sheets OAuth2 credentials.  
     - Document ID: `1WC_awgb-Ohtb0f4o_OJgRcvunTLuS8kFQgk6l8fkR2Q` (replace with your own if needed).  
     - Sheet Name: `Company` (sheet ID 1729280298).  
     - Read all rows or specify desired range.  
   - Connect the Manual Trigger node’s output to this node’s input.

3. **Add an Airtop Node (Extraction):**  
   - Name: `Calculate ICP Scoring`  
   - Credentials: Configure Airtop API key.  
   - Parameters:  
     - URL: Set to an expression referencing current item’s `Linkedin_URL_Company` (e.g., `={{ $json["Linkedin_URL_Company"] }}`).  
     - Prompt: Use the provided detailed LinkedIn Company Analysis Prompt for comprehensive data extraction and scoring.  
     - Operation: Query extraction with new session.  
     - Output Schema: Use the detailed JSON schema defining expected fields.  
   - Connect the Google Sheets node output to this node’s input.

4. **Add a Code Node (JavaScript):**  
   - Name: `Format response`  
   - Mode: Run once for each item (process each company separately).  
   - Code: Use the following logic:  
     ```javascript
     const row_number = $('Get companies').item.json.row_number;
     const Linkedin_URL_Company = $('Get companies').item.json.Linkedin_URL_Company;
     const icp_scoring = JSON.parse($input.item.json.data.modelResponse).icp_scoring;

     return {
       json: {
         row_number,
         Linkedin_URL_Company,
         ICP_Score_Company: icp_scoring.total_score
       }
     };
     ```  
   - Connect the Airtop node output to this node’s input.

5. **Add a Google Sheets Node (Update Operation):**  
   - Name: `Update row`  
   - Operation: Update rows in the same Google Sheet and tab as in step 2.  
   - Configure:  
     - Use the same document ID and sheet name (`Company`).  
     - Matching column: `row_number` to locate the correct row.  
     - Map columns: At least update `ICP_Score_Company` with the new score. Include any other columns as needed.  
     - Use the same Google Sheets OAuth2 credentials.  
   - Connect the Code node output to this node’s input.

6. **Workflow Execution:**  
   - Test by manually triggering the workflow via the manual trigger node.  
   - The workflow will read LinkedIn URLs, query Airtop AI, calculate the ICP score, and update the Google Sheet accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Free Airtop API key required to use the AI extraction node.                                                                      | Obtain from https://portal.airtop.ai/?utm_campaign=n8n                                                        |
| Sample Google Sheets template provided for input and output columns, including `Linkedin_URL_Company` and `ICP_Score_Company`. | Template link: https://docs.google.com/spreadsheets/d/1O69nQkKr4fyWl5AQUrX7y-nwPCMDeFwp-2swG0YW6Cg/edit       |
| ICP scoring criteria and weighting logic are embedded in the Airtop AI prompt.                                                   | Modify prompt for customization of scoring rules as needed.                                                  |
| Recommended best practices include quarterly review of ICP criteria and tiered follow-up strategies.                             | Use this automation to prioritize sales leads effectively and integrate with CRM or notification systems.    |
| Additional customization options include batch processing, conditional scoring, and multi-channel notifications.                | Can be extended by adding nodes and conditional branches in n8n.                                            |

---

This documentation fully captures the workflow’s structure, logic, configuration, and operational context, enabling advanced users or automation agents to understand, reproduce, and customize the company ICP scoring automation using n8n, Airtop, and Google Sheets.