Generate Medical Reports from Emails with Gemini AI & Google Workspace

https://n8nworkflows.xyz/workflows/generate-medical-reports-from-emails-with-gemini-ai---google-workspace-6487


# Generate Medical Reports from Emails with Gemini AI & Google Workspace

### 1. Workflow Overview

This workflow automates the generation of medical reports from incoming emails using Google Workspace and Gemini AI. It is designed for healthcare professionals or administrative staff who receive medical information via email and want to automatically extract key patient data, generate structured reports, and share finalized PDFs efficiently.

Logical blocks included:

- **1.1 Input Reception:** Watches a Gmail inbox for new emails from a specific doctor.
- **1.2 Content Extraction:** Extracts raw email content for AI processing.
- **1.3 AI Processing:** Uses Google Gemini AI via n8n’s Langchain agent to extract structured patient data.
- **1.4 Data Parsing:** Cleans and parses the AI JSON response.
- **1.5 Data Recording:** Appends extracted data to a Google Sheet.
- **1.6 Document Preparation:** Copies a Google Docs template, replaces placeholders with extracted data.
- **1.7 PDF Exporting:** Exports the updated Google Doc as a PDF file.
- **1.8 File Naming:** Renames the PDF with patient name and current date.
- **1.9 Email Sending:** Sends the final PDF medical report via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Triggers workflow execution on incoming emails from a specified doctor’s email address.
- **Nodes Involved:** Gmail Trigger
- **Node Details:**
  - **Gmail Trigger**
    - Type: Gmail Trigger
    - Configuration: Watches emails from "your-doctor-email@example.com" every hour.
    - Input: New Gmail messages filtered by sender.
    - Output: Email data including raw text.
    - Edge Cases: Email format variations, no new emails, Gmail API auth errors.
    - Sticky Note: "🔔 *Triggers* the workflow when a new email is received from a specific sender (e.g. a doctor)."

#### 2.2 Content Extraction

- **Overview:** Extracts the raw textual content from the email for AI analysis.
- **Nodes Involved:** Edit Fields
- **Node Details:**
  - **Edit Fields (Set Node)**
    - Type: Set (field assignment)
    - Configuration: Creates a new field `content` and assigns it the email’s raw text body.
    - Input: Output from Gmail Trigger.
    - Output: JSON including `content` field.
    - Edge Cases: Missing or malformed email text.
    - Sticky Note: "🛠 *Extracts the raw email content* into a new field called `content` for AI processing."

#### 2.3 AI Processing

