Automated Lead Research – From LinkedIn to Ready-to-Send Report

https://n8nworkflows.xyz/workflows/automated-lead-research---from-linkedin-to-ready-to-send-report-4191


# Automated Lead Research – From LinkedIn to Ready-to-Send Report

### 1. Workflow Overview

This workflow automates lead research by extracting prospect data from LinkedIn profiles and generating a ready-to-send report. It is designed for sales, marketing, or business development teams aiming to streamline the collection, analysis, and presentation of lead intelligence. The workflow orchestrates data retrieval from Google Sheets, queries to LinkedIn accounts (personal and company), AI-powered enrichment and analysis, competitor research through Perplexity AI, and final report generation in Google Docs and Sheets.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception and Data Preparation:** Initiates the workflow manually, fetches prospect data from Google Sheets, and prepares it for processing.
- **1.2 Batch Processing and LinkedIn Data Retrieval:** Processes prospect items in batches, queries LinkedIn API endpoints (personal and company accounts), and applies wait times for API rate limiting.
- **1.3 AI-Driven Research and Validation:** Utilizes OpenAI chat models and Perplexity AI to research and validate prospect information, including competitor analysis.
- **1.4 Data Aggregation and Analysis:** Merges data from multiple sources, parses structured AI outputs, and performs prospect analysis.
- **1.5 Report Generation and Output:** Generates a comprehensive report using AI, creates Google Docs, appends content, and updates Google Sheets with final data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Data Preparation

**Overview:**  
Starts the workflow on manual trigger, retrieves prospect data from Google Sheets, and sets it up for batch processing.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Get All Data (Google Sheets)  
- Set Data (Set)  
- Loop Over Items (SplitInBatches)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow during testing or execution.  
  - Configuration: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Connects to “Get All Data”.  
  - Edge Cases: None specific; manual start required.

- **Get All Data**  
  - Type: Google Sheets (Read)  
  - Role: Fetches all prospect data rows from a configured Google Sheet.  
  - Configuration: Reads spreadsheet and worksheet containing lead data; likely configured with appropriate credentials.  
  - Inputs: Trigger from manual node.  
  - Outputs: Provides data items for further processing.  
  - Edge Cases: Authentication issues, empty sheet, API limits.

- **Set Data**  
  - Type: Set  
  - Role: Prepares or transforms retrieved data for batch processing.  
  - Configuration: Likely sets or maps specific fields for the next steps.  
  - Inputs: Data from “Get All Data”.  
  - Outputs: Passes data to “Loop Over Items”.  
  - Edge Cases: Data mapping errors if sheet format changes.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes prospect data in manageable batches to avoid API rate limits or large payloads.  
  - Configuration: Default or configured batch size.  
  - Inputs: Data from “Set Data”.  
  - Outputs: Feeds batch items into LinkedIn API calls and Perplexity requests.  
  - Edge Cases: Batch size too large causing timeouts; empty batches.

---

#### 1.2 Batch Processing and LinkedIn Data Retrieval

**Overview:**  
For each batch, sends requests to both personal and company LinkedIn endpoints, manages API rate limits with wait nodes, and sets results for further processing.

**Nodes Involved:**  
- Personal LinkedIn Account POST (HTTP Request)  
- Wait (Wait)  
- Personal LinkedIn Account GET (HTTP Request)  
- Set Results Personal (Set)  
- Company LinkedIn Account POST (HTTP Request)  
- Wait1 (Wait)  
- Company LinkedIn Account GET (HTTP Request)  
- Set Results Company (Set)

**Node Details:**

- **Personal LinkedIn Account POST**  
  - Type: HTTP Request  
  - Role: Submits POST request to LinkedIn API for personal account data retrieval.  
  - Configuration: Endpoint URL, authentication (likely OAuth2), headers, and payload based on batch item data.  
  - Inputs: From “Loop Over Items”.  
  - Outputs: Triggers “Wait” node.  
  - Edge Cases: API authentication failures, rate limits, invalid payloads.

