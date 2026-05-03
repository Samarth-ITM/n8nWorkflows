Parse & Evaluate HR Candidates with GPT-4.1 and LinkedIn Data in CSV/XLSX

https://n8nworkflows.xyz/workflows/parse---evaluate-hr-candidates-with-gpt-4-1-and-linkedin-data-in-csv-xlsx-8775


# Parse & Evaluate HR Candidates with GPT-4.1 and LinkedIn Data in CSV/XLSX

### 1. Workflow Overview

This workflow automates the evaluation of HR candidates by processing uploaded CSV or XLSX files containing candidate data, enriching this data with recent LinkedIn posts via Apify, and scoring candidates using GPT-4.1 with detailed Hebrew explanations. The workflow integrates Google Drive and Google Sheets for file storage, data manipulation, and result reporting. It is designed for recruitment teams seeking automated, AI-powered candidate analysis incorporating social media insights.

Logical blocks:

- **1.1 Input Reception & File Handling:** Receives candidate data files via form, uploads them to Google Drive, downloads for processing.
- **1.2 File Content Extraction & Data Preparation:** Extracts data from CSV or XLSX formats, creates a new Google Sheet, and prepares the data for further processing.
- **1.3 Google Sheet Formatting & Column Management:** Adds custom columns for evaluation results, formats the sheet with RTL support and styling.
- **1.4 Candidate Data Enrichment & AI Evaluation:** Reads candidate data row by row, fetches recent LinkedIn posts via Apify, and uses GPT-4.1 (LangChain agent) to evaluate and score candidates.
- **1.5 Results Writing & Cleanup:** Writes evaluation scores and explanations back into Google Sheets, handles temporary files, and error notification.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & File Handling

- **Overview:**  
  Handles form submission where users upload candidate data files (CSV/XLSX). Files are uploaded to Google Drive, then downloaded for processing.

- **Nodes Involved:**  
  - On form submission  
  - Upload file (Google Drive)  
  - Download file (Google Drive)  
  - Create Google Sheet  
  - No Operation, do nothing

- **Node Details:**

  - **On form submission**  
    - *Type:* formTrigger  
    - *Role:* Entry point, listens for form submissions with file upload (CSV/XLSX only).  
    - *Config:* Single required file field named "New File Attachment" accepts csv, xlsx, xls.  
    - *Input:* HTTP webhook POST with form data.  
    - *Output:* Passes uploaded file metadata and binary data downstream.  
    - *Failure modes:* File type validation failure, form submission errors, webhook unavailability.  

  - **Upload file**  
    - *Type:* Google Drive node (upload)  
    - *Role:* Uploads received file to designated Google Drive folder ("Demos") with timestamped name.  
    - *Config:* Dynamic filename with current date/time. Upload target folder is fixed by ID. Uses Google Drive OAuth2 credentials.  
    - *Input:* Binary file from formTrigger node.  
    - *Output:* Outputs uploaded file metadata including file ID.  
    - *Failure modes:* OAuth errors, file upload failure, folder permission issues.

  - **Download file**  
    - *Type:* Google Drive node (download)  
    - *Role:* Downloads the uploaded file by file ID for content extraction.  
    - *Config:* Uses file ID from Upload file output. Downloads into binary property "New_File_Attachment".  
    - *Input:* File ID from Upload file node output.  
    - *Output:* Binary file content for extraction.  
    - *Failure modes:* File not found, permissions, OAuth errors.

  - **Create Google Sheet**  
    - *Type:* Google Sheets node (spreadsheet creation)  
    - *Role:* Creates a new Google Sheet to hold candidate data with a timestamped title and a single sheet named "New Job Application Form".  
    - *Config:* Uses Google Sheets OAuth2 credentials. Creates spreadsheet with one sheet. Runs once per trigger.  
    - *Input:* Triggered after file download.  
    - *Output:* Spreadsheet ID and sheet ID for further operations.  
    - *Failure modes:* OAuth errors, API quota limits, sheet creation failure.

  - **No Operation, do nothing**  
    - *Type:* NoOp node  
    - *Role:* Placeholder branch to continue workflow if needed.  
    - *Config:* No params.  
    - *Failure modes:* None.