- **Overview:** Uses Google Gemini AI model to extract structured patient and medical data from the raw email content.
- **Nodes Involved:** AI Agent, Google Gemini Chat Model
- **Node Details:**
  - **AI Agent**
    - Type: Langchain Agent
    - Configuration: Defines prompt instructing AI to extract patient_name, doctor_name (nullable), diagnosis, phone_number (must start with 0111), and date (today's date).
    - Key Expression: Uses `{{$json.content}}` as input to prompt.
    - Output: Raw AI JSON response with extracted fields.
    - Edge Cases: AI misinterpretation, incomplete data, prompt failures.
  - **Google Gemini Chat Model**
    - Type: Langchain Google Gemini LM Chat Model
    - Configuration: Uses model `models/gemini-2.0-flash-lite`.
    - Role: Provides AI language model for the agent.
    - Credentials: Google Gemini (PaLM) API.
    - Output: AI response to agent.
    - Sticky Note: "🤖 *Processes the text using AI* to extract structured information like patient name, doctor name, etc."

#### 2.4 Data Parsing

- **Overview:** Cleans AI response by removing markdown code blocks and converts it into usable JSON.
- **Nodes Involved:** Code (JavaScript)
- **Node Details:**
  - **Code**
    - Type: Code Node (JavaScript)
    - Configuration: Removes any triple backtick markdown formatting around JSON and parses the cleaned string.
    - Input: Raw AI output from AI Agent.
    - Output: Parsed JSON with patient_name, doctor_name, diagnosis, phone_number, date.
    - Edge Cases: Parsing errors if AI returns invalid JSON.
    - Sticky Note: "🧹 *Cleans and parses* the AI response by removing markdown code formatting and turning it into usable JSON."

#### 2.5 Data Recording

- **Overview:** Stores extracted patient and diagnosis data in a Google Sheet for record-keeping.
- **Nodes Involved:** Append row in sheet
- **Node Details:**
  - **Append row in sheet (Google Sheets)**
    - Type: Google Sheets Append
    - Configuration: Appends a new row to a configured sheet with columns Date, Diagnoses, Patient Name, Doctor's Name, Patient Phone Num.
    - Input: Parsed JSON from Code node.
    - Output: Confirmation and details of appended row.
    - Credentials: Google Sheets OAuth2.
    - Edge Cases: Sheet access permissions, API rate limits.
    - Sticky Note: "📄 *Adds the extracted data* (patient info, diagnosis, etc.) to a specific Google Sheet row."

#### 2.6 Document Preparation

- **Overview:** Copies a Google Docs template for the patient, then replaces placeholders with extracted data.
- **Nodes Involved:** Copy file, Update a document1
- **Node Details:**
  - **Copy file (Google Drive)**
    - Type: Google Drive Copy File
    - Configuration: Copies a predefined Google Docs template, naming the copy with the patient’s name.
    - Input: Confirmation of appended data.
    - Output: New document ID.
    - Credentials: Google Drive OAuth2.
    - Edge Cases: Copy failures, permission issues.
    - Sticky Note: "📁 *Makes a copy* of the Google Docs medical report template for the current patient."
  - **Update a document1 (Google Docs)**
    - Type: Google Docs Update
    - Configuration: Inserts tables and replaces placeholders (e.g., `{{date}}`, `{{patient_name}}`) with real data from Google Sheet row.
    - Input: Copy file node output.
    - Output: Updated Google Doc.
    - Credentials: Google Docs OAuth2.
    - Edge Cases: Placeholder mismatches, API limits.
    - Sticky Note: "📝 *Replaces placeholders* (like patient name/date) in the copied Google Doc with real data."

#### 2.7 PDF Exporting

- **Overview:** Exports the updated Google Docs document as a PDF file.
- **Nodes Involved:** HTTP Request
- **Node Details:**
  - **HTTP Request**
    - Type: HTTP Request
    - Configuration: Calls Google Drive API export endpoint to convert Google Doc to PDF MIME type.
    - Input: Updated document ID from previous node.
    - Output: Binary PDF data.
    - Credentials: Google Drive OAuth2.
    - Edge Cases: API export failures, network issues.
    - Sticky Note: "📤 *Exports the updated Google Doc* as a PDF via Google Drive API."

#### 2.8 File Naming

- **Overview:** Renames the exported PDF file with the patient's name and the current date.
- **Nodes Involved:** Code1
- **Node Details:**
  - **Code1**
    - Type: Code Node (JavaScript)
    - Configuration: Sets binary data file name to `{PatientName}_{YYYY-MM-DD}.pdf` and MIME type to `application/pdf`.
    - Input: PDF binary data from HTTP Request.
    - Output: Properly named PDF file metadata.
    - Edge Cases: Missing patient name, date formatting issues.
    - Sticky Note: "📛 *Renames the PDF file* using patient name and date, and attaches metadata."

#### 2.9 Email Sending

- **Overview:** Sends the finalized PDF medical report as an email attachment.
- **Nodes Involved:** Send a message
- **Node Details:**
  - **Send a message (Gmail)**
    - Type: Gmail Send
    - Configuration: Sends email to a fixed recipient with subject including patient name, attaches the PDF file.
    - Input: Binary PDF file from Code1.
    - Output: Email sending confirmation.
    - Credentials: Gmail OAuth2.
    - Edge Cases: Email sending failure, invalid recipient.
    - Sticky Note: "✉️ *Sends the final PDF* (medical report) to the recipient via Gmail."

---

### 3. Summary Table

| Node Name            | Node Type                      | Functional Role                              | Input Node(s)       | Output Node(s)          | Sticky Note                                                                                  |
|----------------------|--------------------------------|----------------------------------------------|---------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Gmail Trigger        | Gmail Trigger                  | Triggers workflow on new email from doctor   | -                   | Edit Fields             | 🔔 *Triggers* the workflow when a new email is received from a specific sender (e.g. a doctor). |
| Edit Fields          | Set                           | Extract raw email content into `content` field | Gmail Trigger       | AI Agent                | 🛠 *Extracts the raw email content* into a new field called `content` for AI processing.      |
| AI Agent             | Langchain Agent               | Extracts structured data from email content  | Edit Fields          | Code                    | 🤖 *Processes the text using AI* to extract structured information like patient name, doctor name, etc. |
| Google Gemini Chat Model | Langchain Google Gemini LM Chat | Provides AI model for text understanding      | -                   | AI Agent                |                                                                                              |
| Code                 | Code (JavaScript)             | Cleans and parses AI response to JSON         | AI Agent             | Append row in sheet      | 🧹 *Cleans and parses* the AI response by removing markdown code formatting and turning it into usable JSON. |
| Append row in sheet  | Google Sheets Append          | Adds extracted data to Google Sheet            | Code                 | Copy file               | 📄 *Adds the extracted data* (patient info, diagnosis, etc.) to a specific Google Sheet row.  |
| Copy file            | Google Drive Copy File        | Copies Google Docs template for patient        | Append row in sheet  | Update a document1      | 📁 *Makes a copy* of the Google Docs medical report template for the current patient.          |
| Update a document1   | Google Docs Update            | Replaces placeholders in copied document       | Copy file            | HTTP Request            | 📝 *Replaces placeholders* (like patient name/date) in the copied Google Doc with real data.  |
| HTTP Request         | HTTP Request                  | Exports updated Google Doc as PDF               | Update a document1   | Code1                   | 📤 *Exports the updated Google Doc* as a PDF via Google Drive API.                            |
| Code1                | Code (JavaScript)             | Renames PDF file with patient name and date    | HTTP Request         | Send a message          | 📛 *Renames the PDF file* using patient name and date, and attaches metadata.                 |
| Send a message       | Gmail Send                    | Sends final PDF medical report via email       | Code1                | -                       | ✉️ *Sends the final PDF* (medical report) to the recipient via Gmail.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**
   - Type: Gmail Trigger
   - Configure credential with Gmail OAuth2.
   - Set filter: sender = "your-doctor-email@example.com".
   - Set polling mode: every hour.

2. **Create Set node (Edit Fields)**
   - Type: Set
   - Add a new string field `content`.
   - Set value: `={{ $json.text }}` (extract raw email text).
   - Connect Gmail Trigger → Edit Fields.

3. **Create Langchain Google Gemini Chat Model node**
   - Type: Langchain LM Chat Google Gemini
   - Configure credential: Google Gemini (PaLM) API.
   - Model: `models/gemini-2.0-flash-lite`.

4. **Create Langchain Agent node (AI Agent)**
   - Type: Langchain Agent
   - Set prompt type: Define
   - Prompt text: extract patient_name, doctor_name (nullable), diagnosis, phone_number (starts 0111), date (today’s date).
   - Use expression: `{{ $json.content }}` as input.
   - Connect Google Gemini Chat Model → AI Agent.
   - Connect Edit Fields → AI Agent.

5. **Create Code node (Parsing)**
   - Type: Code (JavaScript)
   - Code to remove triple backticks and parse JSON from AI Agent output.
   - Connect AI Agent → Code.

6. **Create Google Sheets Append node (Append row in sheet)**
   - Type: Google Sheets
   - Configure credential: Google Sheets OAuth2.
   - Set operation: Append.
   - Document ID: your Google Sheet ID.
   - Sheet Name: your target sheet (e.g., "Sheet2").
   - Map columns: Date, Diagnoses, Patient Name, Doctor's Name, Patient Phone Num. from parsed JSON.
   - Connect Code → Append row in sheet.

7. **Create Google Drive Copy File node (Copy file)**
   - Type: Google Drive
   - Configure credential: Google Drive OAuth2.
   - Operation: Copy.
   - File ID: Google Docs template ID.
   - Name: `your-doc-template-name - {{ $json['Patient Name'] }}`.
   - Connect Append row in sheet → Copy file.

8. **Create Google Docs Update node (Update a document1)**
   - Type: Google Docs
   - Configure credential: Google Docs OAuth2.
   - Operation: Update document.
   - Document URL: copied doc URL (use expression to get new file ID).
   - Add actions to insert tables and replace placeholders like `{{date}}`, `{{patient_name}}` with respective data from Google Sheet.
   - Connect Copy file → Update a document1.

9. **Create HTTP Request node (PDF export)**
   - Type: HTTP Request
   - Configure credential: Google Drive OAuth2.
   - Set URL: `https://www.googleapis.com/drive/v3/files/{{ $node["Copy file"].json["id"] }}/export?mimeType=application/pdf`.
   - Method: GET.
   - Connect Update a document1 → HTTP Request.

10. **Create Code node (Rename PDF)**
    - Type: Code (JavaScript)
    - Rename binary PDF file: `${PatientName}_${YYYY-MM-DD}.pdf`.
    - Set MIME type to `application/pdf`.
    - Connect HTTP Request → Code1.

11. **Create Gmail Send node (Send a message)**
    - Type: Gmail
    - Configure credential: Gmail OAuth2.
    - Recipient: fixed email or dynamic as needed.
    - Subject: `your-doc-template-name - {{ $('Code').item.json.patient_name }}`.
    - Attachments: binary PDF from Code1.
    - Connect Code1 → Send a message.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                        |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| Uses Google Gemini AI (PaLM API) to process medical email content with structured JSON output.                 | AI model selection and prompt design.                                |
| Workflow requires OAuth2 credentials for Gmail, Google Sheets, Google Drive, and Google Docs with appropriate scopes. | Credential management in n8n.                                        |
| Google Docs template must use clear placeholders (e.g., `{{patient_name}}`, `{{date}}`) for text replacement.  | Document template preparation.                                       |
| Polling interval is hourly, adjust based on expected email volume and latency requirements.                    | Gmail Trigger configuration.                                         |
| Ensure phone numbers extracted start with “0111” as per AI prompt constraints to maintain data consistency.    | Data validation in prompt.                                           |

---

This detailed reference document enables understanding, modification, and full reproduction of the "Generate Medical Reports from Emails with Gemini AI & Google Workspace" workflow.