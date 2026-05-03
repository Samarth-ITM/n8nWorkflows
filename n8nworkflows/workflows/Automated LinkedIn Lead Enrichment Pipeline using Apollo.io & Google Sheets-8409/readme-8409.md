Automated LinkedIn Lead Enrichment Pipeline using Apollo.io & Google Sheets

https://n8nworkflows.xyz/workflows/automated-linkedin-lead-enrichment-pipeline-using-apollo-io---google-sheets-8409


# Automated LinkedIn Lead Enrichment Pipeline using Apollo.io & Google Sheets

### 1. Workflow Overview

This n8n workflow automates a LinkedIn lead enrichment pipeline using Apollo.io APIs and Google Sheets as data storage and orchestration points. Its main purpose is to retrieve raw lead data, enrich it with company and contact information, perform AI-based data structuring, and update Google Sheets accordingly. The workflow is designed for sales and marketing teams aiming to automate lead qualification and enrichment processes with minimal manual intervention.

The workflow is logically split into the following blocks:

- **1.1 Trigger & Data Retrieval:** Periodic or webhook-triggered retrieval of lead data from Google Sheets.
- **1.2 Data Processing & Splitting:** Batch processing and splitting of input data for scalable enrichment.
- **1.3 AI-Powered Parsing & Formatting:** Use of Google Gemini AI models and LangChain nodes to parse and format LinkedIn scraped data.
- **1.4 Apollo.io Enrichment Calls:** HTTP requests to Apollo.io endpoints for company domain resolution and people search.
- **1.5 Email Finding & Final Data Updates:** Further enrichment via email finder API and updating enriched data back into Google Sheets.
- **1.6 Control Logic & Error Handling:** Conditional branches and wait nodes to control flow, handle errors gracefully, and avoid rate limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Retrieval

- **Overview:** This block initiates the workflow either through external webhook calls or scheduled triggers, retrieving lead records from Google Sheets for enrichment.
- **Nodes Involved:** `Webhook`, `Schedule Trigger`, `Schedule Trigger1`, `Schedule Trigger2`, `Get row(s) in sheet`, `Get row(s) in sheet1`, `Get row(s) in sheet3`
- **Node Details:**

  - **Webhook**
    - Type: Webhook (Trigger)
    - Role: Entry point for manual or external trigger via HTTP POST.
    - Configuration: Default webhook with unique ID; no parameters set.
    - Inputs: External HTTP request.
    - Outputs: Passes data to `Split Out`.
    - Edge Cases: Missing or malformed webhook payloads may cause downstream failures.

  - **Schedule Trigger, Schedule Trigger1, Schedule Trigger2**
    - Type: Schedule Trigger nodes
    - Role: Periodically trigger workflows for different datasets or sheets.
    - Configuration: Use cron or interval schedules (not detailed), each mapped to distinct Google Sheets.
    - Inputs: Timed triggers.
    - Outputs: Connect to respective `Get row(s) in sheet` nodes.
    - Edge Cases: Missed schedules or time zone mismatches could delay processing.

  - **Get row(s) in sheet, Get row(s) in sheet1, Get row(s) in sheet3**
    - Type: Google Sheets - Read rows
    - Role: Fetch lead data rows from designated Google Sheets.
    - Configuration: Sheet ID and range configured per use case (not explicitly specified).
    - Inputs: From schedule triggers.
    - Outputs: To conditional `If` nodes for filtering.
    - Edge Cases: API quota limits, empty sheets, or permission errors.

#### 2.2 Data Processing & Splitting

- **Overview:** This block splits the incoming data into manageable batches or individual items to process asynchronously and efficiently.
- **Nodes Involved:** `Split Out`, `Loop Over Items`, `Loop Over Items1`, `Loop Over Items2`, `Loop Over Items4`
- **Node Details:**

  - **Split Out**
    - Type: Split Out
    - Role: Splits webhook input data into individual items.
    - Configuration: Default.
    - Inputs: From `Webhook`.
    - Outputs: To `Loop Over Items`.
    - Edge Cases: Empty incoming data or unexpected structure.

  - **Loop Over Items, Loop Over Items1, Loop Over Items2, Loop Over Items4**
    - Type: SplitInBatches
    - Role: Batch processing for scalable handling of large datasets.
    - Configuration: Batch size not explicitly stated; error handling set to continue on some nodes.
    - Inputs/Outputs: Sequentially connected to enrichment and processing nodes.
    - Edge Cases: Batch size too large causing rate limits; error propagation.
  
#### 2.3 AI-Powered Parsing & Formatting

