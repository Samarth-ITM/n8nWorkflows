Extract University Term Dates from Excel using CloudFlare Markdown Conversion

https://n8nworkflows.xyz/workflows/extract-university-term-dates-from-excel-using-cloudflare-markdown-conversion-3505


# Extract University Term Dates from Excel using CloudFlare Markdown Conversion

### 1. Workflow Overview

This workflow automates the extraction of university term dates from an Excel spreadsheet and converts them into an ICS calendar file for easy import into calendar applications like iCal, Google Calendar, or Outlook. It is designed to replace manual data entry by leveraging AI-powered document understanding and external services for Excel parsing.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Excel Download:** Triggering the workflow manually and downloading the term dates Excel file from a university website.
- **1.2 Excel Parsing via Cloudflare Markdown Conversion:** Sending the Excel file to Cloudflare’s Markdown Conversion Service to transform spreadsheet data into markdown tables readable by AI.
- **1.3 AI Extraction and Data Transformation:** Using Google Gemini LLM and the Information Extractor node to parse markdown and extract structured event data, followed by date fixing and sorting.
- **1.4 ICS File Creation and Distribution:** Generating an ICS calendar file from the extracted events, converting it to a binary file, and emailing it to recipients.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Excel Download

- **Overview:**  
  This block initiates the workflow manually and downloads the university term dates Excel file from a specified URL.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Get Term Dates Excel

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Triggers "Get Term Dates Excel" node.  
    - Edge Cases: None specific; manual trigger requires user interaction.

  - **Get Term Dates Excel**  
    - Type: HTTP Request  
    - Role: Downloads the XLSX file from the university URL.  
    - Configuration:  
      - URL set to the university’s term dates Excel file.  
      - Response format set to "file" to receive binary XLSX data.  
    - Inputs: Trigger from manual node.  
    - Outputs: Binary XLSX file passed to "Markdown Conversion Service".  
    - Edge Cases:  
      - URL changes or file unavailable (404 or network errors).  
      - Large file size may cause timeouts.  
      - Requires internet access.  
    - Notes: URL parameter should be updated if the university changes the file location.

---

#### 2.2 Excel Parsing via Cloudflare Markdown Conversion

- **Overview:**  
  Converts the downloaded Excel file to markdown tables using Cloudflare’s Markdown Conversion API, enabling AI to parse spreadsheet data.

- **Nodes Involved:**  
  - Markdown Conversion Service  
  - Extract Target Sheet

- **Node Details:**

  - **Markdown Conversion Service**  
    - Type: HTTP Request  
    - Role: Sends XLSX binary to Cloudflare API to convert sheets to markdown.  
    - Configuration:  
      - POST request to Cloudflare API endpoint with multipart-form-data including the XLSX file.  
      - Uses Cloudflare API credentials for authentication.  
    - Inputs: XLSX binary from "Get Term Dates Excel".  
    - Outputs: JSON response containing markdown conversion results.  
    - Edge Cases:  
      - Authentication failure if Cloudflare credentials invalid.  
      - API rate limits or service downtime.  
      - Incorrect ACCOUNT_ID in URL leads to 403 errors.  
    - Requirements: Cloudflare account and proper API credentials.  
    - Sticky Note: Explains the rationale for using Cloudflare Markdown Conversion and links to documentation.

  - **Extract Target Sheet**  
    - Type: Set  
    - Role: Extracts the markdown string of the target sheet from the Cloudflare API response.  
    - Configuration:  
      - Assigns `target_sheet` by splitting the markdown result string and selecting the 10th section (index 9).  
    - Inputs: JSON from Cloudflare API response.  
    - Outputs: JSON with `target_sheet` string passed to AI extraction.  
    - Edge Cases:  
      - If the markdown response format changes, the index-based extraction may fail or extract wrong data.  
      - No validation on extracted markdown content.  

---

#### 2.3 AI Extraction and Data Transformation