---

#### 1.2 File Content Extraction & Data Preparation

- **Overview:**  
  Determines file type (CSV or XLSX), extracts data accordingly, merges data streams, and appends to the newly created Google Sheet.

- **Nodes Involved:**  
  - Switch (file extension check)  
  - Extract from CSV  
  - Extract from XLSX  
  - Merge1  
  - Convert to Google Sheet  
  - Get Rid of File  
  - Format Columns  
  - Format Columns1  
  - Add Columns

- **Node Details:**

  - **Switch**  
    - *Type:* Switch node  
    - *Role:* Routes processing based on uploaded file extension (.csv or .xlsx).  
    - *Config:* Checks filename string suffix to determine route.  
    - *Input:* Filename string from form submission data.  
    - *Output:* Two branches: CSV and XLSX extraction paths.  
    - *Failure modes:* Unrecognized file type, empty filename.

  - **Extract from CSV**  
    - *Type:* ExtractFromFile  
    - *Role:* Parses CSV binary data into structured JSON rows.  
    - *Config:* Reads from binary property "New_File_Attachment".  
    - *Input:* File binary from Download file node on CSV branch.  
    - *Output:* JSON array of candidate rows.  
    - *Failure modes:* CSV format errors, corrupted files.

  - **Extract from XLSX**  
    - *Type:* ExtractFromFile (xlsx operation)  
    - *Role:* Parses XLSX binary data into structured JSON rows.  
    - *Config:* Reads from binary property "New_File_Attachment".  
    - *Input:* File binary from Download file node on XLSX branch.  
    - *Output:* JSON array of candidate rows.  
    - *Failure modes:* XLSX format issues, corrupt files.

  - **Merge1**  
    - *Type:* Merge node  
    - *Role:* Combines CSV and XLSX extraction outputs for unified downstream processing.  
    - *Config:* Combine by position, includes unpaired items (handles one active branch).  
    - *Input:* Extract from CSV and Extract from XLSX outputs.  
    - *Output:* Single merged data stream of candidate rows.  

  - **Convert to Google Sheet**  
    - *Type:* Google Sheets (appendOrUpdate)  
    - *Role:* Inserts or updates candidate rows into the newly created Google Sheet's "New Job Application Form" sheet.  
    - *Config:* Auto-maps columns, no type conversion, no matching columns specified (append mode).  
    - *Input:* Merged candidate data.  
    - *Output:* Confirmation with spreadsheet and sheet IDs.  
    - *Failure modes:* Wrong mapping, OAuth errors, API limits.

  - **Get Rid of File**  
    - *Type:* Google Drive (move)  
    - *Role:* Moves the uploaded file to a "Temp" folder for cleanup after processing.  
    - *Config:* Uses file ID from Upload file node, target folder ID fixed.  
    - *Input:* File metadata from Upload file node.  
    - *Output:* File metadata in new location.  
    - *Failure modes:* Permission issues, file not found.

  - **Format Columns**  
    - *Type:* Set node  
    - *Role:* Adds two new columns ("Score Explanation" and "Evaluation Score") as a single string value to the workflow data to prepare for Google Sheets column creation.  
    - *Config:* Sets a string with these column names.  
    - *Input:* File cleanup confirmation.  
    - *Output:* Data with new columns definition.  

  - **Format Columns1**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Splits CSV header string into individual column keys with null values to prepare for sheet column creation.  
    - *Config:* Custom JS code splits column names by commas and returns an object with keys set to null.  
    - *Input:* String of columns from previous node.  
    - *Output:* JSON object representing columns as keys.  

  - **Add Columns**  
    - *Type:* Google Sheets (append)  
    - *Role:* Adds the columns defined above to the Google Sheet to enable storing evaluation scores and explanations.  
    - *Config:* Uses a schema of Hebrew-named columns related to candidate attributes, mapping mode set to auto-map input data.  
    - *Input:* Formatted columns object.  
    - *Output:* Confirmation of column addition.  
    - *Failure modes:* API errors, OAuth issues.