- **Wait**  
  - Type: Wait  
  - Role: Introduces delay to comply with LinkedIn API rate limits or processing delays.  
  - Configuration: Duration or webhook-based wait.  
  - Inputs: From POST request.  
  - Outputs: Triggers “Personal LinkedIn Account GET”.  
  - Edge Cases: Timeout or webhook failure.

- **Personal LinkedIn Account GET**  
  - Type: HTTP Request  
  - Role: Retrieves personal LinkedIn data after POST request processing.  
  - Configuration: Endpoint URL, authentication, query parameters referencing POST response.  
  - Inputs: From “Wait”.  
  - Outputs: Feeds into “Set Results Personal”.  
  - Edge Cases: API errors, missing data, timeout.

- **Set Results Personal**  
  - Type: Set  
  - Role: Stores or formats personal LinkedIn data for merging.  
  - Configuration: Maps GET response data fields.  
  - Inputs: From “Personal LinkedIn Account GET”.  
  - Outputs: Connected to “Merge” node.  
  - Edge Cases: Data inconsistencies.

- **Company LinkedIn Account POST**  
  - Type: HTTP Request  
  - Role: Submits POST request to LinkedIn API for company account data retrieval.  
  - Configuration: Similar to personal POST but targeting company data endpoints.  
  - Inputs: From batch processing.  
  - Outputs: Triggers “Wait1” node.  
  - Edge Cases: Same as personal POST.

- **Wait1**  
  - Type: Wait  
  - Role: Delay for company LinkedIn API processing.  
  - Configuration: Duration or webhook-based.  
  - Inputs: From company POST.  
  - Outputs: Triggers “Company LinkedIn Account GET”.  
  - Edge Cases: Same as previous wait.

- **Company LinkedIn Account GET**  
  - Type: HTTP Request  
  - Role: Retrieves company LinkedIn data after POST request.  
  - Configuration: Endpoint, authentication, parameters.  
  - Inputs: From “Wait1”.  
  - Outputs: Feeds into “Set Results Company”.  
  - Edge Cases: Missing data, API errors.

- **Set Results Company**  
  - Type: Set  
  - Role: Stores or formats company LinkedIn data for merging.  
  - Configuration: Maps GET response data fields.  
  - Inputs: From “Company LinkedIn Account GET”.  
  - Outputs: Connects to “Merge” node.  
  - Edge Cases: Data inconsistencies.

---

#### 1.3 AI-Driven Research and Validation

**Overview:**  
Validates and enriches prospect data using Perplexity AI and OpenAI chat models. It checks data quality, performs competitor research, and generates insights.

**Nodes Involved:**  
- Perplexity Search (HTTP Request)  
- check perplexity research (Chain LLM)  
- If (Conditional)  
- Set if TRUE (Set)  
- Set if WRONG (Set)  
- OpenAI Chat Model2 (OpenAI Chat)  
- OpenAI Chat Model (OpenAI Chat)  
- Structured Output Parser (Output Parser)  
- Analyst of prospect (Chain LLM)  
- Perplexity Competitor Research (HTTP Request)  

**Node Details:**

- **Perplexity Search**  
  - Type: HTTP Request  
  - Role: Queries Perplexity AI for additional research on the prospect.  
  - Configuration: API endpoint for Perplexity, authorization, search parameters.  
  - Inputs: From batch processing.  
  - Outputs: Passed to “check perplexity research”.  
  - Edge Cases: Network errors, API limits, malformed queries.  
  - Additional: Retries on failure with 5-second intervals.

- **check perplexity research**  
  - Type: Chain LLM (LangChain)  
  - Role: Analyzes Perplexity search results for validity or insights using AI.  
  - Configuration: Custom LangChain chain with prompt templates.  
  - Inputs: From Perplexity Search and OpenAI Chat Model2.  
  - Outputs: Conditional “If” node.  
  - Edge Cases: AI model response errors, timeout.

- **OpenAI Chat Model2**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Supports “check perplexity research” with AI-generated completions.  
  - Configuration: Model (likely GPT-4 or similar), prompt context.  
  - Inputs: From batch items.  
  - Outputs: Feeds into “check perplexity research”.  
  - Edge Cases: API quota, auth failures.

