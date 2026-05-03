Automated Resume Screening with GPT-4o & Error Handling - Google Sheets & Drive Pipeline

https://n8nworkflows.xyz/workflows/automated-resume-screening-with-gpt-4o---error-handling---google-sheets---drive-pipeline-9048


# Automated Resume Screening with GPT-4o & Error Handling - Google Sheets & Drive Pipeline

### 1. Workflow Overview

This workflow automates the screening of resumes submitted via email, using AI-powered analysis with GPT-4o to evaluate candidate fit and logs results into Google Sheets. It is designed for HR teams, recruiting agencies, and startups to streamline resume processing, candidate evaluation, and error handling. The workflow includes these main logical blocks:

- **1.1 Input Reception & Monitoring**: Watches a Gmail inbox for incoming resumes with attachments.
- **1.2 File Handling & Conversion**: Saves attachments to Google Drive and processes different file formats (PDF, DOCX, TXT) to extract text.
- **1.3 Resume Text Validation & Standardization**: Validates extracted text quality and prepares standardized resume data.
- **1.4 AI-Based Candidate Analysis**: Uses GPT-4o to score and analyze candidates against a defined job description, outputting structured JSON.
- **1.5 Candidate Data Extraction & Validation**: Extracts key candidate info from resume text and validates AI output completeness.
- **1.6 Logging & Reporting**: Logs successful candidate evaluations to Google Sheets and handles various error conditions.
- **1.7 Error Handling & Notifications**: Captures different failure types, sets error context, merges errors, sends email notifications, and logs errors for manual review.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Monitoring

**Overview:**  
Monitors a Gmail inbox continuously for incoming emails with resume attachments. Downloads attachments for further processing.

**Nodes Involved:**  
- Monitor Resumes (Gmail Trigger)  
- Email Monitoring (Sticky Note)

**Node Details:**

- **Monitor Resumes**  
  - Type: Gmail Trigger  
  - Config: Polls inbox every minute, downloads attachments automatically.  
  - Inputs: Incoming Gmail messages  
  - Outputs: Email JSON with attachment data  
  - Edge Cases: Gmail authentication errors, attachment download failures, empty inbox polls.

- **Email Monitoring** (Sticky Note)  
  - Provides documentation on monitoring scope and supported file types (PDF, DOCX, TXT).

---

#### 1.2 File Handling & Conversion

**Overview:**  
Saves the resume attachments to Google Drive and routes files based on MIME type for appropriate text extraction or conversion.

**Nodes Involved:**  
- Save to Drive (Google Drive)  
- Upload Success? (If)  
- Route by File Type (Switch)  
- Extract PDF Text (Extract from File)  
- Convert DOCX to Docs (HTTP Request)  
- Get Doc as Text (Google Drive)  
- Download TXT (Google Drive)  
- Extract TXT Content (Extract from File)  
- Get Doc (Google Drive)  
- File Processing (Sticky Note)

**Node Details:**

- **Save to Drive**  
  - Type: Google Drive  
  - Saves attachment with sanitized filename containing subject and timestamp to root folder.  
  - Credentials: Google Drive OAuth2  
  - Outputs: File metadata including Drive file ID and link.  
  - Edge Cases: Upload failures, permission errors.

- **Upload Success?**  
  - Type: If  
  - Checks if uploaded file JSON contains file ID and webViewLink.  
  - Routes success to file type routing; failures to error handling.

- **Route by File Type**  
  - Type: Switch  
  - Routes based on MIME type: application/pdf, application/vnd.openxmlformats-officedocument.wordprocessingml.document, text/plain.  
  - Outputs: PDF, DOCX, TXT paths.

- **Extract PDF Text**  
  - Type: Extract from File  
  - Extracts up to 10 pages of text from PDF file.  
  - Input: File from Drive (Get Doc node).  
  - Output: Extracted text.

- **Convert DOCX to Docs**  
  - Type: HTTP Request  
  - Uses Google Drive API to copy DOCX file as Google Docs document for easier text extraction.  
  - Credentials: Google Drive OAuth2  
  - Edge Cases: API timeout, conversion failure.

- **Get Doc as Text**  
  - Type: Google Drive  
  - Downloads Google Docs file as plain text.

- **Download TXT**  
  - Type: Google Drive  
  - Downloads the TXT file directly.

- **Extract TXT Content**  
  - Type: Extract from File  
  - Extracts text content from downloaded TXT file.

- **Get Doc**  
  - Type: Google Drive  
  - Downloads DOCX file converted to PDF (used for PDF text extraction).  

