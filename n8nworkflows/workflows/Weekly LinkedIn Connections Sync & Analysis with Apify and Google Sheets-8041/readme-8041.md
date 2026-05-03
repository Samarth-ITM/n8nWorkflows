Weekly LinkedIn Connections Sync & Analysis with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/weekly-linkedin-connections-sync---analysis-with-apify-and-google-sheets-8041


# Weekly LinkedIn Connections Sync & Analysis with Apify and Google Sheets

### 1. Workflow Overview

This workflow automates the weekly synchronization and analysis of LinkedIn connections data. Scheduled to run every Sunday at 2 AM, it triggers a LinkedIn scrape via Apify, monitors the scrape progress, retrieves the connections data once the scrape completes, processes and cleans the data, and finally updates a Google Sheets document with the latest connections and a sync summary.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Triggering the Scrape:** Initiates the workflow on a weekly schedule and starts the LinkedIn scraping process.
- **1.2 Scrape Monitoring and Control:** Extracts the scrape run ID, waits, checks scrape status, and retries as needed until completion.
- **1.3 Data Retrieval and Processing:** Fetches the scraped connections data and processes it for storage.
- **1.4 Google Sheets Data Management:** Clears previous data, saves the new connections data, and generates a summary of the sync.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Triggering the Scrape

- **Overview:** This block sets the workflow to run on a fixed weekly schedule and initiates the LinkedIn scraping job via an HTTP request to Apify or a similar service.
- **Nodes Involved:** Weekly Sync (Sunday 2AM), Start LinkedIn Scrape

##### Weekly Sync (Sunday 2AM)
- **Type & Role:** Cron Trigger node; triggers the workflow every Sunday at 2 AM.
- **Configuration:** Default weekly cron schedule set to Sunday 2:00 AM.
- **Expressions/Variables:** None.
- **Connections:** Outputs to "Start LinkedIn Scrape".
- **Failures:** Cron misconfiguration, time zone issues.
- **Notes:** None.

##### Start LinkedIn Scrape
- **Type & Role:** HTTP Request node; initiates the LinkedIn scrape by calling the scraping API.
- **Configuration:** Likely configured to send a POST request to start the scrape job, includes necessary authentication headers or tokens.
- **Expressions/Variables:** May use credentials or environment variables for API authentication.
- **Connections:** Outputs to "Extract Run ID".
- **Failures:** API authentication failures, network timeouts, incorrect endpoint or payload.
- **Notes:** None.

#### 1.2 Scrape Monitoring and Control

- **Overview:** This block extracts the run ID from the scrape initiation response, waits for the scrape to progress, checks the scrape status periodically, and loops with retries until the scrape completes.
- **Nodes Involved:** Extract Run ID, Wait for Scrape (30s), Check Scrape Status, Scrape Completed?, Wait & Retry (60s)

##### Extract Run ID
- **Type & Role:** Code node; parses HTTP response to extract the unique scrape run ID.
- **Configuration:** Custom JavaScript code to extract the run ID from the JSON response of "Start LinkedIn Scrape".
- **Expressions/Variables:** Uses output data from the previous node.
- **Connections:** Outputs to "Wait for Scrape (30s)".
- **Failures:** Parsing errors if response format changes or is empty.
- **Notes:** None.

##### Wait for Scrape (30s)
- **Type & Role:** Wait node; pauses the workflow for 30 seconds before next status check.
- **Configuration:** Fixed wait time of 30 seconds.
- **Expressions/Variables:** None.
- **Connections:** Outputs to "Check Scrape Status".
- **Failures:** None expected.

##### Check Scrape Status
- **Type & Role:** HTTP Request node; queries the scraping API for the status of the running scrape using the run ID.
- **Configuration:** GET request with run ID as a parameter to check scrape status.
- **Expressions/Variables:** Uses run ID extracted earlier.
- **Connections:** Outputs to "Scrape Completed?".
- **Failures:** API errors, authentication issues, network timeouts.

##### Scrape Completed?
- **Type & Role:** If node; conditional check to determine if scrape has finished.
- **Configuration:** Evaluates the scrape status response to see if the job is complete.
- **Expressions/Variables:** Checks specific fields in HTTP response, e.g., `status === 'SUCCEEDED'`.
- **Connections:** If true, outputs to "Get Scraped Data"; if false, outputs to "Wait & Retry (60s)".
- **Failures:** Logic errors if response structure changes.

