Automated Funding Intelligence: CrunchBase to Google Sheets Tracking Workflow

https://n8nworkflows.xyz/workflows/automated-funding-intelligence--crunchbase-to-google-sheets-tracking-workflow-4730


# Automated Funding Intelligence: CrunchBase to Google Sheets Tracking Workflow

### 1. Workflow Overview

**Purpose:**  
This workflow automates the monitoring and tracking of new funding rounds from Crunchbase for specific industries and regions, and logs the structured data into a Google Sheet. It is designed for venture capitalists, startup scouts, market researchers, or analysts who need up-to-date funding intelligence without manual data gathering.

**Target Use Cases:**  
- Daily automated retrieval of recent funding rounds in Fintech and Healthtech sectors in the United States.  
- Real-time data logging into Google Sheets for easy access, analysis, and reporting.  
- Customizable filters for location, industry, and sorting by newest funding rounds.  

**Logical Blocks:**  
- **1.1 Input Reception & Scheduling:** Trigger execution daily at a fixed time.  
- **1.2 Data Fetching:** Query Crunchbase API to retrieve recent funding rounds filtered by industry and location.  
- **1.3 Data Processing:** Extract and transform the raw API response into a clean, structured format.  
- **1.4 Data Logging:** Append the processed data into a Google Sheets document for storage and analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Scheduling

**Overview:**  
This block controls when the workflow runs. It uses a schedule trigger node set to activate daily at 8:00 AM, initiating the workflow automatically without manual intervention.

**Nodes Involved:**  
- Daily Check for New Funding Rounds

**Node Details:**  
- **Daily Check for New Funding Rounds**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow on a daily schedule.  
  - *Configuration:* Set to trigger once daily at 8:00 AM local time using built-in Cron scheduling options.  
  - *Expressions/Variables:* None.  
  - *Input:* None (trigger node).  
  - *Output:* Triggers the next node, "Fetch Crunchbase Funding Rounds".  
  - *Version-Specific Requirements:* Version 1.2 of Schedule Trigger node used. Compatible with n8n versions supporting Cron scheduling.  
  - *Potential Failures:* Misconfigured time zone or schedule settings could cause missed triggers. No authentication needed.  
  - *Sub-Workflow:* None.

---

#### 1.2 Data Fetching

**Overview:**  
This block queries the Crunchbase API to retrieve the most recent funding rounds filtered by location (United States) and industries (Fintech, Healthtech). It uses an HTTP Request node configured with appropriate query and header parameters.

**Nodes Involved:**  
- Fetch Crunchbase Funding Rounds

**Node Details:**  
- **Fetch Crunchbase Funding Rounds**  
  - *Type:* HTTP Request  
  - *Role:* Sends GET request to Crunchbase API to fetch recent funding rounds.  
  - *Configuration:*  
    - URL: `https://api.crunchbase.com/api/v4/entities/funding_rounds`  
    - Query Parameters:  
      - locations = United States  
      - industry = Fintech, Healthtech  
      - sort_order = created_at DESC (latest first)  
      - page = 1  
      - items_per_page = 25  
    - Header: `X-cb-user-key` with a Crunchbase API key (must be replaced by user).  
    - Request method: GET.  
  - *Expressions/Variables:* Static query and header values configured.  
  - *Input:* Trigger from Schedule Trigger node.  
  - *Output:* JSON response containing funding round entities.  
  - *Version-Specific Requirements:* Uses HTTP Request node version 4.2. Requires valid Crunchbase API key for authentication.  
  - *Potential Failures:*  
    - Authentication failure if API key invalid or missing.  
    - API rate limiting or downtime.  
    - Network timeouts.  
    - Response schema changes by Crunchbase could break parsing.  
  - *Sub-Workflow:* None.

---

#### 1.3 Data Processing

**Overview:**  
This block processes the raw Crunchbase API response, extracting and formatting relevant fields into a simplified JSON structure suitable for logging. It consolidates investor lists and constructs full URLs for each funding round.

**Nodes Involved:**  
- Extract & Format Funding Data