---

#### 1.3 Google Sheet Formatting & Column Management

- **Overview:**  
  Applies visual and functional formatting to the Google Sheet, including right-to-left layout, header styling, column sizing, filters, and sorting by evaluation score descending.

- **Nodes Involved:**  
  - Prepare for Formatting  
  - Get First Row  
  - Apply Formatting

- **Node Details:**

  - **Prepare for Formatting**  
    - *Type:* Set node  
    - *Role:* Prepares or cleans data prior to formatting steps.  
    - *Config:* Empty options, executes once with error continuation.  
    - *Input:* Output from AI Agent.  
    - *Output:* Passes data to next step.

  - **Get First Row**  
    - *Type:* HTTP Request node  
    - *Role:* Retrieves the first row from the created Google Sheet to get column headers for dynamic formatting.  
    - *Config:* GET request to Google Sheets API v4 for range "Sheet1!1:1", authenticated with Google Sheets OAuth2.  
    - *Input:* Data from Prepare for Formatting.  
    - *Output:* JSON array of header values.  
    - *Failure modes:* API quota, auth failure.

  - **Apply Formatting**  
    - *Type:* HTTP Request node (POST)  
    - *Role:* Sends batchUpdate API requests to Google Sheets to:  
      - Set right-to-left text direction  
      - Format header row (background color, bold, alignment, wrap)  
      - Format data rows alignment and wrap  
      - Set basic filter on sheet  
      - Auto-adjust column widths based on header length  
      - Sort rows descending by "Evaluation Score" column  
    - *Config:* Constructs JSON body dynamically using header data from Get First Row node.  
    - *Input:* Headers from Get First Row.  
    - *Output:* API response confirmation.  
    - *Failure modes:* API quota, malformed requests, auth errors.

---

#### 1.4 Candidate Data Enrichment & AI Evaluation

- **Overview:**  
  For each candidate row, enriches data with recent LinkedIn posts via Apify, then evaluates candidate using GPT-4.1 powered AI with a detailed prompt and scoring rubric. Parses AI output and updates candidate rows with evaluation results.

- **Nodes Involved:**  
  - Read Sheet  
  - Run an Actor in Apify  
  - Get dataset items in Apify  
  - Structured Output Parser1  
  - GPT-4.1  
  - AI Agent  
  - Update row in sheet in Google Sheets1