##### Wait & Retry (60s)
- **Type & Role:** Wait node; waits 60 seconds before retrying the status check.
- **Configuration:** Fixed 60-second wait.
- **Expressions/Variables:** None.
- **Connections:** Outputs back to "Check Scrape Status" to form a retry loop.
- **Failures:** None expected.

#### 1.3 Data Retrieval and Processing

- **Overview:** After successful scrape completion, this block fetches the actual connections data, processes it (e.g., formatting, filtering), and prepares it for storage.
- **Nodes Involved:** Get Scraped Data, Process Connections Data

##### Get Scraped Data
- **Type & Role:** HTTP Request node; retrieves the scraped LinkedIn connections data using the run ID.
- **Configuration:** Likely a GET request to the API endpoint that provides the scraped dataset.
- **Expressions/Variables:** Uses run ID to specify which data to retrieve.
- **Connections:** Outputs to "Process Connections Data".
- **Failures:** API errors, incomplete data, timeouts.

##### Process Connections Data
- **Type & Role:** Code node; processes raw connections data, possibly cleaning and structuring it for Google Sheets.
- **Configuration:** Custom JavaScript code that transforms the data array to the required sheet format.
- **Expressions/Variables:** Accesses data from "Get Scraped Data".
- **Connections:** Outputs to "Clear Existing Data" and "Save Connections to Sheets".
- **Failures:** Data format mismatches, code exceptions.

#### 1.4 Google Sheets Data Management

- **Overview:** This block clears outdated data from the Google Sheets document, saves the new connections data, and generates a summary report.
- **Nodes Involved:** Clear Existing Data, Save Connections to Sheets, Generate Sync Summary

##### Clear Existing Data
- **Type & Role:** Google Sheets node; clears the existing connections data range in the target spreadsheet.
- **Configuration:** Configured with Google Sheets credentials, spreadsheet ID, and range to clear.
- **Expressions/Variables:** None.
- **Connections:** Outputs to "Save Connections to Sheets".
- **Failures:** Authentication errors, incorrect sheet or range.

##### Save Connections to Sheets
- **Type & Role:** Google Sheets node; appends or replaces the processed connections data into the spreadsheet.
- **Configuration:** Uses credentials, targets specific sheet and range.
- **Expressions/Variables:** Receives data from "Process Connections Data".
- **Connections:** Outputs to "Generate Sync Summary".
- **Failures:** API quota limits, data format errors.

##### Generate Sync Summary
- **Type & Role:** Code node; creates a summary of the sync operation, possibly including counts, timestamps, or errors.
- **Configuration:** Custom JavaScript to generate summary text or data.
- **Expressions/Variables:** Uses data from previous nodes to summarize.
- **Connections:** Terminal node.
- **Failures:** Code logic errors.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                         | Input Node(s)               | Output Node(s)                 | Sticky Note                                 |
|-------------------------|--------------------|---------------------------------------|----------------------------|-------------------------------|---------------------------------------------|
| Setup Instructions      | Sticky Note        | Workflow instructions or comments     |                            |                               |                                             |
| Weekly Sync (Sunday 2AM) | Cron               | Weekly trigger for workflow            |                            | Start LinkedIn Scrape          |                                             |
| Start LinkedIn Scrape   | HTTP Request       | Initiate LinkedIn scrape               | Weekly Sync (Sunday 2AM)    | Extract Run ID                 |                                             |
| Extract Run ID          | Code               | Parse scrape run ID from response      | Start LinkedIn Scrape       | Wait for Scrape (30s)          |                                             |
| Wait for Scrape (30s)   | Wait               | Pause before status check               | Extract Run ID              | Check Scrape Status            |                                             |
| Check Scrape Status     | HTTP Request       | Check current scrape status             | Wait for Scrape (30s), Wait & Retry (60s) | Scrape Completed?              |                                             |
| Scrape Completed?       | If                 | Branch logic to continue or retry       | Check Scrape Status         | Get Scraped Data (if true), Wait & Retry (60s) (if false) |                                             |
| Wait & Retry (60s)      | Wait               | Wait before retrying scrape status      | Scrape Completed? (false)   | Check Scrape Status            |                                             |
| Get Scraped Data        | HTTP Request       | Retrieve scraped connections data       | Scrape Completed? (true)    | Process Connections Data       |                                             |
| Process Connections Data | Code               | Clean and format connections data       | Get Scraped Data            | Clear Existing Data, Save Connections to Sheets |                                             |
| Clear Existing Data     | Google Sheets      | Clear old connections data in sheet     | Process Connections Data    | Save Connections to Sheets     |                                             |
| Save Connections to Sheets | Google Sheets   | Save new connections data to sheet       | Clear Existing Data, Process Connections Data | Generate Sync Summary          |                                             |
| Generate Sync Summary   | Code               | Create summary of sync operation         | Save Connections to Sheets  |                               |                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**
   - Name: `Weekly Sync (Sunday 2AM)`
   - Type: Cron
   - Settings: Configure to trigger every Sunday at 2:00 AM.
   - No credentials required.

