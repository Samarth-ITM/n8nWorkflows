Screen and Evaluate Job Candidates with GPT-4, Airtable, and Google Drive

https://n8nworkflows.xyz/workflows/screen-and-evaluate-job-candidates-with-gpt-4--airtable--and-google-drive-4481


# Screen and Evaluate Job Candidates with GPT-4, Airtable, and Google Drive

### 1. Workflow Overview

This workflow automates the screening and evaluation of job candidates using a combination of n8n nodes, Airtable, Google Drive, and OpenAI's GPT-4. It is designed to streamline the recruitment process by:

- Collecting candidate application data and resumes via a web form
- Extracting resume text from uploaded PDF files
- Searching the job posting in Airtable matching the candidate’s applied position
- Using a GPT-4 AI agent to assess candidate suitability based on resume, cover letter, skills, and job requirements
- Uploading resumes to Google Drive and managing file permissions for storage
- Creating candidate records in Airtable with AI-generated screening results
- Automatically notifying HR via email if the candidate is suitable

The workflow logically breaks down into the following blocks:

- **1.1 Candidate Input Reception:** Collect candidate data and resume via form and extract resume text.
- **1.2 Job Posting Retrieval:** Find relevant job posting in Airtable based on candidate’s applied position.
- **1.3 AI-Based Candidate Screening:** Use GPT-4 AI agent to analyze candidate data versus job requirements.
- **1.4 Resume File Handling:** Upload the candidate’s resume to Google Drive and set file permissions.
- **1.5 Data Merging and Candidate Creation:** Combine AI results and file data, then create candidate record in Airtable.
- **1.6 Candidate Evaluation Outcome:** Check if candidate is suitable and send notification email to HR.

---

### 2. Block-by-Block Analysis

#### 2.1 Candidate Input Reception

**Overview:**  
This block starts the workflow by capturing candidate data through a web form and extracting text from the uploaded PDF resume for further processing.

**Nodes Involved:**  
- Candidate Application Form  
- Extract Resume PDF  
- Upload File  
- Set File Permission  

**Node Details:**

- **Candidate Application Form**  
  - Type: Form Trigger  
  - Role: Web form to capture candidate details including name, email, position applied for, relevant skills, cover letter, and PDF resume upload (single file, PDF only).  
  - Configuration: Required fields enforced; email field validated; file field accepts only PDFs.  
  - Input: User-submitted form data  
  - Output: JSON data with candidate responses and binary resume file  
  - Edge Cases: Missing required fields, invalid email, upload of unsupported file types  
  - Sticky Note: Explains form fields and usage  

- **Extract Resume PDF**  
  - Type: Extract From File  
  - Role: Converts the uploaded resume PDF file into plain text content for AI analysis  
  - Configuration: Operates on binary property "Resume"  
  - Input: Binary PDF file from form node  
  - Output: Extracted text in JSON  
  - Edge Cases: Corrupted or unreadable PDFs, unsupported file formats  
  - Dependencies: Must follow form node  

- **Upload File**  
  - Type: Google Drive  
  - Role: Uploads the resume PDF to a specified Google Drive folder  
  - Configuration: Uploads file named as in original upload, to folder ID "13BuRkJofybsBlF77oqoS87A2qq4zD2aP" (cached from Google Drive)  
  - Credentials: Google Drive OAuth2  
  - Input: Binary file from form node  
  - Output: File metadata including webContentLink and file ID  
  - Edge Cases: Drive quota exceeded, permission errors, large files  
  - Sticky Note: Describes file upload and permission handling  

- **Set File Permission**  
  - Type: Google Drive  
  - Role: Sets the uploaded file’s sharing permissions to "anyone with the link can read"  
  - Configuration: Operation "share"; role "reader"; type "anyone"  
  - Input: File ID from Upload File node  
  - Output: Confirmation of permission set  
  - Edge Cases: Permission setting failures, API rate limits  

