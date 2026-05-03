Extract Legal Contract Data & Send Alerts with VLM Run, Google Workspace & Slack

https://n8nworkflows.xyz/workflows/extract-legal-contract-data---send-alerts-with-vlm-run--google-workspace---slack-9328


# Extract Legal Contract Data & Send Alerts with VLM Run, Google Workspace & Slack

### 1. Workflow Overview

This workflow automates the extraction of key data from legal contract documents uploaded to Google Drive, stores the extracted data in Google Sheets, sends alerts to a Slack channel, and creates calendar events for contract milestones in Google Calendar. It is designed for legal, real estate, or contract management teams who need automated tracking of contracts for timely reviews and record-keeping.

**Logical Blocks:**

- **1.1 Input Reception & Download**: Watches a specific Google Drive folder for new contract uploads, then downloads these files for processing.

- **1.2 AI Data Extraction**: Uses the VLM Run AI agent to parse contract images or PDFs and extract structured legal data.

- **1.3 Data Formatting & Storage**: Cleans and organizes the AI-extracted data, then appends it to a Google Sheets document serving as a contract expense database.

- **1.4 Notifications & Calendar Integration**: Sends contract summary notifications to a Slack channel and creates Google Calendar events for key contract dates (effective, termination, and renewal reminders).

- **1.5 Manual Webhook Input**: Supports direct contract data input via webhook for alternative or manual contract data submissions.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Download

- **Overview:**  
  Automatically detects new contract files uploaded to a specific Google Drive folder, then downloads them to be processed by the AI extraction.

- **Nodes Involved:**  
  - Monitor Contract Uploads (Google Drive Trigger)  
  - Download Contract File (Google Drive)  
  - 📁 Input Processing Documentation (Sticky Note)

- **Node Details:**

  1. **Monitor Contract Uploads**  
     - *Type:* Google Drive Trigger  
     - *Role:* Watches a designated Google Drive folder (ID: `1S6baavqJn98MjUlbB6KtmARCWuWEekIZ`) for new files created, polling every minute.  
     - *Config Highlights:* Trigger only on specific folder, event type `fileCreated`.  
     - *Credentials:* Google Drive OAuth2.  
     - *Outputs:* Emits metadata for each new file detected.  
     - *Potential Failures:* Authentication issues, API rate limits, missing folder access.  

  2. **Download Contract File**  
     - *Type:* Google Drive Node  
     - *Role:* Downloads the file detected by the trigger as binary data named `data`.  
     - *Config Highlights:* Uses file ID from previous node’s JSON.  
     - *Credentials:* Same Google Drive OAuth2 as above.  
     - *Inputs:* File ID from "Monitor Contract Uploads".  
     - *Outputs:* Binary content of the contract file for AI processing.  
     - *Potential Failures:* File access denied, file deleted before download, network timeouts.

  3. **📁 Input Processing Documentation**  
     - *Type:* Sticky Note  
     - *Role:* Provides contextual documentation explaining that this block monitors and downloads contract files (images, PDFs, scanned docs).  
     - *No inputs or outputs.*

---

#### Block 1.2: AI Data Extraction

- **Overview:**  
  Processes the downloaded contract file with the VLM Run agent to extract structured contract details using OCR and AI parsing.

- **Nodes Involved:**  
  - VLM Run ContractParser (VLM Run node)  
  - 🤖 AI Extraction Documentation (Sticky Note)

- **Node Details:**

  1. **VLM Run ContractParser**  
     - *Type:* @vlm-run/n8n-nodes-vlmrun.vlmRun (Custom AI Agent node)  
     - *Role:* Executes an AI agent prompt that extracts contract data fields from the PDF/image file.  
     - *Config Highlights:*  
       - Prompt requests JSON output with fields: `contract_id, title, parties (with role), property_address, effective_date, termination_date, rent_amount, security_deposit, payment_terms, governing_law`.  
       - Normalizes dates to `YYYY-MM-DD` and numeric amounts with currency.  
       - Returns `null` for missing fields.  
       - Uses a callback URL to securely receive processed output.  
     - *Credentials:* VLM Run API credentials.  
     - *Inputs:* Binary file from "Download Contract File".  
     - *Outputs:* JSON response with extracted contract data.  
     - *Potential Failures:* AI service downtime, malformed input files, callback URL misconfiguration, parsing errors.

  2. **🤖 AI Extraction Documentation**  
     - *Type:* Sticky Note  
     - *Role:* Explains the AI extraction method, expected output fields, and features like OCR and handling poor-quality images.

