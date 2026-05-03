Automated CV Screening and Scoring with AI, Gmail, GoogleDrive & Airtable

https://n8nworkflows.xyz/workflows/automated-cv-screening-and-scoring-with-ai--gmail--googledrive---airtable-8646


# Automated CV Screening and Scoring with AI, Gmail, GoogleDrive & Airtable

### 1. Workflow Overview

This workflow automates the screening and scoring of candidate CVs received for a Senior Software Engineer position by leveraging Gmail, Google Drive, AI models via OpenRouter, Google Sheets, and Airtable. It is designed to streamline recruitment by extracting, analyzing, and storing candidate information and evaluation scores in structured formats.

**Target Use Cases:**  
- HR teams receiving CVs via email or form submissions  
- Automated extraction of candidate contact info and CV content  
- AI-powered summarization and scoring of candidate suitability  
- Storing processed candidate data for easy tracking and reporting  

**Logical Blocks:**

- **1.1 Input Reception:** Triggers from Gmail emails or form submissions containing CV attachments.  
- **1.2 File Management:** Uploads attachments to Google Drive, downloads them for processing.  
- **1.3 Text Extraction:** Converts CV PDFs to plain text.  
- **1.4 AI Data Extraction:** Parallel AI processes extract structured contact info and detailed CV summaries with scoring.  
- **1.5 Post-Processing:** Normalizes AI output, cleans and parses data.  
- **1.6 Data Integration:** Merges extracted information and stores it into Google Sheets and Airtable.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception  
- **Overview:** Listens for incoming CVs via Gmail or form submissions matching the job title "Senior Software Engineer," fetching attachments for further processing.  
- **Nodes Involved:** Gmail Trigger, On form submission  

**Node Details:**

- **Gmail Trigger**  
  - *Type:* Trigger node listening to Gmail inbox  
  - *Configuration:* Polls every minute; filters emails with subject containing "Senior Software Engineer"; downloads attachments with prefix "CV"  
  - *Input:* Email inbox via OAuth2 credentials  
  - *Output:* Email data including attachments  
  - *Failure Modes:* Authentication errors, Gmail API rate limits, missing attachments  
  - *Sticky Note Content:* "GMAIL TRIGGER: Listen from emails or forms submissions matching the CV's received for specific job position and fetch attachments."  

- **On form submission**  
  - *Type:* Trigger node responding to external form submissions  
  - *Configuration:* Watches for submissions with form titled "Senior Software Engineer"  
  - *Input:* Webhook data from form platform  
  - *Output:* Submission data including uploaded file(s)  
  - *Failure Modes:* Webhook misconfiguration, missing form data  

#### 1.2 File Management  
- **Overview:** Uploads received CV attachments to a designated Google Drive folder, then downloads the stored files for processing.  
- **Nodes Involved:** Upload file, Download file  

**Node Details:**

- **Upload file**  
  - *Type:* Google Drive node for upload  
  - *Configuration:* Uploads attachment named after sender's file name into specific Drive folder "Software Engineer Resume"  
  - *Credentials:* Google Drive OAuth2  
  - *Input:* Attachment binary data from trigger nodes  
  - *Output:* Metadata including file ID  
  - *Failure Modes:* Auth errors, file size limits, folder permission issues  
  - *Sticky Note Content:* "UPLOAD THE FILE: Incoming attachment (CV) is uploaded to the configured Google Drive folder and named from the sender."  

- **Download file**  
  - *Type:* Google Drive node for download  
  - *Configuration:* Downloads file by ID from previous upload node  
  - *Credentials:* Same Google Drive OAuth2  
  - *Input:* File ID from Upload file node output  
  - *Output:* Binary content of the CV PDF  
  - *Failure Modes:* File not found, download timeout  
  - *Sticky Note Content:* "DOWNLOAD THE ATTACHMENT (CV): The stored file is downloaded by ID so it can be read."  

#### 1.3 Text Extraction  
- **Overview:** Converts the downloaded CV PDF into plain text for downstream AI processing.  
- **Nodes Involved:** Extract from File  

**Node Details:**

- **Extract from File**  
  - *Type:* Extract from File node  
  - *Configuration:* Operation set to PDF text extraction without additional options  
  - *Input:* Binary PDF from Download file node  
  - *Output:* Extracted plain text of CV  
  - *Failure Modes:* Corrupt PDF, unsupported file format  
  - *Sticky Note Content:* "EXTRACT FROM FILE: Extract from File converts the CV (PDF) into plain text."  

