Lead Enrichment Pipeline: Leadfeeder to Apollo to Google Sheets

https://n8nworkflows.xyz/workflows/lead-enrichment-pipeline--leadfeeder-to-apollo-to-google-sheets-7510


# Lead Enrichment Pipeline: Leadfeeder to Apollo to Google Sheets

### 1. Workflow Overview

This workflow automates the process of enriching lead data by integrating Leadfeeder, Apollo.io, and Google Sheets. It is designed to run daily, fetch leads from Leadfeeder using pagination, enrich these leads with detailed information from Apollo.io, and save the enriched data into Google Sheets for further analysis or use.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger and Initialization**: Starts the workflow daily at 9 AM and initiates the Leadfeeder data retrieval process.
- **1.2 Leadfeeder Data Retrieval with Pagination**: Fetches Leadfeeder account ID, generates a pagination sequence, and retrieves lead data page by page.
- **1.3 Lead Data Enrichment via Apollo.io**: Sends the extracted leads for enrichment through Apollo.io API calls.
- **1.4 Data Storage in Google Sheets**: Saves the enriched leads into Google Sheets.
- **1.5 Pagination Control and Rate Limiting**: Manages pagination loops and introduces delays to handle API rate limits.
- **1.6 Error Monitoring and Notification**: Listens for workflow errors, formats error reports, and sends alerts through Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Initialization

- **Overview:**  
  This block initiates the workflow execution daily at 9 AM and starts the process by fetching the Leadfeeder account ID.

- **Nodes Involved:**  
  - Daily Trigger (9 AM)  
  - Fetch Leadfeeder Account ID

- **Node Details:**  
  - **Daily Trigger (9 AM)**  
    - Type: Schedule Trigger  
    - Role: Triggers the workflow every day at 9 AM local time.  
    - Configuration: Default daily schedule, no additional parameters.  
    - Connections: Outputs to "Fetch Leadfeeder Account ID".  
    - Edge Cases: Workflow will not run outside scheduled time; ensure timezone settings align with intended execution time.

  - **Fetch Leadfeeder Account ID**  
    - Type: HTTP Request  
    - Role: Sends an API request to Leadfeeder to retrieve the account ID necessary for subsequent API calls.  
    - Configuration: Configured with Leadfeeder API credentials and endpoint to get account details.  
    - Input: Trigger from Daily Trigger (9 AM)  
    - Output: Passes the response data to "Generate Pagination Sequence".  
    - Edge Cases: Possible authentication errors, API downtime, or invalid response format.

#### 1.2 Leadfeeder Data Retrieval with Pagination

- **Overview:**  
  This block handles pagination for fetching leads from Leadfeeder, controlling batch sizes and page numbers to sequentially retrieve all lead data.

- **Nodes Involved:**  
  - Generate Pagination Sequence  
  - Pagination Controller  
  - Assign Current Page Number  
  - Retrieve Lead Data (Leadfeeder)  
  - Check Last Page Condition

- **Node Details:**  
  - **Generate Pagination Sequence**  
    - Type: Code  
    - Role: Generates a sequence of page numbers for pagination based on the total number of pages or a predefined limit.  
    - Configuration: Custom JavaScript code that outputs page numbers for iteration.  
    - Input: Output from "Fetch Leadfeeder Account ID"  
    - Output: Feeds the sequence into "Pagination Controller".  
    - Edge Cases: Incorrect page count calculation can cause missing data or excessive calls.

  - **Pagination Controller**  
    - Type: Split In Batches  
    - Role: Controls the batch processing of pagination sequence, handling one page (or batch) at a time.  
    - Configuration: Batch size set to 1 to process one page per iteration.  
    - Input: Pagination sequence from "Generate Pagination Sequence"  
    - Output: On batch completion, loops back or proceeds to next node "Assign Current Page Number".  
    - Edge Cases: Batch processing failure or incomplete batches can break pagination.

  - **Assign Current Page Number**  
    - Type: Set  
    - Role: Assigns the current page number from the batch to be used in the API request for lead data.  
    - Configuration: Sets a variable with the current page number extracted from the batch.  
    - Input: From "Pagination Controller"  
    - Output: Passes page number to "Retrieve Lead Data (Leadfeeder)".  
    - Edge Cases: Failure to assign page number leads to invalid API requests.

  - **Retrieve Lead Data (Leadfeeder)**  
    - Type: HTTP Request  
    - Role: Fetches lead data for the assigned page number from Leadfeeder API.  
    - Configuration: Configured with API endpoint incorporating the current page number and authentication credentials.  
    - Input: Current page number from "Assign Current Page Number"  
    - Output: Passes lead data to "Check Last Page Condition".  
    - Edge Cases: API request failures, data inconsistencies, or unexpected response structure.

  - **Check Last Page Condition**  
    - Type: If  
    - Role: Checks if the current page is the last page to branch the flow accordingly.  
    - Configuration: Condition based on page number comparison or response metadata.  
    - Input: Lead data from "Retrieve Lead Data (Leadfeeder)"  
    - Output:  
      - True branch: "Extract Leads (Last Page)"  
      - False branch: "Extract Leads (Full Page)"  
    - Edge Cases: Incorrect condition logic may cause premature termination or infinite pagination loops.