**Node Details:**  
- **Extract & Format Funding Data**  
  - *Type:* Code (Function) Node  
  - *Role:* Parses the API response and transforms it into a clean, tabular format.  
  - *Configuration:*  
    - JavaScript code extracts:  
      - company_name (from funded_organization_identifier.value)  
      - industry (concatenated from industry_group_identifiers array)  
      - funding_round_type  
      - announced_date  
      - money_raised_usd  
      - investors (concatenated investor_identifiers array)  
      - crunchbase_url (constructed by appending permalink to base Crunchbase URL)  
    - Handles missing fields by substituting "N/A" or zero values.  
  - *Key Expressions:* Uses `items[0].json.data.entities` to iterate over funding rounds.  
  - *Input:* JSON array from HTTP Request node.  
  - *Output:* Array of simplified JSON objects, one per funding round.  
  - *Version-Specific Requirements:* Function node version 2 used; compatible with n8n JavaScript environment.  
  - *Potential Failures:*  
    - Errors if expected JSON structure changes.  
    - Unexpected null or missing fields causing runtime errors.  
    - Improper string joining if arrays are empty or undefined.  
  - *Sub-Workflow:* None.

---

#### 1.4 Data Logging

**Overview:**  
This block appends the processed funding data as new rows into a specified Google Sheets document, creating a live, up-to-date database of funding rounds.

**Nodes Involved:**  
- Log to Google Sheets

**Node Details:**  
- **Log to Google Sheets**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends data rows to a Google Sheet for storage and later analysis.  
  - *Configuration:*  
    - Operation: Append  
    - Target Spreadsheet: ID `11bGAwfhLtR1FZwEZUAi-pddixpJintqFXAT1eWp_QQ0`  
    - Sheet Name: `Sheet1` (gid=0)  
    - Column Mapping: Maps processed JSON fields to sheet columns:  
      - company name  
      - industry  
      - funding round type  
      - announced date  
      - money raised usd  
      - investors  
      - crunchbase url  
    - Uses Google OAuth2 credentials for authentication.  
  - *Expressions/Variables:* Maps each JSON property from the previous node directly into the corresponding sheet column.  
  - *Input:* Processed funding data from Function node.  
  - *Output:* Confirmation of rows appended (metadata).  
  - *Version-Specific Requirements:* Google Sheets node version 4.5; requires OAuth2 credential setup with proper scopes for sheet editing.  
  - *Potential Failures:*  
    - Authentication failures if OAuth token is invalid or expired.  
    - Permission issues on the target spreadsheet.  
    - Sheet name or document ID errors.  
    - Data type mismatches if input data is malformed.  
  - *Sub-Workflow:* None.

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                     | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                  |
|-------------------------------|-----------------------|-----------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Daily Check for New Funding Rounds | Schedule Trigger      | Initiates daily workflow execution | None                         | Fetch Crunchbase Funding Rounds | 🔶 **SECTION ONE: Data Fetching & Triggering** - Daily schedule trigger at 08:00 AM           |
| Fetch Crunchbase Funding Rounds   | HTTP Request          | Fetches funding round data from Crunchbase API | Daily Check for New Funding Rounds | Extract & Format Funding Data | 🔶 **SECTION ONE: Data Fetching & Triggering** - Pulls fresh funding round data, filters by location and industry |
| Extract & Format Funding Data      | Function (Code)       | Extracts and formats API response | Fetch Crunchbase Funding Rounds | Log to Google Sheets          | 🔷 **SECTION TWO: Processing & Logging** - Cleans and reformats Crunchbase data into structured rows |
| Log to Google Sheets              | Google Sheets         | Logs formatted data into Google Sheets | Extract & Format Funding Data | None                        | 🔷 **SECTION TWO: Processing & Logging** - Appends funding data rows to Google Sheets         |
| Sticky Note                      | Sticky Note           | Documentation / explanation        | None                         | None                        | 🔶 **SECTION ONE: Data Fetching & Triggering** detailed explanation included                   |
| Sticky Note1                     | Sticky Note           | Documentation / explanation        | None                         | None                        | 🔷 **SECTION TWO: Processing & Logging** detailed explanation included                         |
| Sticky Note4                     | Sticky Note           | Full workflow explanation and summary | None                         | None                        | Complete step-by-step explanation of workflow sections and benefits                           |
| Sticky Note9                     | Sticky Note           | Workflow assistance contact info   | None                         | None                        | Contact info and external resources links                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Daily Check for New Funding Rounds`  
   - Set to trigger daily at 8:00 AM using Cron schedule options.

3. **Add an HTTP Request node:**  
   - Name: `Fetch Crunchbase Funding Rounds`  
   - Method: GET  
   - URL: `https://api.crunchbase.com/api/v4/entities/funding_rounds`  
   - Query Parameters:  
     - locations = `United States`  
     - industry = `Fintech,Healthtech`  
     - sort_order = `created_at DESC`  
     - page = `1`  
     - items_per_page = `25`  
   - Headers:  
     - `X-cb-user-key` = Your Crunchbase API key (set this via credentials or directly)  
   - Connect output of Schedule Trigger to input of this HTTP Request node.

