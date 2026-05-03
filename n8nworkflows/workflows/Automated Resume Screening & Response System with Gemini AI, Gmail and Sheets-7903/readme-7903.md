Automated Resume Screening & Response System with Gemini AI, Gmail and Sheets

https://n8nworkflows.xyz/workflows/automated-resume-screening---response-system-with-gemini-ai--gmail-and-sheets-7903


# Automated Resume Screening & Response System with Gemini AI, Gmail and Sheets

### 1. Workflow Overview

This workflow automates the process of screening job applications submitted via a form, using Google Gemini AI to analyze resumes, generate evaluation scores, determine acceptance status, and compose personalized email responses. It integrates form submission, PDF resume extraction, AI processing, email communication via Gmail, and candidate data logging in Google Sheets.

**Target Use Cases:**  
- Automating hiring workflows for roles such as Executive Assistant, IT Specialist, and Manager.  
- Reducing manual resume screening by leveraging AI to score candidates and generate customized email replies.  
- Centralizing application data and communication status tracking in a Google Sheet.

**Logical Blocks:**  
- **1.1 Input Reception:** Capture candidate applications through a custom form including name, email, job choice, and resume upload.  
- **1.2 Resume Processing:** Extract textual content from the uploaded resume PDF.  
- **1.3 AI Evaluation & Email Generation:** Use Google Gemini AI to evaluate resume suitability, assign acceptance status, and draft personalized emails.  
- **1.4 Information Extraction:** Parse the AI output into structured fields (score, status, email subject/body).  
- **1.5 Communication:** Send evaluation outcome emails to candidates via Gmail.  
- **1.6 Data Logging:** Append candidate details, evaluation results, and email status to Google Sheets.  
- **1.7 Setup & Documentation:** Sticky notes provide setup instructions and tutorial references.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow when a candidate submits an application form. It collects personal info, job applied for, and a resume file.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - Type: Form Trigger  
  - Configuration:  
    - Form titled "Apply For Executive Assistant at SAMSUNG"  
    - Fields: Name (required), Email Address (required), Job (dropdown: Executive Assistant, IT Specialist, Manager; required), Resume (single PDF file upload; required)  
    - Form description: "Give your Info carefully"  
  - Input: HTTP webhook triggered by form submission  
  - Output: JSON object with form responses and binary resume file  
  - Edge cases: Missing required fields, unsupported file types, form submission errors  
  - Version-specific: n8n form trigger v2.2 supports these field types and file uploads

#### 2.2 Resume Processing

**Overview:**  
Extracts text content from the uploaded PDF resume to prepare for AI evaluation.

**Nodes Involved:**  
- Extract from File

**Node Details:**  
- **Extract from File**  
  - Type: Extract From File  
  - Operation: PDF text extraction  
  - Binary property: "Resume" (from form upload)  
  - Input: Binary file from form submission  
  - Output: JSON with extracted text in the `text` property  
  - Edge cases: Corrupted PDFs, unreadable format, extraction failures  
  - Version-specific: v1 supports PDF extraction with binary input

#### 2.3 AI Evaluation & Email Generation

**Overview:**  
Uses Google Gemini AI via the LangChain AI Agent node to score the candidate’s resume, decide acceptance, and generate a tailored email response.

**Nodes Involved:**  
- Google 2.5 Flash (Google Gemini Language Model)  
- AI Agent

**Node Details:**  
- **Google 2.5 Flash**  
  - Type: LangChain LM Chat (Google Gemini)  
  - Credentials: Google Gemini(PaLM) API OAuth2 credential configured  
  - Role: Provides the AI language model for the AI Agent node  
  - Input/Output: Receives prompts and returns AI-generated text  
  - Edge cases: API auth failures, rate limits, latency, malformed prompt responses

- **AI Agent**  
  - Type: LangChain Agent  
  - Prompt configuration:  
    - Inputs: Candidate Name, Job Title, Resume text (all via expressions)  
    - System message instructs the agent to:  
      1. Rate the resume suitability on a scale of 0-10  
      2. Assign status “Accepted” if score ≥7, else “Rejected”  
      3. Compose email subject and body accordingly  
    - Example output format provided for clarity  
  - Input: Extracted resume text and form data  
  - Output: AI-generated text containing score, status, and email content  
  - Edge cases: AI misinterpretation, incomplete output, API errors, prompt failures  
  - Version-specific: LangChain agent v2 supports text prompts and system messages  

#### 2.4 Information Extraction