#### 1.3 Lead Data Enrichment via Apollo.io

- **Overview:**  
  This block extracts lead data from the response and sends it for enrichment through Apollo.io API calls, handling both full pages and the last page differently to optimize processing.

- **Nodes Involved:**  
  - Extract Leads (Last Page)  
  - Enrich with Apollo (Last Page)  
  - Extract Leads (Full Page)  
  - Enrich with Apollo (Full Page)

- **Node Details:**  
  - **Extract Leads (Last Page)**  
    - Type: Split Out  
    - Role: Extracts individual lead records from the last page data for processing.  
    - Input: True branch from "Check Last Page Condition"  
    - Output: Leads passed to "Enrich with Apollo (Last Page)".  
    - Edge Cases: Data extraction errors due to unexpected data format.

  - **Enrich with Apollo (Last Page)**  
    - Type: HTTP Request  
    - Role: Sends extracted leads from the last page to Apollo.io API for enrichment.  
    - Configuration: Configured with Apollo API credentials, endpoint, and appropriate payload structure.  
    - Input: Leads from "Extract Leads (Last Page)"  
    - Output: Enriched lead data to "Save Leads to Google Sheets (Last Page)".  
    - Edge Cases: API rate limiting, authentication errors, or malformed payloads.

  - **Extract Leads (Full Page)**  
    - Type: Split Out  
    - Role: Similar to the last page extractor but for all intermediate pages.  
    - Input: False branch from "Check Last Page Condition"  
    - Output: Leads passed to "Enrich with Apollo (Full Page)".  
    - Edge Cases: Same as for last page extraction.

  - **Enrich with Apollo (Full Page)**  
    - Type: HTTP Request  
    - Role: Sends full page leads to Apollo.io for enrichment.  
    - Configuration: Same as last page enrichment node.  
    - Input: Leads from "Extract Leads (Full Page)"  
    - Output: Enriched leads to "Save Leads to Google Sheets (Full Page)".  
    - Edge Cases: Same as last page enrichment.

#### 1.4 Data Storage in Google Sheets

- **Overview:**  
  This block saves the enriched leads data into designated Google Sheets documents, with separate nodes handling data from full pages and last pages.

- **Nodes Involved:**  
  - Save Leads to Google Sheets (Last Page)  
  - Save Leads to Google Sheets (Full Page)

- **Node Details:**  
  - **Save Leads to Google Sheets (Last Page)**  
    - Type: Google Sheets  
    - Role: Inserts enriched last page leads into a Google Sheets spreadsheet.  
    - Configuration: Configured with Google Sheets credentials, target spreadsheet ID, and sheet name or range.  
    - Input: Enriched data from "Enrich with Apollo (Last Page)"  
    - Output: Passes control to "Rate Limit Delay (40s)".  
    - Edge Cases: Authentication issues, quota limits, or invalid spreadsheet ranges.

  - **Save Leads to Google Sheets (Full Page)**  
    - Type: Google Sheets  
    - Role: Inserts enriched full page leads into Google Sheets.  
    - Configuration: Similar to last page node but may target a different sheet or range.  
    - Input: Enriched data from "Enrich with Apollo (Full Page)"  
    - Output: Passes control to "Rate Limit Delay (40s)".  
    - Edge Cases: Same as last page node.

#### 1.5 Pagination Control and Rate Limiting

- **Overview:**  
  This block manages the iterative pagination loop and enforces a wait period to comply with API rate limits and prevent overloading services.

- **Nodes Involved:**  
  - Rate Limit Delay (40s)  
  - Pagination Controller (revisited)

- **Node Details:**  
  - **Rate Limit Delay (40s)**  
    - Type: Wait  
    - Role: Introduces a 40-second delay between batches/pages to manage API rate limits.  
    - Configuration: Wait time set to 40 seconds.  
    - Input: Output from Google Sheets nodes  
    - Output: Loops back to "Pagination Controller" to process the next page batch.  
    - Edge Cases: Delays too short may cause rate limit errors; too long increases total workflow runtime.

  - **Pagination Controller** (Revisited)  
    - Manages the next page batch after delay, continuing the pagination cycle until all pages processed.

#### 1.6 Error Monitoring and Notification

- **Overview:**  
  Captures any errors during workflow execution, formats an error report, and sends an alert message via Telegram to notify administrators promptly.