- **Overview:** Uses Google Gemini chat model and LangChain nodes to parse unstructured LinkedIn data into structured formats suitable for system ingestion.
- **Nodes Involved:** `Google Gemini Chat Model`, `Structured Output Parser`, `Linkedin Scraped Formater`
- **Node Details:**

  - **Google Gemini Chat Model**
    - Type: LangChain LM Chat (Google Gemini)
    - Role: AI model providing natural language processing for raw LinkedIn data.
    - Inputs: Batched lead data.
    - Outputs: Parsed output to `Structured Output Parser`.
    - Edge Cases: API timeouts, model quota limits.

  - **Structured Output Parser**
    - Type: LangChain Output Parser Structured
    - Role: Converts AI chat responses into structured JSON objects.
    - Inputs: From Google Gemini Chat Model.
    - Outputs: To `Linkedin Scraped Formater`.
    - Edge Cases: Parsing errors if AI output is malformed.

  - **Linkedin Scraped Formater**
    - Type: LangChain Agent
    - Role: Formats enriched LinkedIn data for downstream storage.
    - Inputs: Structured parsed output.
    - Outputs: To `Store values` (Google Sheets) and back to `Loop Over Items` for further processing.
    - Error Handling: Configured to continue on error.
    - Edge Cases: Unexpected formatted data or incomplete fields.

#### 2.4 Apollo.io Enrichment Calls

- **Overview:** This block enriches leads by resolving company domains and performing people searches via Apollo.io APIs.
- **Nodes Involved:** `Company Name to Domain`, `People Search`
- **Node Details:**

  - **Company Name to Domain**
    - Type: HTTP Request
    - Role: Resolves company name to domain via Apollo.io API.
    - Configuration: API endpoint configured with necessary authentication headers (OAuth2 or API Key).
    - Inputs: From `Loop Over Items1`.
    - Outputs: To `Update row in sheet`.
    - Edge Cases: API authentication failure, rate limits, or no domain found.
  
  - **People Search**
    - Type: HTTP Request
    - Role: Searches for people associated with the company domain.
    - Configuration: Uses Apollo.io people search endpoint with query parameters.
    - Inputs: From `Loop Over Items2`.
    - Outputs: To `Wait1` node.
    - Error Handling: Continues on regular output even on errors.
    - Edge Cases: Empty search results or API errors.

#### 2.5 Email Finding & Final Data Updates

- **Overview:** This block attempts to find emails for leads and updates all enriched data back into the relevant Google Sheets.
- **Nodes Involved:** `Email Finder`, `Update row in sheet`, `Update row in sheet1`, `Update row in sheet2`, `Update row in sheet3`, `Store values`
- **Node Details:**

  - **Email Finder**
    - Type: HTTP Request
    - Role: Calls an email finder API to retrieve emails for leads.
    - Inputs: From `Code` node after batch processing.
    - Outputs: To `Code1` node.
    - Error Handling: Continues on error.
    - Edge Cases: Email not found, API limits.

  - **Update row in sheet, Update row in sheet1, Update row in sheet2, Update row in sheet3**
    - Type: Google Sheets - Update row
    - Role: Writes enriched data back to Google Sheets.
    - Inputs: Connected from wait nodes or batch loops.
    - Error Handling: Some nodes configured to continue on error to avoid workflow halt.
    - Edge Cases: Row locking, concurrent updates, API rate limits.

  - **Store values**
    - Type: Google Sheets - Append or Update
    - Role: Stores intermediate formatted data during processing.
    - Inputs: From `Linkedin Scraped Formater`.
    - Outputs: To `Loop Over Items`.
    - Edge Cases: Google Sheets API limits.

#### 2.6 Control Logic & Error Handling

- **Overview:** Contains conditional nodes and wait nodes to orchestrate processing flow, error handling, and pacing API calls to avoid rate limiting.
- **Nodes Involved:** `If`, `If1`, `If2`, `Wait`, `Wait1`, `Wait2`, `Code`, `Code1`
- **Node Details:**

  - **If, If1, If2**
    - Type: If (Conditional)
    - Role: Branching logic to decide if enrichment steps should proceed based on criteria (e.g., presence of data or status flags).
    - Inputs: From Google Sheets data retrieval.
    - Outputs: To different batch processing loops.
    - Edge Cases: Misconfigured conditions can cause data to be skipped or processed unnecessarily.

  - **Wait, Wait1, Wait2**
    - Type: Wait
    - Role: Pauses workflow execution to control processing rate and avoid API throttling.
    - Inputs/Outputs: Between update nodes and loop nodes.
    - Edge Cases: Long wait times may delay overall processing; improper wait duration may cause API errors.

  - **Code, Code1**
    - Type: Code (JavaScript)
    - Role: Custom logic for data transformation or conditional checks.
    - Inputs/Outputs: Interleaved between HTTP requests and waits.
    - Edge Cases: Syntax errors or runtime exceptions; configured to continue on error in some cases.