- **Node Details:**

  - **Read Sheet**  
    - *Type:* Google Sheets (read)  
    - *Role:* Reads candidate data rows from the Google Sheet for AI evaluation.  
    - *Config:* Reads from the sheet ID created earlier, returns first match.  
    - *Input:* After columns are added.  
    - *Output:* Candidate rows data.  
    - *Failure modes:* OAuth errors, API limits.

  - **Run an Actor in Apify**  
    - *Type:* Apify actor execution  
    - *Role:* Runs an Apify actor "Linkedin Profile Posts Scraper" to fetch up to 3 recent LinkedIn posts for each candidate based on LinkedIn profile URL.  
    - *Config:* Actor ID fixed, input JSON contains candidate's LinkedIn URL and post limit. Uses Apify API credentials.  
    - *Input:* Candidate LinkedIn URL from Google Sheets data.  
    - *Output:* Actor run metadata.  
    - *Failure modes:* Actor failure, API quota, invalid URL.

  - **Get dataset items in Apify**  
    - *Type:* Apify dataset fetch  
    - *Role:* Retrieves output dataset items from the last run of the Apify actor to get scraped LinkedIn posts.  
    - *Config:* Dataset ID dynamically received from actor run.  
    - *Input:* Actor run output.  
    - *Output:* Up to 3 posts per candidate.  
    - *Failure modes:* Dataset missing, API errors.

  - **Structured Output Parser1**  
    - *Type:* LangChain structured output parser  
    - *Role:* Parses GPT-4.1 AI text output into structured JSON with keys: Evaluation_Score (0-100), Score_Explanation (Hebrew sentence), Row_Number_to_Match (numeric).  
    - *Config:* Uses JSON schema example for validation.  
    - *Input:* GPT-4.1 raw output.  
    - *Output:* Parsed structured data for sheet update.  
    - *Failure modes:* Parsing errors if AI output is malformed.

  - **GPT-4.1**  
    - *Type:* LangChain OpenAI chat model  
    - *Role:* Provides base language model for the AI Agent node.  
    - *Config:* Uses "gpt-4.1" model with default options. Requires OpenAI API key credentials.  
    - *Input:* Prompt from AI Agent.  
    - *Output:* Raw AI response with candidate evaluation.  
    - *Failure modes:* API limits, auth failure, timeout.

  - **AI Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Orchestrates candidate evaluation logic: reads candidate data, triggers Apify LinkedIn scraping, fetches posts, applies scoring rubric, outputs structured evaluation with Hebrew explanation, updates Google Sheet via Update row node.  
    - *Config:* Complex system prompt defining workflow rules, LinkedIn enrichment steps, scoring rubric, error handling, output contract in Hebrew and English. Includes multiple AI tools (GPT and Apify) and output parser integration.  
    - *Input:* Candidate rows from Read Sheet, Apify outputs.  
    - *Output:* Structured evaluation data for each candidate.  
    - *Failure modes:* AI reasoning errors, Apify failures, network errors, prompt misconfiguration.

  - **Update row in sheet in Google Sheets1**  
    - *Type:* Google Sheets Tool node (update)  
    - *Role:* Updates existing candidate rows with evaluation results: "Evaluation Score" and "Score Explanation" columns.  
    - *Config:* Matches rows by "row_number" column, writes two new columns. Uses Google Sheets OAuth2 credentials.  
    - *Input:* Structured output from AI Agent.  
    - *Output:* Confirmation of row update.  
    - *Failure modes:* Row not found, API quota, auth errors.

---

#### 1.5 Results Writing & Cleanup

- **Overview:**  
  Completes formatting after AI evaluation, cleans temporary files, and handles errors by sending email alerts.

- **Nodes Involved:**  
  - Prepare for Formatting  
  - Get First Row  
  - Apply Formatting  
  - Get Rid of File  
  - Error Trigger  
  - Send a message (Gmail)

- **Node Details:**

  - **Error Trigger**  
    - *Type:* Error trigger node  
    - *Role:* Captures any workflow errors to trigger alerting.  
    - *Input:* Any node failure in workflow.  
    - *Output:* Triggers Send a message node.  

  - **Send a message**  
    - *Type:* Gmail node  
    - *Role:* Sends alert email to admin ("elay.g@82labs.io") with error details and timestamp.  
    - *Config:* Uses Gmail OAuth2 credentials, plain text email.  
    - *Failure modes:* Gmail auth errors, email delivery failure.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                              | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                   |
