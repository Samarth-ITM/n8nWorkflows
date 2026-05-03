Extract & Process Invoices with Gemini AI, Google Sheets & Gmail Notifications

https://n8nworkflows.xyz/workflows/extract---process-invoices-with-gemini-ai--google-sheets---gmail-notifications-8548


# Extract & Process Invoices with Gemini AI, Google Sheets & Gmail Notifications

### 1. Workflow Overview

This workflow automates the extraction, processing, validation, and storage of invoice data received as chat messages with file uploads. It leverages AI-powered image analysis (Google Gemini), structured data formatting, Google Sheets for database management, Google Drive for archival, and Gmail for notifications. The workflow ensures data quality by validating mandatory fields and preventing duplicates.

Logical blocks:

- **1.1 Input Reception & File Upload**  
  Triggering on chat message receipt, accepting invoice files.

- **1.2 AI Invoice Extraction & Structuring**  
  Using Google Gemini Vision Model and an AI Agent to extract and format invoice data into a strict JSON schema.

- **1.3 Invoice Archival**  
  Uploading the original invoice file to Google Drive for backup.

- **1.4 Invoice Existence Check**  
  Querying Google Sheets to verify if the invoice already exists, to avoid duplication.

- **1.5 Data Preparation & Validation**  
  Structuring AI output into JSON, validating mandatory fields, and deciding workflow path based on validation results.

- **1.6 Data Persistence & Notifications**  
  Adding new invoice data to Google Sheets and sending appropriate Gmail notifications for success, duplicates, or missing data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & File Upload

- **Overview:**  
  This block triggers the workflow when a chat message is received, allowing file uploads of invoices. The uploaded file is simultaneously sent for AI processing and archived in Google Drive.

- **Nodes Involved:**  
  - When chat message received  
  - Analyze image1  
  - Upload invoice to drive

- **Node Details:**

  - **When chat message received**  
    - Type: Langchain chatTrigger (Trigger node)  
    - Config: Accepts file uploads; MIME types allowed: images, text, PDFs  
    - Inputs: External chat message with optional files  
    - Outputs: Triggers downstream nodes with binary file data under property `data0`  
    - Edge cases: Unsupported file types, no file uploaded, webhook errors  
    - Role: Entry point of workflow

  - **Analyze image1**  
    - Type: Langchain googleGemini (AI image analysis)  
    - Config: Model "gemini-1.5-flash" Vision Model analyzes binary `data0`  
    - Prompt: Extracts all invoice data and formats it as listed invoice fields  
    - Inputs: Binary file from trigger node  
    - Outputs: Text data with invoice content in JSON-like structured text  
    - Credentials: Google Gemini API  
    - Edge cases: API timeout, malformed images, unrecognized data

  - **Upload invoice to drive**  
    - Type: Google Drive node  
    - Config: Uploads incoming binary file as "invoice_data" to user's root Drive folder  
    - Inputs: Binary file `data0` from trigger  
    - Outputs: Confirmation of upload  
    - Credentials: Google Drive OAuth2  
    - Edge cases: Drive quota exceeded, auth errors

---

#### 1.2 AI Invoice Extraction & Structuring

- **Overview:**  
  This block takes raw text output from the image analysis and processes it with an AI Agent to produce a strict JSON invoice structure. It uses Google Gemini Chat model as the AI backend.

- **Nodes Involved:**  
  - AI Agent1  
  - Google Gemini Chat Model1  
  - Make data in json structure format

- **Node Details:**

  - **AI Agent1**  
    - Type: Langchain agent  
    - Config: Receives extracted invoice text and system message instructing strict JSON formatting including mandatory and optional invoice fields  
    - Inputs: Text from "Analyze image1" node  
    - Outputs: Structured JSON text under `$json.content.parts[0].text`  
    - Uses Google Gemini Chat Model1 as language model backend  
    - Edge cases: AI misunderstanding prompt, malformed JSON output

  - **Google Gemini Chat Model1**  
    - Type: Langchain lmChatGoogleGemini (AI language model)  
    - Config: Configured to process prompts from AI Agent  
    - Inputs: AI Agent prompt and text  
    - Outputs: JSON invoice data as string  
    - Credentials: Google Gemini API  
    - Edge cases: API failures, rate limits

  - **Make data in json structure format**  
    - Type: Set node  
    - Config: Assigns AI Agent output JSON text to an `output` field for easier downstream usage  
    - Inputs: AI Agent1 JSON output  
    - Outputs: Single JSON object under `output` key  
    - Edge cases: Missing or malformed JSON from AI Agent

---

#### 1.3 Invoice Archival