#### 1.4 AI Data Extraction  
- **Overview:** Runs two parallel AI processes: one for quick structured extraction of contact info, and another for detailed CV summarization and candidate scoring.  
- **Nodes Involved:** Information Extractor, AI Agent, OpenRouter Chat Model, OpenRouter Chat Model1  

**Node Details:**

- **Information Extractor**  
  - *Type:* LangChain Information Extractor node  
  - *Configuration:* Uses a manual JSON schema to extract candidate_name (string), email_address (email format), and contact_number (regex pattern for phone numbers) from the extracted CV text  
  - *Input:* Plain text from Extract from File  
  - *Output:* Structured contact info JSON  
  - *Failure Modes:* Schema mismatch, extraction errors, invalid formats  
  - *Sticky Note Content:* Part of AI parallel path for quick structured extraction  

- **AI Agent**  
  - *Type:* LangChain Agent node using OpenRouter GPT-OSS-20B free model  
  - *Configuration:* Large system prompt instructing the AI to summarize the CV into educational qualifications, job history, skill set, evaluate candidate suitability with a score (1-10), and provide justification. Input text prepends "CV:\n" to the extracted text.  
  - *Input:* Plain text from Extract from File  
  - *Output:* Text summary including sections and scoring  
  - *Failure Modes:* Model timeouts, API limits, prompt formatting errors  
  - *Sticky Note Content:* "Two parallel AI paths: Quick structured extraction: Information Extractor uses a small schema (name, email, phone) + LM helper to pull contact fields. Full CV analysis: AI Agent runs a large system prompt to summarize Education, Job History, Skills and to assign a suitability score (1–10)."  

- **OpenRouter Chat Model** (linked to Information Extractor)  
  - *Type:* Language Model node (OpenRouter GPT-OSS-20B free)  
  - *Role:* Provides language model processing for Information Extractor node  
  - *Credentials:* OpenRouter API  
  - *Failure Modes:* API rate limits, auth failure  

- **OpenRouter Chat Model1** (linked to AI Agent)  
  - *Type:* Language Model node (OpenRouter GPT-OSS-20B free)  
  - *Role:* Provides language model processing for AI Agent node  
  - *Credentials:* OpenRouter API  
  - *Failure Modes:* API rate limits, auth failure  

#### 1.5 Post-Processing  
- **Overview:** Normalizes and cleans the AI Agent output to extract structured data sections and score for further merging.  
- **Nodes Involved:** Edit Fields, Code  

**Node Details:**

- **Edit Fields**  
  - *Type:* Set node  
  - *Configuration:* Maps AI Agent's output text field into a normalized field called "output"  
  - *Input:* Raw AI Agent text output  
  - *Output:* Normalized text under "output" key  
  - *Failure Modes:* Missing or null AI output  
  - *Sticky Note Content:* "Normalize agent output: Edit Fields maps the agent response into output."  

- **Code**  
  - *Type:* Code node (JavaScript)  
  - *Configuration:* Extracts specific sections (Educational Qualifications, Job History, Skill Set, Candidate Evaluation) by regex from AI Agent text. Also extracts numeric score and justification from the evaluation section. Returns structured JSON with these fields.  
  - *Input:* Normalized AI Agent output from Edit Fields  
  - *Output:* JSON with parsed fields: educationalQualification, jobHistory, skillSet, score, justification  
  - *Failure Modes:* Regex misses section, unexpected output format, empty input  
  - *Sticky Note Content:* "Parse & clean: Code runs JS to extract the three summary sections plus score and justification from the agent text (regex-based)."  

#### 1.6 Data Integration  
- **Overview:** Combines the quick schema extraction (contact info) with the detailed AI-parsed summary and score, then stores the complete record to Google Sheets and Airtable.  
- **Nodes Involved:** Merge, Append row in sheet, Create a record  

**Node Details:**

- **Merge**  
  - *Type:* Merge node  
  - *Configuration:* Combines inputs from Information Extractor and Code node outputs by "combineAll" mode  
  - *Input:* Contact info JSON and parsed AI summary JSON  
  - *Output:* Combined JSON object with full candidate data  
  - *Failure Modes:* Data mismatch, missing inputs  
  - *Sticky Note Content:* "Merge datasets: Merge combines the schema extraction (contact info) with the AI-parsed summary/score."  

- **Append row in sheet**  
  - *Type:* Google Sheets node  
  - *Configuration:* Appends a row to a configured Google Sheets document and sheet (Sheet1) with candidate fields mapped from merged data: candidate_name, email_address, contact_number, Educational Qualifications, Job History, skill set, score, Justification  
  - *Credentials:* Google Sheets OAuth2  
  - *Input:* Merged candidate data  
  - *Output:* Confirmation of row append  
  - *Failure Modes:* Sheet access errors, API limits  
  - *Sticky Note Content:* "Store results: Final record is appended to Google Sheets and inserted into Airtable for tracking, reporting, or downstream workflows."  