---

#### 2.2 Job Posting Retrieval

**Overview:**  
Finds the job posting in Airtable that matches the position the candidate applied for, enabling the AI agent to compare candidate data against the correct job description.

**Nodes Involved:**  
- Search Job Posting  

**Node Details:**

- **Search Job Posting**  
  - Type: Airtable  
  - Role: Searches the "Job Posting" table in the "HR Database" Airtable base for a job title matching the candidate’s applied position (case-insensitive substring match)  
  - Configuration: Filter formula uses Airtable's FIND and LOWER functions to match applied position to Job Title field  
  - Input: Position Applied For from Candidate Application Form  
  - Output: Job posting record including job title, description, required skills  
  - Credentials: Airtable Personal Access Token  
  - Edge Cases: No matching job found, API errors, rate limits  
  - Sticky Note: Describes job matching logic  

---

#### 2.3 AI-Based Candidate Screening

**Overview:**  
Uses OpenAI GPT-4 via a LangChain agent to analyze candidate information against job requirements and generate screening results including suitability status, match percentage, and detailed notes.

**Nodes Involved:**  
- OpenAI Chat Model  
- Structured Output Parser  
- Candidate Screener AI Agent  

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Provides GPT-4 model interface for natural language processing  
  - Configuration: Model set to "gpt-4o-mini" (GPT-4 variant)  
  - Credentials: OpenAI API key  
  - Input: Prompt from Candidate Screener AI Agent  
  - Output: Raw AI chat completion  
  - Edge Cases: API quota exhaustion, timeout, prompt formatting errors  

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI response JSON with schema example specifying screening_status, match_percentage, and screening_notes fields  
  - Configuration: JSON schema example provided to guide parsing  
  - Input: Raw AI completion from OpenAI Chat Model  
  - Output: Structured parsed JSON for downstream use  
  - Edge Cases: Parsing failures if AI output is malformed or incomplete  

- **Candidate Screener AI Agent**  
  - Type: LangChain Agent  
  - Role: Combines candidate and job data into a prompt, sends to OpenAI Chat Model, and parses output  
  - Configuration:  
    - System message defines AI’s role and output expectations  
    - Input prompt includes candidate’s position applied, relevant skills, cover letter, resume text, job description, and required skills from Airtable  
    - Output fields: Screening Status (Suitable/Not Suitable/Under Review), Match Percentage (numeric), Screening Notes (string)  
  - Input: Job posting data (from Search Job Posting), candidate form data, extracted resume text  
  - Output: Parsed AI screening results  
  - Edge Cases: Mismatches in field references, AI hallucination, partial data availability  
  - Sticky Note: Details AI agent’s function and expected outputs  

---

#### 2.4 Resume File Handling

**Overview:**  
Prepares file data for Airtable by formatting the uploaded resume’s Google Drive link and filename into the structure required for Airtable file uploads.

**Nodes Involved:**  
- Set File Array for airtable  

**Node Details:**

- **Set File Array for airtable**  
  - Type: Set  
  - Role: Creates a JSON array containing the public Google Drive file URL and filename, matching Airtable’s file upload format  
  - Configuration: Raw JSON output with dynamic expressions referencing Upload File node’s webContentLink and name fields  
  - Input: Output of Set File Permission node (file metadata)  
  - Output: JSON object with "file_array" property for Airtable  
  - Edge Cases: Missing file URL or name, incorrect formatting causing Airtable API errors  
  - Sticky Note: Explains file preparation for Airtable upload  

---

#### 2.5 Data Merging and Candidate Creation

**Overview:**  
Combines AI screening results and formatted file data, then creates a new candidate record in Airtable’s “Candidates” table with all relevant information.

**Nodes Involved:**  
- Merge  
- Create Candidate  

**Node Details:**

