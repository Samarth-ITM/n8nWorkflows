Generate Tailored Interview Questions with GPT-4 Based on CV, JD, and Round

https://n8nworkflows.xyz/workflows/generate-tailored-interview-questions-with-gpt-4-based-on-cv--jd--and-round-6767


# Generate Tailored Interview Questions with GPT-4 Based on CV, JD, and Round

### 1. Workflow Overview

This workflow, titled **"Smart Interview Assistant: Tailored Questions Based on CV, JD, and Round"**, is designed to automate and streamline the preparation of tailored interview questions for hiring teams. It targets recruiters, hiring managers, and interviewers who need personalized interview guidance based on candidate resumes, job descriptions, and the specific interview round.

The workflow logically divides into the following functional blocks:

- **1.1 Input Reception and Form Submission**  
  Collect candidate CV (PDF), job role selection, and interview round via a custom application form.

- **1.2 Candidate Profile Extraction and Analysis**  
  Extract and parse structured candidate information from the uploaded resume PDF using AI.

- **1.3 Job Description Retrieval and Processing**  
  Lookup the job description PDF URL from a Google Sheet based on the selected role, download the PDF, and extract its contents.

- **1.4 Data Merging and Interview Round Metadata Assignment**  
  Combine candidate profile data with job description content and add detailed metadata about the interview round.

- **1.5 Interview Preparation Generation with GPT-4**  
  Use a GPT-4 powered agent to generate a tailored interview prep report including candidate summary, job description highlights, interview round context, and 5-8 customized interview questions with expected answers.

- **1.6 Report Formatting and Delivery**  
  Format the AI-generated content into an HTML report and send it to the hiring team via email.

The workflow integrates Google Drive and Google Sheets for document management, OpenAI GPT-4 for AI analysis and generation, and SMTP for email delivery.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Form Submission

**Overview:**  
This block captures the hiring manager’s input including the candidate’s CV (PDF), the applied job role from a predefined dropdown list, and the interview round type.

**Nodes Involved:**  
- Application form  
- Sticky Note1 (documentation)

**Node Details:**  

- **Application form**  
  - Type: Form Trigger  
  - Role: Receives user input via a web form with three fields: CV upload (PDF), Job Role dropdown, Interview Round dropdown.  
  - Configuration:  
    - CV: Single PDF file, required  
    - Job Role: Dropdown with configurable roles matching the Google Sheet positions  
    - Interview Round: Dropdown with four options (Initial Screening, Technical/Functional, Managerial/Team Fit, Final Interview)  
    - Custom CSS applied for styling  
  - Input: User form submission  
  - Output: JSON with form data including file binary for CV  
  - Edge cases: Missing required fields, unsupported file type, multiple file uploads rejected.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Explains form fields and selection context for users and maintainers.

---

#### 2.2 Candidate Profile Extraction and Analysis

**Overview:**  
This block extracts textual content from the candidate CV PDF and processes it with GPT-4 to generate structured candidate profile data.

**Nodes Involved:**  
- Extract profile  
- gpt4-1 model  
- json parser  
- Profile Analyzer Agent  
- Sticky Note2

**Node Details:**  

- **Extract profile**  
  - Type: Extract From File (PDF)  
  - Role: Converts uploaded PDF (binary from Application form) into raw text for AI processing  
  - Input: Binary file from Application form (CV)  
  - Output: JSON with extracted text content  
  - Edge cases: Corrupt or scanned PDFs may fail extraction.

- **gpt4-1 model**  
  - Type: OpenAI GPT-4 Chat Language Model  
  - Role: Processes raw CV text to extract candidate profile information  
  - Configuration: Model set to "gpt-4.1-mini" variant with default options  
  - Credentials: OpenAI API key required  
  - Input: Extracted CV text  
  - Output: AI-generated candidate profile data (likely JSON string)  
  - Edge cases: API rate limits, timeouts, malformed input.

- **json parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses GPT-4 output into a structured JSON format matching a predefined schema (candidate info including education, skills, experience, etc.)  
  - Input: GPT-4 output  
  - Output: Structured JSON candidate profile  
  - Edge cases: Parsing failures if GPT output deviates from expected format.

