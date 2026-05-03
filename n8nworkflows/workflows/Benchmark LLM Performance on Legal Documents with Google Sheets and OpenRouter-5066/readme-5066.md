Benchmark LLM Performance on Legal Documents with Google Sheets and OpenRouter

https://n8nworkflows.xyz/workflows/benchmark-llm-performance-on-legal-documents-with-google-sheets-and-openrouter-5066


# Benchmark LLM Performance on Legal Documents with Google Sheets and OpenRouter

### 1. Workflow Overview

This workflow benchmarks the performance of Large Language Models (LLMs) on legal document analysis tasks by integrating Google Sheets, Google Drive, and OpenRouter LLM APIs. Its primary use case is to evaluate LLM outputs against source legal documents, judge their correctness, and store evaluation results back into Google Sheets. The workflow is designed for legal AI model benchmarking and quality assurance.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception and Test Case Fetching:** Initiates either manually or via webhook, fetches test cases from Google Sheets.
- **1.2 Filtering and PDF Verification:** Filters test cases to focus on those referencing PDF files.
- **1.3 PDF Retrieval and Text Extraction:** Downloads and extracts text from PDF legal documents stored in Google Drive.
- **1.4 LLM Evaluation Chain:** Sends source text, task input, and LLM output to an LLM evaluation prompt via OpenRouter, parsing results.
- **1.5 Result Aggregation and Storage:** Merges evaluation results with original data and updates an output Google Sheet with pass/fail decisions and reasoning.
- **1.6 Subworkflow Execution:** Uses an external HTTP webhook to process test cases in batch, supporting asynchronous operation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Test Case Fetching

**Overview:**  
This block starts the workflow either manually ("When clicking ‘Test workflow’" node) or by receiving HTTP POST requests via a webhook. It fetches test cases (input prompts, outputs, and source references) from a Google Sheet containing legal evaluation tasks.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Webhook  
- Get Tests  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual execution for testing  
  - *Config:* No parameters  
  - *Connections:* Outputs to Get Tests node  
  - *Edge cases:* Manual trigger does not scale for production; webhook preferred.

- **Webhook**  
  - *Type:* HTTP Webhook (POST)  
  - *Role:* Receives external POST requests to start workflow with test data  
  - *Config:* Path fixed to webhook ID; responds with all entries from last node  
  - *Connections:* Outputs to Keep Original Data and Save Input/Output nodes  
  - *Failure modes:* HTTP errors, unauthorized requests, malformed POST bodies

- **Get Tests**  
  - *Type:* Google Sheets (Read)  
  - *Role:* Reads test cases from a predefined Google Sheet (Info Extraction Tasks sheet)  
  - *Config:* Reads from sheet with gid=0, document ID set; uses Google Sheets OAuth2 credentials  
  - *Expressions:* None complex; static sheet and document ID  
  - *Connections:* Outputs to Is PDF? node  
  - *Failure modes:* Google Sheets auth failures, missing file/sheet, rate limits

---

#### 2.2 Filtering and PDF Verification

**Overview:**  
Filters fetched test cases to only those referencing PDF files in the "Relevant Source Reference" column and with valid row numbers.

**Nodes Involved:**  
- Is PDF?  
- Limit (for testing) [disabled]  

**Node Details:**

- **Is PDF?**  
  - *Type:* If Condition  
  - *Role:* Checks if test case references a PDF file by testing if the "Relevant Source Reference" contains ".pdf" and row number > 0  
  - *Config:* String contains ".pdf" and numeric row_number > 0  
  - *Connections:* Main output to Limit (for testing) node  
  - *Edge cases:* Empty or malformed references, non-PDF files must be excluded to avoid processing errors

- **Limit (for testing)** (disabled)  
  - *Type:* Limit  
  - *Role:* Limits the number of test cases processed to 3 (for testing)  
  - *Config:* Max items 3, disabled in production  
  - *Connections:* Outputs to Execute Subworkflow  
  - *Note:* Disabled for full runs; useful for debugging small batches