|----------------------------|----------------------------------|----------------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| On form submission         | formTrigger                      | Receives form file submission (CSV/XLSX)    | -                                | Upload file                     |                                                                                              |
| Upload file                | Google Drive (upload)             | Uploads file to Google Drive folder          | On form submission               | Download file                   |                                                                                              |
| Download file              | Google Drive (download)           | Downloads uploaded file for extraction       | Upload file                     | Create Google Sheet, No Operation |                                                                                              |
| Create Google Sheet        | Google Sheets (spreadsheet create) | Creates new Google Sheet for candidate data  | Download file                   | Merge                          |                                                                                              |
| No Operation, do nothing   | NoOp                            | Placeholder branch                            | Download file                   | Merge                          |                                                                                              |
| Switch                    | Switch                          | Routes processing based on file extension     | Merge                          | Extract from CSV, Extract from XLSX |                                                                                              |
| Extract from CSV           | ExtractFromFile                 | Parses CSV binary to JSON                      | Switch (CSV branch)             | Merge1                         |                                                                                              |
| Extract from XLSX          | ExtractFromFile                 | Parses XLSX binary to JSON                     | Switch (XLSX branch)            | Merge1                         |                                                                                              |
| Merge1                    | Merge                          | Merges CSV and XLSX extraction outputs        | Extract from CSV, Extract from XLSX | Convert to Google Sheet        |                                                                                              |
| Convert to Google Sheet    | Google Sheets (appendOrUpdate)  | Appends candidate data to Google Sheet         | Merge1                         | Get Rid of File                |                                                                                              |
| Get Rid of File            | Google Drive (move)             | Moves processed file to Temp folder            | Convert to Google Sheet         | Format Columns                 |                                                                                              |
| Format Columns             | Set                            | Defines new columns for evaluation output      | Get Rid of File                 | Format Columns1                |                                                                                              |
| Format Columns1            | Code                           | Converts columns string to object keys         | Format Columns                 | Add Columns                   |                                                                                              |
| Add Columns                | Google Sheets (append)          | Adds evaluation columns to Google Sheet        | Format Columns1                | Read Sheet                    |                                                                                              |
| Read Sheet                 | Google Sheets (read)            | Reads candidate rows for AI evaluation          | Add Columns                   | AI Agent                     |                                                                                              |
| Run an Actor in Apify      | Apify actor execution           | Runs LinkedIn posts scraper actor               | AI Agent                     | Get dataset items in Apify   |                                                                                              |
| Get dataset items in Apify | Apify dataset fetch             | Retrieves scraped LinkedIn posts                 | Run an Actor in Apify         | AI Agent                     |                                                                                              |
| GPT-4.1                   | OpenAI LangChain model          | Provides base GPT-4.1 model for AI Agent        | AI Agent                     | AI Agent                     |                                                                                              |
| AI Agent                  | LangChain Agent                 | Orchestrates candidate evaluation and updates   | Read Sheet, GPT-4.1, Apify nodes | Prepare for Formatting          |                                                                                              |
| Prepare for Formatting    | Set                            | Prepares data for formatting steps               | AI Agent                     | Get First Row                |                                                                                              |
| Get First Row             | HTTP Request                   | Gets Google Sheet headers for formatting         | Prepare for Formatting        | Apply Formatting             |                                                                                              |
| Apply Formatting          | HTTP Request                   | Applies styling, RTL support, filters, sorting   | Get First Row                 | -                            |                                                                                              |
| Structured Output Parser1 | LangChain Output Parser        | Parses AI output JSON structure                    | GPT-4.1                      | AI Agent                     |                                                                                              |
| Update row in sheet in Google Sheets1 | Google Sheets Tool (update) | Updates candidate rows with evaluation results    | AI Agent                     | -                            |                                                                                              |
| Error Trigger             | Error Trigger                  | Catches workflow errors                            | -                            | Send a message               |                                                                                              |
| Send a message            | Gmail                         | Sends email alert on error                         | Error Trigger                | -                            |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: formTrigger  
   - Configure with a form titled "Upload CSV File"  
   - Add a file field named "New File Attachment"  
   - Accept file types: csv, xlsx, xls  
   - Mark it as required  
   - Connect webhook for form submission.

2. **Add Google Drive Upload Node**  
   - Type: Google Drive (upload)  
   - Configure to upload file from "New_File_Attachment" binary property  
   - Set filename dynamically as "New Job Application Form {{ $now.format('d-LL-yyyy, H:mm') }}"  
   - Target folder: Use folder ID "17r0X1Y16pvY1NXhKFUzzjS0mflnTy_k0" (Demos)  
   - Use Google Drive OAuth2 credentials  
   - Connect output of formTrigger node here.

