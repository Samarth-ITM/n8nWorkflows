Resume Screening & Evaluation System with Gemini AI, Google Sheets & Drive for HR

https://n8nworkflows.xyz/workflows/resume-screening---evaluation-system-with-gemini-ai--google-sheets---drive-for-hr-5453


# Resume Screening & Evaluation System with Gemini AI, Google Sheets & Drive for HR

### 1. Workflow Overview

This workflow automates the resume screening and evaluation process for HR teams by integrating form submissions, AI-powered text extraction and summarization, and Google Workspace tools. It accepts candidate applications via a custom form including CV upload, extracts and processes candidate data using Google Gemini AI and LangChain nodes, compares candidate profiles against job role criteria stored in Google Sheets, scores candidates, and logs results along with CVs in Google Drive and Google Sheets.

Logical blocks included:

- **1.1 Input Reception:** Captures candidate submissions via a web form with CV upload.
- **1.2 File Handling & Data Extraction:** Uploads CVs to Google Drive, extracts text from PDFs, and extracts detailed candidate information (qualifications and personal data) using AI.
- **1.3 Data Summarization & Merging:** Summarizes extracted candidate data into concise profiles and merges relevant outputs.
- **1.4 Job Role Lookup:** Retrieves job profile details from Google Sheets based on candidate’s selected role.
- **1.5 Candidate Evaluation:** Uses AI to score and evaluate candidates against the desired profile.
- **1.6 Result Storage:** Saves evaluation results and candidate summaries back to Google Sheets for HR review.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block handles the initial candidate input via a hosted form, capturing personal details and the CV file upload.

**Nodes Involved:**  
- On form submission

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point capturing candidate name, email, CV (PDF), and job role selection.  
  - Configuration: Custom CSS styling for form UI; required fields include Name, Email, and CV upload; Job Role is a dropdown with predefined options.  
  - Input: HTTP webhook triggered on form submit  
  - Output: JSON with form fields and binary CV file  
  - Edge Cases: Missing required fields cause form rejection; file upload issues if non-PDF files submitted; webhook must be activated for trigger to work.

---

#### 1.2 File Handling & Data Extraction

**Overview:**  
Manages CV upload to Google Drive, extracts text from uploaded PDF, and uses AI to extract structured candidate qualifications and personal data.

**Nodes Involved:**  
- Upload CV  
- Extract from File  
- Qualifications  
- Personal Data

**Node Details:**

- **Upload CV**  
  - Type: Google Drive (File Upload)  
  - Role: Saves the uploaded CV file to a designated Google Drive folder for storage and record-keeping.  
  - Configuration: File named dynamically as "cv-[original filename]"; uploads to specific folder "work cvs" in Google Drive.  
  - Input: Binary CV file from form submission  
  - Output: Metadata about uploaded file  
  - Credentials: Google Drive OAuth2 required  
  - Edge Cases: Upload failure due to auth or quota limits; filename collisions managed by unique naming.

- **Extract from File**  
  - Type: Extract From File  
  - Role: Extracts text content from the uploaded PDF CV for AI processing.  
  - Configuration: Operation set to PDF extraction on binary property "CV"  
  - Input: Binary PDF from form  
  - Output: Extracted text in JSON  
  - Edge Cases: PDF parsing errors if file corrupt or scanned images without OCR; missing binary property.

- **Qualifications**  
  - Type: LangChain Information Extractor  
  - Role: Extracts structured data on educational qualifications, job history, and technical skills from the CV text.  
  - Configuration: Custom system prompt instructing to extract relevant info, attributes defined with descriptions and max word limits for summary fields.  
  - Input: Extracted text JSON from "Extract from File"  
  - Output: JSON with extracted attributes (Educational qualification, Job History, Skills)  
  - Edge Cases: Missing or ambiguous data in CV; extraction confidence variability.

- **Personal Data**  
  - Type: LangChain Information Extractor  
  - Role: Extracts personal candidate data such as telephone, city, and birthdate from CV text.  
  - Configuration: Uses a manual JSON schema specifying expected properties; extraction guided by system prompt.  
  - Input: Extracted text JSON from "Extract from File"  
  - Output: JSON with personal data fields  
  - Edge Cases: Data missing in CV or poorly formatted; schema mismatches.

---

