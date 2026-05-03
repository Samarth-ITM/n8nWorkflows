Fast-track candidate CV screening with Gemini AI analysis from Gmail to Slack and Google Sheets

https://n8nworkflows.xyz/workflows/fast-track-candidate-cv-screening-with-gemini-ai-analysis-from-gmail-to-slack-and-google-sheets-9764


# Fast-track candidate CV screening with Gemini AI analysis from Gmail to Slack and Google Sheets

### 1. Workflow Overview

This workflow, titled **"First-Round Fast Track AI Recruiter Assistant"**, automates the initial screening of candidate CVs received via Gmail by leveraging Google Gemini AI for intelligent analysis and job description (JD) matching. It streamlines recruitment by:

- Monitoring Gmail for incoming CV attachments labeled for screening.
- Automatically handling CVs in PDF and Word formats, converting and standardizing them.
- Matching candidates to relevant job descriptions using a two-stage AI approach (email-based priority, fallback to CV content).
- Performing an AI-driven detailed screening of candidate fit against the selected JD.
- Extracting key candidate information.
- Logging results into a Google Sheet for audit and tracking.
- Sending actionable summaries and decision buttons to a Slack channel.

The workflow is logically grouped into these blocks:

- **1.1 Input Reception and File Handling:** Gmail trigger, file type detection, upload and conversion of CVs.
- **1.2 CV Text Extraction and Standardization:** Extract text from CVs, prepare data for AI analysis.
- **1.3 Job Description Matching:** Two-stage AI agent logic to select relevant JDs, including fallback and refinement.
- **1.4 Detailed Candidate Screening:** AI analysis of candidate CV against the selected JD, generating strengths, weaknesses, risk/reward, and fit scoring.
- **1.5 Candidate Information Extraction:** Parsing candidate personal details from CV text.
- **1.6 Data Logging and Notification:** Append results to Google Sheets and send Slack messages with interactive decision buttons.
- **1.7 Auxiliary and Utility Nodes:** Various helper nodes for formatting, looping, error handling, and credential management.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and File Handling

**Overview:**  
This block listens to Gmail for new candidate CV emails, detects attachment file types (PDF or Word), and initiates appropriate processing paths including uploading files to Google Drive and converting Word docs to Google Docs.

**Nodes Involved:**  
- Receive CV via Email  
- Switch - File Type  
- Stream Doc/Docx File  
- Preserve CV file  
- Merge  
- Upload CV - PDF  
- Download CV - PDF  
- Download CV - GDoc as PDF  
- Trigger Google Docs Conversion  
- Get Web Link

**Node Details:**

- **Receive CV via Email**  
  - Type: Gmail Trigger  
  - Role: Watches Gmail inbox label for incoming candidate CV emails with attachments. Downloads attachments for processing.  
  - Config: Filters by Gmail label (e.g., "CV-Screening"), downloads attachments, prefixing attachment data for clarity.  
  - Inputs: None (trigger node)  
  - Outputs: Email data with attachments  
  - Edge Cases: Missing attachments, label misconfigured, Gmail API auth errors.

- **Switch - File Type**  
  - Type: Switch  
  - Role: Branches workflow based on attachment MIME type (PDF vs Word doc).  
  - Config: Checks MIME type of binary data; routes Word docs to conversion, PDFs to direct upload.  
  - Inputs: Output from Gmail trigger  
  - Outputs: Two outputs—"Doc-Docx" and "PDF" paths  
  - Edge Cases: Unknown MIME types, corrupt files.

- **Stream Doc/Docx File**  
  - Type: HTTP Request (Google Drive API resumable upload)  
  - Role: Uploads Word/Docx file as Google Doc to destination Drive folder.  
  - Config: Creates Google Doc with naming convention based on sender and timestamp; uploads binary data.  
  - Inputs: Word doc binary from Switch node  
  - Outputs: HTTP response with new Google Doc metadata  
  - Edge Cases: Google Drive API quota limits, upload failures.