- **Overview:**  
  This step stores the original invoice file into Google Drive for future reference and audit.

- **Nodes Involved:**  
  - Upload invoice to drive (also part of 1.1)

---

#### 1.4 Invoice Existence Check

- **Overview:**  
  This block checks if the invoice already exists in the Google Sheet database by matching the invoice ID.

- **Nodes Involved:**  
  - Get data  
  - check if Data exist or not in table

- **Node Details:**

  - **Get data**  
    - Type: Google Sheets node  
    - Config: Filters rows where `Entry_ID` matches the current invoice’s `invoice_id` from AI output  
    - Document & Sheet: Specific Google Sheet and sheet tab configured (Expense_Tracker)  
    - Inputs: JSON invoice data from previous step  
    - Outputs: Rows matching invoice ID (if any)  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Sheet access issues, rate limits, no matches

  - **check if Data exist or not in table**  
    - Type: If node  
    - Config: Checks if the output from Get data is empty (no existing invoice) or not  
    - Inputs: Rows from Get data  
    - Outputs: Branch to either new data path (empty) or duplicate notification (exists)  
    - Edge cases: False positives if sheet format changes

---

#### 1.5 Data Preparation & Validation

- **Overview:**  
  This block prepares the invoice JSON payload for insertion and validates mandatory fields before deciding to append or send error notifications.

- **Nodes Involved:**  
  - New data add using payload  
  - Check Mandatory fields  
  - If - check missing field  
  - no missing field - new data add using payload 2

- **Node Details:**

  - **New data add using payload**  
    - Type: Code node  
    - Logic: If Google Sheets returned no existing invoice, use AI JSON payload; else use existing sheet data  
    - Inputs: Output from check if Data exist or not in table  
    - Outputs: Invoice JSON to validate  
    - Edge cases: Mismatched data sources

  - **Check Mandatory fields**  
    - Type: Code node  
    - Logic: Validates presence of mandatory fields (`invoice_id`, `shop_name`, `date`, `Total`, `items`) in invoice JSON  
    - Outputs: JSON with status "ok" if all present or "error" with missing fields list  
    - Edge cases: Missing fields, empty arrays, malformed JSON

  - **If - check missing field**  
    - Type: If node  
    - Logic: Routes workflow based on `status` field from Check Mandatory fields node  
    - True branch: Continue processing  
    - False branch: Send error notification email  
    - Edge cases: Unexpected status values

  - **no missing field - new data add using payload 2**  
    - Type: Code node  
    - Logic: Passes only valid invoice JSON forward for appending; errors propagate to notification  
    - Inputs: Result from If node  
    - Outputs: Validated invoice JSON for Google Sheets append  
    - Edge cases: Status not "ok"

---

#### 1.6 Data Persistence & Notifications

- **Overview:**  
  This final block appends validated invoice data to the Google Sheet and sends Gmail notifications for successful addition, duplicates, or missing data errors.

- **Nodes Involved:**  
  - Append data to sheet  
  - Send successful email  
  - Duplicate entry send mail  
  - Send missing field error on mail

