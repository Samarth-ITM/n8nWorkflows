Parse CVs from Emails with OCR & GPT for Notion Database

https://n8nworkflows.xyz/workflows/parse-cvs-from-emails-with-ocr---gpt-for-notion-database-6326


# Parse CVs from Emails with OCR & GPT for Notion Database

### 1. Workflow Overview

This workflow automates the processing of CVs (resumes) received via email attachments, extracting candidate information using OCR and AI, and storing it in a Notion database. It is designed for HR teams or recruiters who want to streamline candidate data collection from emails.

The workflow is logically divided into the following blocks:

- **1.1 Email Input Reception**  
  Watches a Gmail inbox for incoming emails with attachments.

- **1.2 PDF Attachment Filtering and OCR Processing**  
  Filters PDF attachments from emails and extracts their text content using the OCR.space API.

- **1.3 AI-Powered CV Parsing**  
  Sends the extracted OCR text to OpenAI’s GPT model to detect if the document is a CV and to extract structured candidate fields.

- **1.4 Data Mapping and Duplicate Verification**  
  Maps extracted fields to the workflow's internal structure and queries the Notion database to check for existing entries by email to avoid duplicates.

- **1.5 Notion Database Entry Creation**  
  Creates a new candidate record in Notion if no duplicate is found.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Input Reception

**Overview:**  
This block triggers the workflow upon receiving a new email with attachments in a Gmail inbox.

**Nodes Involved:**  
- Gmail Trigger  
- Code (Attachment Filter)

**Node Details:**

- **Gmail Trigger**  
  - Type: Trigger node for Gmail  
  - Configuration: Filters emails with attachments (`has:attachment`), downloads attachments, polls every minute.  
  - Input: None (trigger)  
  - Output: Email data including attachments in binary form.  
  - Edge Cases: OAuth2 token expiration, Gmail API rate limits.

- **Code (Attachment Filter)**  
  - Type: Function (JavaScript) node  
  - Role: Filters email attachments to only PDF files.  
  - Key Logic: Iterates over binary attachments and outputs only those with mime type `application/pdf`.  
  - Input: Gmail Trigger output  
  - Output: One item per PDF attachment with binary data.  
  - Edge Cases: Emails without PDFs lead to empty output; code assumes `binary` data exists; malformed emails might cause errors.

#### 2.2 PDF Attachment Filtering and OCR Processing

**Overview:**  
This block sends the PDF attachment binary data to OCR.space to extract textual content.

**Nodes Involved:**  
- HTTP Request (OCR API)  
- If1 (HTTP status check)

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request node  
  - Role: Sends PDF binary data as multipart/form-data to OCR.space API for OCR text extraction.  
  - Configuration:  
    - URL: `https://api.ocr.space/parse/image`  
    - Method: POST  
    - Headers: API key dynamically read from variable `OCR_SPACE_API_KEY`  
    - Timeout: 40 seconds  
  - Input: PDF binary from Code node  
  - Output: OCR response including parsed text.  
  - Edge Cases: API timeouts, invalid API key, malformed PDF files, OCR errors.

- **If1**  
  - Type: Conditional node  
  - Role: Checks HTTP response status code is between 200 and 299 (success).  
  - Input: HTTP Request output  
  - Output: Proceeds to AI parsing if success, terminates otherwise.  
  - Edge Cases: Handles non-2xx responses gracefully by skipping downstream.

#### 2.3 AI-Powered CV Parsing

**Overview:**  
Sends OCR text content to a GPT model to classify if the document is a CV and extract structured candidate data fields.

**Nodes Involved:**  
- Message a model (OpenAI GPT node)  
- If (CV check)

**Node Details:**

- **Message a model**  
  - Type: OpenAI GPT node (via LangChain integration)  
  - Role: Provides instructions and OCR text to GPT-4.1-NANO model to parse CV fields.  
  - Key prompt details:  
    - Check if document is a CV  
    - Extract fields: full_name, email, phone, location, job_title, years_experience, skills (array), education (array), languages, certifications, linkedin, availability  
    - Return JSON or `{ "is_cv": false }` if not a CV  
  - Input: OCR parsed text (`ParsedResults[0].ParsedText`) from HTTP Request  
  - Output: JSON with candidate data or `is_cv: false`  
  - Credentials: OpenAI API key  
  - Edge Cases: Model misinterpretation, rate limiting, incomplete data extraction.