- **Merge**  
  - Type: Merge  
  - Role: Combines data from AI agent output and file array into a single JSON object for candidate record creation  
  - Configuration: Mode "combine", combining all inputs  
  - Inputs: Output from Candidate Screener AI Agent and Set File Array for airtable  
  - Output: Combined JSON with candidate info, AI screening data, and resume file array  
  - Edge Cases: Mismatched input lengths, merge conflicts, missing data  

- **Create Candidate**  
  - Type: Airtable  
  - Role: Creates a new record in the "Candidates" table of the "HR Database" Airtable base  
  - Configuration: Maps fields including resume (file array), full name, cover letter, date applied (current timestamp), email, relevant skills, screening notes, match percentage, position applied, and screening status  
  - Credentials: Airtable Personal Access Token  
  - Input: Merged JSON from Merge node  
  - Output: Confirmation of record creation  
  - Edge Cases: API errors, required field omissions, incorrect field mapping  
  - Sticky Note: Describes merging and candidate creation process  

---

#### 2.6 Candidate Evaluation Outcome

**Overview:**  
Evaluates the AI screening status and conditionally sends an email notification to the HR team if the candidate is suitable.

**Nodes Involved:**  
- Check if candidate is suitable  
- Send email to HR  

**Node Details:**

- **Check if candidate is suitable**  
  - Type: If (Conditional)  
  - Role: Checks if screening_status from merged data equals "Suitable"  
  - Configuration: Condition with strict equality check on screening_status field  
  - Input: Output from Create Candidate node (merged AI screening output)  
  - Output: Two branches — if true proceeds to send email, if false ends flow  
  - Edge Cases: Missing or malformed screening_status field causing false negatives  

- **Send email to HR**  
  - Type: Gmail  
  - Role: Sends notification email to HR team with candidate details upon suitability confirmation  
  - Configuration:  
    - Recipient: hr_team_n8n@yopmail.com  
    - Subject: Includes candidate full name and job title  
    - Message body: Summary of candidate, match percentage, and screening notes  
    - Email type: Text  
  - Credentials: Gmail OAuth2  
  - Input: Candidate and job data from previous nodes  
  - Edge Cases: Email delivery failure, credential expiration, rate limits  
  - Sticky Note: Explains candidate qualification check and email notification  

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                           | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                                    |
|-----------------------------|---------------------------------|-----------------------------------------|-------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| Candidate Application Form   | Form Trigger                    | Collect candidate application data      | —                             | Extract Resume PDF, Upload File  | Web form for candidate application submission including resume PDF upload                                   |
| Extract Resume PDF           | Extract From File               | Extract text from uploaded PDF resume   | Candidate Application Form     | Search Job Posting               | —                                                                                                             |
| Upload File                 | Google Drive                   | Upload resume PDF to Google Drive       | Candidate Application Form     | Set File Permission             | Uploads resume file and manages permissions                                                                 |
| Set File Permission          | Google Drive                   | Set public read permission on file      | Upload File                   | Set File Array for airtable      | —                                                                                                             |
| Search Job Posting           | Airtable                       | Find matching job posting in Airtable   | Extract Resume PDF             | Candidate Screener AI Agent      | Finds matching job posting for the applied position                                                          |
| OpenAI Chat Model            | LangChain LM Chat OpenAI       | Interface to GPT-4 language model       | Candidate Screener AI Agent    | Candidate Screener AI Agent      | Used by AI agent for candidate screening analysis                                                           |
| Structured Output Parser     | LangChain Output Parser        | Parse structured AI output JSON          | OpenAI Chat Model             | Candidate Screener AI Agent      | Parses AI screening results into JSON                                                                        |
| Candidate Screener AI Agent  | LangChain Agent                | Analyze candidate vs job requirements    | Search Job Posting, Extract Resume PDF, Candidate Application Form | Merge                    | AI-powered evaluation and scoring of candidate suitability                                                   |
| Set File Array for airtable  | Set                           | Prepare file JSON array for Airtable     | Set File Permission           | Merge                          | Formats Google Drive file link and name for Airtable file upload                                            |
| Merge                       | Merge                         | Combine AI output and file data           | Candidate Screener AI Agent, Set File Array for airtable | Create Candidate               | Combines candidate screening results and resume file info                                                   |
| Create Candidate             | Airtable                      | Create candidate record in Airtable      | Merge                         | Check if candidate is suitable   | Saves candidate data and AI screening results in Airtable                                                  |
| Check if candidate is suitable | If                           | Conditional check on candidate suitability | Create Candidate              | Send email to HR                | Checks if candidate is suitable to trigger email notification                                               |
| Send email to HR             | Gmail                         | Notify HR team about suitable candidate  | Check if candidate is suitable | —                              | Sends email notification to HR when candidate is suitable                                                  |
| Sticky Note                  | Sticky Note                   | Documentation and explanations           | —                             | —                              | Multiple sticky notes provide detailed explanations and links (see block descriptions)                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Candidate Application Form" node:**
   - Type: Form Trigger  
   - Configure form title: "Smart Candidate Screening Form"  
   - Fields (all required except resume file type validation):  
     - Full Name (text)  
     - Email Address (email)  
     - Position Applied For (text)  
     - Relevant Skills (textarea)  
     - Cover Letter (textarea)  
     - Resume (file upload, single, accept only ".pdf")  
   - Save and note generated webhook URL for form submissions.