- **If**  
  - Type: Conditional  
  - Role: Branches workflow based on evaluation of Perplexity research quality.  
  - Configuration: Expression evaluating AI response for TRUE or FALSE conditions.  
  - Inputs: From “check perplexity research”.  
  - Outputs: “Set if TRUE” or “Set if WRONG”.  
  - Edge Cases: Expression failures, unexpected data.

- **Set if TRUE**  
  - Type: Set  
  - Role: Sets variables or flags when research is validated positively.  
  - Inputs: From “If” TRUE branch.  
  - Outputs: Connects to “Merge”.  
  - Edge Cases: None specific.

- **Set if WRONG**  
  - Type: Set  
  - Role: Sets variables or flags when research is invalid or questionable.  
  - Inputs: From “If” FALSE branch.  
  - Outputs: Connects to “Merge”.  
  - Edge Cases: None specific.

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Provides AI completions for prospect analysis.  
  - Configuration: Model and prompt configured to analyze merged data.  
  - Inputs: From “Merge”.  
  - Outputs: Feeds “Analyst of prospect”.  
  - Edge Cases: API errors, invalid prompts.

- **Structured Output Parser**  
  - Type: Output Parser (LangChain)  
  - Role: Parses AI output into structured data formats for further processing.  
  - Configuration: Parsing schema defined in LangChain.  
  - Inputs: From “OpenAI Chat Model”.  
  - Outputs: To “Analyst of prospect”.  
  - Edge Cases: Parsing errors if AI output format deviates.

- **Analyst of prospect**  
  - Type: Chain LLM (LangChain)  
  - Role: Deep analysis of prospect data combining LinkedIn, Perplexity, and AI insights.  
  - Configuration: Complex LangChain chain with multiple prompts and logic.  
  - Inputs: From “Structured Output Parser” and “OpenAI Chat Model”.  
  - Outputs: Triggers “Perplexity Competitor Research”.  
  - Edge Cases: AI model failures, timeout.

- **Perplexity Competitor Research**  
  - Type: HTTP Request  
  - Role: Performs competitor research queries on Perplexity AI based on prospect analysis.  
  - Configuration: API endpoint, auth, competitor query parameters.  
  - Inputs: From “Analyst of prospect”.  
  - Outputs: Feeds into “Generate Report”.  
  - Edge Cases: Network or API failures, retries on fail with 5 seconds delay.

---

#### 1.4 Data Aggregation and Analysis

**Overview:**  
Merges all data streams (personal, company, AI research) and prepares consolidated input for report generation.

**Nodes Involved:**  
- Merge (Merge)

**Node Details:**

- **Merge**  
  - Type: Merge  
  - Role: Combines personal LinkedIn data, company data, and AI research results into a single unified dataset.  
  - Configuration: Default merge mode (likely ‘Wait for all’ or ‘Merge by index’).  
  - Inputs: From “Set if TRUE”, “Set if WRONG”, “Set Results Personal”, and “Set Results Company”.  
  - Outputs: Feeds into “OpenAI Chat Model” for analysis.  
  - Edge Cases: Missing inputs leading to incomplete merge.

---

#### 1.5 Report Generation and Output

**Overview:**  
Generates the final prospect report using AI, creates and populates Google Docs, and updates Google Sheets with the report data.

**Nodes Involved:**  
- Generate Report (Chain LLM)  
- OpenAI Chat Model1 (OpenAI Chat)  
- Create doc (Google Docs)  
- Add report to doc (Google Docs)  
- Add to sheets (Google Sheets)

**Node Details:**

- **Generate Report**  
  - Type: Chain LLM (LangChain)  
  - Role: Generates a comprehensive textual report based on competitor research and prospect analysis.  
  - Configuration: LangChain chain with report template and completion parameters.  
  - Inputs: From “Perplexity Competitor Research”.  
  - Outputs: Triggers “Create doc”.  
  - Edge Cases: AI model failures, token limits.

- **OpenAI Chat Model1**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Provides language model completions to assist report generation.  
  - Configuration: Model, prompt set for report writing.  
  - Inputs: Feeds into “Generate Report”.  
  - Outputs: To “Generate Report”.  
  - Edge Cases: API limits, auth errors.