---

#### Block 1.3: Data Formatting & Storage

- **Overview:**  
  Structures the AI-extracted data into a clean format suitable for spreadsheet storage and appends this data to a centralized Google Sheet database.

- **Nodes Involved:**  
  - Format Contract Data (Set node)  
  - Save to Expense Database (Google Sheets append)  
  - 📊 Storage Documentation (Sticky Note)

- **Node Details:**

  1. **Format Contract Data**  
     - *Type:* Set node  
     - *Role:* Transforms raw AI JSON output into spreadsheet-friendly fields with clear names: Contract ID, Title, Parties (concatenated with roles), Effective Date, Termination Date.  
     - *Config Highlights:*  
       - Uses expressions to safely extract fields from nested JSON.  
       - Concatenates parties array into a semicolon-separated string with roles.  
       - Keeps only the set fields for output.  
     - *Inputs:* JSON from VLM Run node or webhook.  
     - *Outputs:* Clean JSON with contract data fields.  
     - *Potential Failures:* Expression errors if data missing or malformed.  

  2. **Save to Expense Database**  
     - *Type:* Google Sheets node  
     - *Role:* Appends the formatted contract data as a new row to a specified Google Sheets document (ID: `1lg0aJKvd7E2pbhumHNjcgxUfEQKvlBs9h1zZbhSeqas`) and sheet (gid=0).  
     - *Config Highlights:* Uses defined column mapping for Contract ID, Title, Parties, Effective Date, Termination Date.  
     - *Credentials:* Google Sheets OAuth2.  
     - *Inputs:* Data from "Format Contract Data".  
     - *Outputs:* Confirmation of append operation, can include links or metadata.  
     - *Potential Failures:* Sheet access denied, API limits, schema mismatch.  

  3. **📊 Storage Documentation**  
     - *Type:* Sticky Note  
     - *Role:* Describes the purpose of structured storage, fields saved, and benefits like real-time tracking and mobile access.

---

#### Block 1.4: Notifications & Calendar Integration

- **Overview:**  
  Notifies the team via Slack about new contracts and creates Google Calendar all-day events for contract milestones including effective date, termination date, and renewal reminders.

- **Nodes Involved:**  
  - Send a message (Slack node)  
  - Prepare Calendar Events (Code node)  
  - Create an event (Google Calendar)  
  - 📊 Storage Documentation1 (Sticky Note)

- **Node Details:**

  1. **Send a message**  
     - *Type:* Slack node  
     - *Role:* Sends a formatted message to a Slack channel summarizing the newly processed contract.  
     - *Config Highlights:*  
       - Message includes Contract ID, Title, Parties, Effective Date, Termination Date, and a Drive link if available.  
       - Posts to channel ID `C081Z0KL546`.  
     - *Credentials:* Slack OAuth2 API.  
     - *Inputs:* Data from "Save to Expense Database".  
     - *Outputs:* Slack API response.  
     - *Potential Failures:* Slack API rate limits, invalid channel, auth failures.

  2. **Prepare Calendar Events**  
     - *Type:* Code (Function) node  
     - *Role:* Reads formatted contract data and creates up to three calendar event objects: Effective Date, Termination Date, and a Renewal Review Reminder 60 days before termination.  
     - *Config Highlights:*  
       - Converts date strings to all-day event start/end in `YYYY-MM-DD`.  
       - Creates event summaries and descriptions with contract info and links.  
       - Supports missing dates gracefully by skipping event creation.  
     - *Inputs:* Data from "Format Contract Data".  
     - *Outputs:* Array of calendar event JSON objects for next node.  
     - *Potential Failures:* Date parsing errors, empty fields.

  3. **Create an event**  
     - *Type:* Google Calendar node  
     - *Role:* Creates Google Calendar all-day events for each event object created in the previous node.  
     - *Config Highlights:*  
       - Uses calendar `sayonaraistata@gmail.com`.  
       - Sets event summary, description, start, end, and all-day flag.  
     - *Credentials:* Google Calendar OAuth2.  
     - *Inputs:* Events array from "Prepare Calendar Events".  
     - *Outputs:* Event creation confirmation.  
     - *Potential Failures:* Calendar API errors, permission issues, invalid dates.

  4. **📊 Storage Documentation1**  
     - *Type:* Sticky Note  
     - *Role:* Explains notification methods and calendar event details for contract milestone tracking.