- **Preserve CV file**  
  - Type: Code  
  - Role: Passes binary data unchanged; ensures binary file persists alongside converted Google Doc data for merging.  
  - Inputs: Word doc binary  
  - Outputs: Same binary data  
  - Edge Cases: None significant.

- **Merge**  
  - Type: Merge  
  - Role: Combines the Google Doc metadata and original binary attachment into one item for further processing.  
  - Inputs: From Stream Doc/Docx File and Preserve CV file  
  - Outputs: Single combined item  
  - Edge Cases: Merge conflicts if input arrays mismatch.

- **Upload CV - PDF**  
  - Type: Google Drive  
  - Role: Uploads PDF CV directly to Google Drive folder for CV storage.  
  - Config: Uses naming convention with sender name and timestamp; stores in "Candidate CVs" folder.  
  - Inputs: PDF binary from Switch node  
  - Outputs: Google Drive file metadata  
  - Edge Cases: Drive quota, permissions.

- **Download CV - PDF**  
  - Type: Google Drive  
  - Role: Downloads just-uploaded PDF CV for text extraction.  
  - Inputs: File ID from Upload CV - PDF  
  - Outputs: Binary PDF file  
  - Edge Cases: File not found, permissions.

- **Download CV - GDoc as PDF**  
  - Type: Google Drive  
  - Role: Downloads Google Doc CV as converted PDF for consistent extraction downstream.  
  - Inputs: Google Doc file ID from Stream Doc/Docx File  
  - Outputs: PDF binary  
  - Edge Cases: Conversion errors, API limits.

- **Trigger Google Docs Conversion**  
  - Type: HTTP Request (Google Drive API)  
  - Role: PUT request to update metadata or trigger conversion? (Used here possibly to finalize Google Doc upload)  
  - Inputs: Google Drive file metadata  
  - Outputs: Updated file metadata  
  - Edge Cases: API errors.

- **Get Web Link**  
  - Type: HTTP Request (Google Drive API)  
  - Role: Retrieves webViewLink for uploaded Google Doc CV to share with stakeholders.  
  - Inputs: File ID from Trigger Google Docs Conversion  
  - Outputs: File metadata including webViewLink  
  - Edge Cases: Permission errors, missing file.

---

#### 1.2 CV Text Extraction and Standardization

**Overview:**  
Extracts textual content from the CV PDF or Google Doc and standardizes output to a common format for AI analysis.

**Nodes Involved:**  
- Extract from PDF  
- Extract from PDF Download  
- Extract from File  
- Extract from File1  
- Standardize Web Link and CV Text (PDF)  
- Standardize Web Link and CV Text (GDoc)  
- Standardize

**Node Details:**

- **Extract from PDF**, **Extract from PDF Download**  
  - Type: Extract From File  
  - Role: Parses PDF binary to extract readable text from candidate CV.  
  - Inputs: PDF binary from Google Drive download nodes  
  - Outputs: Text extracted from CV  
  - Edge Cases: Poor PDF quality, scanned images not recognized.

- **Extract from File**, **Extract from File1**  
  - Type: Extract From File  
  - Role: Extracts text from PDFs of Job Descriptions.  
  - Inputs: Downloaded JD PDF binary  
  - Outputs: Extracted JD text  
  - Edge Cases: As above.

- **Standardize Web Link and CV Text (PDF)**, **Standardize Web Link and CV Text (GDoc)**  
  - Type: Set  
  - Role: Sets standardized JSON properties `cv_text` and `cv_webviewlink` for downstream use.  
  - Inputs: Extracted CV text and webViewLink from respective nodes  
  - Outputs: Structured JSON for AI nodes  
  - Edge Cases: Missing text or links.

- **Standardize**  
  - Type: Set  
  - Role: Final normalization step setting `cv_text` and `cv_webviewlink` fields to unify data structure.  
  - Inputs: From either PDF or GDoc paths  
  - Outputs: Standardized candidate CV data.

---

#### 1.3 Job Description Matching

**Overview:**  
Uses AI agents to analyze candidate email and CV text to identify the most relevant job description(s) from a Google Drive folder. It prioritizes email content for direct JD matching and falls back to CV content otherwise. If multiple JDs match, a detailed AI agent selects the single best JD.