- **File Processing** (Sticky Note)  
  - Documents multi-format routing strategy converging into standardized text output.

---

#### 1.3 Resume Text Validation & Standardization

**Overview:**  
Validates extracted text length and content quality, then consolidates it into a standard format with references to original email and Drive link.

**Nodes Involved:**  
- Text Extracted? (If)  
- Standardize Resume Data (Set)  
- Resume Quality Check (If)  
- Set Extraction Error (Set)  
- Set Quality Error (Set)

**Node Details:**

- **Text Extracted?**  
  - Type: If  
  - Validates if extracted text length > 50 characters.  
  - Routes valid text to standardization; invalid to extraction error.

- **Standardize Resume Data**  
  - Type: Set  
  - Assigns candidateResume field with extracted text from any source (PDF, DOCX, TXT).  
  - Also stores original email JSON and Google Drive link for reference.

- **Resume Quality Check**  
  - Type: If  
  - Checks if candidateResume length > 100 and contains keywords like "experience", "skills", or "education".  
  - Routes valid resumes forward to analysis; poor quality triggers quality error.

- **Set Extraction Error**  
  - Type: Set  
  - Sets error_type to "Text Extraction Failed" with message and file info.

- **Set Quality Error**  
  - Type: Set  
  - Sets error_type to "Poor Resume Quality" with a snippet of resume data for diagnostics.

---

#### 1.4 AI-Based Candidate Analysis

**Overview:**  
Feeds standardized resume text and a fixed job description into GPT-4o AI to generate detailed candidate evaluation scored from 1 to 10, structured as JSON.

**Nodes Involved:**  
- Job Description (Set)  
- AI Recruiter Analysis (LangChain Agent)  
- GPT-4o Model (LM Chat OpenAI)  
- Structured Output Parser (LangChain Output Parser)  
- AI Analysis Success? (If)  
- Set AI Error (Set)  
- AI Analysis Engine (Sticky Note)

**Node Details:**

- **Job Description**  
  - Type: Set  
  - Contains fixed detailed job description for Senior Software Engineer with required and preferred skills and responsibilities.

- **AI Recruiter Analysis**  
  - Type: LangChain Agent  
  - Prompt includes candidate resume and job description.  
  - System message frames AI as expert recruiter evaluating specific dimensions with output in JSON format.  
  - Output expected: detailed analysis and scoring.

- **GPT-4o Model**  
  - Type: LangChain LM Chat OpenAI node  
  - Uses model "gpt-4o-mini" with temperature 0.3 and max tokens 2000.  
  - Credentials: OpenAI API key.  
  - Input: passed from AI Recruiter Analysis node.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Validates and parses AI output JSON against strict schema defining strengths, weaknesses, risk, scores, and recommendations.

- **AI Analysis Success?**  
  - Type: If  
  - Checks if AI output exists and contains overall_score and candidate_strengths.  
  - Routes success to candidate info extraction; failures to AI error.

- **Set AI Error**  
  - Type: Set  
  - Sets error_type "AI Processing Failed" with error message and AI output snapshot.

- **AI Analysis Engine** (Sticky Note)  
  - Documents AI evaluation framework and detailed output expectations.

---

#### 1.5 Candidate Data Extraction & Validation

**Overview:**  
Extracts key candidate information fields from standardized resume text and validates overall output completeness before logging.

**Nodes Involved:**  
- Extract Candidate Info (LangChain Information Extractor)  
- Final Data Valid? (If)  
- Set Validation Error (Set)

**Node Details:**

- **Extract Candidate Info**  
  - Type: LangChain Information Extractor  
  - Extracts full_name, email_address, phone_number, current_title, years_experience, key_skills from resume text.  
  - CandidateResume text passed as input.

- **Final Data Valid?**  
  - Type: If  
  - Validates presence of full_name, email_address, and AI overall_score.  
  - Routes valid data to logging; invalid to validation error.

- **Set Validation Error**  
  - Type: Set  
  - Sets error_type "Data Validation Failed" with message and a JSON snapshot of candidate info and AI analysis outputs.

---

#### 1.6 Logging & Reporting

**Overview:**  
Logs successfully processed candidate data into a Google Sheets spreadsheet for dashboard and records.

**Nodes Involved:**  
- Log Successful Processing (Google Sheets)  
- Setup Guide (Sticky Note)

**Node Details:**