---

#### 2.3 PDF Retrieval and Text Extraction

**Overview:**  
Downloads PDF files from Google Drive using URLs in the test cases and extracts their text content for evaluation.

**Nodes Involved:**  
- Save Input/Output  
- Google Drive  
- Extract from File  

**Node Details:**

- **Save Input/Output**  
  - *Type:* Set  
  - *Role:* Saves the raw HTTP webhook body data for later use in prompt assembly  
  - *Config:* Saves entire JSON body from webhook into node output  
  - *Connections:* Output to Google Drive node  
  - *Edge cases:* Missing or malformed body data can cause downstream failures

- **Google Drive**  
  - *Type:* Google Drive (Download)  
  - *Role:* Downloads the PDF file using extracted file ID from URL in the test case  
  - *Config:* File ID extracted by regex from the "URL" field in JSON input  
  - *Credentials:* Google Drive OAuth2 credentials required  
  - *Connections:* Outputs to Extract from File node  
  - *Edge cases:* File not found, permission denied, invalid file ID, network errors

- **Extract from File**  
  - *Type:* Extract from File (PDF)  
  - *Role:* Extracts text content from the downloaded PDF file for LLM input  
  - *Config:* PDF extraction operation selected  
  - *Connections:* Outputs to Basic LLM Chain1 for evaluation prompt input  
  - *Edge cases:* Corrupted or scanned PDFs may result in missing or incomplete text extraction

---

#### 2.4 LLM Evaluation Chain

**Overview:**  
Uses an OpenRouter-hosted GPT-4.1 model to evaluate the LLM output against the source text and task prompt, determining pass/fail with reasoning. Output is parsed strictly as JSON.

**Nodes Involved:**  
- Basic LLM Chain1  
- OpenRouter Chat Model  
- Structured Output Parser2  
- Merge1  
- Keep Original Data  

**Node Details:**

- **Basic LLM Chain1**  
  - *Type:* LangChain LLM Chain with defined prompt and output parser  
  - *Role:* Sends a structured prompt to the LLM model, providing input task, source extracted text, and LLM output for evaluation  
  - *Config:*  
    - Prompt templates crafted to enforce three accuracy standards: factual correctness, relevance, completeness  
    - Includes detailed instructions and failure patterns for the AI judge  
    - Output expected in JSON with keys "reasoning" and "decision"  
  - *Expressions:* Uses saved input/output from previous node, injected JSON source text  
  - *Connections:* Receives AI model from OpenRouter Chat Model, output parser from Structured Output Parser2, outputs to Merge1  
  - *Edge cases:* Model errors, timeout, malformed JSON output, unexpected output format, OpenRouter API limits or credentials errors

- **OpenRouter Chat Model**  
  - *Type:* LangChain OpenRouter LLM Chat Model  
  - *Role:* Provides GPT-4.1 access via OpenRouter API  
  - *Config:* Model set to "openai/gpt-4.1"  
  - *Credentials:* OpenRouter API credentials required  
  - *Connections:* Provides AI model input to Basic LLM Chain1  
  - *Edge cases:* API key invalid, quota exceeded, network failures

- **Structured Output Parser2**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Enforces and validates JSON schema compliance for the LLM output  
  - *Config:* JSON schema example includes "reasoning" (string) and "decision" (string) keys  
  - *Connections:* Connects output parser to Basic LLM Chain1  
  - *Edge cases:* Parsing errors if LLM output is not valid JSON, incomplete output

- **Merge1**  
  - *Type:* Merge  
  - *Role:* Combines original data and evaluation results by position  
  - *Config:* Combine mode: "combineByPosition" to merge arrays element-wise  
  - *Connections:* Inputs from Basic LLM Chain1 and Keep Original Data nodes; outputs to final return or storage  
  - *Edge cases:* Mismatched array lengths, data loss if indexes misaligned