**Nodes Involved:**  
- JD Matching Agent  
- Structured Output Parser-1  
- JD Match w/Email? (If node)  
- Download Selected JD  
- Transform for Multiple JDs  
- Loop Over Items  
- Detailed JD Matching Agent  
- Structured Output Parser-2  
- Set  
- Match Selected JD Name with Full Text  
- Gemini 2.5 Flash  
- Gemini 2.5 Pro-1

**Node Details:**

- **JD Matching Agent**  
  - Type: LangChain AI Agent (Google Gemini)  
  - Role: Matches candidate to relevant JDs using email subject/body and CV text as fallback, returning structured JSON with either a single email match or multiple CV matches.  
  - Inputs: Standardized CV text, email content, list of JD files (from Google Drive Tool)  
  - Outputs: Structured JSON with match_type and matched JD(s) info  
  - Edge Cases: Ambiguous email content, missing JD files, parsing errors.

- **Structured Output Parser-1**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates and parses JSON output from JD Matching Agent into structured data for downstream nodes.  
  - Inputs: Raw AI agent output  
  - Outputs: Structured JSON  
  - Edge Cases: Parsing failures if AI output deviates.

- **JD Match w/Email?**  
  - Type: If  
  - Role: Branches workflow based on whether JD matching was from email or fallback CV match (match_type).  
  - Inputs: Parsed output from JD Matching Agent  
  - Outputs: Two paths—email_match or cv_match.

- **Download Selected JD**  
  - Type: Google Drive Download  
  - Role: Downloads the JD file identified by email_match for detailed analysis.  
  - Inputs: JD file ID from email_match  
  - Outputs: JD PDF binary.

- **Transform for Multiple JDs**  
  - Type: Code  
  - Role: Converts AI output array (up to 3 matched JDs) into individual items for looping. For email match, returns a single item.  
  - Inputs: Parsed AI output  
  - Outputs: Array of JD objects with filename and file ID.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes each matched JD separately for detailed analysis.  
  - Inputs: Array of matched JDs  
  - Outputs: Single JD item per iteration.

- **Detailed JD Matching Agent**  
  - Type: LangChain AI Agent (Google Gemini, Pro model)  
  - Role: Analyzes candidate CV against up to 3 matched JDs to select the single best JD match.  
  - Inputs: Candidate CV text, multiple JD texts and filenames  
  - Outputs: Selected JD filename in structured JSON  
  - Edge Cases: Ambiguous best match, AI errors.

- **Structured Output Parser-2**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses output from detailed JD matching agent.  
  - Inputs: Raw AI output  
  - Outputs: Structured JSON with selected JD filename.

- **Set**  
  - Type: Set  
  - Role: Prepares JD data for detailed screening by combining JD filename, file ID, and text.  
  - Inputs: Loop item with JD info and extracted text  
  - Outputs: Structured JD data.

- **Match Selected JD Name with Full Text**  
  - Type: Code  
  - Role: Finds and returns full details of the selected JD by filename from all looped JDs for final screening.  
  - Inputs: Selected JD filename, all JD data from loop  
  - Outputs: Selected JD filename, file ID, and full text.

- **Gemini 2.5 Flash**, **Gemini 2.5 Pro-1**  
  - Type: LangChain AI Language Model (Google Gemini)  
  - Role: Provide AI inference for JD matching agents. Flash version for initial matching, Pro for detailed selection.  
  - Inputs: Prompt text with candidate and JD data  
  - Outputs: AI-generated JSON responses.

---

#### 1.4 Detailed Candidate Screening

**Overview:**  
This block uses AI to analyze the candidate's CV against the selected JD to generate a detailed screening report that includes strengths, weaknesses, risk/reward factors, overall fit score, and justification.

**Nodes Involved:**  
- Recruiter Scoring Agent  
- Structured Output Parser-3  
- Gemini 2.5 Pro-2

**Node Details:**