- **Log Successful Processing**  
  - Type: Google Sheets  
  - Appends a row with date, email, resume link, full name, strengths, key skills, weaknesses, overall fit score, risk factor, justification, and reward factor.  
  - Uses a shared Google Sheets template (link provided in Setup Guide).  
  - Credentials: Google Sheets OAuth2.  
  - Edge Cases: Spreadsheet access or quota errors.

- **Setup Guide** (Sticky Note)  
  - Provides overview, features, setup instructions, and use cases.

---

#### 1.7 Error Handling & Notifications

**Overview:**  
Centralizes error setting from all failure points, merges errors, sends email notifications about errors, and logs error details to a dedicated Google Sheet for manual review.

**Nodes Involved:**  
- Set Upload Error (Set)  
- Set Extraction Error (Set)  
- Set Quality Error (Set)  
- Set AI Error (Set)  
- Set Validation Error (Set)  
- Merge All Errors (Merge)  
- Send Error Notification (Gmail)  
- Log Error to Sheet (Google Sheets)  
- Error Handling (Sticky Note)

**Node Details:**

- **Set Upload Error**  
  - Sets error_type "Upload Failed" with message and original email data.

- **Merge All Errors**  
  - Combines all error outputs into one stream for notification.

- **Send Error Notification**  
  - Sends Gmail to admin email with error type, message, timestamp, original email metadata, and suggested next steps.  
  - Credentials: Gmail OAuth2.

- **Log Error to Sheet** (Disabled)  
  - Intended to log errors with timestamp, status, file link, error details, and original email info to a Google Sheet.  
  - Currently disabled.

- **Error Handling** (Sticky Note)  
  - Documents multi-layer validation, error recovery strategies including notifications and manual review triggers.

---

### 3. Summary Table