---

### 3. Summary Table

| Node Name                 | Node Type                  | Functional Role                          | Input Node(s)                 | Output Node(s)                | Sticky Note                  |
|---------------------------|----------------------------|----------------------------------------|------------------------------|------------------------------|------------------------------|
| Webhook                   | Webhook                    | Entry point for external trigger       | -                            | Split Out                    |                              |
| Split Out                 | Split Out                  | Split webhook data into items          | Webhook                      | Loop Over Items              |                              |
| Loop Over Items           | SplitInBatches             | Batch processing of items               | Split Out, Linkedin Scraped Formater (secondary) | Linkedin Scraped Formater, Store values |                              |
| Google Gemini Chat Model  | LangChain LM Chat          | AI model for parsing LinkedIn data     | Linkedin Scraped Formater     | Structured Output Parser      |                              |
| Structured Output Parser  | LangChain Output Parser    | Parse AI output into structured format | Google Gemini Chat Model      | Linkedin Scraped Formater     |                              |
| Linkedin Scraped Formater | LangChain Agent            | Format parsed data for storage          | Loop Over Items              | Store values, Loop Over Items |                              |
| Store values              | Google Sheets              | Store formatted data into Google Sheets | Linkedin Scraped Formater     | Loop Over Items              |                              |
| Schedule Trigger          | Schedule Trigger           | Timed trigger for sheet data retrieval | -                            | Get row(s) in sheet          |                              |
| Get row(s) in sheet       | Google Sheets              | Read lead data rows                     | Schedule Trigger             | If1                         |                              |
| If1                       | If                         | Condition check on sheet data           | Get row(s) in sheet          | Loop Over Items1             |                              |
| Loop Over Items1          | SplitInBatches             | Batch processing for company domain    | If1                         | Company Name to Domain       |                              |
| Company Name to Domain    | HTTP Request               | Apollo.io API call to get domain       | Loop Over Items1             | Update row in sheet          |                              |
| Update row in sheet       | Google Sheets              | Update sheet with domain data           | Company Name to Domain       | Wait                        |                              |
| Wait                      | Wait                       | Pause to avoid rate limit                | Update row in sheet          | Loop Over Items1             |                              |
| Schedule Trigger1         | Schedule Trigger           | Timed trigger for sheet1 data           | -                            | Get row(s) in sheet1         |                              |
| Get row(s) in sheet1      | Google Sheets              | Read lead data for people search        | Schedule Trigger1            | If                         |                              |
| If                        | If                         | Condition check on sheet1 data           | Get row(s) in sheet1         | Loop Over Items2             |                              |
| Loop Over Items2          | SplitInBatches             | Batch processing for people search      | If                         | People Search               |                              |
| People Search             | HTTP Request               | Apollo.io people search API call        | Loop Over Items2             | Wait1                       |                              |
| Wait1                     | Wait                       | Pause after people search API call      | People Search               | Update row in sheet1         |                              |
| Update row in sheet1      | Google Sheets              | Update sheet1 with people search results| Wait1                       | Loop Over Items2             |                              |
| Schedule Trigger2         | Schedule Trigger           | Timed trigger for email finder processing| -                            | Get row(s) in sheet3         |                              |
| Get row(s) in sheet3      | Google Sheets              | Read lead data for email finder          | Schedule Trigger2            | If2                        |                              |
| If2                       | If                         | Condition check on sheet3 data            | Get row(s) in sheet3         | Loop Over Items4             |                              |
| Loop Over Items4          | SplitInBatches             | Batch processing for email finder         | If2                        | Code                       |                              |
| Code                      | Code                       | Custom logic for email finder processing | Loop Over Items4             | Email Finder                |                              |
| Email Finder              | HTTP Request               | Email finder API call                      | Code                       | Code1                      |                              |
| Code1                     | Code                       | Post-processing after email finder         | Email Finder                | Wait2                      |                              |
| Wait2                     | Wait                       | Pause before final update                  | Code1                      | Update row in sheet2        |                              |
| Update row in sheet2      | Google Sheets              | Update sheet2 with email finder data       | Wait2                      | Loop Over Items4            |                              |
| Linkedin Scraped Formater  | LangChain Agent            | Format LinkedIn data                        | Loop Over Items             | Store values, Loop Over Items|                              |
| Store values              | Google Sheets              | Save formatted LinkedIn data                | Linkedin Scraped Formater    | Loop Over Items             |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node**
   - Type: Webhook (Trigger)
   - Set method to POST.
   - Save webhook URL for external triggering.