- **Recruiter Scoring Agent**  
  - Type: LangChain AI Agent (Google Gemini, Pro model)  
  - Role: Performs an expert-level evaluation of candidate CV vs JD, producing structured screening output with strengths, weaknesses, risk factor, reward factor, overall fit rating (0-10), and justification.  
  - Inputs: Candidate CV text, selected JD filename and text  
  - Outputs: Structured JSON report  
  - Edge Cases: AI hallucinations, missing candidate or JD data.

- **Structured Output Parser-3**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses and validates the JSON screening report from AI.  
  - Inputs: Raw AI output  
  - Outputs: Structured screening data.

- **Gemini 2.5 Pro-2**  
  - Type: LangChain AI Language Model (Google Gemini, Pro model)  
  - Role: Provides inference for the Recruiter Scoring Agent.  
  - Inputs: Prompt text with CV and JD data  
  - Outputs: Screening report JSON.

---

#### 1.5 Candidate Information Extraction

**Overview:**  
Extracts candidate personal details such as first name, last name, and email address from the CV text for record keeping.

**Nodes Involved:**  
- Information Extractor

**Node Details:**

- **Information Extractor**  
  - Type: LangChain Information Extractor  
  - Role: Uses AI to parse CV text and extract key candidate information fields.  
  - Config: Extracts First Name (required), Last Name (required), Email Address (optional).  
  - Inputs: Standardized CV text  
  - Outputs: Candidate personal details in structured JSON  
  - Edge Cases: Poor CV text quality, missing email.

---

#### 1.6 Data Logging and Notification

**Overview:**  
Logs the candidate screening data into a Google Sheet and sends a formatted notification message with interactive action buttons to a Slack channel for recruiting team decisions.

**Nodes Involved:**  
- Append row in sheet  
- Send Candidate Screening Confirmation

**Node Details:**

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Appends a new row with candidate data and AI screening results into the "AI Candidate Screening" Google Sheet.  
  - Config: Maps fields like CV link, date, candidate names, JD match, strengths, weaknesses, risk/reward factors, overall fit, justification, submission ID, and sender email.  
  - Inputs: Candidate info, screening results from AI nodes  
  - Outputs: Sheet append confirmation  
  - Edge Cases: Sheet access permissions, column mismatch.

- **Send Candidate Screening Confirmation**  
  - Type: Slack  
  - Role: Sends a Slack message to a specified channel summarizing candidate screening results with formatted blocks and interactive Proceed/Reject buttons.  
  - Config: Uses Slack Block Kit with dynamic content placeholders; includes a link to the Google Sheet.  
  - Inputs: Data from appended sheet row  
  - Outputs: Slack message posted  
  - Edge Cases: Slack API permissions, invalid channel ID.

---

#### 1.7 Auxiliary and Utility Nodes

**Overview:**  
Helper nodes for data transformation, looping, code evaluation, and user notes.

**Nodes Involved:**  
- Set  
- Code (various)  
- Split In Batches (Loop Over Items)  
- Sticky Notes (documentation and instructions)

**Node Details:**

- **Set**  
  - Used for setting or transforming variables and preparing data for subsequent nodes.

- **Code**  
  - Custom JavaScript for transforming AI output or matching selected JD to looped data.

- **Split In Batches**  
  - Processes arrays of matched JDs one at a time for detailed analysis.

- **Sticky Notes**  
  - Provide workflow documentation, instructions, troubleshooting tips, acknowledgments, and screenshots.

---

### 3. Summary Table