- **Overview:**  
  Uses Google Gemini LLM and the Information Extractor node to parse the markdown table into structured event data, then fixes date formats and sorts events chronologically.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Extract Key Events and Dates  
  - Events to Items  
  - Fix Dates  
  - Sort Events by Date

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: Langchain Google Gemini Chat Model  
    - Role: Provides AI language model capabilities to the Information Extractor node.  
    - Configuration:  
      - Model set to "models/gemini-2.5-pro-preview-03-25".  
      - Credentials: Google Palm API OAuth2.  
    - Inputs: Receives the markdown text from "Extract Target Sheet".  
    - Outputs: AI-powered extraction results to "Extract Key Events and Dates".  
    - Edge Cases:  
      - Authentication failure or quota exceeded on Google API.  
      - Model version changes may affect output format.  

  - **Extract Key Events and Dates**  
    - Type: Information Extractor (Langchain)  
    - Role: Extracts structured event data (week number, week beginning, title) from markdown text.  
    - Configuration:  
      - Input text: `{{$json.target_sheet}}`.  
      - System prompt instructs to capture values as-is without date conversion.  
      - Schema defines expected output as an array of objects with properties: week_number (number), week_beginning (string), title (string).  
    - Inputs: Markdown text from "Extract Target Sheet" via Google Gemini.  
    - Outputs: JSON array of extracted events to "Events to Items".  
    - Edge Cases:  
      - Extraction errors if markdown format varies.  
      - Schema mismatch or unexpected AI output structure.  

  - **Events to Items**  
    - Type: Split Out  
    - Role: Splits the array of events into individual items for further processing.  
    - Configuration: Splits on `output` field (the extracted events array).  
    - Inputs: JSON array from "Extract Key Events and Dates".  
    - Outputs: Individual event JSON objects to "Fix Dates".  
    - Edge Cases: Empty arrays or malformed data cause no output or errors.

  - **Fix Dates**  
    - Type: Set  
    - Role: Converts Excel serial date numbers to ISO 8601 UTC date strings for the `week_beginning` field.  
    - Configuration:  
      - Uses expression to convert Excel date number to JavaScript Date, adjusted for the base date (45915 offset).  
      - Keeps other fields intact.  
    - Inputs: Individual event JSON from "Events to Items".  
    - Outputs: Events with fixed `week_beginning` dates to "Sort Events by Date".  
    - Edge Cases:  
      - Invalid or missing `week_beginning` values cause conversion errors.  
      - Timezone assumptions may cause slight date shifts.

  - **Sort Events by Date**  
    - Type: Sort  
    - Role: Sorts the events chronologically by `week_beginning`.  
    - Configuration: Sort field set to `week_beginning`.  
    - Inputs: Fixed date events from "Fix Dates".  
    - Outputs: Sorted event list to "Events to ICS Document".  
    - Edge Cases: Missing or malformed dates may affect sorting order.

---

#### 2.4 ICS File Creation and Distribution

- **Overview:**  
  Converts the sorted events into an ICS calendar document using Python code, then converts it to a binary file and emails it as an attachment.

- **Nodes Involved:**  
  - Events to ICS Document  
  - Create ICS File  
  - Send Email with Attachment