---

#### Block 1.5: Manual Webhook Input

- **Overview:**  
  Allows manual or external systems to POST contract data directly into the workflow via a webhook, bypassing Google Drive upload trigger.

- **Nodes Involved:**  
  - Receive Contract (Webhook node)  
  - Format Contract Data (Set node, reused from Block 1.3)

- **Node Details:**

  1. **Receive Contract**  
     - *Type:* Webhook node  
     - *Role:* Accepts POST requests at path `b905e71d-8ea5-4fc2-a773-b0f92e5398e4` to input contract data manually or from external systems.  
     - *Config Highlights:* HTTP POST method, no special authentication configured by default.  
     - *Outputs:* Raw JSON payload forwarded to "Format Contract Data".  
     - *Potential Failures:* Unauthorized access if not secured, malformed JSON input.

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                               | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                          |
|----------------------------|--------------------------------|----------------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| 📁 Input Processing Documentation | Sticky Note                   | Documents Input Reception & Download Process | —                           | —                           | Explains monitoring folder, trigger on new uploads, supported formats                             |
| Monitor Contract Uploads    | Google Drive Trigger           | Watches Drive folder for new contract files  | —                           | Download Contract File       | Monitors Google Drive folder for new receipt uploads and triggers processing automatically        |
| Download Contract File      | Google Drive                  | Downloads files for AI processing             | Monitor Contract Uploads     | VLM Run ContractParser       | Downloads receipt files from Google Drive for AI processing                                      |
| 🤖 AI Extraction Documentation | Sticky Note                   | Documents AI Extraction step                   | —                           | —                           | Explains VLM Run extraction fields and features                                                  |
| VLM Run ContractParser      | VLM Run Node                  | Extracts structured contract data via AI      | Download Contract File       | Format Contract Data         |                                                                                                    |
| Format Contract Data        | Set                           | Cleans & structures AI data for storage       | VLM Run ContractParser, Receive Contract | Save to Expense Database     | Transforms AI-extracted receipt data into clean, structured format for spreadsheet storage        |
| Save to Expense Database    | Google Sheets                 | Appends contract data to Google Sheets         | Format Contract Data         | Send a message, Prepare Calendar Events | Automatically saves extracted receipt data to Google Sheets for expense tracking                   |
| Send a message              | Slack                         | Posts contract summary to Slack channel        | Save to Expense Database     | —                           |                                                                                                    |
| Prepare Calendar Events     | Code (Function)               | Creates calendar event objects for contract dates | Save to Expense Database     | Create an event             |                                                                                                    |
| Create an event             | Google Calendar               | Creates Google Calendar events                  | Prepare Calendar Events      | —                           |                                                                                                    |
| 📊 Data Storage             | Sticky Note                   | Documents Data Storage process                   | —                           | —                           | Explains structured storage benefits and fields                                                  |
| 📊 Storage Documentation1   | Sticky Note                   | Documents Notifications & Calendar integration | —                           | —                           | Explains Slack notifications and Google Calendar event creation                                  |
| Receive Contract            | Webhook                       | Receives manual/external contract data          | —                           | Format Contract Data         |                                                                                                    |
| Sticky Note3                | Sticky Note                   | Instructions for setting the VLM Run callback URL | —                           | —                           | Advises to use production URL in VLM Run callback, not localhost/dev URLs                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Drive Trigger Node ("Monitor Contract Uploads")**  
   - Type: Google Drive Trigger  
   - Event: `fileCreated`  
   - Poll frequency: Every minute  
   - Folder: Set to the desired Google Drive folder ID to watch (e.g., your contract upload folder)  
   - Credentials: Configure Google Drive OAuth2 with access to that folder  

2. **Create a Google Drive Node ("Download Contract File")**  
   - Type: Google Drive  
   - Operation: Download  
   - File ID: Use expression `{{$json["id"]}}` from previous node output  
   - Binary Property Name: `data`  
   - Credentials: Same Google Drive OAuth2 as above  
   - Connect from "Monitor Contract Uploads"  

3. **Create a VLM Run Node ("VLM Run ContractParser")**  
   - Type: Custom VLM Run node  
   - Operation: `executeAgent`  
   - Agent Prompt: Use detailed prompt to extract contract JSON fields (contract_id, title, parties, dates, amounts, etc.) as described  
   - Agent Callback URL: Set to your n8n webhook URL (production URL, not localhost)  
   - Credentials: VLM Run API credentials  
   - Connect from "Download Contract File"  