| Node Name                | Node Type                         | Functional Role                  | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                                    |
|--------------------------|----------------------------------|---------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------|
| Setup Guide              | Sticky Note                      | Workflow overview and instructions | -                                | -                                | # 🎯 AI Resume Screening & Candidate Pipeline with error handling; Setup instructions and use cases           |
| Monitor Resumes          | Gmail Trigger                   | Incoming email monitoring       | -                                | Save to Drive                    | Watches Gmail inbox for resumes; supports PDF, DOCX, TXT                                                      |
| Email Monitoring         | Sticky Note                     | Email monitoring documentation  | -                                | -                                | Describes supported file formats and monitoring scope                                                        |
| Save to Drive            | Google Drive                    | Save attachments to Drive       | Monitor Resumes                  | Upload Success?                  | Saves resume files with sanitized names                                                                       |
| Upload Success?          | If                             | Verify successful upload        | Save to Drive                   | Route by File Type / Set Upload Error | Checks Drive upload success                                                                                    |
| Route by File Type       | Switch                         | Route by file MIME type         | Upload Success?                 | Get Doc / Convert DOCX to Docs / Download TXT | Routes PDF, DOCX, TXT to appropriate text extraction paths                                                    |
| Extract PDF Text         | Extract from File               | Extract text from PDF           | Get Doc                        | Text Extracted?                  | Extracts up to 10 pages of PDF text                                                                            |
| Convert DOCX to Docs     | HTTP Request                   | Convert DOCX to Google Docs     | Route by File Type              | Get Doc as Text                  | Uses Google Drive API to convert DOCX                                                                          |
| Get Doc as Text          | Google Drive                   | Download Google Docs as text    | Convert DOCX to Docs            | Text Extracted?                  | Downloads converted DOCX document as plain text                                                                |
| Download TXT             | Google Drive                   | Download TXT file               | Route by File Type              | Extract TXT Content              | Downloads TXT file                                                                                              |
| Extract TXT Content      | Extract from File               | Extract text from TXT           | Download TXT                   | Text Extracted?                  | Extracts text content from TXT                                                                                  |
| Get Doc                  | Google Drive                   | Download DOCX as PDF for PDF extraction | Route by File Type              | Extract PDF Text                | Downloads DOCX file converted to PDF                                                                            |
| File Processing          | Sticky Note                    | File format routing overview   | -                              | -                              | Describes multi-format processing converging to text extraction                                               |
| Text Extracted?          | If                             | Validate extracted text presence | Extract PDF Text / Extract TXT Content / Get Doc as Text | Standardize Resume Data / Set Extraction Error | Validates if extracted text >50 chars                                                                           |
| Standardize Resume Data  | Set                            | Consolidate resume text and metadata | Text Extracted?                | Resume Quality Check             | Prepares candidateResume field and attaches email and Drive info                                              |
| Resume Quality Check     | If                             | Validate resume content quality | Standardize Resume Data         | Job Description / Set Quality Error | Checks for minimum length and presence of keywords                                                             |
| Set Extraction Error     | Set                            | Set extraction error context   | Text Extracted? (failure)       | Merge All Errors                | Error info: text extraction failure                                                                            |
| Set Quality Error        | Set                            | Set quality error context      | Resume Quality Check (failure)  | Merge All Errors                | Error info: poor resume quality                                                                                 |
| Job Description          | Set                            | Provide job description text   | Resume Quality Check            | AI Recruiter Analysis           | Static job description for candidate evaluation                                                                |
| AI Recruiter Analysis    | LangChain Agent                | AI evaluation of candidate     | Job Description                | AI Analysis Success?            | Runs GPT-4o analysis with detailed prompt and system message                                                  |
| GPT-4o Model            | LangChain LM Chat OpenAI        | OpenAI GPT-4o model invocation | AI Recruiter Analysis           | AI Recruiter Analysis           | Uses GPT-4o-mini model with temperature 0.3                                                                    |
| Structured Output Parser | LangChain Output Parser         | Parse AI output JSON           | AI Recruiter Analysis           | AI Recruiter Analysis (output)  | Validates AI output against strict JSON schema                                                                 |
| AI Analysis Success?     | If                             | Validate AI output completeness | AI Recruiter Analysis           | Extract Candidate Info / Set AI Error | Checks for presence of score and strengths                                                                     |
| Set AI Error             | Set                            | Set AI processing error        | AI Analysis Success? (failure)  | Merge All Errors                | Error info: AI analysis failure                                                                                 |
| AI Analysis Engine       | Sticky Note                    | AI analysis logic documentation | -                              | -                              | Describes AI evaluation framework and output requirements                                                      |
| Extract Candidate Info   | LangChain Information Extractor | Extract key candidate data     | AI Analysis Success?            | Final Data Valid?               | Extracts full name, email, phone, title, experience, skills                                                    |
| Final Data Valid?        | If                             | Validate final data completeness | Extract Candidate Info          | Log Successful Processing / Set Validation Error | Ensures candidate info and AI score are present                                                                |
| Set Validation Error     | Set                            | Set data validation error      | Final Data Valid? (failure)     | Merge All Errors                | Error info: missing candidate or AI data                                                                        |
| Log Successful Processing | Google Sheets                 | Log candidate data to sheet   | Final Data Valid?               | -                              | Appends candidate evaluation data to Google Sheets dashboard                                                  |
| Set Upload Error         | Set                            | Set upload error context      | Upload Success? (failure)       | Merge All Errors                | Error info: file upload failure                                                                                  |
| Merge All Errors         | Merge                          | Combine all error streams     | Set Upload Error / Set Extraction Error / Set Quality Error / Set AI Error / Set Validation Error | Send Error Notification | Consolidates errors for notification                                                                             |
| Send Error Notification  | Gmail                          | Notify admin of errors        | Merge All Errors               | Log Error to Sheet              | Sends email with error details and next steps to admin                                                        |
| Log Error to Sheet       | Google Sheets (disabled)        | Log errors for manual review  | Send Error Notification         | -                              | Disabled node intended to log errors in Google Sheets                                                          |
| Error Handling           | Sticky Note                    | Error handling overview       | -                              | -                              | Documents multi-level validation, graceful failure, notifications, manual review                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail account.  
   - Set to poll every minute and download email attachments.

2. **Add Google Drive Node “Save to Drive”**  
   - Type: Google Drive  
   - Configure OAuth2 credentials for Google Drive.  
   - Set filename to sanitized email subject with timestamp.  
   - Save to root folder or desired folder.  
   - Input data field: attachment_0 from Gmail trigger.

3. **Add If Node “Upload Success?”**  
   - Check if output JSON from Google Drive contains `id` and `webViewLink`.  
   - True: proceed to file type routing; False: route to error setting.

4. **Add Switch Node “Route by File Type”**  
   - Route based on MIME type:  
     - PDF: application/pdf  
     - DOCX: application/vnd.openxmlformats-officedocument.wordprocessingml.document  
     - TXT: text/plain

5. **For PDFs:**  
   - Add Google Drive Node “Get Doc” to download file.  
   - Add Extract from File Node “Extract PDF Text” to extract text (max 10 pages).