- **Profile Analyzer Agent**  
  - Type: Langchain Agent  
  - Role: Coordinates the GPT-4 model and parser to extract and validate candidate profile data  
  - Input: Extracted profile text from "Extract profile" node  
  - Output: Structured candidate profile JSON  
  - Edge cases: Agent misinterpreting content, partial data extraction.

- **Sticky Note2**  
  - Explains candidate profile extraction and transformation to expected JSON format.

---

#### 2.3 Job Description Retrieval and Processing

**Overview:**  
Retrieves the job description PDF URL based on selected job role from Google Sheets, downloads the PDF from Google Drive, and extracts its textual content.

**Nodes Involved:**  
- Get position JD  
- Download file  
- Extract Job Description  
- Sticky Note3

**Node Details:**  

- **Get position JD**  
  - Type: Google Sheets  
  - Role: Queries the "Positions" Google Sheet for the selected job role to obtain the associated Job Description file URL  
  - Configuration: Filters rows where "Job Role" matches the form selection  
  - Credentials: Google Sheets OAuth2  
  - Input: Job Role from Application form  
  - Output: JSON including Job Description file URL  
  - Edge cases: Role not found, API permission error.

- **Download file**  
  - Type: Google Drive  
  - Role: Downloads the Job Description PDF file using the URL retrieved from the sheet  
  - Credentials: Google Drive OAuth2  
  - Input: File ID parsed from URL  
  - Output: Binary PDF file  
  - Edge cases: File deleted or access denied.

- **Extract Job Description**  
  - Type: Extract From File (PDF)  
  - Role: Converts Job Description PDF binary to plain text for AI processing  
  - Input: Binary from Download file  
  - Output: Extracted job description text  
  - Edge cases: Corrupt PDF, scanned images.

- **Sticky Note3**  
  - Describes the process of retrieving and extracting the Job Description PDF.

---

#### 2.4 Data Merging and Interview Round Metadata Assignment

**Overview:**  
Merges candidate profile data and job description text, then enriches with detailed metadata about the chosen interview round.

**Nodes Involved:**  
- Merge  
- Transform output  
- Interview round metadata  
- Sticky Note5

**Node Details:**  

- **Merge**  
  - Type: Merge  
  - Role: Combines two input streams: candidate profile JSON and job description text  
  - Input: Output from Profile Analyzer Agent and Extract Job Description nodes  
  - Output: Array containing candidate profile and job description items

- **Transform output**  
  - Type: Code (JavaScript)  
  - Role: Processes merged data to create unified JSON objects containing candidate profile, job description, applied position, and interview round from form inputs  
  - Input: Merged array  
  - Output: Array of JSON objects ready for AI interview prep generation  
  - Code notes: Assumes last item is job description, others candidate profiles

- **Interview round metadata**  
  - Type: Set  
  - Role: Adds detailed textual metadata describing the purpose, focus areas, and goals of the selected interview round (e.g., Initial Screening, Technical Interview)  
  - Input: Transformed output from previous node  
  - Output: JSON enriched with interview round descriptions  
  - Configuration: Predefined string mappings for each round type, assigned as variables

- **Sticky Note5**  
  - Summarizes the preparation of input material for the Interview Expert Agent.

---

#### 2.5 Interview Preparation Generation with GPT-4

**Overview:**  
Uses GPT-4 (via a Langchain agent) to generate a comprehensive interview preparation report based on the candidate profile, job description, and interview round metadata.

**Nodes Involved:**  
- Interview Expert Agent  
- gpt-4-1 model 2  
- Structured Output Parser  
- Sticky Note6

**Node Details:**  

- **Interview Expert Agent**  
  - Type: Langchain Chain LLM  
  - Role: Generates interview preparation document including:  
    - Candidate summary (name, role, experience, skills)  
    - Job description summary (role, team, requirements)  
    - Interview round context (purpose, focus)  
    - 5-8 tailored interview questions with expected answers  
  - Configuration: Defines system prompt targeting TA teams; inputs include candidate profile JSON, job description text, interview round metadata  
  - Input: JSON with candidate, JD, and round data from Transform output and Interview round metadata  
  - Output: AI-generated structured JSON report  
  - Edge cases: API limits, prompt misinterpretation, output parsing errors.