4. **Create a Webhook Node ("Receive Contract")**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., a UUID or descriptive string)  
   - No credential required unless securing externally  
   - This node allows manual or external contract data input into the workflow  

5. **Create a Set Node ("Format Contract Data")**  
   - Type: Set  
   - Keep Only Set: Enabled  
   - Set fields:  
     - `Contract ID` = `{{$json.body.response.contract_id}}`  
     - `Title` = `{{$json.body.response.title}}`  
     - `Parties` = `{{$json.body.parties && $json.body.parties.length ? $json.body.parties.map(p => p.name + " (" + p.role + ")").join("; ") : ""}}`  
     - `Effective Date` = `{{$json.body.response.effective_date}}`  
     - `Termination Date` = `{{$json.body.response.termination_date}}`  
   - Connect from "VLM Run ContractParser" and from "Receive Contract" (parallel input)  

6. **Create a Google Sheets Node ("Save to Expense Database")**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Your Google Sheets ID for contract storage  
   - Sheet Name: e.g., `Sheet1` or by GID  
   - Columns mapping:  
     - Contract ID → `{{$json["Contract ID"]}}`  
     - Title → `{{$json["Title"]}}`  
     - Parties → `{{$json["Parties"]}}`  
     - Effective Date → `{{$json["Effective Date"]}}`  
     - Termination Date → `{{$json["Termination Date"]}}`  
   - Credentials: Google Sheets OAuth2 authorized for the document  
   - Connect from "Format Contract Data"  

7. **Create a Slack Node ("Send a message")**  
   - Type: Slack  
   - Channel: Set to your team's Slack channel ID  
   - Text: Use template with contract info as shown, e.g.:  
     ```
     =*New Contract Processed* 📄
     • Contract ID: {{$node["Format Contract Data"].json["Contract ID"] || "N/A"}}
     • Title: {{$node["Format Contract Data"].json["Title"] || "N/A"}}
     • Parties: {{$node["Format Contract Data"].json["Parties"] || "N/A"}}
     • Effective: {{$node["Format Contract Data"].json["Effective Date"] || "N/A"}}
     • Termination: {{$node["Format Contract Data"].json["Termination Date"] || "N/A"}}
     🔗 {{$node["Format Contract Data"].json["Drive Link"] || $node["Save to Expense Database"].json["driveLink"] || "No drive link available"}}
     ```  
   - Credentials: Slack OAuth2 API with posting rights  
   - Connect from "Save to Expense Database"  

8. **Create a Code Node ("Prepare Calendar Events")**  
   - Type: Function (JavaScript)  
   - Code: Implement logic to generate event objects for Effective Date, Termination Date, and Renewal Reminder (60 days prior), using the formatted contract data  
   - Connect from "Save to Expense Database"  

9. **Create a Google Calendar Node ("Create an event")**  
   - Type: Google Calendar  
   - Calendar ID: Your Google Calendar email or ID  
   - Set:  
     - Summary: `{{$json["summary"]}}`  
     - Description: `{{$json["description"]}}`  
     - Start Date: `{{$json["start"]}}`  
     - End Date: `{{$json["end"]}}`  
     - All-day event enabled  
   - Credentials: Google Calendar OAuth2 authorized for this calendar  
   - Connect from "Prepare Calendar Events"  

10. **Add Sticky Notes**  
    - Add descriptive sticky notes at relevant workflow positions for documentation: explaining input processing, AI extraction, data storage, notifications, and webhook setup.  

11. **Set VLM Run Callback URL**  
    - Ensure that the VLM Run agent is configured with the production n8n webhook URL from the "Receive Contract" node (not localhost or development URLs) to receive AI results securely.  

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Make sure to paste the **Production URL** of your n8n Webhook node into the **Callback URL** field in VLM Run. Do **not** use localhost or development URLs, as they won’t be reachable by VLM Run’s servers. | Sticky Note near webhook and VLM Run nodes                                                          |
| Workflow supports multiple input methods: automated via Google Drive trigger and manual/external via webhook.| Enables flexibility for contract ingestion from diverse sources                                      |
| Slack notifications enable team visibility for all processed contracts.                                      | Useful for collaboration and audit trails                                                             |
| Google Calendar events automate key milestone tracking (effective date, termination date, renewal reminders). | Helps prevent contract lapses and missed renewals                                                    |

---

**Disclaimer:** The text provided is generated exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.