2. **Create HTTP Request Node to Start Scrape**
   - Name: `Start LinkedIn Scrape`
   - Type: HTTP Request
   - Configure with the LinkedIn scraping service API endpoint to initiate a scrape (likely POST).
   - Set authentication headers or credentials as required by the API.
   - Connect output from `Weekly Sync (Sunday 2AM)`.

3. **Create Code Node to Extract Run ID**
   - Name: `Extract Run ID`
   - Type: Code (JavaScript)
   - Write code to parse the run ID from the response of `Start LinkedIn Scrape`.
   - Connect output from `Start LinkedIn Scrape`.

4. **Create Wait Node (30 seconds)**
   - Name: `Wait for Scrape (30s)`
   - Type: Wait
   - Set waiting time to 30 seconds.
   - Connect output from `Extract Run ID`.

5. **Create HTTP Request Node to Check Scrape Status**
   - Name: `Check Scrape Status`
   - Type: HTTP Request
   - Configure GET request to query scrape status endpoint, using run ID from previous nodes.
   - Include authentication as necessary.
   - Connect output from `Wait for Scrape (30s)` and later from `Wait & Retry (60s)`.

6. **Create If Node to Check Completion**
   - Name: `Scrape Completed?`
   - Type: If
   - Configure condition to check if scrape status equals "SUCCEEDED" or equivalent.
   - Connect output from `Check Scrape Status`.

7. **Create HTTP Request Node to Get Scraped Data**
   - Name: `Get Scraped Data`
   - Type: HTTP Request
   - Configure GET request to fetch the scraped connections data using the run ID.
   - Connect "true" output from `Scrape Completed?`.

8. **Create Code Node to Process Data**
   - Name: `Process Connections Data`
   - Type: Code
   - Write JavaScript to transform raw data into sheet-ready format (array of arrays or objects).
   - Connect output from `Get Scraped Data`.

9. **Create Google Sheets Node to Clear Old Data**
   - Name: `Clear Existing Data`
   - Type: Google Sheets
   - Configure with Google Sheets OAuth2 credentials.
   - Set spreadsheet ID and range to clear existing connections data.
   - Connect output from `Process Connections Data`.

10. **Create Google Sheets Node to Save New Data**
    - Name: `Save Connections to Sheets`
    - Type: Google Sheets
    - Configure with same credentials and spreadsheet details.
    - Set to write processed connections data to the target sheet and range.
    - Connect outputs from both `Clear Existing Data` and `Process Connections Data`.

11. **Create Code Node to Generate Summary**
    - Name: `Generate Sync Summary`
    - Type: Code
    - Write JavaScript to summarize the sync operation (e.g., number of connections processed, timestamp).
    - Connect output from `Save Connections to Sheets`.

12. **Create Wait Node for Retry (60 seconds)**
    - Name: `Wait & Retry (60s)`
    - Type: Wait
    - Set waiting time to 60 seconds.
    - Connect "false" output from `Scrape Completed?`.
    - Connect output back to `Check Scrape Status` to form retry loop.

13. **Connect all nodes as per the described flow.**

14. **Credentials:**
    - Configure appropriate credentials for:
      - HTTP Request nodes (API keys or OAuth tokens for scraping API).
      - Google Sheets nodes (OAuth2 with access to the target spreadsheet).

15. **Testing:**
    - Test each node independently, especially API requests and Google Sheets operations.
    - Verify the retry logic by simulating incomplete scrape status.

---

### 5. General Notes & Resources

| Note Content                                                | Context or Link                                                   |
|-------------------------------------------------------------|------------------------------------------------------------------|
| Workflow automates LinkedIn connections sync using Apify API and Google Sheets integration. | Workflow purpose description                                      |
| The schedule is set to Sunday 2 AM to avoid peak LinkedIn usage hours. | Scheduling rationale                                             |
| Google Sheets credentials require OAuth2 with edit permissions on the target sheet. | Credential setup note                                             |
| For LinkedIn scraping, ensure compliance with LinkedIn's terms of service and API usage guidelines. | Compliance reminder                                              |
| No sticky notes with content were provided in the workflow JSON. | Sticky Note placeholder                                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.