#### 1.3 Data Summarization & Merging

**Overview:**  
Combines extracted candidate attributes and personal data, then generates a concise summary of the candidate profile.

**Nodes Involved:**  
- Merge  
- Summarization Chain  
- Merge2

**Node Details:**

- **Merge**  
  - Type: Merge (combine)  
  - Role: Combines outputs from "Qualifications" and "Personal Data" to consolidate candidate information.  
  - Configuration: Mode "combine" merges all inputs into a single JSON object.  
  - Input: Outputs from Qualifications and Personal Data nodes  
  - Output: Combined JSON object with all extracted data  
  - Edge Cases: Mismatched data lengths or missing inputs.

- **Summarization Chain**  
  - Type: LangChain Summarization Chain  
  - Role: Generates a concise, conversational summary of candidate details including city, birthdate, educational background, work history, and skills.  
  - Configuration: Custom prompt with placeholders for dynamic insertion of combined data; limits summary to 100 words or less.  
  - Input: Combined data from "Merge" node  
  - Output: Text summary of candidate profile  
  - Edge Cases: AI model timeouts or incomplete data.

- **Merge2**  
  - Type: Merge (combine by position)  
  - Role: Combines the summarized candidate profile with the job profile data (from job roles lookup).  
  - Configuration: Mode "combineByPosition" to combine data items by array position.  
  - Input: From Summarization Chain and job roles lookup  
  - Output: Combined JSON of candidate summary and job profile  
  - Edge Cases: Unequal array lengths causing misalignment.

---

#### 1.4 Job Role Lookup

**Overview:**  
Retrieves the profile description and criteria of the job role selected by the candidate from a Google Sheet.

**Nodes Involved:**  
- job roles

**Node Details:**

- **job roles**  
  - Type: Google Sheets (Read)  
  - Role: Looks up the job profile matching the candidate's selected job role from a predefined Google Sheet.  
  - Configuration: Filters rows by matching "Role" column to candidate's selected "Job Role"; reads from sheet "Sheet1" in document "JobProfiles".  
  - Input: Candidate form submission JSON ("Job Role")  
  - Output: JSON with job profile details including "Profile Wanted" field.  
  - Credentials: Google Sheets OAuth2 required  
  - Edge Cases: Role not found in sheet; multiple matches; auth failures.

---

#### 1.5 Candidate Evaluation

**Overview:**  
Uses AI to compare the candidate summary against the desired job profile and produce a numeric score and textual considerations.

**Nodes Involved:**  
- HR Expert  
- Structured Output Parser

**Node Details:**

- **HR Expert**  
  - Type: LangChain LLM Chain  
  - Role: AI agent acting as an HR expert to assess candidate fit relative to job profile, generating a score 1-10 and explanation.  
  - Configuration: Prompt includes the job profile and candidate summary; expects output parsed by structured output parser; output fields "vote" and "consideration".  
  - Input: Combined data from Merge2  
  - Output: AI-generated evaluation JSON  
  - Edge Cases: Model response format issues; ambiguous scoring; API rate limits.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses the HR Expert’s LLM output into a strict JSON schema with "vote" and "consideration".  
  - Configuration: Manual schema requiring string types for vote and consideration.  
  - Input: Text output from HR Expert node  
  - Output: Structured JSON for downstream use  
  - Edge Cases: Parsing failures due to malformed AI output.

---

#### 1.6 Result Storage

**Overview:**  
Stores candidate evaluation results, summaries, and key data into a Google Sheet for HR review.

**Nodes Involved:**  
- Google Sheets

**Node Details:**

- **Google Sheets**  
  - Type: Google Sheets (Append)  
  - Role: Appends a new row in the "recommended candidates" Google Sheet with candidate info, summary, evaluation score, and other extracted data.  
  - Configuration: Maps multiple columns such as candidate name, email, city, vote, skills, education, job history, summary text, date, and consideration notes; sheet and document IDs predefined.  
  - Input: JSON output from Structured Output Parser and other merged data nodes  
  - Output: Confirmation of row append operation  
  - Credentials: Google Sheets OAuth2 required  
  - Edge Cases: API write errors, schema mismatch, quota limits.

---

### 3. Summary Table