- **Node Details:**

  - **Append data to sheet**  
    - Type: Google Sheets node  
    - Operation: Append new row with mapped invoice fields including mandatory and optional fields  
    - Inputs: Validated invoice JSON from no missing field - new data add using payload 2  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Sheet access issues, data mapping errors

  - **Send successful email**  
    - Type: Gmail node  
    - Config: Sends HTML formatted confirmation email with invoice details after successful append  
    - Inputs: Invoice JSON from Append data to sheet node  
    - Credentials: Gmail OAuth2  
    - Edge cases: Mail delivery failure

  - **Duplicate entry send mail**  
    - Type: Gmail node  
    - Config: Sends HTML email warning about duplicate invoice detected, no data changed  
    - Inputs: Invoice JSON from check if Data exist or not in table node (duplicate branch)  
    - Credentials: Gmail OAuth2  
    - Edge cases: Mail delivery failure

  - **Send missing field error on mail**  
    - Type: Gmail node  
    - Config: Sends error notification listing missing mandatory fields  
    - Inputs: JSON error from If - check missing field node (false branch)  
    - Credentials: Gmail OAuth2  
    - Edge cases: Mail delivery failure

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                                   | Input Node(s)                      | Output Node(s)                            | Sticky Note                                                                                                      |
|-----------------------------------|----------------------------------|-------------------------------------------------|----------------------------------|------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| When chat message received         | Langchain chatTrigger            | Workflow trigger; receives chat and file upload | —                                | Analyze image1, Upload invoice to drive  | Entry point; triggers workflow on chat message with file upload                                                  |
| Analyze image1                    | Langchain googleGemini           | Extracts invoice data from image using Gemini   | When chat message received        | AI Agent1                                | Uses Gemini Vision to extract structured text from invoice image                                                |
| Upload invoice to drive           | Google Drive                     | Stores original invoice file in Google Drive    | When chat message received        | —                                        | Uploads invoice file to Drive for archival                                                                       |
| AI Agent1                        | Langchain agent                  | Converts extracted text into strict JSON format | Analyze image1                   | Make data in json structure format       | AI processing step; formats invoice data strictly as JSON                                                       |
| Google Gemini Chat Model1         | Langchain lmChatGoogleGemini     | AI language model backend                        | AI Agent1 (ai_languageModel)      | AI Agent1                                | Powers AI Agent with Gemini Chat model                                                                            |
| Make data in json structure format| Set                             | Packages AI output into one JSON object          | AI Agent1                        | Get data                                | Organizes AI output into `output` JSON object                                                                    |
| Get data                         | Google Sheets                   | Checks if invoice exists in Google Sheets        | Make data in json structure format | check if Data exist or not in table     | Queries Google Sheet for invoice ID to prevent duplicates                                                        |
| check if Data exist or not in table| If                             | Decides if invoice is new or duplicate           | Get data                        | New data add using payload, Duplicate entry send mail | Checks existence of invoice in sheet                                                                              |
| New data add using payload       | Code                            | Chooses invoice data payload for next steps      | check if Data exist or not in table | Check Mandatory fields                   | Ensures invoice JSON payload is ready for validation                                                             |
| Check Mandatory fields           | Code                            | Validates presence of mandatory invoice fields   | New data add using payload        | If - check missing field                  | Validates mandatory fields like invoice_id, shop_name, date, Total, items                                        |
| If - check missing field         | If                              | Routes workflow based on validation result       | Check Mandatory fields            | no missing field - new data add using payload 2, Send missing field error on mail | Decision point for valid vs incomplete invoice data                                                               |
| no missing field - new data add using payload 2 | Code                | Passes validated invoice JSON forward             | If - check missing field (true)   | Append data to sheet                      | Passes clean invoice data for final append                                                                        |
| Append data to sheet             | Google Sheets                   | Appends new invoice row to Google Sheet          | no missing field - new data add using payload 2 | Send successful email                  | Saves validated invoice data into expense tracker sheet                                                          |
| Send successful email            | Gmail                          | Sends success notification email                  | Append data to sheet             | —                                        | Confirms invoice addition with detailed HTML email                                                               |
| Duplicate entry send mail        | Gmail                          | Sends duplicate invoice alert email               | check if Data exist or not in table (false) | —                                        | Alerts user about duplicate invoice detected                                                                      |
| Send missing field error on mail | Gmail                          | Sends missing mandatory fields error email        | If - check missing field (false)  | —                                        | Notifies user of missing mandatory invoice fields                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "When chat message received"**  
   - Type: Langchain chatTrigger  
   - Configure webhook to receive chat messages with file uploads enabled  
   - Set allowed MIME types: images, text, PDF  
   - Position: Start of workflow

2. **Add "Analyze image1" Node**  
   - Type: Langchain googleGemini  
   - Connect input from "When chat message received"  
   - Set model to "gemini-1.5-flash" vision model  
   - Operation: Analyze binary file from property `data0`  
   - Prompt: Extract all invoice data into formatted structured output  
   - Add Google Gemini API credentials

3. **Add "Upload invoice to drive" Node**  
   - Type: Google Drive  
   - Connect input from "When chat message received" (same binary file)  
   - Set upload folder to Drive root or desired folder  
   - Use Google Drive OAuth2 credentials