2. **Add "Extract Resume PDF" node:**
   - Type: Extract From File  
   - Set operation to "pdf"  
   - Binary Property to extract: "Resume" (from form node)  
   - Connect from "Candidate Application Form" main output.

3. **Add "Upload File" node (Google Drive):**
   - Set file name to: `{{$json.Resume.filename}}`  
   - Set Drive ID to "My Drive"  
   - Set Folder ID to the target folder (e.g., "13BuRkJofybsBlF77oqoS87A2qq4zD2aP")  
   - Input data field: "Resume" (binary file from form node)  
   - Connect from "Candidate Application Form" main output (parallel to Extract Resume PDF).

4. **Add "Set File Permission" node (Google Drive):**
   - Operation: "share"  
   - Permissions: role = "reader", type = "anyone"  
   - File ID: `={{$json.id}}` (from Upload File node output)  
   - Connect from "Upload File".

5. **Add "Set File Array for airtable" node:**
   - Type: Set node  
   - Mode: Raw JSON with output:  
     ```json
     {
       "file_array": [
         {
           "url": "{{ $('Upload File').item.json.webContentLink }}",
           "filename": "{{ $('Upload File').item.json.name }}"
         }
       ]
     }
     ```  
   - Connect from "Set File Permission".

6. **Add "Search Job Posting" node (Airtable):**
   - Base: "HR Database" (app ID: appgVjZcaRP8BsKf0)  
   - Table: "Job Posting" (table ID: tblla4rBCW3BhPtRO)  
   - Operation: Search  
   - Limit: 1  
   - Filter by formula:  
     `=FIND(LOWER("{{ $('Candidate Application Form').item.json['Position Applied For'] }}"), LOWER({Job Title})) > 0`  
   - Connect from "Extract Resume PDF".

7. **Add "Candidate Screener AI Agent" node (LangChain Agent):**
   - Text prompt:  
     ```
     Candidate Screening Information:

     ///
     User Side:

     Position Applied:  
     {{ $('Search Job Posting').item.json['Job Title'] }}

     Relevant Skills the Candidate Possesses:
     {{ $('Candidate Application Form').item.json['Relevant Skills'] }}

     Cover Letter:  
     {{ $('Candidate Application Form').item.json['Cover Letter'] }}

     Resume Content:  
     {{ $('Extract Resume PDF').item.json.text }}

     ///

     Employer Side:

     Position Description:  
     {{ $('Search Job Posting').item.json['Job Description'] }}

     Skills Required:  
     {{ $('Search Job Posting').item.json['Required Skills'] }}

     ///
     ```
   - System message: (As described in block 2.3 overview) specifying evaluation criteria and output fields  
   - Enable output parser  
   - Connect from "Search Job Posting".