- **Keep Original Data**  
  - *Type:* Set  
  - *Role:* Passes original webhook data unchanged for merging later  
  - *Connections:* Input from Webhook, output to Merge1  
  - *Edge cases:* None significant; ensures original context preserved

---

#### 2.5 Result Aggregation and Storage

**Overview:**  
Appends or updates the Google Sheet storing evaluation results with original test data plus judge decision and reasoning.

**Nodes Involved:**  
- Update Results  
- Execute Subworkflow  

**Node Details:**

- **Update Results**  
  - *Type:* Google Sheets (Append or Update)  
  - *Role:* Writes evaluation results to a results sheet ("Results" tab) in the same Google Sheet document  
  - *Config:*  
    - Uses "ID" column to match existing rows for update or appends new rows  
    - Maps fields: ID, URL, Input, Output, Decision, Test No., Reasoning, AI Platform, Relevant Source Reference  
    - Uses Google Sheets OAuth2 credentials  
  - *Connections:* Input from Execute Subworkflow node, no further outputs  
  - *Edge cases:* Sheet permission issues, rate limits, data mismatches, partial updates

- **Execute Subworkflow**  
  - *Type:* HTTP Request (POST)  
  - *Role:* Sends batches of test case data to an external webhook-based subworkflow for processing  
  - *Config:*  
    - POST to a specified external webhook URL  
    - Batching enabled: batch size 1, interval 500ms  
    - Sends JSON body dynamically from input data  
    - Error handling set to continue on error, max 2 tries  
  - *Connections:* Input from Limit (for testing) node, output to Update Results node  
  - *Edge cases:* Network failures, webhook timeouts, incorrect payloads leading to subworkflow errors

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                                      | Input Node(s)                                 | Output Node(s)                        | Sticky Note                                                                                                                                            |
|-----------------------------|----------------------------------|-----------------------------------------------------|----------------------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger                   | Manual start of workflow for testing                 | None                                         | Get Tests                           | Start Here: Make sure to click "Execute Workflow" here, not underneath, to set webhook listening mode                                                  |
| Webhook                     | HTTP Webhook (POST)              | Receive external test data via HTTP POST             | None                                         | Keep Original Data, Save Input/Output |                                                                                                                                                        |
| Get Tests                   | Google Sheets                   | Fetch test cases from Google Sheet                    | When clicking ‘Test workflow’                 | Is PDF?                            | Fetch test cases from Google Sheet. Only PDF rows are processed.                                                                                       |
| Is PDF?                     | If Condition                    | Filter test cases referencing PDFs                    | Get Tests                                    | Limit (for testing)                 | Data filter: Only rows where Relevant Source Reference contains ".pdf" and row_number > 0                                                              |
| Limit (for testing)          | Limit                          | Limit number of test cases processed (disabled)      | Is PDF?                                      | Execute Subworkflow                | Disabled by default; limits items to 3 for testing                                                                                                    |
| Save Input/Output           | Set                            | Save raw webhook body for prompt assembly             | Webhook                                      | Google Drive                      |                                                                                                                                                        |
| Google Drive                | Google Drive (Download)         | Download PDF file from Drive using URL from test case | Save Input/Output                            | Extract from File                 | Download PDF for text extraction                                                                                                                      |
| Extract from File           | Extract from File (PDF)         | Extract text content from PDF                          | Google Drive                                | Basic LLM Chain1                  | Extract text from downloaded PDF                                                                                                                     |
| Basic LLM Chain1            | LangChain LLM Chain             | Evaluate LLM output against source and task          | Extract from File, OpenRouter Chat Model, Structured Output Parser2 | Merge1                           | Uses OpenRouter GPT-4.1 to judge LLM output; expects JSON output with reasoning and decision                                                          |
| OpenRouter Chat Model       | LangChain OpenRouter LLM Chat   | Provide GPT-4.1 LLM model via OpenRouter API         | None                                         | Basic LLM Chain1 (ai_languageModel) | OpenRouter credentials required                                                                                                                      |
| Structured Output Parser2   | LangChain Structured Output Parser | Parse and validate JSON output from LLM               | None                                         | Basic LLM Chain1 (ai_outputParser) | Enforce JSON schema for evaluation output                                                                                                           |
| Keep Original Data          | Set                            | Pass original webhook data unchanged                  | Webhook                                      | Merge1                           | Preserve original input for combining with evaluation results                                                                                        |
| Merge1                     | Merge                          | Combine original data with evaluation output          | Basic LLM Chain1, Keep Original Data          | None (final output)               | Combine pass/fail decision with original data                                                                                                        |
| Execute Subworkflow         | HTTP Request                   | Send test cases to external webhook subworkflow       | Limit (for testing)                           | Update Results                   | Runs immediately with batching, waits for result before next step                                                                                     |
| Update Results             | Google Sheets                   | Append or update evaluation results in Google Sheet  | Execute Subworkflow                           | None                            | Write decisions and reasoning back to results sheet                                                                                                  |
| Sticky Note3               | Sticky Note                    | Explanation of Execute Subworkflow node role          | None                                         | None                            | ## 2. Execute Subworkflow: Runs immediately (batching), waits for results before next step                                                           |
| Sticky Note6               | Sticky Note                    | Data format explanation for Tests Sheet                | None                                         | None                            | ## Data format: Columns include ID, Test No., AI Platform, Relevant Source, URL, Input, Output                                                      |
| Sticky Note7               | Sticky Note                    | Explanation of test cases fetch step                    | None                                         | None                            | ## 1. Fetch test cases: Grab list from Google Sheet, only PDFs processed                                                                             |
| Sticky Note11              | Sticky Note                    | Explanation of PDF download and extraction              | None                                         | None                            | ## 3. Grab the PDF as text: Download from Google Drive, extract text, filter empty files                                                             |
| Sticky Note12              | Sticky Note                    | Explanation of writing results to Google Sheets         | None                                         | None                            | ## 5. Update results: Append original data with judge decision and reasoning                                                                         |
| Sticky Note13              | Sticky Note                    | Explanation of LLM evaluation prompt and output parser | None                                         | None                            | ## 4. Judge LLM outputs: Prompt judges input/output, uses OpenRouter, output parsed as JSON                                                         |
| Sticky Note14              | Sticky Note                    | Explanation of data merging and returning combined data | None                                         | None                            | ## 5. Combine data and return: Merge pass/fail + reason with original data, keep all context                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually for testing  
   - No parameters needed  
   - Connect output to **Get Tests**