**Overview:**  
Parses the AI Agent’s raw text output into structured JSON fields for easy downstream usage.

**Nodes Involved:**  
- Information Extractor

**Node Details:**  
- **Information Extractor**  
  - Type: LangChain Information Extractor  
  - Input: AI Agent output text  
  - Attributes extracted:  
    - Score (number)  
    - Status (string; Accepted/Rejected)  
    - Mail Subject (string)  
    - Mail Body (string)  
  - Output: Structured JSON with these attributes  
  - Edge cases: Parsing failures if AI output format deviates, missing fields  
  - Version-specific: v1.2 supports attribute extraction with required flags

#### 2.5 Communication

**Overview:**  
Sends the generated acceptance or rejection email to the candidate's email address using Gmail.

**Nodes Involved:**  
- Gmail

**Node Details:**  
- **Gmail**  
  - Type: Gmail node  
  - Parameters:  
    - Recipient: Email from the form submission  
    - Subject: Extracted email subject from Information Extractor  
    - Message body: Extracted email body text  
    - Email type: Plain text  
    - Append Attribution: Disabled for clean emails  
  - Credentials: OAuth2 Gmail account authorization required  
  - Input: Parsed email content and candidate email  
  - Output: Email sent status  
  - Edge cases: Authentication errors, quota limits, invalid email addresses, network failures  
  - Version-specific: v2.1 supports OAuth2 and text emails

#### 2.6 Data Logging

**Overview:**  
Appends a record of the candidate’s application, evaluation score, status, and email sending status to a Google Sheet for tracking.

**Nodes Involved:**  
- Google Sheets

**Node Details:**  
- **Google Sheets**  
  - Type: Google Sheets Append operation  
  - Target Sheet: "Form responses 1" tab, specific Google Sheets document ID provided  
  - Columns mapped:  
    - Job, Name, Email (from form)  
    - Score, Status (from Information Extractor)  
    - Email Status: Set to "SENT ✅" on success  
  - Credentials: Google Sheets OAuth2 credentials must be configured  
  - Input: Candidate details and evaluation results  
  - Output: Appended row confirmation  
  - Edge cases: Permission errors, sheet locking, schema mismatch, network issues  
  - Version-specific: v4.6 supports schema mapping and append operations

#### 2.7 Setup & Documentation

**Overview:**  
Sticky notes provide visual guidance, setup instructions, and tutorial references embedded in the workflow canvas.

**Nodes Involved:**  
- Sticky Note3  
- Sticky Note

**Node Details:**  
- **Sticky Note3**  
  - Content: Link to a YouTube tutorial video on building an AI Lead Finder agent (thumbnail and clickable link)  
  - Purpose: Visual entry point for beginners to learn setup steps

- **Sticky Note**  
  - Content: Detailed step-by-step setup instructions covering form customization, node configuration, credential linking (Google Gemini, Gmail, Google Sheets), and email template editing  
  - Purpose: Ensures user can properly configure the workflow before activation

---

### 3. Summary Table

| Node Name           | Node Type                           | Functional Role                    | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                                  |
|---------------------|-----------------------------------|----------------------------------|------------------------|-----------------------|--------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger                      | Receives candidate form inputs   | —                      | Extract from File       | See setup instructions sticky note for form customization                                                   |
| Extract from File    | Extract From File                 | Extracts text from resume PDF    | On form submission      | AI Agent               | Resume upload limited to PDF; see setup instructions                                                        |
| AI Agent            | LangChain Agent                  | AI evaluation and email draft    | Extract from File       | Information Extractor  | Uses Google Gemini via LangChain; see setup instructions for API key setup                                   |
| Information Extractor| LangChain Information Extractor  | Parses AI text output to fields  | AI Agent                | Gmail                  | Requires structured AI output format; see setup instructions                                                |
| Gmail               | Gmail node                       | Sends acceptance/rejection email| Information Extractor   | Google Sheets          | Gmail OAuth2 required; see setup instructions                                                               |
| Google Sheets       | Google Sheets Append             | Logs candidate data and status   | Gmail                   | —                      | Google Sheets OAuth2 required; spreadsheet must have specific columns; see setup instructions                |
| Google 2.5 Flash     | LangChain LM Chat (Google Gemini)| Provides AI language model       | —                      | AI Agent, Information Extractor (AI LM input) | Credential for Google Gemini API; see setup instructions                                                   |
| Sticky Note3        | Sticky Note                      | Tutorial video link              | —                      | —                      | YouTube tutorial for workflow setup                                                                         |
| Sticky Note         | Sticky Note                      | Setup instructions              | —                      | —                      | Detailed setup guide including form, credentials, and node configurations                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node ("On form submission"):**  
   - Set form title: "Apply For Executive Assistant at SAMSUNG"  
   - Add fields:  
     - Name (text, required)  
     - Email Address (text, required)  
     - Job (dropdown with options: Executive Assistant, IT Specialist, Manager; required)  
     - Resume (file upload, accept only `.pdf`, single file, required)  
   - Save and note the webhook URL for form integration.