| Node Name            | Node Type                              | Functional Role                                     | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                      |
|----------------------|--------------------------------------|----------------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------|
| On form submission    | Form Trigger                         | Captures candidate data and CV upload via form     | -                           | job roles, Upload CV, Extract from File | 🎯 AI Resume Screener – Form-Based HR Automation (covers entire workflow setup and purpose)                      |
| Upload CV            | Google Drive (File Upload)           | Uploads candidate CV to Google Drive                | On form submission          | Extract from File            | The CV is uploaded to Google Drive and converted so that it can be processed                                    |
| Extract from File    | Extract From File                    | Extracts text from uploaded PDF CV                   | Upload CV                   | Qualifications, Personal Data | The essential information for evaluating the candidate is collected in two different chains                      |
| Qualifications       | LangChain Information Extractor      | Extracts candidate qualifications from CV text      | Extract from File           | Merge                       | The essential information for evaluating the candidate is collected in two different chains                      |
| Personal Data        | LangChain Information Extractor      | Extracts personal info (phone, city, birthdate)      | Extract from File           | Merge                       | The essential information for evaluating the candidate is collected in two different chains                      |
| Merge                | Merge (combine)                      | Combines Qualifications and Personal Data            | Qualifications, Personal Data | Summarization Chain          | Summary of relevant information useful for classifying the candidate                                            |
| Summarization Chain  | LangChain Summarization Chain        | Summarizes candidate data into concise profile       | Merge                      | Merge2                      | Summary of relevant information useful for classifying the candidate                                            |
| job roles            | Google Sheets (Read)                 | Retrieves desired job profile info from Google Sheets | On form submission          | Merge2                      | Characteristics of the profile sought by the company that intends to hire the candidate                         |
| Merge2               | Merge (combineByPosition)            | Combines candidate summary with job profile          | Summarization Chain, job roles | HR Expert                   | Characteristics of the profile sought by the company that intends to hire the candidate                         |
| HR Expert            | LangChain LLM Chain                  | AI scores candidate fit and provides evaluation      | Merge2                     | Google Sheets               | Candidate evaluation with vote and considerations of the HR agent relating the profile sought with candidate's skills |
| Structured Output Parser | LangChain Structured Output Parser | Parses HR Expert's evaluation into structured JSON   | HR Expert                  | Google Sheets               | Candidate evaluation with vote and considerations of the HR agent relating the profile sought with candidate's skills |
| Google Sheets        | Google Sheets (Append)               | Stores candidate evaluation and summary data         | Structured Output Parser, Merge2 | -                          | Candidate evaluation with vote and considerations of the HR agent relating the profile sought with candidate's skills |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node**  
   - Type: Form Trigger  
   - Configure form titled "Candidate Form" with fields: Name (text, required), Email (email, required), CV (file, required, accept .pdf), Job Role (dropdown: Sales, Security, Operations, Reception).  
   - Apply custom CSS for styling (optional).  
   - Activate webhook.

2. **Add Google Drive Upload node ("Upload CV")**  
   - Type: Google Drive (File Upload)  
   - Configure to upload the binary CV file from form submission under property `CV`.  
   - Set file name to `cv-{{ $json.CV[0].filename }}`.  
   - Target folder: "work cvs" (use folder ID from your Drive).  
   - Provide Google Drive OAuth2 credentials.

3. **Add Extract From File node ("Extract from File")**  
   - Type: Extract From File  
   - Set operation to PDF extraction.  
   - Binary property name: `CV`.  
   - Input from "Upload CV" binary data.

4. **Add LangChain Information Extractor node ("Qualifications")**  
   - Type: Information Extractor  
   - Input text from extracted file text (`{{$json.text}}`).  
   - Configure attributes to extract:  
     - Educational qualification (required, max 100 words + grade if applicable)  
     - Job History (required, max 100 words)  
     - Skills (required, bulleted list)  
   - Use system prompt emphasizing relevance and optional omission if data unknown.

5. **Add LangChain Information Extractor node ("Personal Data")**  
   - Type: Information Extractor  
   - Input text from extracted file text (`{{$json.text}}`).  
   - Define manual JSON schema for fields: telephone (string), city (string), birthdate (string).  
   - Use system prompt to extract these personal details.

6. **Add Merge node ("Merge")**  
   - Type: Merge  
   - Mode: Combine All  
   - Inputs: From "Qualifications" and "Personal Data".