- **Node Details:**

  - **Events to ICS Document**  
    - Type: Code (Python)  
    - Role: Generates ICS file content from event JSON objects.  
    - Configuration:  
      - Python code parses each event’s `week_beginning` date and title.  
      - Creates ICS VEVENT entries with UID, timestamps, DTSTART, DTEND, and SUMMARY fields.  
      - Handles parsing errors gracefully by skipping invalid events.  
      - Encodes ICS content to base64 string for binary conversion.  
    - Inputs: Sorted event JSON from "Sort Events by Date".  
    - Outputs: Base64-encoded ICS content to "Create ICS File".  
    - Edge Cases:  
      - Date parsing errors handled by try-except.  
      - Missing required fields cause event skip.  
      - Python environment must support async and base64 modules.  

  - **Create ICS File**  
    - Type: Convert to File  
    - Role: Converts base64 ICS content to a binary file with `.ics` extension.  
    - Configuration:  
      - File name derived from original Excel file name with `.ics` extension.  
      - MIME type set to `text/calendar`.  
      - Source property set to `data` (base64 string).  
    - Inputs: Base64 ICS content from "Events to ICS Document".  
    - Outputs: Binary ICS file to "Send Email with Attachment".  
    - Edge Cases:  
      - Incorrect base64 input causes file corruption.  

  - **Send Email with Attachment**  
    - Type: Gmail  
    - Role: Sends the ICS file as an email attachment.  
    - Configuration:  
      - Recipient email set to `jim@example.com` (should be customized).  
      - Subject and message body predefined.  
      - Attaches the ICS binary file.  
      - Uses Gmail OAuth2 credentials.  
    - Inputs: Binary ICS file from "Create ICS File".  
    - Outputs: Email sent confirmation.  
    - Edge Cases:  
      - Authentication failure or token expiration.  
      - Attachment size limits or email delivery failures.  
      - Email address must be valid.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                              | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                           |
|----------------------------|----------------------------------|----------------------------------------------|----------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Starts workflow manually                      | None                       | Get Term Dates Excel            |                                                                                                     |
| Get Term Dates Excel        | HTTP Request                     | Downloads XLSX file from university website  | When clicking ‘Test workflow’ | Markdown Conversion Service     |                                                                                                     |
| Markdown Conversion Service | HTTP Request                     | Converts XLSX to markdown via Cloudflare API | Get Term Dates Excel        | Extract Target Sheet            | Explains Cloudflare Markdown Conversion service and links to docs                                  |
| Extract Target Sheet        | Set                              | Extracts target markdown sheet from response | Markdown Conversion Service | Extract Key Events and Dates    |                                                                                                     |
| Google Gemini Chat Model    | Langchain Google Gemini Chat     | Provides LLM for AI extraction                | Extract Target Sheet        | Extract Key Events and Dates (ai_languageModel) |                                                                                                     |
| Extract Key Events and Dates| Information Extractor (Langchain)| Extracts structured event data from markdown | Extract Target Sheet (via Gemini) | Events to Items                | Explains AI extraction of term dates to events and links to docs                                   |
| Events to Items             | Split Out                       | Splits array of events into individual items | Extract Key Events and Dates | Fix Dates                      |                                                                                                     |
| Fix Dates                  | Set                              | Converts Excel serial dates to ISO strings   | Events to Items             | Sort Events by Date             |                                                                                                     |
| Sort Events by Date         | Sort                            | Sorts events by date                          | Fix Dates                   | Events to ICS Document          |                                                                                                     |
| Events to ICS Document      | Code (Python)                   | Generates ICS calendar content from events   | Sort Events by Date         | Create ICS File                 | Explains ICS document creation with code node and links to docs                                    |
| Create ICS File             | Convert to File                 | Converts ICS content to binary file           | Events to ICS Document      | Send Email with Attachment      | Explains Convert to File node for ICS binary creation and links to docs                            |
| Send Email with Attachment  | Gmail                          | Sends ICS file as email attachment            | Create ICS File             | None                          |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Create HTTP Request Node "Get Term Dates Excel"**  
   - URL: `https://www.westminster.ac.uk/sites/default/public-files/general-documents/undergraduate-term-dates-2025%E2%80%932026.xlsx` (update as needed)  
   - Response Format: File (binary)  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node "Markdown Conversion Service"**  
   - Method: POST  
   - URL: `https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/ai/tomarkdown` (replace `{ACCOUNT_ID}` with your Cloudflare account ID)  
   - Authentication: Use Cloudflare API credentials  
   - Content-Type: multipart/form-data  
   - Body Parameters: Add form binary data parameter named `files` linked to binary XLSX data from previous node.  
   - Connect output of "Get Term Dates Excel" to this node.

4. **Create Set Node "Extract Target Sheet"**  
   - Add a string field `target_sheet`  
   - Value: Expression to extract the 10th markdown section from Cloudflare response:  
     `={{ $json.result[0].data.split('##')[9] }}`  
   - Connect output of "Markdown Conversion Service" to this node.