6. **For DOCX:**  
   - Add HTTP Request Node “Convert DOCX to Docs” using Google Drive API to copy the file as Google Docs MIME type.  
   - Add Google Drive Node “Get Doc as Text” to download converted doc as plain text.

7. **For TXT:**  
   - Add Google Drive Node “Download TXT” to download the file.  
   - Add Extract from File Node “Extract TXT Content” to extract text.

8. **Add If Node “Text Extracted?”**  
   - Check if extracted text length > 50 characters (check multiple possible keys).  
   - True: proceed to standardize data; False: set extraction error.

9. **Add Set Node “Standardize Resume Data”**  
   - Set `candidateResume` to extracted text.  
   - Store original email JSON from Gmail trigger.  
   - Store Google Drive file link.

10. **Add If Node “Resume Quality Check”**  
    - Verify if candidateResume length > 100 and contains keywords “experience”, “skills”, or “education”.  
    - True: proceed; False: set quality error.

11. **Add Set Node “Job Description”**  
    - Set static job description text for target position (e.g., Senior Software Engineer).

12. **Add LangChain Agent Node “AI Recruiter Analysis”**  
    - Configure prompt with candidateResume text and jobDescription.  
    - Set system message as expert recruiter with detailed analysis framework.  
    - Connect to GPT-4o model node.

13. **Add LangChain LM Chat OpenAI Node “GPT-4o Model”**  
    - Use model “gpt-4o-mini” with temperature 0.3, max tokens 2000.  
    - Configure OpenAI API credentials.

14. **Add Structured Output Parser Node**  
    - Define JSON schema for AI output with strengths, weaknesses, risk, score, justification, next steps.

15. **Add If Node “AI Analysis Success?”**  
    - Check for presence of overall_score and candidate_strengths in AI output.  
    - True: proceed to candidate info extraction; False: set AI error.

16. **Add LangChain Information Extractor Node “Extract Candidate Info”**  
    - Extract full_name, email_address, phone_number, current_title, years_experience, key_skills from candidateResume.

17. **Add If Node “Final Data Valid?”**  
    - Validate presence of full_name, email_address, and AI overall_score.  
    - True: proceed to logging; False: set validation error.

18. **Add Google Sheets Node “Log Successful Processing”**  
    - Connect to Google Sheets with OAuth2 credentials.  
    - Append row with date, email, resume link, full name, strengths, key skills, weaknesses, overall fit, risk, justification, reward factor.

19. **Create Error Handling Set Nodes:**  
    - “Set Upload Error”: error_type="Upload Failed"  
    - “Set Extraction Error”: error_type="Text Extraction Failed"  
    - “Set Quality Error”: error_type="Poor Resume Quality"  
    - “Set AI Error”: error_type="AI Processing Failed"  
    - “Set Validation Error”: error_type="Data Validation Failed"  
    - Each sets relevant error messages and context.

20. **Add Merge Node “Merge All Errors”**  
    - Combine all error outputs into one stream.

21. **Add Gmail Node “Send Error Notification”**  
    - Configure Gmail OAuth2 credentials.  
    - Set recipient email for error notification.  
    - Email body includes error type, message, timestamp, original email details, and next steps.

22. **(Optional) Add Google Sheets Node “Log Error to Sheet”**  
    - Append error details to a dedicated error log sheet (disabled in original).

23. **Add Sticky Notes for documentation**  
    - Provide overview, file processing logic, AI analysis framework, error handling details, and setup instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                                   |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Copy the [Google Sheet Template](https://docs.google.com/spreadsheets/d/1vucZgBrULNToEQMAQrFyWczpOzXxBU6rzCX0waIZyeM/edit?gid=0) for logging parsed data | Required for workflow to write candidate data                                                                                     |
| Workflow created by David Olusola                                                                                  | Author credit                                                                                                                    |
| Error handling includes multi-layer validation and graceful failure recovery with email notifications             | See “Error Handling” sticky note for details                                                                                     |
| AI model used is GPT-4o-mini with a custom prompt framing analysis as expert technical recruiter                   | See “AI Analysis Engine” sticky note for analysis framework                                                                      |
| Workflow supports multiple resume formats: PDF, DOCX, TXT                                                         | See “Email Monitoring” and “File Processing” sticky notes                                                                        |
| Email notifications for errors include detailed context and recommended next steps                                 | Configured in “Send Error Notification” Gmail node                                                                                |

---

This structured documentation enables comprehensive understanding, modification, and reproduction of the Automated Resume Screening workflow with GPT-4o and robust error handling.