7. **Add LangChain Summarization Chain node ("Summarization Chain")**  
   - Type: Summarization Chain  
   - Input: Combined data from "Merge".  
   - Prompt: Compose a concise summary using city, birthdate, educational qualification, job history, and skills with a limit of 100 words.  
   - Use placeholders to dynamically insert extracted data.

8. **Add Google Sheets Read node ("job roles")**  
   - Type: Google Sheets  
   - Configure to read from your "JobProfiles" Google Sheet.  
   - Filter rows where "Role" column matches the candidate’s "Job Role" from form submission.  
   - Provide Google Sheets OAuth2 credentials.

9. **Add Merge node ("Merge2")**  
   - Type: Merge  
   - Mode: Combine By Position  
   - Inputs: "Summarization Chain" and "job roles".

10. **Add LangChain LLM Chain node ("HR Expert")**  
    - Type: Chain LLM  
    - Input prompt: Provide "Profile Wanted" from job roles and candidate summary from summarization chain.  
    - Task: Score candidate fit from 1 to 10 and provide textual considerations why.  
    - Output parser: Enable, linked to next parser node.

11. **Add LangChain Structured Output Parser node ("Structured Output Parser")**  
    - Type: Structured Output Parser  
    - Define manual schema: object with string fields "vote" and "consideration".  
    - Input: AI output from "HR Expert".

12. **Add Google Sheets Append node ("Google Sheets")**  
    - Type: Google Sheets  
    - Append mode to your "recommended candidates" sheet.  
    - Map columns: Candidate name, email, phone (cleaned), city, birthdate, educational qualification, job history, skills, summarized profile, vote, consideration, and current date.  
    - Provide Google Sheets OAuth2 credentials.

13. **Connect nodes in order:**  
    - On form submission → job roles, Upload CV, Extract from File  
    - Upload CV → Extract from File  
    - Extract from File → Qualifications, Personal Data  
    - Qualifications + Personal Data → Merge  
    - Merge → Summarization Chain  
    - Summarization Chain + job roles → Merge2  
    - Merge2 → HR Expert  
    - HR Expert → Structured Output Parser  
    - Structured Output Parser + Merge2 → Google Sheets

14. **Configure AI credentials:**  
    - Set up Google Gemini (PaLM) API credentials for all LangChain nodes using AI models.  
    - Ensure API keys have sufficient quota and permissions.

15. **Testing & Activation:**  
    - Activate the webhook for form trigger.  
    - Test with sample CVs and form submissions.  
    - Monitor logs and handle errors such as auth failures, extraction issues, or AI timeouts.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                       | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| 🎯 AI Resume Screener – Form-Based HR Automation: This workflow automates resume screening using AI, extracting key details, comparing to job profiles, and logging scores for HR review. Setup requires Google Sheets with job profiles, Google Drive, and OpenAI/Gemini API keys. | Workflow sticky note for overview and setup instructions                                                     |
| The CV is uploaded to Google Drive and converted for processing — ensures the PDF text can be extracted and analyzed.                                                                                                                             | Sticky note near "Upload CV" node                                                                            |
| Essential candidate information is collected via two parallel extraction chains: qualifications and personal data.                                                                                                                                 | Sticky note near "Extract from File", "Qualifications", and "Personal Data" nodes                            |
| Summary produced is used to classify the candidate efficiently and is combined with job profile data for evaluation.                                                                                                                               | Sticky notes near "Merge" and "Summarization Chain" nodes                                                   |
| Job profile characteristics are retrieved from Google Sheets to guide scoring and evaluation.                                                                                                                                                      | Sticky note near "job roles" and "Merge2" nodes                                                              |
| Candidate evaluation outputs a numeric vote and textual considerations to justify the score, facilitating transparent HR decisions.                                                                                                               | Sticky notes near "HR Expert", "Structured Output Parser", and "Google Sheets" nodes                         |
| Contact for assistance or customization: tharwat.elsayed2000@gmail.com; WhatsApp: +20106 180 3236                                                                                                                                                   | Workflow sticky note                                                                                          |
| Optional extensions: add email or Slack notifications, multi-language support, or replace form trigger with Google Drive/Airtable triggers for alternative input sources.                                                                            | Workflow sticky note                                                                                          |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.