| Node Name                       | Node Type                         | Functional Role                         | Input Node(s)                          | Output Node(s)                          | Sticky Note                                          |
|--------------------------------|----------------------------------|---------------------------------------|--------------------------------------|----------------------------------------|------------------------------------------------------|
| Receive CV via Email            | Gmail Trigger                    | Receive CV emails with attachments    | None                                 | Switch - File Type                     |                                                      |
| Switch - File Type              | Switch                          | Routes based on CV file MIME type     | Receive CV via Email                  | Stream Doc/Docx File, Upload CV - PDF | Sticky Note2 (Get CV via PDF Format), Sticky Note3 (Get CV via Word/Doc/Docx Format) |
| Stream Doc/Docx File            | HTTP Request (Google Drive API) | Upload Word docs as Google Docs       | Switch - File Type                   | Merge                                 |                                                      |
| Preserve CV file               | Code                            | Pass binary unchanged                  | Switch - File Type                   | Merge                                 |                                                      |
| Merge                         | Merge                           | Combine original file and converted doc | Stream Doc/Docx File, Preserve CV file | Trigger Google Docs Conversion       |                                                      |
| Trigger Google Docs Conversion  | HTTP Request (Google Drive API) | Finalize Google Docs conversion        | Merge                               | Get Web Link                          |                                                      |
| Get Web Link                   | HTTP Request (Google Drive API) | Retrieve webViewLink for Google Doc   | Trigger Google Docs Conversion       | Standardize Web Link and CV Text (GDoc) |                                                      |
| Upload CV - PDF               | Google Drive                    | Upload PDF CV directly                 | Switch - File Type                   | Download CV - PDF                     |                                                      |
| Download CV - PDF             | Google Drive                    | Download uploaded PDF CV               | Upload CV - PDF                     | Extract from PDF                      |                                                      |
| Extract from PDF              | Extract From File               | Extract text from PDF CV               | Download CV - PDF                   | Standardize Web Link and CV Text (PDF) |                                                      |
| Extract from PDF Download     | Extract From File               | Extract text from downloaded GDoc PDF | Download CV - GDoc as PDF           | Standardize Web Link and CV Text (GDoc) |                                                      |
| Standardize Web Link and CV Text (PDF) | Set                         | Set standardized CV text & web link   | Extract from PDF                    | Standardize                          |                                                      |
| Standardize Web Link and CV Text (GDoc) | Set                         | Set standardized CV text & web link   | Extract from PDF Download           | Standardize                          |                                                      |
| Standardize                  | Set                             | Final normalization of CV data        | Standardize Web Link and CV Text (PDF/GDoc) | JD Matching Agent                    |                                                      |
| JD Matching Agent             | LangChain AI Agent              | Match candidate to relevant JDs       | Standardize, Access JD Files         | JD Match w/Email?                    | Sticky Note (Job Description Matching with Candidate's CV) |
| Structured Output Parser-1    | LangChain Output Parser        | Parse JD Matching Agent output         | JD Matching Agent                   | JD Match w/Email?                    |                                                      |
| JD Match w/Email?             | If                             | Branch on match type (email vs CV)    | Structured Output Parser-1          | Download Selected JD / Transform for Multiple JDs |                                                      |
| Download Selected JD          | Google Drive                   | Download single JD PDF identified by email match | JD Match w/Email? (email path)      | Extract from File                   |                                                      |
| Transform for Multiple JDs    | Code                           | Transform AI output for looping       | JD Match w/Email? (cv_match path)  | Loop Over Items                     |                                                      |
| Loop Over Items               | Split In Batches               | Loop over matched JDs for detailed analysis | Transform for Multiple JDs          | Detailed JD Matching Agent           |                                                      |
| Detailed JD Matching Agent    | LangChain AI Agent             | Select best JD from multiple matches  | Loop Over Items                    | Match Selected JD Name with Full Text |                                                      |
| Structured Output Parser-2    | LangChain Output Parser        | Parse detailed JD matching output      | Detailed JD Matching Agent          | Match Selected JD Name with Full Text |                                                      |
| Set                          | Set                            | Prepare JD data for screening agent   | Extract from File1, Loop Over Items | Recruiter Scoring Agent             |                                                      |
| Match Selected JD Name with Full Text | Code                           | Find full text of selected JD          | Detailed JD Matching Agent          | Recruiter Scoring Agent             |                                                      |
| Recruiter Scoring Agent       | LangChain AI Agent             | Analyze candidate CV vs selected JD   | Match Selected JD Name with Full Text, Standardize | Information Extractor               | Sticky Note1 (3. CV Analysis and Feedback)            |
| Structured Output Parser-3    | LangChain Output Parser        | Parse scoring agent output             | Recruiter Scoring Agent             | Information Extractor               |                                                      |
| Information Extractor         | LangChain Information Extractor | Extract candidate personal details     | Recruiter Scoring Agent             | Append row in sheet                 |                                                      |
| Append row in sheet           | Google Sheets                  | Log candidate data and screening report | Information Extractor               | Send Candidate Screening Confirmation |                                                      |
| Send Candidate Screening Confirmation | Slack                         | Send Slack message with screening summary and action buttons | Append row in sheet                 | None                               | Sticky Note10 (Slack confirmation message)            |
| Access JD Files               | Google Drive Tool             | Retrieve list of Job Description files | JD Matching Agent (ai_tool input)   | JD Matching Agent                   |                                                      |
| Gemini 2.5 Flash              | LangChain AI Language Model  | AI inference for initial JD matching  | JD Matching Agent (ai_languageModel) | JD Matching Agent                   |                                                      |
| Gemini 2.5 Pro-1             | LangChain AI Language Model  | AI inference for detailed JD matching | Detailed JD Matching Agent (ai_languageModel) | Detailed JD Matching Agent          |                                                      |
| Gemini 2.5 Pro-2             | LangChain AI Language Model  | AI inference for detailed candidate screening | Recruiter Scoring Agent (ai_languageModel) | Recruiter Scoring Agent             |                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure: Use OAuth2 credentials for Gmail.  
   - Set filter label to the label used for candidate CV emails (e.g., "CV-Screening").  
   - Enable attachment download, prefix attachment data with "cv_".  
   - Set polling interval to every minute.

2. **Add Switch Node 'Switch - File Type'**  
   - Connect from Gmail Trigger.  
   - Configure two outputs based on attachment MIME type:  
     - "Doc-Docx": for Word MIME types (`application/vnd.openxmlformats-officedocument.wordprocessingml.document` or `application/msword`)  
     - "PDF": for `application/pdf`

3. **For Word Documents Path:**  
   - Add HTTP Request node "Stream Doc/Docx File"  
     - Method: POST to `https://www.googleapis.com/upload/drive/v3/files?uploadType=resumable&supportsAllDrives=true`  
     - JSON Body: Set name using sender’s name and timestamp; mimeType as Google Docs; specify parent folder ID (Candidate CVs folder).  
     - Authenticate with Google Drive OAuth2.  
   - Add Code node "Preserve CV file" to pass binary data unchanged.  
   - Add Merge node to combine Google Doc metadata and original binary.  
   - Add HTTP Request node "Trigger Google Docs Conversion" (PUT) to update doc metadata if needed.  
   - Add HTTP Request node "Get Web Link" to get Google Doc webViewLink.

4. **For PDF Path:**  
   - Add Google Drive node "Upload CV - PDF"  
     - Operation: Upload file  
     - Folder: Candidate CVs folder ID  
     - Name: Use sender’s name and timestamp.  
   - Add Google Drive node "Download CV - PDF"  
     - Operation: Download file by ID.  
   - Add Extract From File node "Extract from PDF"  
     - Operation: PDF text extraction.

5. **Add Set Nodes for Standardization:**  
   - "Standardize Web Link and CV Text (PDF)" for PDF path outputs.  
   - "Standardize Web Link and CV Text (GDoc)" for Google Doc path outputs.  
   - Both set variables: `cv_text` and `cv_webviewlink`.

6. **Add Set node "Standardize"**  
   - Assign `cv_text` and `cv_webviewlink` from whichever path was used.  
   - Connect both standardized paths to this node.

7. **Access Job Description Files:**  
   - Add Google Drive Tool node "Access JD Files"  
     - Filter by folder ID for Job Descriptions folder.  
   - Connect output as AI tool input for JD Matching Agent.

8. **Add LangChain Agent Node "JD Matching Agent"**  
   - Configure prompt to analyze email subject/body first for JD match, fallback to CV text.  
   - Use Google Gemini (Flash) model credentials.  
   - Attach Structured Output Parser node to parse response.

9. **Add If Node "JD Match w/Email?"**  
   - Condition: `match_type` equals "email_match".  
   - True branch: Download the single matched JD file (Google Drive Download node).  
   - False branch: Transform AI output to array and loop over matched JDs.

10. **Loop Over Items Node**  
    - Process each JD matched for detailed analysis.

11. **Detailed JD Matching Agent (LangChain Agent)**  
    - Use Google Gemini Pro model.  
    - Prompt compares candidate CV against up to 3 matched JDs to choose the single best match.  
    - Attach Structured Output Parser for result.

12. **Add Code node "Match Selected JD Name with Full Text"**  
    - Locate full text of selected JD from looped JDs for final screening.

13. **Recruiter Scoring Agent (LangChain Agent)**  
    - Use Google Gemini Pro model.  
    - Prompt to analyze candidate CV against selected JD, output strengths, weaknesses, risk/reward, overall fit, and justification.  
    - Attach Structured Output Parser.

14. **Information Extractor Node**  
    - Extract candidate first name, last name, and email from standardized CV text.

15. **Google Sheets Node "Append row in sheet"**  
    - Append a new row to "AI Candidate Screening" sheet.  
    - Map fields: CV link, date/time, candidate info, JD match, strengths, weaknesses, risk/reward, overall fit, justification, submission ID, sender email.  
    - Use Google Sheets OAuth2 credentials.

16. **Slack Node "Send Candidate Screening Confirmation"**  
    - Send formatted Slack message to configured channel using Slack OAuth2 credentials.  
    - Include interactive buttons for Proceed/Reject candidate with confirmation dialogs.  
    - Include links to CV and Google Sheet.

17. **Credential Setup:**  
    - Gmail OAuth2 with API access  
    - Google Drive OAuth2 for file uploads/downloads  
    - Google Sheets OAuth2 for logging  
    - Slack OAuth2 for messaging  
    - Google Gemini API key for AI nodes

18. **Folder Setup:**  
    - Create Google Drive folders for Candidate CVs and Job Descriptions.  
    - Update folder IDs in relevant nodes accordingly.

19. **Testing and Activation:**  
    - Test each branch with sample emails and CVs in both PDF and Word formats.  
    - Validate AI outputs and Slack messages.  
    - Activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Workflow automates CV screening from Gmail, JD matching, AI analysis, and Slack notification with interactive decision buttons | Workflow overview and intended use                                                                                      |
| Handles PDF and Word CVs with automatic conversion and text extraction                                                         | File processing capabilities                                                                                            |
| Two-stage JD matching: email-priority, fallback to CV content; max 3 JDs matched, then best single JD selected                   | AI logic for job description matching                                                                                   |
| Uses Google Gemini API for AI tasks: JD matching, detailed candidate screening, info extraction                                  | AI integration details                                                                                                   |
| Candidate data logged in Google Sheets for audit and tracking                                                                    | Data persistence and audit trail                                                                                        |
| Slack messages include rich formatting and interactive buttons for team decision-making                                          | Notification and collaboration                                                                                          |
| Setup requires Gmail, Google Drive, Google Sheets, Slack OAuth2 credentials, Google Gemini API key                              | Prerequisites                                                                                                           |
| Troubleshooting tips included in Sticky Note8                                                                                    | Common error resolutions and checks                                                                                     |
| Inspired by Nate Herk’s YouTube demo with enhancements: dynamic JD matching, Slack Block Kit, updated Drive API usage            | Acknowledgments and references                                                                                           |
| Workflow screenshots and sample outputs linked in Sticky Notes (e.g., Google Sheets sample, Slack message screenshots)           | Visual references for user orientation                                                                                   |
| Google AI Pricing info link provided for understanding Google Gemini API costs                                                   | https://ai.google.dev/pricing                                                                                           |
| Google Gemini API key acquisition link: https://makersuite.google.com/app/apikey                                                | API credential setup                                                                                                     |

---

_Disclaimer: The text above is generated from an automated n8n workflow export designed for legal and public data processing in compliance with content policies._