5. **Create Langchain Google Gemini Chat Model Node**  
   - Model Name: `models/gemini-2.5-pro-preview-03-25`  
   - Credentials: Google Palm API OAuth2  
   - Connect output of "Extract Target Sheet" to this node as AI language model input.

6. **Create Information Extractor Node "Extract Key Events and Dates"**  
   - Input Text: `={{ $json.target_sheet }}`  
   - System Prompt: "Capture the values as seen. Do not convert dates."  
   - Schema Type: Manual  
   - Input Schema: JSON schema defining array of objects with properties:  
     - `week_number` (number)  
     - `week_beginning` (string)  
     - `title` (string)  
   - Connect output of "Extract Target Sheet" to this node’s main input.  
   - Connect output of Google Gemini Chat Model node to this node’s AI language model input.

7. **Create Split Out Node "Events to Items"**  
   - Field to split out: `output` (the extracted events array)  
   - Connect output of "Extract Key Events and Dates" to this node.

8. **Create Set Node "Fix Dates"**  
   - Add string field `week_beginning` with expression:  
     ```
     ={
       new Date(2025,8,15,0,0,0).toDateTime().toUTC()
         .plus({ 'day': $json.week_beginning - 45915 })
     }
     ```  
   - Include other fields unchanged.  
   - Connect output of "Events to Items" to this node.

9. **Create Sort Node "Sort Events by Date"**  
   - Sort field: `week_beginning` ascending  
   - Connect output of "Fix Dates" to this node.

10. **Create Code Node "Events to ICS Document"**  
    - Language: Python  
    - Paste the provided Python code that converts JSON events to ICS format, encodes to base64 string.  
    - Connect output of "Sort Events by Date" to this node.

11. **Create Convert to File Node "Create ICS File"**  
    - Operation: toBinary  
    - Source Property: `data` (base64 string from previous node)  
    - File Name: Expression to use original XLSX file name with `.ics` extension:  
      `={{ $('Get Term Dates Excel').first().binary.data.fileName }}.ics`  
    - MIME Type: `text/calendar`  
    - Connect output of "Events to ICS Document" to this node.

12. **Create Gmail Node "Send Email with Attachment"**  
    - Recipient: Update to desired email address (default is `jim@example.com`)  
    - Subject: "Undergraduate Terms Dates Calendar 2025/2026"  
    - Message:  
      ```
      Hey,

      Please find attached calendar for Undergraduate terms dates 2025/2026.

      Thanks
      ```  
    - Attachments: Attach binary data from "Create ICS File" node.  
    - Credentials: Gmail OAuth2 account  
    - Connect output of "Create ICS File" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Cloudflare Account Required: Add your Cloudflare {ACCOUNT_ID} to the Markdown Conversion Service URL.                                                                                                                        | Sticky Note near Markdown Conversion Service node                                                      |
| Learn more about Cloudflare's Markdown Conversion Service for converting Excel to markdown tables.                                                                                                                           | https://developers.cloudflare.com/workers-ai/markdown-conversion/                                     |
| Learn more about the Information Extractor node for structured AI data extraction.                                                                                                                                             | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.information-extractor |
| Learn more about the Code node for custom scripting and transformations.                                                                                                                                                       | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/                               |
| Learn more about the Convert to File node for creating binary files from data.                                                                                                                                                | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.converttofile/                      |
| This workflow template is designed to automate manual data entry for university term dates and can be adapted for other Excel-based document extraction scenarios using AI and external parsing services.                     | Provided in main workflow description and sticky notes                                                 |
| Join the n8n community for support and discussion.                                                                                                                                                                            | Discord: https://discord.com/invite/XPKeKXeB7d, Forum: https://community.n8n.io/                        |

---

This documentation provides a detailed and structured reference for understanding, reproducing, and modifying the workflow to automate university term dates extraction and calendar creation using AI and external services.