- **Create doc**  
  - Type: Google Docs  
  - Role: Creates a new Google Document to store the report.  
  - Configuration: Document title, folder, and optional metadata.  
  - Inputs: From “Generate Report”.  
  - Outputs: Connects to “Add report to doc”.  
  - Edge Cases: Google API quota, permissions.

- **Add report to doc**  
  - Type: Google Docs  
  - Role: Appends the generated report content into the created Google Doc.  
  - Configuration: Uses document ID from “Create doc” and content from AI nodes.  
  - Inputs: From “Create doc”.  
  - Outputs: Feeds “Add to sheets”.  
  - Edge Cases: Document write errors.

- **Add to sheets**  
  - Type: Google Sheets  
  - Role: Updates a Google Sheet with the final report data and metadata (e.g., document link).  
  - Configuration: Spreadsheet and worksheet, row/column mapping, authentication.  
  - Inputs: From “Add report to doc”.  
  - Outputs: Loops back to “Loop Over Items” for next batch.  
  - Edge Cases: Sheet write errors, quota exceeded.

---

### 3. Summary Table

| Node Name                       | Node Type                         | Functional Role                        | Input Node(s)                        | Output Node(s)                            | Sticky Note                              |
|--------------------------------|----------------------------------|-------------------------------------|------------------------------------|------------------------------------------|-----------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                   | Workflow entry point                 | None                               | Get All Data                             |                                         |
| Get All Data                   | Google Sheets                    | Fetch prospect data                  | When clicking ‘Test workflow’       | Set Data                                |                                         |
| Set Data                      | Set                             | Prepare data for batching            | Get All Data                      | Loop Over Items                          |                                         |
| Loop Over Items               | SplitInBatches                  | Batch processing of prospects       | Set Data                         | Personal LinkedIn Account POST, Company LinkedIn Account POST, Perplexity Search |                                         |
| Personal LinkedIn Account POST | HTTP Request                    | Submit personal LinkedIn POST       | Loop Over Items                   | Wait                                    |                                         |
| Wait                         | Wait                            | Delay for LinkedIn API               | Personal LinkedIn Account POST    | Personal LinkedIn Account GET            |                                         |
| Personal LinkedIn Account GET  | HTTP Request                    | Retrieve personal LinkedIn data     | Wait                             | Set Results Personal                     |                                         |
| Set Results Personal           | Set                             | Store personal LinkedIn results     | Personal LinkedIn Account GET     | Merge                                   |                                         |
| Company LinkedIn Account POST  | HTTP Request                    | Submit company LinkedIn POST        | Loop Over Items                   | Wait1                                   |                                         |
| Wait1                        | Wait                            | Delay for company LinkedIn API      | Company LinkedIn Account POST     | Company LinkedIn Account GET             |                                         |
| Company LinkedIn Account GET   | HTTP Request                    | Retrieve company LinkedIn data      | Wait1                            | Set Results Company                      |                                         |
| Set Results Company            | Set                             | Store company LinkedIn results      | Company LinkedIn Account GET      | Merge                                   |                                         |
| Perplexity Search             | HTTP Request                    | Query Perplexity AI for research    | Loop Over Items                   | check perplexity research                 |                                         |
| OpenAI Chat Model2            | OpenAI Chat Model (LangChain)   | AI completions for Perplexity check | Loop Over Items                   | check perplexity research                 |                                         |
| check perplexity research      | Chain LLM (LangChain)            | Analyze Perplexity results           | Perplexity Search, OpenAI Chat Model2 | If                                   |                                         |
| If                           | Conditional                    | Branch on Perplexity research result | check perplexity research         | Set if TRUE, Set if WRONG                 |                                         |
| Set if TRUE                  | Set                             | Handle valid research flag           | If (TRUE branch)                 | Merge                                   |                                         |
| Set if WRONG                 | Set                             | Handle invalid research flag         | If (FALSE branch)                | Merge                                   |                                         |
| Merge                        | Merge                           | Combine personal, company, AI data   | Set if TRUE, Set if WRONG, Set Results Personal, Set Results Company | OpenAI Chat Model               |                                         |
| OpenAI Chat Model            | OpenAI Chat Model (LangChain)   | AI analysis of combined data         | Merge                           | Structured Output Parser                  |                                         |
| Structured Output Parser     | Output Parser (LangChain)       | Parse AI output into structured data | OpenAI Chat Model               | Analyst of prospect                       |                                         |
| Analyst of prospect          | Chain LLM (LangChain)            | Deep prospect analysis                | Structured Output Parser, OpenAI Chat Model | Perplexity Competitor Research        |                                         |
| Perplexity Competitor Research | HTTP Request                   | Competitor research using Perplexity | Analyst of prospect             | Generate Report                          |                                         |
| OpenAI Chat Model1           | OpenAI Chat Model (LangChain)   | AI completions for report generation | Generate Report                 | Generate Report                          |                                         |
| Generate Report              | Chain LLM (LangChain)            | Generate textual prospect report     | Perplexity Competitor Research   | Create doc                              |                                         |
| Create doc                   | Google Docs                     | Create new Google Doc for report     | Generate Report                 | Add report to doc                        |                                         |
| Add report to doc            | Google Docs                     | Append report content to Google Doc  | Create doc                     | Add to sheets                           |                                         |
| Add to sheets                | Google Sheets                   | Update Google Sheet with report data | Add report to doc              | Loop Over Items                         |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Manual start of workflow.