3. **Add Google Drive Download Node**  
   - Type: Google Drive (download)  
   - Configure to download file by ID from Upload file node  
   - Set binary property name "New_File_Attachment"  
   - Use same Google Drive OAuth2 credentials  
   - Connect Upload file node output here.

4. **Add Google Sheets Node to Create Spreadsheet**  
   - Type: Google Sheets (spreadsheet create)  
   - Title: "New Job Application Google Sheet Form {{ $now.format('d-LL-yyyy, H:mm') }}"  
   - Create one sheet named "New Job Application Form"  
   - Use Google Sheets OAuth2 credentials  
   - Connect Download file output here.

5. **Add Switch Node for File Type**  
   - Type: Switch  
   - Add two cases checking file extension of uploaded file:  
     - Ends with ".csv" output "CSV"  
     - Ends with ".xlsx" output "XLSX"  
   - Connect Create Google Sheet node output to Switch.

6. **Add Extract from CSV Node**  
   - Type: ExtractFromFile  
   - Configure for CSV  
   - Binary property: "New_File_Attachment"  
   - Connect Switch "CSV" output here.

7. **Add Extract from XLSX Node**  
   - Type: ExtractFromFile  
   - Configure operation: XLSX  
   - Binary property: "New_File_Attachment"  
   - Connect Switch "XLSX" output here.

8. **Add Merge Node (Merge1)**  
   - Type: Merge  
   - Mode: Combine  
   - Combine by position  
   - Include unpaired items: true  
   - Connect Extract from CSV and Extract from XLSX outputs here.

9. **Add Google Sheets AppendOrUpdate Node**  
   - Type: Google Sheets (appendOrUpdate)  
   - Document ID and Sheet Name from Create Google Sheet node outputs  
   - Map columns matching candidate data fields (Hebrew headers)  
   - Use Google Sheets OAuth2 credentials  
   - Connect Merge1 output here.

10. **Add Google Drive Move Node (Get Rid of File)**  
    - Type: Google Drive (move)  
    - Move uploaded file to folder ID "1cmnKI80W2CN67l8HOMeLdRLx-rFcxbzB" (Temp)  
    - Use Google Drive credentials  
    - Connect Convert to Google Sheet output here.

11. **Add Set Node (Format Columns)**  
    - Type: Set  
    - Add string field "columns" with value "Score Explanation, Evaluation Score"  
    - Connect Get Rid of File output here.

12. **Add Code Node (Format Columns1)**  
    - Type: Code (JavaScript)  
    - Input: columns string from previous node  
    - Script: Split columns string by commas, trim, create object keys with null values, output as JSON object  
    - Connect Format Columns output here.

13. **Add Google Sheets Append Node (Add Columns)**  
    - Type: Google Sheets (append)  
    - Document and sheet from Create Google Sheet node  
    - Append columns defined in Format Columns1 node  
    - Use Google Sheets credentials  
    - Connect Format Columns1 output here.

14. **Add Google Sheets Read Node (Read Sheet)**  
    - Type: Google Sheets (read)  
    - Read from created sheet, auto detect range  
    - Use Google Sheets credentials  
    - Connect Add Columns output here.

15. **Add Apify Actor Node (Run an Actor in Apify)**  
    - Type: Apify actor execution  
    - Actor ID: "LQQIXN9Othf8f7R5n" (Linkedin Profile Posts Scraper)  
    - Input JSON: includes candidate LinkedIn URL and limit 3 posts  
    - Use Apify API credentials  
    - Connect AI Agent input or Read Sheet output as per design.

16. **Add Apify Dataset Node (Get dataset items in Apify)**  
    - Type: Apify dataset fetch  
    - Dataset ID from last run of Apify actor node  
    - Use Apify API credentials  
    - Connect Run an Actor node output here.