- **If**  
  - Type: Conditional node  
  - Role: Checks if the OpenAI response contains a non-null email field (indicating a CV was detected).  
  - Input: GPT output  
  - Output: Continues if CV detected, stops otherwise.  
  - Edge Cases: False negatives if email missing; expression errors if response malformed.

#### 2.4 Data Mapping and Duplicate Verification

**Overview:**  
Maps GPT-extracted fields into a structured format and queries Notion to check for existing candidate entries by email.

**Nodes Involved:**  
- Map Fields (Set node)  
- Get many entries (Notion query)  
- If2 (Duplicate check)

**Node Details:**

- **Map Fields**  
  - Type: Set node  
  - Role: Assigns GPT-extracted fields to a structured JSON format under `message.content` namespace, e.g., `full_name`, `email`, `phone`, `location`, `years_experience`, `education` (array), `languages` (array), `linkedin`.  
  - Input: GPT output JSON  
  - Output: Mapped structured data for downstream use  
  - Edge Cases: Null or missing fields handled with defaults (`null` or empty arrays).

- **Get many entries**  
  - Type: Notion node (database query)  
  - Role: Queries Notion "Candidates" database filtering by email to find duplicates.  
  - Input: Mapped data from Map Fields  
  - Output: List of matching entries (empty if none)  
  - Credentials: Notion API integration  
  - Edge Cases: Notion API errors, rate limits, incorrect database ID.

- **If2**  
  - Type: Conditional node  
  - Role: Checks if No duplicates exist (result length = 0) to proceed with creation.  
  - Input: Notion query output  
  - Output: Creates new entry if no duplicates found, skips otherwise.  
  - Edge Cases: Duplicate detection logic relies on exact email match; partial duplicates may be missed.

#### 2.5 Notion Database Entry Creation

**Overview:**  
Creates a new candidate record in the Notion database with the parsed and mapped candidate data.

**Nodes Involved:**  
- Create entry (Notion create database page)

**Node Details:**

- **Create entry**  
  - Type: Notion node (create database page)  
  - Role: Inserts a new page in the "Candidates" database with mapped candidate properties.  
  - Properties mapped:  
    - Name (title)  
    - Email (email)  
    - Phone (phone_number)  
    - Location (rich_text)  
    - Years of Experience (rich_text)  
    - Education (joined string from array)  
    - Languages (joined string)  
    - LinkedIn (url)  
    - Application Date (current date/time in Europe/Madrid timezone)  
  - Credentials: Notion API integration  
  - Edge Cases: Notion API errors, improper data formatting, missing required fields.

---

### 3. Summary Table

| Node Name         | Node Type                        | Functional Role                           | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                                  |
|-------------------|---------------------------------|-----------------------------------------|-----------------------|----------------------|--------------------------------------------------------------------------------------------------------------|
| Gmail Trigger     | Gmail Trigger                   | Trigger workflow on new email with attachments | None                  | Code                 | ### 🧠 CV Parser - Email to Notion: Workflow overview and purpose                                            |
| Code              | Function (Code)                 | Filter attachments to PDFs only         | Gmail Trigger         | HTTP Request         | ### 📌 Notes: Only processes emails with PDF attachments; skips non-CV docs automatically                    |
| HTTP Request      | HTTP Request                   | Send PDF binary to OCR.space API for text extraction | Code                  | If1                  |                                                                                                              |
| If1               | If Condition                   | Check OCR API HTTP status (200-299)     | HTTP Request          | Message a model       |                                                                                                              |
| Message a model   | OpenAI GPT (LangChain node)    | Parse OCR text to detect CV and extract candidate data | If1                   | If                   |                                                                                                              |
| If                | If Condition                   | Check if AI detected a CV (email present) | Message a model        | Map Fields            |                                                                                                              |
| Map Fields        | Set                            | Map AI-extracted candidate fields to structured format | If                    | Get many entries      |                                                                                                              |
| Get many entries  | Notion (Database query)        | Query Notion database for duplicates by email | Map Fields             | If2                  |                                                                                                              |
| If2               | If Condition                   | Proceed only if no duplicate found       | Get many entries       | Create entry          |                                                                                                              |
| Create entry      | Notion (Create database page) | Create new candidate record in Notion   | If2                   | None                 |                                                                                                              |
| Sticky Note       | Sticky Note                    | Documentation and workflow overview      | None                  | None                 | ### 🧠 CV Parser - Email to Notion: This workflow automates extraction and organization of candidate data.    |
| Sticky Note1      | Sticky Note                    | Additional notes on processing and duplicates | None                  | None                 | ### 📌 Notes: Only processes emails with PDF attachments; skips non-CV; checks duplicates by email.           |
| Sticky Note2      | Sticky Note                    | Requirements and setup notes              | None                  | None                 | ### ✅ Requirements: Gmail OAuth2, OCR.space API Key, OpenAI API Key, Notion integration with proper fields.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail.  
   - Set filter query: `has:attachment`  
   - Enable attachment download.  
   - Set polling interval to every minute.