2. **Create Webhook Node**  
   - Type: HTTP Webhook (POST)  
   - Path: Fixed unique ID (e.g., "1cbce320-d28e-4e97-8663-bf2c6a36a358")  
   - HTTP Method: POST  
   - Response Mode: Return all entries from last node  
   - Connect output to **Keep Original Data** and **Save Input/Output**

3. **Create Google Sheets Node "Get Tests"**  
   - Operation: Read rows  
   - Document ID: "10l_gMtPsge00eTTltGrgvAo54qhh3_twEDsETrQLAGU"  
   - Sheet Name: gid=0 (first sheet)  
   - Credentials: Google Sheets OAuth2  
   - Connect output to **Is PDF?**

4. **Create If Node "Is PDF?"**  
   - Conditions:  
     - "Relevant Source Reference" contains ".pdf" (case sensitive)  
     - "row_number" > 0  
   - Connect true output to **Limit (for testing)** (optional) or directly to **Execute Subworkflow**

5. **Create Limit Node (optional for testing)**  
   - Max Items: 3  
   - Disabled for production  
   - Connect output to **Execute Subworkflow**

6. **Create HTTP Request Node "Execute Subworkflow"**  
   - Method: POST  
   - URL: External webhook URL (e.g., "https://webhook-processor-production-48f8.up.railway.app/webhook/1cbce320-d28e-4e97-8663-bf2c6a36a358")  
   - Options: Enable batching, batch size=1, batch interval=500ms  
   - Send body as JSON of input data  
   - Error handling: Continue on error, max 2 retries  
   - Connect output to **Update Results**