4. **Add a Function node:**  
   - Name: `Extract & Format Funding Data`  
   - Paste the following JavaScript code:  
     ```javascript
     const baseUrl = "https://www.crunchbase.com/funding-round/";
     const output = [];

     for (const item of items[0].json.data.entities) {
       const props = item.properties;
       const id = item.identifier;

       const companyName = props.funded_organization_identifier?.value || "N/A";
       const industry = props.industry_group_identifiers?.map(ind => ind.value).join(", ") || "N/A";
       const fundingType = props.funding_type || "N/A";
       const date = props.announced_on || "N/A";
       const amount = props.money_raised_usd || 0;
       const investors = props.investor_identifiers?.map(inv => inv.value).join(", ") || "N/A";
       const url = baseUrl + id.permalink;

       output.push({
         json: {
           company_name: companyName,
           industry: industry,
           funding_round_type: fundingType,
           announced_date: date,
           money_raised_usd: amount,
           investors: investors,
           crunchbase_url: url
         }
       });
     }

     return output;
     ```  
   - Connect output of HTTP Request node to input of this Function node.

5. **Add a Google Sheets node:**  
   - Name: `Log to Google Sheets`  
   - Operation: Append  
   - Document ID: `11bGAwfhLtR1FZwEZUAi-pddixpJintqFXAT1eWp_QQ0` (replace with your spreadsheet ID)  
   - Sheet Name: `Sheet1` (or your target sheet)  
   - Map columns as follows:  
     - company name → `{{$json.company_name}}`  
     - industry → `{{$json.industry}}`  
     - funding round type → `{{$json.funding_round_type}}`  
     - announced date → `{{$json.announced_date}}`  
     - money raised usd → `{{$json.money_raised_usd}}`  
     - investors → `{{$json.investors}}`  
     - crunchbase url → `{{$json.crunchbase_url}}`  
   - Set up Google Sheets OAuth2 credentials with edit permissions on the target spreadsheet.  
   - Connect output of Function node to input of Google Sheets node.

6. **Save and activate the workflow.**  
   - Confirm that the Schedule Trigger is active and that the workflow runs as expected.  
   - Optionally, run the workflow manually for testing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| For any questions or support related to this workflow, contact Yaron at Yaron@nofluff.online.                                                                                                                                                                                                                                                                          | Support contact                                                            |
| Explore additional tips and tutorials on YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                     | External learning resources                                                |
| This workflow provides a robust foundation for automated funding intelligence and can be customized to include other industries, countries, or integration points such as Slack notifications or email alerts.                                                                                                                                                          | Customization advice                                                       |
| Remember to replace placeholder API keys with your valid credentials and ensure you have proper permissions for Google Sheets API access.                                                                                                                                                                                                                              | Credential setup reminder                                                  |
| To improve reliability, consider adding error handling nodes for API failures, rate limiting, or Google Sheets write errors.                                                                                                                                                                                                                                          | Error handling best practices                                             |
| This workflow is designed with no-code and low-code logic, making it accessible to users without programming experience.                                                                                                                                                                                                                                              | Accessibility note                                                        |

---

**Disclaimer:**  
The text above is generated from an automated n8n workflow analysis tool. All data handled is legal, public, and compliant with content policies. The workflow respects Crunchbase and Google API usage guidelines.