- **gpt-4-1 model 2**  
  - Used internally by Interview Expert Agent as the LLM backend with OpenAI credentials.

- **Structured Output Parser**  
  - Parses Interview Expert Agent’s output into a structured JSON conforming to a predefined schema for candidate summary, job description summary, interview round, and questions.

- **Sticky Note6**  
  - Explains the purpose of this block: generating a tailored interview prep report.

---

#### 2.6 Report Formatting and Delivery

**Overview:**  
Formats the generated interview prep report into HTML and sends it via email to the hiring team.

**Nodes Involved:**  
- Build interview prep report  
- Send interview prep report to hiring team  
- Sticky Note8

**Node Details:**  

- **Build interview prep report**  
  - Type: Code (JavaScript)  
  - Role: Converts JSON report into a styled HTML document including sections for candidate summary, job description summary, interview round details, and interview questions with expected answers  
  - Input: Interview Expert Agent output JSON  
  - Output: JSON with an `html` property containing the full report  
  - Edge cases: Missing fields, malformed JSON

- **Send interview prep report to hiring team**  
  - Type: Email Send (SMTP)  
  - Role: Sends the formatted HTML interview prep report as the email body to a predefined recipient list  
  - Configuration:  
    - Subject dynamically includes candidate name and position  
    - From and To emails configured (example: no-reply@example.com to hiring-team@example.com)  
    - SMTP credentials required  
  - Input: HTML report from the previous node  
  - Output: Email sent confirmation  
  - Edge cases: SMTP auth failure, invalid email addresses, email delivery issues

- **Sticky Note8**  
  - Notes the final parsing and delivery step.

---

### 3. Summary Table