7. **Create Google Sheets Node "Update Results"**  
   - Operation: Append or Update  
   - Document ID: Same as above  
   - Sheet Name: Results tab (gid=537199982)  
   - Map columns: ID, Test No., AI Platform, Relevant Source Reference, URL, Input, Output, Decision, Reasoning  
   - Use "ID" as matching column for updates  
   - Credentials: Google Sheets OAuth2

8. **Inside Subworkflow (external webhook):**  
   - Receive test case data  
   - **Save Input/Output** node: Save raw input body  
   - **Google Drive** node: Download PDF file by extracting file ID from "URL" field  
     - Credentials: Google Drive OAuth2  
   - **Extract from File** node: Extract text from PDF  
   - **OpenRouter Chat Model** node: Configure with OpenRouter API credentials, model "openai/gpt-4.1"  
   - **Structured Output Parser2** node: Define JSON schema with "reasoning" and "decision" keys  
   - **Basic LLM Chain1** node:  
     - Configure prompt with task, source text, and output  
     - Use OpenRouter Chat Model as AI model  
     - Use Structured Output Parser for output validation  
   - **Keep Original Data** node: Pass original input for merging  
   - **Merge1** node: Combine evaluation output with original input by position  
   - Return merged output as HTTP response

9. **Credentials Setup:**  
   - Google Sheets OAuth2 credentials must be created and authorized for reading and writing the designated spreadsheet  
   - Google Drive OAuth2 credentials must have access to files referenced in test cases  
   - OpenRouter API credentials must be configured with an active key supporting GPT-4.1 model access

10. **Testing and Execution:**  
    - Start workflow manually or send POST requests to the webhook URL  
    - Monitor logs for errors such as auth failures, missing files, parsing errors  
    - Adjust batch sizes or retry parameters as needed  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                              | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The Tests Sheet contains columns: ID, Test No., AI Platform, Relevant Source Reference, URL, Input, Output. Only PDF files are processed currently; DOCX handling is not implemented.                                                                                                    | Sticky Note6                                                                                      |
| Start the workflow by clicking "Execute Workflow" on the Manual Trigger node to ensure webhook listeners are properly initiated.                                                                                                                                                         | Sticky Note                                                                                       |
| The subworkflow executes immediately with batching enabled and waits for results, ensuring controlled API usage and avoiding overload.                                                                                                                                                   | Sticky Note3                                                                                      |
| The LLM evaluation prompt checks for factual correctness, relevance, and completeness, returning a JSON object with "reasoning" and "decision". The prompt includes detailed instructions and failure patterns for accurate judgment.                                                   | Sticky Note13                                                                                     |
| Results are appended or updated in the "Results" tab of the same Google Sheet, preserving original inputs plus evaluation decision and reasoning.                                                                                                                                         | Sticky Note12                                                                                     |
| The merged output from evaluation and original data is returned from the subworkflow to the main workflow for final storage and response. This preserves full context for each test case.                                                                                                | Sticky Note14                                                                                     |
| OpenRouter API is used here to facilitate easy switching of LLMs and better control over the language model provider.                                                                                                                                                                    | Sticky Note13                                                                                     |
| Workflow designed for legal domain AI evaluation and benchmarking; ensuring input/output consistency and error handling is critical, particularly for document retrieval and text extraction.                                                                                             | Entire workflow context                                                                          |
| Google Sheets and Drive OAuth2 credentials require proper scopes and refresh tokens; ensure these are configured before running the workflow.                                                                                                                                             | Credential setup                                                                                  |
| The workflow assumes file URLs are valid Google Drive links; file ID extraction uses a regex to parse the file ID. Ensure URLs conform to expected format.                                                                                                                                 | Google Drive node configuration                                                                   |

---

*Disclaimer:* The provided text is derived exclusively from an automated workflow built with n8n, a tool for integration and automation. This processing fully complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.