2. **Add Google Sheets Node to Read Data**  
   - Name: `Get All Data`  
   - Operation: Read rows from configured spreadsheet and worksheet containing prospect data.  
   - Credentials: Google Sheets credentials with read permissions.  
   - Connect output from manual trigger.

3. **Add Set Node to Prepare Data**  
   - Name: `Set Data`  
   - Configure to map or clean data fields as needed for LinkedIn API calls.  
   - Connect input from `Get All Data`.

4. **Add SplitInBatches Node**  
   - Name: `Loop Over Items`  
   - Set batch size to a reasonable number (e.g., 5-10) to manage API rate limits.  
   - Connect input from `Set Data`.

5. **Add HTTP Request Node for Personal LinkedIn POST**  
   - Name: `Personal LinkedIn Account POST`  
   - Configure API endpoint for LinkedIn personal data POST request.  
   - Set headers, authentication (OAuth2), and body from batch item data.  
   - Connect input from `Loop Over Items`.

6. **Add Wait Node**  
   - Name: `Wait`  
   - Set fixed wait time or webhook wait to ensure LinkedIn API data availability.  
   - Connect input from `Personal LinkedIn Account POST`.

7. **Add HTTP Request Node for Personal LinkedIn GET**  
   - Name: `Personal LinkedIn Account GET`  
   - Configure API endpoint to retrieve personal LinkedIn data post-processing.  
   - Connect input from `Wait`.

8. **Add Set Node to Store Personal Results**  
   - Name: `Set Results Personal`  
   - Map relevant fields from GET response for further merging.  
   - Connect input from `Personal LinkedIn Account GET`.

9. **Repeat steps 5-8 for Company LinkedIn Account**  
   - Nodes: `Company LinkedIn Account POST`, `Wait1`, `Company LinkedIn Account GET`, `Set Results Company`.  
   - Configure endpoints and auth for company LinkedIn data.  
   - Connect inputs/outputs accordingly.

10. **Add HTTP Request Node for Perplexity Search**  
    - Name: `Perplexity Search`  
    - Configure Perplexity AI API endpoint, authorization, and query based on batch items.  
    - Enable retry on failure with 5-second delay.  
    - Connect input from `Loop Over Items`.

11. **Add OpenAI Chat Model Node for Perplexity Check**  
    - Name: `OpenAI Chat Model2`  
    - Configure with appropriate OpenAI credentials, model (e.g., GPT-4), and prompt for Perplexity data validation.  
    - Connect input from `Loop Over Items`.