3. **Add an Extract From File node ("Extract from File"):**  
   - Operation: PDF text extraction  
   - Binary Property Name: Use "Resume" (matches form field)  
   - Connect input from "On form submission".

4. **Add a LangChain LM Chat node ("Google 2.5 Flash"):**  
   - Select Google Gemini (PaLM) API credentials (OAuth2)  
   - No additional configuration needed; used as AI language model provider.

5. **Add a LangChain Agent node ("AI Agent"):**  
   - Set the prompt text using expressions:  
     ```
     Name: {{ $('On form submission').item.json.Name }}
     Job Name: {{ $('On form submission').item.json.Job }}
     Resume: {{ $json.text }}
     ```  
   - System message instructs the AI to score the resume out of 10, determine status (Accepted if score ≥7, else Rejected), and generate email subject/body accordingly (see workflow example for exact phrasing).  
   - Set the AI language model node to "Google 2.5 Flash" (link LM node).  
   - Connect input from "Extract from File".

6. **Add a LangChain Information Extractor node ("Information Extractor"):**  
   - Input text: `={{ $json.output }}` (from AI Agent output)  
   - Define attributes to extract:  
     - Score (number, required)  
     - Status (string, required)  
     - Mail Subject (string, required)  
     - Mail Body (string)  
   - Connect input from "AI Agent".

7. **Add a Gmail node ("Gmail"):**  
   - Authenticate using OAuth2 for Gmail.  
   - Send To: `={{ $('On form submission').item.json['Email Address'] }}`  
   - Subject: `={{ $json.output['Mail Subject'] }}`  
   - Message: `={{ $json.output['Mail Body'] }}`  
   - Email Type: Text  
   - Disable "Append Attribution" to avoid footer text.  
   - Connect input from "Information Extractor".

8. **Add a Google Sheets node ("Google Sheets"):**  
   - Authenticate with Google Sheets OAuth2.  
   - Operation: Append  
   - Set Document ID to your target Google Sheet.  
   - Sheet Name: "Form responses 1" (or your sheet tab name).  
   - Define columns mapping:  
     - Job: `={{ $('On form submission').item.json.Job }}`  
     - Name: `={{ $('On form submission').item.json.Name }}`  
     - Email: `={{ $('On form submission').item.json['Email Address'] }}`  
     - Score: `={{ $('Information Extractor').item.json.output.Score }}`  
     - Status: `={{ $('Information Extractor').item.json.output.Status }}`  
     - Email Status: Set static value "SENT ✅"  
   - Connect input from "Gmail".

9. **Connect node chain:**  
   `On form submission` → `Extract from File` → `AI Agent` → `Information Extractor` → `Gmail` → `Google Sheets`.

10. **Add sticky notes for setup guidance and tutorial links as needed for user reference.**

11. **Test end-to-end:**  
    - Submit a form entry with a PDF resume.  
    - Verify text extraction, AI scoring, email sending, and sheet logging.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Setup guide includes detailed steps for form customization, credential configuration (Google Gemini, Gmail, Sheets), and emails. | Embedded in the workflow via sticky note titled "🛠 Setup Guide".                                   |
| Tutorial video linked: "I Built an Auto Lead Finder AI Agent" on YouTube.                                                         | [https://youtu.be/5bXPl3ud-VA](https://youtu.be/5bXPl3ud-VA)                                        |
| Google Gemini API key required for AI Agent node; signup and key setup at Google AI Studio.                                       | [https://aistudio.google.com/apikey](https://aistudio.google.com/apikey)                            |
| Gmail OAuth2 authorization needed to send emails automatically from the workflow.                                                 | [https://mail.google.com/](https://mail.google.com/)                                               |
| Google Sheets must have columns: Name, Email, Job, Score, Status, Email Status for proper logging.                                  | [https://docs.google.com/spreadsheets/](https://docs.google.com/spreadsheets/)                      |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow. It complies fully with legal and content policies. All data processed is public and lawful.