8. **Configure the "OpenAI Chat Model" node:**
   - Model: "gpt-4o-mini" (GPT-4 variant)  
   - Credentials: OpenAI API key  
   - Connect as AI language model node to "Candidate Screener AI Agent".

9. **Configure "Structured Output Parser" node:**
   - JSON schema example for expected output:  
     ```json
     {
       "screening_status": "Suitable",
       "match_percentage": 88,
       "screening_notes": "Detailed explanation..."
     }
     ```  
   - Connect as AI output parser node to "Candidate Screener AI Agent".

10. **Add "Merge" node:**
    - Mode: combine  
    - Combine inputs: outputs from "Candidate Screener AI Agent" and "Set File Array for airtable"  
    - Connect from both nodes accordingly.

11. **Add "Create Candidate" node (Airtable):**
    - Base: "HR Database"  
    - Table: "Candidates" (tblQy83erGR5lQj5c)  
    - Operation: Create  
    - Map fields:  
      - Resume: `={{ $json.file_array }}`  
      - Full Name: `={{ $('Candidate Application Form').item.json['Full Name '] }}`  
      - Cover Letter: `={{ $('Candidate Application Form').item.json['Cover Letter'] }}`  
      - Date Applied: `={{ $now }}` (current date/time)  
      - Email Address: `={{ $('Candidate Application Form').item.json['Email Address'] }}`  
      - Relevant Skills: `={{ $('Candidate Application Form').item.json['Relevant Skills'] }}`  
      - Screening Notes: `={{ $json.output.screening_notes }}`  
      - Match Percentage: `={{ $json.output.match_percentage }}`  
      - Position Applied: `={{ $('Candidate Application Form').item.json['Position Applied For'] }}`  
      - Screening Status: `={{ $json.output.screening_status }}`  
    - Credentials: Airtable API key  
    - Connect from "Merge".

12. **Add "Check if candidate is suitable" node (If):**
    - Condition: check if `={{ $('Merge').item.json.output.screening_status }}` equals "Suitable" (case sensitive, strict)  
    - Connect from "Create Candidate".

13. **Add "Send email to HR" node (Gmail):**
    - Recipient: hr_team_n8n@yopmail.com  
    - Subject: `Candidate {{ $('Candidate Application Form').item.json['Full Name '] }} - Suitable for {{ $('Search Job Posting').item.json['Job Title'] }}`  
    - Message: Include candidate details, match percentage, and screening notes dynamically from AI agent outputs  
    - Credentials: Gmail OAuth2  
    - Connect from "Check if candidate is suitable" (true branch).

Ensure all credentials (Airtable Personal Access Token, OpenAI API key, Google Drive OAuth2, Gmail OAuth2) are configured and tested.  
Test each node connection and data passing before proceeding to the next.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Author: Billy Christi - Creator of this workflow template                                                                                                | [Billy Christi Profile](https://n8n.io/creators/billy/)                                                             |
| Airtable Base Structure: Copy the Airtable base used for job postings and candidates, including table schema                                           | [Airtable Base Link](https://airtable.com/appgVjZcaRP8BsKf0/tblla4rBCW3BhPtRO/viw19l5BlEW7NnZJQ)                      |
| Workflow video and branding notes are included as sticky notes in the workflow for user guidance                                                      | Sticky notes within workflow nodes                                                                                   |
| Requires API credentials for Airtable, OpenAI GPT-4, Google Drive, and Gmail for full functionality                                                    | Credential setup needed in n8n Credentials manager                                                                   |
| Workflow automates the entire hiring pipeline from application submission to HR team notification                                                     | Provides end-to-end automation for candidate screening and evaluation                                                |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. The processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.