- **Create a record**  
  - *Type:* Airtable node  
  - *Configuration:* Creates a new record in a specified Airtable base and table with the same candidate fields as Google Sheets  
  - *Credentials:* Airtable Personal Access Token  
  - *Input:* Merged candidate data  
  - *Output:* Confirmation of record creation  
  - *Failure Modes:* Airtable API limits, auth errors, schema mismatch  
  - *Sticky Note Content:* Same as Append row in sheet  

---

### 3. Summary Table

| Node Name           | Node Type                       | Functional Role                         | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                                    |
|---------------------|--------------------------------|---------------------------------------|-----------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| Gmail Trigger       | Trigger (n8n-nodes-base.gmailTrigger) | Listen for emails with matching CVs   | -                           | Upload file                     | GMAIL TRIGGER: Listen from emails or forms submissions matching the CV's received for specific job position and fetch attachments. |
| On form submission  | Trigger (n8n-nodes-base.formTrigger)  | Listen for form submission CV uploads | -                           | Upload file                     | GMAIL TRIGGER: Listen from emails or forms submissions matching the CV's received for specific job position and fetch attachments. |
| Upload file         | Google Drive upload node          | Upload CV attachment to Drive folder   | Gmail Trigger, On form submission | Download file                  | UPLOAD THE FILE: Incoming attachment (CV) is uploaded to the configured Google Drive folder and named from the sender. |
| Download file       | Google Drive download node        | Download stored CV PDF file            | Upload file                  | Extract from File               | DOWNLOAD THE ATTACHMENT (CV): The stored file is downloaded by ID so it can be read.                           |
| Extract from File   | File text extraction node         | Extract plain text from CV PDF         | Download file                | Information Extractor, AI Agent | EXTRACT FROM FILE: Extract from File converts the CV (PDF) into plain text.                                   |
| Information Extractor | LangChain Information Extractor  | Extract contact info (name, email, phone) | Extract from File            | Merge                          | Two parallel AI paths: Quick structured extraction: Information Extractor uses a small schema (name, email, phone) + LM helper to pull contact fields. |
| AI Agent            | LangChain Agent AI node           | Summarize CV, score candidate suitability | Extract from File            | Edit Fields                    | Two parallel AI paths: Full CV analysis: AI Agent runs a large system prompt to summarize Education, Job History, Skills and to assign a suitability score (1–10). |
| OpenRouter Chat Model | Language Model node (OpenRouter) | Provides LM for Information Extractor  | -                           | Information Extractor          |                                                                                                               |
| OpenRouter Chat Model1 | Language Model node (OpenRouter) | Provides LM for AI Agent                | -                           | AI Agent                      |                                                                                                               |
| Edit Fields         | Set node                        | Normalize AI Agent raw output           | AI Agent                    | Code                          | Normalize agent output: Edit Fields maps the agent response into output.                                     |
| Code                | Code (JavaScript) node          | Parse AI text output into structured fields | Edit Fields                 | Merge                         | Parse & clean: Code runs JS to extract the three summary sections plus score and justification from the agent text (regex-based). |
| Merge               | Merge node                     | Combine contact info and AI summary     | Information Extractor, Code  | Append row in sheet, Create a record | Merge datasets: Merge combines the schema extraction (contact info) with the AI-parsed summary/score.          |
| Append row in sheet | Google Sheets append node       | Store candidate data in Google Sheets   | Merge                       | -                             | Store results: Final record is appended to Google Sheets and inserted into Airtable for tracking, reporting, or downstream workflows. |
| Create a record     | Airtable create record node     | Store candidate data in Airtable base   | Merge                       | -                             | Store results: Final record is appended to Google Sheets and inserted into Airtable for tracking, reporting, or downstream workflows. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail access  
   - Set filter query to `"Senior Software Engineer"`  
   - Enable attachment download with prefix `"CV"`  
   - Poll every minute  

2. **Create On form submission Node:**  
   - Type: Form Trigger  
   - Set form title to `"Senior Software Engineer"`  
   - Ensure webhook is configured and exposed externally  

3. **Create Upload file Node:**  
   - Type: Google Drive node with "Upload" operation  
   - OAuth2 credentials for Google Drive  
   - Target folder: Google Drive folder ID for "Software Engineer Resume"  
   - File name: Set dynamically from incoming attachment name (expression: `{{$json.from.value[0].name}}`)  
   - Input binary data field: Attachment from trigger nodes (e.g., `CV0`)  