4. **Add "AI Agent1" Node**  
   - Type: Langchain agent  
   - Connect input from "Analyze image1"  
   - Configure system message with strict JSON schema for invoice data, disallowing `\n` and `\`  
   - Use "Google Gemini Chat Model1" as language model backend  
   - Add Google Gemini API credentials

5. **Add "Google Gemini Chat Model1" Node**  
   - Type: Langchain lmChatGoogleGemini  
   - Connect as `ai_languageModel` input to "AI Agent1"  
   - Use same Google Gemini API credentials

6. **Add "Make data in json structure format" Node**  
   - Type: Set  
   - Connect input from "AI Agent1"  
   - Assign AI output JSON text to a single field called `output`

7. **Add "Get data" Node**  
   - Type: Google Sheets (read)  
   - Connect input from "Make data in json structure format"  
   - Configure document ID and sheet name for invoice database  
   - Set filter: lookup `Entry_ID` column equals `={{ $json.output.invoice_id }}`  
   - Use Google Sheets OAuth2 credentials

8. **Add "check if Data exist or not in table" Node**  
   - Type: If  
   - Connect input from "Get data"  
   - Condition: Check if returned JSON is empty (no existing invoice)  
   - Output true branch: "New data add using payload"  
   - Output false branch: "Duplicate entry send mail"

9. **Add "New data add using payload" Node**  
   - Type: Code  
   - Connect input from "check if Data exist or not in table"  
   - Logic: If no existing data, use fresh invoice JSON from "Make data in json structure format" else use existing sheet data

10. **Add "Check Mandatory fields" Node**  
    - Type: Code  
    - Connect input from "New data add using payload"  
    - Validate mandatory fields: `invoice_id`, `shop_name`, `date`, `Total`, and `items`  
    - Output status and missing fields list

11. **Add "If - check missing field" Node**  
    - Type: If  
    - Connect input from "Check Mandatory fields"  
    - Condition: `status === 'ok'`  
    - True branch: "no missing field - new data add using payload 2"  
    - False branch: "Send missing field error on mail"

12. **Add "no missing field - new data add using payload 2" Node**  
    - Type: Code  
    - Connect input from "If - check missing field" true branch  
    - Pass only valid invoice JSON forward

13. **Add "Append data to sheet" Node**  
    - Type: Google Sheets (append)  
    - Connect input from "no missing field - new data add using payload 2"  
    - Map invoice JSON fields to sheet columns (`Entry_ID`, `Date`, `Shop_Name`, `Item_Name`, `Quantity`, `Total_Amount`, etc.)  
    - Set operation to append row  
    - Use Google Sheets OAuth2 credentials

14. **Add "Send successful email" Node**  
    - Type: Gmail  
    - Connect input from "Append data to sheet"  
    - Configure recipient email, subject, and HTML body using invoice data placeholders  
    - Use Gmail OAuth2 credentials

15. **Add "Duplicate entry send mail" Node**  
    - Type: Gmail  
    - Connect input from "check if Data exist or not in table" false branch  
    - Configure recipient email, subject, and HTML body warning about duplicate invoice  
    - Use Gmail OAuth2 credentials

16. **Add "Send missing field error on mail" Node**  
    - Type: Gmail  
    - Connect input from "If - check missing field" false branch  
    - Configure recipient email, subject, and message listing missing mandatory fields  
    - Use Gmail OAuth2 credentials

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow relies on Google Gemini (PaLM API) for both image analysis and natural language processing. Proper API access and credentials setup are required.                                                                | https://developers.generativeai.google/                                                                                                                                        |
| Google Sheets acts as the primary invoice database. The sheet must have columns matching the mapped fields, especially `Entry_ID` for duplicate detection.                                                                    | Google Sheets documentation: https://support.google.com/docs                                                                                                                  |
| Emails are sent via Gmail OAuth2 credentials. Ensure the Gmail account used has appropriate sending permissions and API access.                                                                                               | Gmail API: https://developers.google.com/gmail/api                                                                                                                             |
| The AI Agent’s system prompt is critical to ensuring the JSON output conforms to the schema. Adjust the prompt carefully if invoice formats or fields change.                                                                  | System prompt located in AI Agent1 node                                                                                                                                        |
| The workflow validates mandatory fields to avoid incomplete data entries, and branches accordingly to maintain database integrity and user notification.                                                                       | Validation in "Check Mandatory fields" node                                                                                                                                     |
| The original invoice files are preserved in Google Drive for audit and reference, providing a backup independent from the processed data.                                                                                    | Upload node "Upload invoice to drive"                                                                                                                                           |
| For large volume or complex invoices, consider API rate limits and potential timeouts in Gemini and Gmail nodes. Add error handling or retries if necessary.                                                                   | Monitor API usage and errors in n8n execution logs                                                                                                                              |
| The workflow uses consistent JSON field naming conventions; ensure Google Sheets column names and email placeholders match exactly to avoid data mismatches.                                                                  | See Append data to sheet node’s column mapping                                                                                                                                |
| Gmail notifications use rich HTML formatting for clarity and professionalism. Customize templates as needed for localization or branding.                                                                                    | Email body templates in Send successful email and Duplicate entry send mail nodes                                                                                                |
| Sticky notes in the workflow provide detailed explanations per node, useful for onboarding or maintenance.                                                                                                                     | Sticky notes cover nodes from trigger to final email nodes                                                                                                                      |

---

**Disclaimer:** The text provided derives solely from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.