2. **Add Split Out Node**
   - Connect from Webhook.
   - Default settings to split incoming array data into individual items.

3. **Add Loop Over Items Node**
   - Type: SplitInBatches.
   - Connect from Split Out node.
   - Configure batch size (default or based on expected input size).

4. **Add Google Gemini Chat Model Node**
   - Type: LangChain LM Chat (Google Gemini).
   - Connect from Loop Over Items.
   - Configure API credentials for Google Gemini.
   - Set prompt or input to process LinkedIn raw data.

5. **Add Structured Output Parser Node**
   - Type: LangChain Output Parser Structured.
   - Connect from Google Gemini Chat Model.
   - Configure expected output schema for parsing.

6. **Add Linkedin Scraped Formater Node**
   - Type: LangChain Agent.
   - Connect from Structured Output Parser.
   - Set instructions to format data as needed.

7. **Add Store Values Node**
   - Type: Google Sheets (Append or Update).
   - Connect from Linkedin Scraped Formater.
   - Configure Google Sheets credentials, sheet ID, and target range/columns.

8. **Add Schedule Trigger Nodes (x3)**
   - Create three Schedule Triggers for different time intervals or datasets.
   - Connect each to corresponding Get row(s) in sheet nodes.

9. **Add Get Row(s) in Sheet Nodes (x3)**
   - Configure each to read from different sheets or ranges.
   - Connect each to respective If conditional node.

10. **Add If Nodes (x3)**
    - Configure conditions to filter rows needing enrichment.
    - Connect outputs to respective Loop Over Items nodes for batch processing.

11. **Add Loop Over Items1, Loop Over Items2, Loop Over Items4 Nodes**
    - Each a SplitInBatches node connected from respective If nodes.
    - Configure batch sizes and error handling (continue on error).

12. **Add Company Name to Domain Node**
    - Type: HTTP Request.
    - Connect from Loop Over Items1.
    - Configure Apollo.io company domain API endpoint.
    - Set OAuth2 or API key credentials.

13. **Add Update Row in Sheet Node**
    - Connect from Company Name to Domain.
    - Configure to update rows in Google Sheets with domain info.

14. **Add Wait Node**
    - Connect from Update Row in Sheet.
    - Configure appropriate delay to avoid API rate limiting.
    - Connect back to Loop Over Items1 to continue processing.

15. **Add People Search Node**
    - Type: HTTP Request.
    - Connect from Loop Over Items2.
    - Configure Apollo.io people search API endpoint.

16. **Add Wait1 Node**
    - Connect from People Search.
    - Configure delay as per API rate limits.

17. **Add Update Row in Sheet1 Node**
    - Connect from Wait1.
    - Update Google Sheets with people search results.

18. **Add Email Finder Node**
    - Type: HTTP Request.
    - Connect from custom Code node.
    - Configure email finder API (e.g., Apollo.io or other provider).

19. **Add Code Nodes**
    - Add two Code nodes for custom data transformation between HTTP requests and wait nodes.
    - Connect outputs appropriately.

20. **Add Wait2 Node**
    - Connect from Code1.
    - Configure delay before final update.

21. **Add Update Row in Sheet2 Node**
    - Connect from Wait2.
    - Update Google Sheets with email finder data.

22. **Configure Credentials**
    - Set up Google Sheets OAuth2 credentials.
    - Set up Apollo.io API credentials or OAuth2.
    - Set up Google Gemini API credentials for LangChain LM Chat.

23. **Validate Node Connections**
    - Ensure all nodes are connected in proper sequence as per outlined dependencies.
    - Confirm error handling configurations (continue on error where appropriate).

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow relies on Google Gemini API integrated via LangChain nodes for advanced AI-based parsing and formatting.       | https://developers.google.com/ai/gemini                                                                    |
| Apollo.io API credentials must be configured separately with appropriate scopes for company and people search endpoints.    | https://apollo.io/api                                                                                 |
| Google Sheets API requires OAuth2 credentials configured in n8n for reading and updating sheets.                             | https://developers.google.com/sheets/api                                                              |
| Error handling with "continue on error" is used to ensure partial failures do not stop the entire workflow.                  | n8n documentation on error workflows                                                                |
| Consider API rate limits for Apollo.io and Google Gemini; wait nodes are introduced to space out requests accordingly.       |                                                                                                    |

---

Disclaimer: The provided description and analysis are derived exclusively from the supplied n8n workflow JSON. The workflow follows content policies strictly and handles only legal, public data.