4. **Create Download file Node:**  
   - Type: Google Drive node with "Download" operation  
   - Use file ID from Upload file node output (expression: `{{$json.id}}`)  
   - OAuth2 credentials: same as Upload file  

5. **Create Extract from File Node:**  
   - Type: Extract from File  
   - Operation: PDF text extraction  
   - Input: Binary PDF from Download file node  

6. **Create OpenRouter Chat Model Node (for Information Extractor):**  
   - Type: LangChain LM Chat OpenRouter  
   - Model: `openai/gpt-oss-20b:free`  
   - Credentials: OpenRouter API key  

7. **Create Information Extractor Node:**  
   - Type: LangChain Information Extractor  
   - Link OpenRouter Chat Model as language model  
   - Configure manual JSON schema with these fields:  
     - candidate_name (string)  
     - email_address (string, email format)  
     - contact_number (string, regex pattern for phone number)  
   - Input: text from Extract from File  

8. **Create OpenRouter Chat Model1 Node (for AI Agent):**  
   - Type: LangChain LM Chat OpenRouter  
   - Model: `openai/gpt-oss-20b:free`  
   - Credentials: OpenRouter API key  

9. **Create AI Agent Node:**  
   - Type: LangChain Agent node  
   - Link OpenRouter Chat Model1 as language model  
   - System message: Paste the detailed prompt for CV summarization, scoring, and output formatting as provided in the original workflow  
   - Input: Text from Extract from File prefixed with `"CV:\n"`  

10. **Create Edit Fields Node:**  
    - Type: Set node  
    - Create a new field `output` and assign it the AI Agent’s output text (`{{$json.output}}`)  

11. **Create Code Node:**  
    - Type: Code (JavaScript) node  
    - Paste the JavaScript code that extracts sections (Educational Qualifications, Job History, Skill Set, Candidate Evaluation) and parses score and justification from the `output` field  
    - Input: Output from Edit Fields node  

12. **Create Merge Node:**  
    - Type: Merge node  
    - Mode: combineAll  
    - Inputs: Information Extractor and Code node outputs  

13. **Create Append row in sheet Node:**  
    - Type: Google Sheets node (Append operation)  
    - Credentials: Google Sheets OAuth2  
    - Document ID: Google Sheets document for HR tracking  
    - Sheet name: `"Sheet1"` or equivalent  
    - Map columns: candidate_name, email_address, contact_number, Educational Qualifications, Job History, skill set, score, Justification from merged data  

14. **Create Create a record Node:**  
    - Type: Airtable node (Create operation)  
    - Credentials: Airtable Personal Access Token  
    - Base and table set to target Airtable base and table  
    - Map fields identical to Google Sheets mapping  

15. **Connect Nodes:**  
    - Connect Gmail Trigger and On form submission nodes to Upload file node  
    - Connect Upload file to Download file  
    - Download file to Extract from File  
    - Extract from File to Information Extractor and AI Agent nodes in parallel  
    - Connect OpenRouter Chat Model to Information Extractor  
    - Connect OpenRouter Chat Model1 to AI Agent  
    - AI Agent to Edit Fields  
    - Edit Fields to Code  
    - Information Extractor and Code to Merge  
    - Merge to Append row in sheet and Create a record  

16. **Activate Workflow:**  
    - Test each step for proper execution and error handling  
    - Monitor for API quotas, permission issues, and data format consistency  

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The detailed system prompt for AI Agent is essential to produce consistent CV summaries and scoring.                                   | Embedded in AI Agent node’s system message parameter                                                        |
| For Gmail Trigger, ensure OAuth2 credentials have appropriate scopes for reading emails and downloading attachments.                   | Gmail API documentation: https://developers.google.com/gmail/api                                                |
| Google Drive folder ID and Google Sheets document ID must be replaced with your own Drive and Sheet for production use.               | Google Drive & Sheets API docs: https://developers.google.com/drive, https://developers.google.com/sheets/api |
| Airtable Personal Access Token requires scopes for base and table access matching the configured base/table IDs.                      | Airtable API docs: https://airtable.com/api                                                                  |
| The JavaScript in the Code node uses regex to parse AI output—format changes in AI output may break parsing and require updates.        | Monitor AI output consistency to maintain regex accuracy                                                    |
| The parallel AI paths enhance resilience and data completeness by splitting contact info extraction and deep CV analysis.             | Sticky notes explain design rationale                                                                       |

---

*Disclaimer: The provided text is extracted solely from an automated workflow created with n8n, respecting all applicable content policies. All processed data is legal and publicly accessible.*