2. **Add Code Node to Filter PDF Attachments**  
   - Type: Function (Code) node  
   - Paste JavaScript code to filter attachments with mime type `application/pdf`.  
   - Connect Gmail Trigger output to this node.

3. **Add HTTP Request Node to OCR.space API**  
   - Type: HTTP Request  
   - Set URL: `https://api.ocr.space/parse/image`  
   - Method: POST  
   - Content-Type: multipart/form-data  
   - Add header parameter `apikey` with value from variable `OCR_SPACE_API_KEY` (create this variable in n8n or set manually).  
   - Configure body to send binary data under field `file`.  
   - Set timeout to 40000 ms.  
   - Connect Code node output to this node.

4. **Add If Node to Check OCR API Response Status**  
   - Type: If node (version 2)  
   - Condition: `statusCode >= 200 AND statusCode < 300`  
   - Connect HTTP Request node output to this node.

5. **Add OpenAI GPT Node (LangChain OpenAi)**  
   - Type: OpenAi (LangChain) node  
   - Model: `gpt-4.1-nano`  
   - Credentials: Configure with OpenAI API Key.  
   - Message content:  
     ```
     You're an HR assistant. I will provide you with text from a PDF document.

     1. First, determine if it's a CV/resume.
     2. If it is, extract the following fields:
     - full_name
     - email
     - phone
     - location
     - job_title
     - years_experience
     - skills (as an array)
     - education (as an array of strings)
     - languages
     - certifications
     - linkedin
     - availability

     3. Return the result as a JSON object with those fields.
     If it's not a CV, just return: { "is_cv": false }
     Here is the document text:

     {{ $json.body.ParsedResults[0].ParsedText }}
     ```
   - Connect the successful branch of the If node (OCR response check) to this node.

6. **Add If Node to Check if Email Field Exists in GPT Output**  
   - Type: If node (version 2)  
   - Condition: Check if `message.content.email !== null`  
   - Connect GPT node output to this node.

7. **Add Set Node to Map Fields**  
   - Type: Set node  
   - Map fields from GPT output JSON to structured keys under `message.content` for: full_name, email, phone, location, years_experience, education, languages, linkedin, etc.  
   - Connect positive branch of previous If node.

8. **Add Notion Node to Query Database for Existing Entries**  
   - Type: Notion (Database Page - Get All)  
   - Credentials: Connect to Notion integration with access to your Candidates database.  
   - Set database ID to your Notion Candidates DB.  
   - Filter: `Email` equals `={{ $json.message.content.email }}`  
   - Connect Map Fields node output to this node.

9. **Add If Node to Check for Duplicates**  
   - Type: If node (version 2)  
   - Condition: Check if the number of returned entries is zero (no duplicates).  
   - Connect Notion query output to this node.

10. **Add Notion Node to Create New Candidate Entry**  
    - Type: Notion (Create Database Page)  
    - Credentials: Same Notion integration.  
    - Database ID: Candidates DB.  
    - Map properties from mapped fields to Notion properties: Name, Email, Phone, Location, Years of Experience, Education (joined by `/`), Languages (joined by `, `), LinkedIn URL, Application Date (set current date/time, timezone Europe/Madrid).  
    - Connect positive branch of previous If node.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                           |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Workflow automates CV extraction from Gmail attachments, including OCR and GPT-powered parsing into Notion | Sticky Note on workflow overview                           |
| Only processes PDF attachments; skips non-CV documents automatically; checks duplicates by email          | Sticky Note1                                               |
| Requires Gmail OAuth2 credentials, OCR.space API key, OpenAI API key, and Notion integration with proper database fields | Sticky Note2                                               |
| Notion database should contain fields: Name, Email, Phone, Location, Experience, Education, Languages, LinkedIn | Configured in Notion create and query nodes                |

---

This structured documentation provides a comprehensive understanding of the workflow, enabling reproduction, modification, and anticipation of potential errors or integration issues.