12. **Add Chain LLM Node for Perplexity Research Check**  
    - Name: `check perplexity research`  
    - Configure LangChain with prompt chain to analyze Perplexity results and OpenAI completions.  
    - Connect inputs from `Perplexity Search` and `OpenAI Chat Model2`.

13. **Add Conditional Node**  
    - Name: `If`  
    - Configure expression to branch based on research validity (TRUE/FALSE).  
    - Connect input from `check perplexity research`.

14. **Add Set Nodes for TRUE/FALSE branches**  
    - Names: `Set if TRUE`, `Set if WRONG`  
    - Configure each to set flags or data accordingly.  
    - Connect from corresponding `If` branches.

15. **Add Merge Node**  
    - Name: `Merge`  
    - Configure to combine data from `Set if TRUE`, `Set if WRONG`, `Set Results Personal`, and `Set Results Company`.  
    - Connect inputs accordingly.

16. **Add OpenAI Chat Model Node for Analysis**  
    - Name: `OpenAI Chat Model`  
    - Configure LangChain with prompt to analyze merged data.  
    - Connect input from `Merge`.

17. **Add Structured Output Parser**  
    - Name: `Structured Output Parser`  
    - Configure parser to convert AI output into structured JSON or similar format.  
    - Connect input from `OpenAI Chat Model`.

18. **Add Chain LLM Node for Prospect Analysis**  
    - Name: `Analyst of prospect`  
    - Configure LangChain chain with prompts to deeply analyze prospect data.  
    - Connect input from `Structured Output Parser` and `OpenAI Chat Model`.

19. **Add HTTP Request Node for Perplexity Competitor Research**  
    - Name: `Perplexity Competitor Research`  
    - Configure Perplexity AI endpoint to research competitors based on analysis.  
    - Enable retry on fail with 5-second delay.  
    - Connect input from `Analyst of prospect`.

20. **Add OpenAI Chat Model Node for Report Generation Assistance**  
    - Name: `OpenAI Chat Model1`  
    - Configure model and prompt for report writing.  
    - Connect input to feed into `Generate Report`.

21. **Add Chain LLM Node for Report Generation**  
    - Name: `Generate Report`  
    - Configure LangChain chain to create full report text.  
    - Connect input from `Perplexity Competitor Research` and `OpenAI Chat Model1`.

22. **Add Google Docs Node to Create Document**  
    - Name: `Create doc`  
    - Configure to create a new Google Document with appropriate title and folder.  
    - Connect input from `Generate Report`.

23. **Add Google Docs Node to Append Report**  
    - Name: `Add report to doc`  
    - Configure to append report content to the created document using its ID.  
    - Connect input from `Create doc`.

24. **Add Google Sheets Node to Update Sheet**  
    - Name: `Add to sheets`  
    - Configure to write final report metadata and links back to Google Sheets.  
    - Connect input from `Add report to doc`.

25. **Connect back output of “Add to sheets” to `Loop Over Items`** for next batch processing.

26. **Credential Setup:**  
    - Configure Google Sheets and Google Docs credentials with appropriate scopes.  
    - Set up LinkedIn API OAuth2 credentials.  
    - Configure OpenAI API credentials for LangChain nodes.  
    - Configure Perplexity API access tokens.

27. **Testing and Validation:**  
    - Run the manual trigger node.  
    - Monitor execution for errors in API calls, AI completions, or document creation.  
    - Adjust batch sizes and wait times if rate limits or timeouts occur.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                    |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Workflow automates lead research from LinkedIn data enriched with AI models and Perplexity AI.  | Workflow purpose                                   |
| Uses LangChain nodes for advanced AI prompt chaining and output parsing.                         | AI processing nodes                                |
| Google Docs and Sheets integration for final report output and record keeping.                   | Output generation                                  |
| Handles API rate limits using Wait nodes with webhook support for asynchronous processing.      | LinkedIn API integration                           |
| Includes retry logic for Perplexity API calls with 5-second intervals on failure.                | Robustness for external API failures               |
| Requires OAuth2 credentials setup for LinkedIn and Google services; OpenAI API key for AI nodes. | Credential management                              |
| Workflow is initiated manually via a trigger node for testing or scheduled runs.                 | Execution control                                  |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.