- **Nodes Involved:**  
  - Workflow Error Listener  
  - Format Error Report  
  - Send Alert to Telegram

- **Node Details:**  
  - **Workflow Error Listener**  
    - Type: Error Trigger  
    - Role: Listens for any errors that occur in the workflow.  
    - Configuration: Default error trigger settings.  
    - Output: Passes error data to "Format Error Report".  
    - Edge Cases: May not capture errors outside workflow runtime or in external systems.

  - **Format Error Report**  
    - Type: Code  
    - Role: Formats the error details into a human-readable message.  
    - Configuration: Custom JavaScript code to extract relevant error info and build the notification text.  
    - Input: Error data from "Workflow Error Listener"  
    - Output: Formatted message to "Send Alert to Telegram".  
    - Edge Cases: Code errors or missing error details can lead to incomplete reports.

  - **Send Alert to Telegram**  
    - Type: Telegram  
    - Role: Sends the formatted error message to a predefined Telegram chat or user.  
    - Configuration: Configured with Telegram bot credentials and chat ID.  
    - Input: Formatted error message from "Format Error Report".  
    - Output: None (terminal node)  
    - Edge Cases: Telegram API downtime, incorrect chat ID, or bot permission issues.

---

### 3. Summary Table

| Node Name                         | Node Type          | Functional Role                            | Input Node(s)                          | Output Node(s)                        | Sticky Note |
|----------------------------------|--------------------|------------------------------------------|--------------------------------------|-------------------------------------|-------------|
| Daily Trigger (9 AM)              | Schedule Trigger   | Initiates daily workflow execution       |                                      | Fetch Leadfeeder Account ID          |             |
| Fetch Leadfeeder Account ID       | HTTP Request       | Retrieves Leadfeeder account ID          | Daily Trigger (9 AM)                  | Generate Pagination Sequence          |             |
| Generate Pagination Sequence      | Code               | Creates pagination sequence for leads    | Fetch Leadfeeder Account ID           | Pagination Controller                 |             |
| Pagination Controller             | Split In Batches   | Controls batch processing of pages       | Generate Pagination Sequence, Rate Limit Delay (40s) | Assign Current Page Number            |             |
| Assign Current Page Number        | Set                | Sets current page number for API call    | Pagination Controller                 | Retrieve Lead Data (Leadfeeder)       |             |
| Retrieve Lead Data (Leadfeeder)   | HTTP Request       | Fetches leads data from Leadfeeder       | Assign Current Page Number            | Check Last Page Condition             |             |
| Check Last Page Condition         | If                 | Determines if current page is last       | Retrieve Lead Data (Leadfeeder)       | Extract Leads (Last Page), Extract Leads (Full Page) |             |
| Extract Leads (Last Page)         | Split Out           | Extracts leads from last page             | Check Last Page Condition (true)     | Enrich with Apollo (Last Page)        |             |
| Enrich with Apollo (Last Page)    | HTTP Request       | Enriches last page leads via Apollo      | Extract Leads (Last Page)             | Save Leads to Google Sheets (Last Page) |             |
| Save Leads to Google Sheets (Last Page) | Google Sheets      | Saves enriched last page leads            | Enrich with Apollo (Last Page)        | Rate Limit Delay (40s)                |             |
| Extract Leads (Full Page)         | Split Out           | Extracts leads from full page             | Check Last Page Condition (false)    | Enrich with Apollo (Full Page)        |             |
| Enrich with Apollo (Full Page)    | HTTP Request       | Enriches full page leads via Apollo      | Extract Leads (Full Page)             | Save Leads to Google Sheets (Full Page) |             |
| Save Leads to Google Sheets (Full Page) | Google Sheets      | Saves enriched full page leads            | Enrich with Apollo (Full Page)        | Rate Limit Delay (40s)                |             |
| Rate Limit Delay (40s)            | Wait               | Delays execution to handle rate limits   | Save Leads to Google Sheets (Full/Last Page) | Pagination Controller                |             |
| Workflow Error Listener           | Error Trigger      | Detects workflow errors                   |                                      | Format Error Report                   |             |
| Format Error Report               | Code               | Formats error details for notification   | Workflow Error Listener               | Send Alert to Telegram                |             |
| Send Alert to Telegram            | Telegram           | Sends error alerts to Telegram            | Format Error Report                   |                                     |             |
| Sticky Note3                     | Sticky Note         |                                          |                                      |                                     |             |
| Sticky Note                      | Sticky Note         |                                          |                                      |                                     |             |
| Sticky Note1                     | Sticky Note         |                                          |                                      |                                     |             |
| Sticky Note2                     | Sticky Note         |                                          |                                      |                                     |             |
| Sticky Note4                     | Sticky Note         |                                          |                                      |                                     |             |
| Sticky Note5                     | Sticky Note         |                                          |                                      |                                     |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: "Daily Trigger (9 AM)"  
   - Type: Schedule Trigger  
   - Set to trigger daily at 9:00 AM local time.  
   - No credentials needed.