17. **Add LangChain GPT-4.1 Node**  
    - Type: LangChain OpenAI chat model  
    - Model: "gpt-4.1"  
    - Use OpenAI API credentials  
    - Connect AI Agent node.

18. **Add LangChain Agent Node (AI Agent)**  
    - Type: LangChain Agent  
    - Configure system prompt with detailed instructions for candidate evaluation, Apify integration, scoring rubric, Hebrew output, error handling, and output format  
    - Integrate GPT-4.1 as language model, Apify as AI tool, and structured output parser  
    - Connect Read Sheet, Apify nodes, GPT-4.1, and structured output parser accordingly.

19. **Add LangChain Structured Output Parser Node**  
    - Type: Output Parser Structured  
    - JSON schema example with keys: Evaluation_Score (number), Score_Explanation (string), Row_Number_to_Match (number)  
    - Connect GPT-4.1 output to this parser.

20. **Add Google Sheets Update Node (Update row in sheet in Google Sheets1)**  
    - Type: Google Sheets Tool (update)  
    - Configure to update rows by matching "row_number" column  
    - Write "Evaluation Score" and "Score Explanation" columns with AI results  
    - Use Google Sheets credentials  
    - Connect AI Agent output here.

21. **Add Set Node (Prepare for Formatting)**  
    - Type: Set  
    - Prepare data before formatting (empty config)  
    - Connect AI Agent main output here.

22. **Add HTTP Request Node (Get First Row)**  
    - Type: HTTP Request (GET)  
    - URL: Google Sheets API to get first row values from created sheet  
    - Auth with Google Sheets OAuth2  
    - Connect Prepare for Formatting output here.

23. **Add HTTP Request Node (Apply Formatting)**  
    - Type: HTTP Request (POST)  
    - URL: Google Sheets batchUpdate API  
    - JSON body: batchUpdate commands for RTL, header styling, wrap, filters, column sizing, sorting by "Evaluation Score" descending  
    - Auth with Google Sheets OAuth2  
    - Connect Get First Row output here.

24. **Add Gmail Node (Send a message)**  
    - Type: Gmail  
    - Sends error alert to admin email with timestamp and message  
    - Use Gmail OAuth2 credentials  

25. **Add Error Trigger Node**  
    - Type: Error Trigger  
    - Connect to Send a message node to handle workflow failures.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| AI-Powered Candidate Evaluation and LinkedIn Enrichment in CSV/XLSX Format by Elay Guez. Features include form upload, Google Sheets integration with RTL support, LinkedIn scraping via Apify, GPT-4.1 scoring with Hebrew explanations. | Creator LinkedIn: https://www.linkedin.com/in/elay-g                                                                                                            |
| Quick setup instructions: Connect Google Drive/Sheets, OpenAI API key, Apify API key, Gmail for error alerts, test and activate workflow. Estimated cost ~$0.05 per candidate, saves 15-20 minutes per candidate.                     | Workflow sticky note content                                                                                                                                    |
| Google Sheets API limits and OAuth credentials must be properly configured to avoid authorization errors.                                                                                                                      | Google Sheets API docs: https://developers.google.com/sheets/api                                                                                                 |
| Apify actor used: "Linkedin Profile Posts Scraper [NO COOKIES]" (apimaestro/linkedin-profile-posts) requires valid LinkedIn URLs and proper Apify API credentials.                                                               | Apify actor link: https://console.apify.com/actors/LQQIXN9Othf8f7R5n                                                                                             |
| GPT-4.1 model used with LangChain integration; ensure OpenAI API key has GPT-4.1 access and sufficient quota.                                                                                                                  | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4                                                                                                  |
| Error handling includes email alerts to admin on failures triggered by n8n’s errorTrigger node.                                                                                                                                 | Gmail API docs: https://developers.google.com/gmail/api                                                                                                        |

---

**Disclaimer:** The provided content is derived solely from an n8n automated workflow. It complies fully with current content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.