| Node Name                               | Node Type                          | Functional Role                                    | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                                                                    |
|----------------------------------------|----------------------------------|---------------------------------------------------|-------------------------------|----------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note1                           | Sticky Note                      | Explains form inputs                               |                               |                                        | ## 1. Hiring manager select candidate profile, applied position and interview round - Profile will be candidate resume in PDF format; Job role dropdown; Interview round options |
| Application form                      | Form Trigger                    | Collects CV, Job Role, and Interview Round         |                               | Get position JD, Extract profile       |                                                                                                                                                |
| Extract profile                       | Extract From File (PDF)          | Extract text from candidate CV PDF                  | Application form (binary CV)   | Profile Analyzer Agent                  |                                                                                                                                                |
| gpt4-1 model                         | OpenAI GPT-4 Chat Model          | Processes CV text to extract structured candidate profile | Extract profile               | json parser                            |                                                                                                                                                |
| json parser                         | Langchain Structured Output Parser | Parses GPT-4 output into structured JSON            | gpt4-1 model                  | Profile Analyzer Agent                  |                                                                                                                                                |
| Profile Analyzer Agent               | Langchain Agent                  | Coordinates candidate profile extraction             | Extract profile, gpt4-1 model, json parser | Merge                                 | ## 2.1. Candidate profile analyzer - Extract candidate info from PDF; transform to expected JSON format                                         |
| Get position JD                      | Google Sheets                   | Lookup JD file URL based on selected job role       | Application form              | Download file                          |                                                                                                                                                |
| Download file                       | Google Drive                    | Download JD PDF file                                 | Get position JD               | Extract Job Description                |                                                                                                                                                |
| Extract Job Description             | Extract From File (PDF)          | Extract text from JD PDF                             | Download file                 | Merge                                 | ## 2.2. Download selected job description - Get position name & JD URL from Google Sheet; download and extract JD PDF                           |
| Merge                              | Merge                          | Merge candidate profile and JD text inputs          | Profile Analyzer Agent, Extract Job Description | Transform output                      |                                                                                                                                                |
| Transform output                   | Code (JS)                      | Prepare unified JSON with candidate, JD, position, round | Merge                        | Interview round metadata               |                                                                                                                                                |
| Interview round metadata           | Set                            | Assign detailed metadata strings describing interview rounds | Transform output             | Interview Expert Agent                 |                                                                                                                                                |
| Interview Expert Agent             | Langchain Chain LLM             | Generate interview prep report with questions        | Interview round metadata      | Build interview prep report            | ## 4. Interview Expert Agent generate interview prep report - Tailored with profile, JD, interview round inputs                                  |
| gpt-4-1 model 2                   | OpenAI GPT-4 Chat Model          | Backend LLM for Interview Expert Agent               | Interview Expert Agent (internal) | Structured Output Parser             |                                                                                                                                                |
| Structured Output Parser          | Langchain Structured Output Parser | Parses Interview Expert Agent output to JSON          | Interview Expert Agent        | Build interview prep report            |                                                                                                                                                |
| Build interview prep report        | Code (JS)                      | Format JSON report as styled HTML                     | Interview Expert Agent        | Send interview prep report to hiring team |                                                                                                                                                |
| Send interview prep report to hiring team | Email Send (SMTP)               | Email report to hiring team                            | Build interview prep report   |                                        | ## 5. Parse the output and send email to hiring team                                                                                           |
| Sticky Note3                       | Sticky Note                      | Explains JD retrieval and extraction process          |                               |                                        | ![JD Screenshot](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-07-29+at+12.54.54%E2%80%AFPM.png)                        |
| Sticky Note5                       | Sticky Note                      | Explains data preparation for Interview Expert Agent  |                               |                                        | ## 3. Prepare material for Interview Expert Agent - Candidate profile, JD, interview round                                                      |
| Sticky Note6                       | Sticky Note                      | Explains Interview Expert Agent output generation     |                               |                                        | ## 4. Interview Expert Agent generate interview prep report                                                                                    |
| Sticky Note8                       | Sticky Note                      | Final step explanation for parsing & sending email    |                               |                                        | ## 5. Parse the output and send email to hiring team                                                                                           |
| Sticky Note9                       | Sticky Note                      | Visual screenshots illustrating workflow outputs       |                               |                                        | ![Interview Prep Screenshots](https://wisestackai.s3.ap-southeast-1.amazonaws.com/interview-prep-2.png) and ![Interview Prep Screenshots](https://wisestackai.s3.ap-southeast-1.amazonaws.com/interview-prep-3.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Application Form Node**  
   - Type: Form Trigger  
   - Configure form fields:  
     - CV: File upload, accept `.pdf`, single file, required  
     - Job Role: Dropdown with options:  
       - Talent acquisition Specialist (Intermediate)  
       - Senior Tester/QC (Web Apps, Mobile, API, Database)  
       - Lead Full-stack Developer (.NET, Angular/React.js)  
       - Manual Tester/QC (Senior level)  
     - Interview Round: Dropdown with options:  
       - Initial Screening  
       - Technical/Functional Interview  
       - Managerial/Team Fit Interview  
       - Final Interview  
   - Apply custom CSS styling as per workflow.

2. **Add Extract From File Node (Extract profile)**  
   - Operation: PDF  
   - Input binary property: `CV` from Application form  
   - Output: Text content of CV.

3. **Add OpenAI GPT-4 Chat Model Node (gpt4-1 model)**  
   - Model: gpt-4.1-mini  
   - Credentials: Connect your OpenAI API key  
   - Input: Text from Extract profile node.

4. **Add Langchain Structured Output Parser Node (json parser)**  
   - Paste the JSON schema example for candidate profile extraction (as per workflow).  
   - Input: GPT-4 output.

5. **Add Langchain Agent Node (Profile Analyzer Agent)**  
   - Configure to use gpt4-1 model and json parser nodes.  
   - Input: Extract profile output.  
   - Output: Structured candidate profile JSON.

6. **Add Google Sheets Node (Get position JD)**  
   - Connect Google Sheets account with OAuth2.  
   - Document ID: Your Positions sheet ID.  
   - Sheet Name: `Sheet1` or as named.  
   - Filter: Lookup "Job Role" column matching Application form's Job Role.  
   - Output: Job Description file URL.

7. **Add Google Drive Node (Download file)**  
   - Connect Google Drive OAuth2.  
   - Operation: Download file.  
   - File ID: Parse from Job Description URL output of Get position JD.

8. **Add Extract From File Node (Extract Job Description)**  
   - Operation: PDF  
   - Input binary property: from Download file node  
   - Output: Text content of Job Description PDF.

9. **Add Merge Node**  
   - Mode: Merge by index or append, combining:  
     - Profile Analyzer Agent output (candidate profile)  
     - Extract Job Description output (job description text).

10. **Add Code Node (Transform output)**  
    - JavaScript code to:  
      - Extract job description text from last merged item  
      - Loop through candidate profiles in merged items  
      - Compose JSON objects with candidate profile, job description text, applied position, and interview round (from Application form)  
    - Output: Array of unified JSON objects.

11. **Add Set Node (Interview round metadata)**  
    - Add string assignments for each interview round describing purpose, focus, and goals (copy exact text from workflow).  
    - Input: Transform output node.  
    - Output: JSON enriched with interview round metadata.

12. **Add Langchain Chain LLM Node (Interview Expert Agent)**  
    - System prompt: AI Interview Prep Assistant as described; inputs: candidate profile, job description, interview round, and interview round description.  
    - Model: gpt-4.1-mini  
    - Credentials: OpenAI API  
    - Output parser: Structured Output Parser node (configured with interview prep JSON schema).  
    - Input: Interview round metadata node.

13. **Add Langchain Structured Output Parser Node (Structured Output Parser)**  
    - Use JSON schema for interview prep report (candidate summary, JD summary, interview round, questions).  
    - Input: Interview Expert Agent output.

14. **Add Code Node (Build interview prep report)**  
    - Run once per item.  
    - JavaScript to build styled HTML containing:  
      - Candidate summary (name, position, experience, highlights, key skills)  
      - Job description summary (role, team, requirements)  
      - Interview round details (type, purpose, conducted by, focus areas)  
      - List of interview questions with expected answers  
    - Input: Parsed structured JSON from Interview Expert Agent.

15. **Add Email Send Node (Send interview prep report to hiring team)**  
    - Credentials: SMTP account (host, username, password).  
    - From: no-reply@example.com (or your sender email)  
    - To: hiring-team@example.com (or actual recipient list)  
    - Subject: "Interview Preparation Guide for [Candidate Name] – [Position]" (dynamic from Interview Expert Agent output)  
    - HTML body: Use output from Build interview prep report node.

16. **Connect all nodes as per logical flow:**  
    - Application form → Extract profile & Get position JD  
    - Get position JD → Download file → Extract Job Description  
    - Extract profile → Profile Analyzer Agent  
    - Profile Analyzer Agent & Extract Job Description → Merge → Transform output → Interview round metadata → Interview Expert Agent → Structured Output Parser → Build interview prep report → Send email node

17. **Testing and Validation:**  
    - Test with sample CV PDFs and job roles.  
    - Validate AI outputs and email delivery.  
    - Handle errors such as missing files or API timeouts gracefully.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Workflow designed for recruiters, hiring managers, and technical interviewers to automate interview question generation based on CV, job description, and interview round.                                                        | Sticky Note7 content                                                                                                             |
| Google Drive folder structure recommended: `/jd/` for job descriptions and a Google Sheet named `Positions` mapping job roles to JD file URLs.                                                                                | Sticky Note7 and Sticky Note3                                                                                                    |
| Sample Positions Sheet link: https://docs.google.com/spreadsheets/d/1pW0muHp1NXwh2GiRvGVwGGRYCkcMR7z8NyS9wvSPYjs/edit?usp=sharing                                                                                              | Sticky Note7                                                                                                                     |
| Must configure and enable Google Sheets API, Google Drive API, and OpenAI GPT-4 API key in n8n credentials. SMTP credentials required for email sending.                                                                        | Sticky Note7                                                                                                                     |
| Dropdown options and interview round metadata can be customized to include additional rounds or details like region, language, or job level.                                                                                     | Sticky Note7 “How to Customize” section                                                                                          |
| The workflow uses Langchain nodes for advanced AI prompt management and output parsing, requiring n8n version supporting these nodes (Langchain integration).                                                                     | Version-specific requirement for Langchain nodes                                                                                 |
| For visual reference, screenshots of form UI and report outputs are included as sticky notes with example images.                                                                                                               | Sticky Note3, Sticky Note9                                                                                                       |

---

**Disclaimer:**  
The provided content is extracted exclusively from an automated n8n workflow. It fully respects all applicable content policies and contains no illegal or protected material. All processed data is legal and public.