2. **Add an HTTP Request Node**  
   - Name: "Fetch Leadfeeder Account ID"  
   - Type: HTTP Request  
   - Configure with Leadfeeder API credentials (e.g., API key in headers).  
   - Set endpoint to Leadfeeder API endpoint for retrieving account ID.  
   - Method: GET  
   - Connect output of "Daily Trigger (9 AM)" to this node.

3. **Add a Code Node**  
   - Name: "Generate Pagination Sequence"  
   - Type: Code  
   - Write JavaScript to generate an array of page numbers based on total pages expected or API response.  
   - Input: Response from "Fetch Leadfeeder Account ID".  
   - Output: Array of page numbers for pagination.

4. **Add a Split In Batches Node**  
   - Name: "Pagination Controller"  
   - Type: Split In Batches  
   - Set batch size to 1 to process one page at a time.  
   - Connect input to "Generate Pagination Sequence".

5. **Add a Set Node**  
   - Name: "Assign Current Page Number"  
   - Type: Set  
   - Configure to assign the current page number from the batch item to a variable (e.g., `currentPage`).  
   - Connect input to "Pagination Controller" output.

6. **Add an HTTP Request Node**  
   - Name: "Retrieve Lead Data (Leadfeeder)"  
   - Type: HTTP Request  
   - Configure with Leadfeeder API credentials.  
   - Method: GET  
   - Endpoint includes query parameter for page number using the variable from "Assign Current Page Number".  
   - Connect input to "Assign Current Page Number".

7. **Add an If Node**  
   - Name: "Check Last Page Condition"  
   - Type: If  
   - Configure condition to check if current page number equals total pages (or last page indicator).  
   - Connect input to "Retrieve Lead Data (Leadfeeder)".

8. **Add Split Out Nodes**  
   - Name: "Extract Leads (Last Page)" and "Extract Leads (Full Page)"  
   - Type: Split Out  
   - Configure each to extract lead array from API response.  
   - Connect "Check Last Page Condition" true branch to "Extract Leads (Last Page)".  
   - Connect false branch to "Extract Leads (Full Page)".

9. **Add HTTP Request Nodes for Apollo Enrichment**  
   - Names: "Enrich with Apollo (Last Page)" and "Enrich with Apollo (Full Page)"  
   - Type: HTTP Request  
   - Configure with Apollo.io API credentials and endpoints for lead enrichment.  
   - Method: POST or as required by Apollo API.  
   - Input: Extracted leads from respective previous nodes.

10. **Add Google Sheets Nodes**  
    - Names: "Save Leads to Google Sheets (Last Page)" and "Save Leads to Google Sheets (Full Page)"  
    - Type: Google Sheets  
    - Configure with Google Sheets OAuth2 credentials.  
    - Provide spreadsheet ID and sheet name/range for lead data insertion.  
    - Connect input from respective Apollo enrichment nodes.

11. **Add a Wait Node**  
    - Name: "Rate Limit Delay (40s)"  
    - Type: Wait  
    - Set wait time to 40 seconds to manage API rate limits.  
    - Connect input from both Google Sheets nodes' outputs.

12. **Loop Back Connection**  
    - Connect output of "Rate Limit Delay (40s)" back to the "Pagination Controller" to process the next page batch.

13. **Add Error Handling Nodes**  
    - Add an Error Trigger Node named "Workflow Error Listener" to catch any workflow errors.  
    - Add a Code Node named "Format Error Report" to format error details into a message.  
    - Add a Telegram Node named "Send Alert to Telegram" to send the error report to a specified Telegram chat.  
    - Configure Telegram node with bot token and chat ID credentials.  
    - Connect "Workflow Error Listener" → "Format Error Report" → "Send Alert to Telegram".

14. **Validate all credentials** (Leadfeeder API key, Apollo.io API key, Google Sheets OAuth2, Telegram Bot token) and test API endpoints independently before running the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow requires valid API credentials for Leadfeeder, Apollo.io, Google Sheets, and Telegram Bot.         | Credential setup in n8n for each integration.   |
| To avoid hitting API rate limits, a 40-second wait has been introduced between pagination batches.               | API rate limit management best practice.        |
| Telegram alerts provide immediate notification of workflow errors, aiding in quick troubleshooting.              | Telegram Bot API documentation: https://core.telegram.org/bots/api |
| Pagination is controlled with batch size 1 to sequentially process each page, ensuring ordered data retrieval.   | n8n Split In Batches node documentation.         |
| The workflow assumes Leadfeeder API responses include pagination metadata to determine the last page.            | Leadfeeder API docs should be referenced.       |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.