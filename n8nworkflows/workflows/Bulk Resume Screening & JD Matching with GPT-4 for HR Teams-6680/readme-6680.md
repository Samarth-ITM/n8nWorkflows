Bulk Resume Screening & JD Matching with GPT-4 for HR Teams

https://n8nworkflows.xyz/workflows/bulk-resume-screening---jd-matching-with-gpt-4-for-hr-teams-6680


# Bulk Resume Screening & JD Matching with GPT-4 for HR Teams

### 1. Workflow Overview

This workflow, titled **"TalentFlow AI – Bulk Resume Screening with JD Matching"**, is designed for HR and Talent Acquisition (TA) teams to automate the process of screening multiple candidate resumes against job descriptions (JDs) using GPT-4 AI models. It streamlines bulk resume evaluation, generates structured candidate assessments, and updates evaluation tracking sheets while optionally sending notifications via Slack or email.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception**: HR/TA uploads multiple resumes and selects the job role via a custom form.
- **2.1 Candidate Profile Extraction & Parsing**: Uploaded resumes are saved to Google Drive, then parsed with GPT-4 to extract structured candidate profile data.
- **2.2 Job Description Retrieval & Parsing**: Based on the selected role, the corresponding job description PDF is fetched from Google Drive and parsed.
- **3. AI Candidate Evaluation**: The candidate profile and job description are combined and sent to an AI agent to evaluate fit, strengths, gaps, and provide recommendations.
- **4. Output & Notifications**: Evaluation results are mapped and appended to a Google Sheets tracking sheet, and qualified candidates trigger Slack notifications and optionally emails.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception

**Overview:**  
Facilitates HR/TA team input by allowing bulk upload of resumes and selection of the corresponding job role through a web form.

**Nodes Involved:**  
- Application form  
- Code  
- Upload to Google Drive  
- Sticky Note1 (documentation)

**Node Details:**

- **Application form**  
  - *Type:* Form Trigger (webhook)  
  - *Role:* Captures multiple candidate resumes (.pdf) and selected job role from HR/TA via a web UI form.  
  - *Configuration:* Custom CSS styling for branding; fields include file upload (accepts PDF only) and a dropdown with predefined job roles.  
  - *Inputs:* External user form submission.  
  - *Outputs:* Form data including uploaded binary files and selected job role.  
  - *Edge Cases:* Large file uploads may timeout or exceed limits; invalid file types rejected by form input.  
  - *Notes:* Triggers downstream processing upon submission.

- **Code**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Splits the incoming form data containing multiple uploaded resumes into individual items, each holding one resume binary and JSON metadata. This enables parallel processing of each candidate's resume.  
  - *Key Logic:* Iterates over binary data fields starting with "CV_" to create separate items.  
  - *Input:* Application form output.  
  - *Output:* Multiple items, each with one candidate’s resume binary file and metadata.  
  - *Edge Cases:* If no CV files are uploaded, output will be empty causing downstream nodes to have no input.

- **Upload to Google Drive**  
  - *Type:* Google Drive node  
  - *Role:* Saves each uploaded resume PDF to a specified Google Drive folder for archival and further processing.  
  - *Configuration:* Dynamically names files using timestamp and original filename; targets a fixed Drive folder URL.  
  - *Input:* Individual resume items from Code node.  
  - *Output:* Metadata about uploaded file including Drive file ID, used downstream.  
  - *Edge Cases:* Drive API errors, permission issues, or network failures may cause upload failures.

- **Sticky Note1**  
  - *Type:* Sticky Note (documentation)  
  - *Content:* Indicates this block’s purpose as the HR/TA upload interface with job role selection.

---

#### Block 2.1: Candidate Profile Extraction & Parsing

**Overview:**  
Extracts textual content from uploaded resume PDFs and processes it with GPT-4 to produce structured candidate profile JSON.

**Nodes Involved:**  
- Extract profile  
- gpt4-1 model  
- json parser  
- Profile Analyzer Agent  
- Remove bad data CV  
- Sticky Note (descriptive)  
- Sticky Note3 (visual aid)

**Node Details:**

- **Extract profile**  
  - *Type:* Extract From File (PDF)  
  - *Role:* Converts uploaded resume PDF binary files into plain text content for AI processing.  
  - *Input:* Uploaded resumes from Google Drive node output (file metadata and binary).  
  - *Output:* Extracted raw text from PDF resumes.  
  - *Edge Cases:* Complex PDF formatting or scanned images may reduce extraction accuracy.

- **gpt4-1 model**  
  - *Type:* LangChain OpenAI Chat Model (GPT-4.1-mini)  
  - *Role:* Processes extracted resume text to extract relevant candidate profile information.  
  - *Configuration:* Uses GPT-4.1-mini model with default options, authenticated via OpenAI API credentials.  
  - *Input:* Resume text from Extract profile node.  
  - *Output:* AI-generated structured candidate data (raw JSON string).  
  - *Edge Cases:* API rate limits, malformed text inputs, or unexpected resume formats may degrade output quality.

- **json parser**  
  - *Type:* LangChain Output Parser Structured  
  - *Role:* Parses the AI-generated JSON string into a structured JSON object per a predefined candidate profile schema (including personal details, education, experience, skills, etc.).  
  - *Input:* Raw AI output from gpt4-1 model.  
  - *Output:* Parsed candidate profile JSON object.  
  - *Edge Cases:* Invalid or incomplete JSON from AI may cause parse failures.

- **Profile Analyzer Agent**  
  - *Type:* LangChain Agent (AI agent)  
  - *Role:* Validates and refines the candidate profile extraction by applying a defined prompt requesting all relevant candidate info extraction.  
  - *Input:* Parsed JSON profile from json parser and extracted resume text.  
  - *Output:* Finalized candidate profile JSON.  
  - *Edge Cases:* AI misinterpretation or incomplete extraction.

- **Remove bad data CV**  
  - *Type:* Filter node  
  - *Role:* Filters out candidate profiles where critical data (e.g., full_name) is missing, ensuring only valid profiles proceed.  
  - *Input:* Candidate profiles from Profile Analyzer Agent.  
  - *Output:* Filtered list of valid candidate profiles.  
  - *Edge Cases:* Overly strict filtering may remove valid candidates; insufficient validation may allow bad data.

- **Sticky Notes**  
  - Provide documentation and screenshots illustrating candidate profile analysis and extraction steps.

---

#### Block 2.2: Job Description Retrieval & Parsing

**Overview:**  
Fetches the job description PDF for the selected job role from Google Sheets and Google Drive, then extracts its text content for evaluation.

**Nodes Involved:**  
- Get position JD  
- Download file  
- Extract Job Description  
- Sticky Note2 (documentation)

**Node Details:**

- **Get position JD**  
  - *Type:* Google Sheets node  
  - *Role:* Looks up the selected job role in a Google Sheets document to retrieve the corresponding job description file URL or ID.  
  - *Configuration:* Filters rows by the job role selected in the application form; accesses a specific sheet within a Google Sheets document.  
  - *Input:* Job role from Application form node.  
  - *Output:* Job description file metadata (file ID/URL).  
  - *Edge Cases:* Missing or incorrect job role entries, Google Sheets API errors.

- **Download file**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the job description PDF file from Google Drive using the file ID obtained.  
  - *Input:* File ID from Get position JD node.  
  - *Output:* Job description PDF binary.  
  - *Edge Cases:* Permissions issues, missing file, or API errors.

- **Extract Job Description**  
  - *Type:* Extract From File (PDF)  
  - *Role:* Converts job description PDF binary into plain text for AI evaluation.  
  - *Input:* PDF binary from Download file node.  
  - *Output:* Extracted job description text.  
  - *Edge Cases:* Poor PDF formatting or scanned files.

- **Sticky Note2**  
  - Explains the job description download and extraction process.

---

#### Block 3: AI Candidate Evaluation

**Overview:**  
Combines candidate profiles and job description text, then uses GPT-4-based AI to evaluate each candidate's fit, strengths, gaps, and recommendation.

**Nodes Involved:**  
- Merge  
- Code1  
- HR Expert Agent  
- gpt-4-1 model 2  
- json parser 2  
- Map Columns  
- Sticky Note5 (documentation)

**Node Details:**

- **Merge**  
  - *Type:* Merge node (wait mode)  
  - *Role:* Combines processed candidate profiles and the extracted job description text into a single dataset for evaluation.  
  - *Input:* Candidate profiles (filtered) and extracted job description text.  
  - *Output:* Combined data for AI evaluation.  
  - *Edge Cases:* Timing issues if one input stream is delayed.

- **Code1**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Prepares an array of evaluation input objects, each containing a candidate profile, the job description text, and the applied position.  
  - *Input:* Merged data from candidate profiles and JD text.  
  - *Output:* Array of evaluation objects, one per candidate.  
  - *Edge Cases:* Incorrect indexing or missing data could cause evaluation failures.

- **HR Expert Agent**  
  - *Type:* LangChain Chain LLM (GPT-4 agent)  
  - *Role:* Evaluates each candidate against the job description by parsing job requirements, matching candidate data, and generating a structured evaluation with scores and recommendations.  
  - *Configuration:* Uses a detailed prompt defining the evaluation criteria and output format.  
  - *Input:* Candidate profile + job description from Code1 node.  
  - *Output:* Raw evaluation JSON.  
  - *Edge Cases:* Model API limits, ambiguous job descriptions, or incomplete candidate data.

- **gpt-4-1 model 2**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Underlying language model used by HR Expert Agent to generate AI evaluations.  
  - *Input:* Prompt from HR Expert Agent.  
  - *Output:* AI-generated evaluation content.  
  - *Edge Cases:* Same as HR Expert Agent.

- **json parser 2**  
  - *Type:* LangChain Output Parser Structured  
  - *Role:* Parses AI-generated evaluation text into structured JSON format as defined by schema (including fit score, recommendation, strengths, gaps, notes).  
  - *Input:* Raw AI evaluation output.  
  - *Output:* Parsed structured evaluation object.  
  - *Edge Cases:* Parsing errors if AI output deviates from schema.

- **Map Columns**  
  - *Type:* Code node  
  - *Role:* Transforms evaluation JSON into a flat JSON object matching Google Sheets columns for appending.  
  - *Input:* Parsed evaluation JSON.  
  - *Output:* Mapped evaluation data with fields like full_name, email, position, fit score, recommendation, strengths, concerns, final notes, and timestamp.  
  - *Edge Cases:* Missing fields could result in empty values.

- **Sticky Note5**  
  - Describes the AI candidate evaluation purpose and outputs.

---

#### Block 4: Output & Notifications

**Overview:**  
Updates evaluation results to a Google Sheets tracking sheet, sends Slack notifications for qualified candidates, and optionally emails unqualified candidates.

**Nodes Involved:**  
- Update evaluation sheet  
- Candidate qualified?  
- Send message via Slack to the hiring team  
- Send email to candidate about the result (disabled by default)  
- Sticky Note6, Sticky Note8 (documentation)

**Node Details:**

- **Update evaluation sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends each candidate’s evaluation results to a designated Google Sheets document for tracking and auditing.  
  - *Configuration:* Maps multiple columns (e.g., fit score, recommendation, skills) based on auto-mapping from input data; targets a specific Google Sheets URL and sheet name.  
  - *Input:* Mapped evaluation data from Map Columns node.  
  - *Output:* Confirmation of append operation.  
  - *Edge Cases:* API quota limits, malformed data, or access issues.

- **Candidate qualified?**  
  - *Type:* If node (conditional)  
  - *Role:* Checks if the candidate’s overall fit score is greater than or equal to 8 to determine qualification.  
  - *Input:* Evaluation data from Update evaluation sheet.  
  - *Output:* Two branches: true (qualified), false (unqualified).  
  - *Edge Cases:* Missing or non-numeric scores could cause incorrect branching.

- **Send message via Slack to the hiring team**  
  - *Type:* Slack node  
  - *Role:* Sends a formatted Slack message with candidate evaluation summary to a designated channel when a candidate is qualified.  
  - *Configuration:* Uses OAuth2 authentication; message includes candidate name, position, fit score, recommendation, and strengths; channel is preconfigured.  
  - *Input:* True branch from Candidate qualified? node.  
  - *Output:* Slack API response.  
  - *Edge Cases:* Slack API errors, invalid channel, or auth token expiry.

- **Send email to candidate about the result**  
  - *Type:* SendGrid node (disabled by default)  
  - *Role:* Sends an email to unqualified candidates informing them of the decision politely.  
  - *Configuration:* HTML email template personalized with candidate name and position; uses SendGrid API credentials.  
  - *Input:* False branch from Candidate qualified? node.  
  - *Output:* Email send confirmation.  
  - *Edge Cases:* Email delivery failures, invalid email addresses.

- **Sticky Note6 and Sticky Note8**  
  - Describe the update and notification process, including screenshots of evaluation sheets.

---

### 3. Summary Table

| Node Name                       | Node Type                           | Functional Role                                | Input Node(s)                     | Output Node(s)                                | Sticky Note                                              |
|--------------------------------|-----------------------------------|------------------------------------------------|----------------------------------|-----------------------------------------------|----------------------------------------------------------|
| Application form               | Form Trigger                      | Collect resumes and job role from HR/TA       | -                                | Code, Get position JD                         | ## 1. HR/TA upload multiple resumes & select the position (job role) |
| Code                          | Code                             | Split multiple uploaded resumes into single items | Application form                | Upload to Google Drive, Extract profile       | ## 1. HR/TA upload multiple resumes & select the position (job role) |
| Upload to Google Drive         | Google Drive                     | Save uploaded resumes to Drive folder          | Code                            | Extract profile                              | ## 2.1 Candidate profiles analyzer                        |
| Extract profile               | Extract From File (PDF)           | Extract text from resumes                       | Upload to Google Drive           | gpt4-1 model                                  | ## 2.1 Candidate profiles analyzer                        |
| gpt4-1 model                  | LangChain OpenAI Model (GPT-4)   | Generate structured candidate profile          | Extract profile                 | json parser                                   | ## 2.1 Candidate profiles analyzer                        |
| json parser                  | LangChain Output Parser Structured | Parse AI JSON output into structured profile   | gpt4-1 model                   | Profile Analyzer Agent                         | ## 2.1 Candidate profiles analyzer                        |
| Profile Analyzer Agent         | LangChain Agent                  | Finalize candidate profile extraction           | json parser                    | Remove bad data CV                            | ## 2.1 Candidate profiles analyzer                        |
| Remove bad data CV             | Filter                          | Remove profiles missing critical data           | Profile Analyzer Agent          | Merge                                         | ## 2.1 Candidate profiles analyzer                        |
| Get position JD               | Google Sheets                   | Fetch job description file ID based on job role | Application form               | Download file                                 | ## 2.2 Download selected job description                  |
| Download file                | Google Drive                   | Download the job description PDF                 | Get position JD                | Extract Job Description                       | ## 2.2 Download selected job description                  |
| Extract Job Description       | Extract From File (PDF)           | Extract text from job description PDF            | Download file                 | Merge                                         | ## 2.2 Download selected job description                  |
| Merge                        | Merge node                     | Combine candidate profiles and job description   | Remove bad data CV, Extract Job Description | Code1                                  | ## 3. HR AI Agent evaluate candidate matching             |
| Code1                        | Code                            | Prepare evaluation input objects for AI          | Merge                         | HR Expert Agent                               | ## 3. HR AI Agent evaluate candidate matching             |
| HR Expert Agent              | LangChain Chain LLM             | Evaluate candidate fit against job description   | Code1                         | Map Columns                                   | ## 3. HR AI Agent evaluate candidate matching             |
| gpt-4-1 model 2               | LangChain OpenAI Model (GPT-4)   | Underlying model for HR Expert Agent             | HR Expert Agent               | json parser 2                                 | ## 3. HR AI Agent evaluate candidate matching             |
| json parser 2                | LangChain Output Parser Structured | Parse AI evaluation output into structured JSON  | gpt-4-1 model 2               | Map Columns                                   | ## 3. HR AI Agent evaluate candidate matching             |
| Map Columns                 | Code                            | Map evaluation JSON to Google Sheets columns     | HR Expert Agent               | Update evaluation sheet, Candidate qualified? | ## 4. Update evaluation result to target platform         |
| Update evaluation sheet      | Google Sheets                   | Append candidate evaluation results to sheet     | Map Columns                  | Candidate qualified?                          | ## 4. Update evaluation result to target platform         |
| Candidate qualified?         | If                             | Check if candidate score >= 8 to branch workflow | Update evaluation sheet       | Send message via Slack, Send email to candidate | ## 4. Update evaluation result to target platform         |
| Send message via Slack       | Slack                          | Notify hiring team of qualified candidate         | Candidate qualified? (true)    | -                                             | ## 4. Update evaluation result to target platform         |
| Send email to candidate about the result | SendGrid (disabled)             | Email unqualified candidates                       | Candidate qualified? (false)   | -                                             | ## 4. Update evaluation result to target platform         |
| Sticky Note1                 | Sticky Note                    | Documentation for block 1 input reception          | -                            | -                                             | ## 1. HR/TA upload multiple resumes & select the position (job role) |
| Sticky Note                  | Sticky Note                    | Documentation for candidate profile extraction     | -                            | -                                             | ## 2.1 Candidate profiles analyzer                        |
| Sticky Note2                 | Sticky Note                    | Documentation for job description download         | -                            | -                                             | ## 2.2 Download selected job description                  |
| Sticky Note3                 | Sticky Note                    | Visual aid for candidate profile extraction        | -                            | -                                             | ## 2.1 Candidate profiles analyzer                        |
| Sticky Note5                 | Sticky Note                    | Documentation for AI candidate evaluation           | -                            | -                                             | ## 3. HR AI Agent evaluate candidate matching             |
| Sticky Note6                 | Sticky Note                    | Documentation for update and notification block     | -                            | -                                             | ## 4. Update evaluation result to target platform         |
| Sticky Note7                 | Sticky Note                    | Overall workflow description and setup instructions | -                            | -                                             | Full workflow overview and instructions                   |
| Sticky Note8                 | Sticky Note                    | Screenshot of evaluation sheet sample data          | -                            | -                                             | ## 4. Update evaluation result to target platform         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("Application form")**:  
   - Type: Form Trigger  
   - Configure form fields:  
     - File upload field, label "CV", required, accept ".pdf"  
     - Dropdown field, label "Job Role", required, options:  
       - Talent acquisition Specialist (Intermediate)  
       - Senior Tester/QC (Web Apps, Mobile, API, Database)  
       - Lead Full-stack Developer (.NET, Angular/React.js)  
       - Manual Tester/QC (Senior level)  
   - Customize the form appearance with CSS as desired.

2. **Add a Code node ("Code")** after the form to split multiple resumes:  
   - JavaScript code: iterate over binary inputs starting with "CV_" and create separate items per resume.

3. **Add a Google Drive Upload node ("Upload to Google Drive")**:  
   - Configure with your Google Drive OAuth2 credentials.  
   - Set file name to `cv-{{ $now.toFormat("yyyyLLdd-HHmmss") }}-{{$binary.data.fileName}}`.  
   - Select target Drive folder URL for storing resumes.

4. **Add an Extract From File node ("Extract profile")**:  
   - Operation: PDF Text Extraction.  
   - Input: Uploaded resume file binary from previous step.

5. **Add a LangChain OpenAI Chat Model node ("gpt4-1 model")**:  
   - Use GPT-4.1-mini model.  
   - Connect OpenAI credentials.  
   - Input: Extracted resume text.

6. **Add a LangChain Output Parser Structured node ("json parser")**:  
   - Define the expected JSON schema for candidate profiles (as per example in the workflow).  
   - Input: AI output from GPT-4 model.

7. **Add a LangChain Agent node ("Profile Analyzer Agent")**:  
   - Prompt the AI to extract all relevant candidate info from the resume text.  
   - Use the parsed JSON from previous step as input.

8. **Add a Filter node ("Remove bad data CV")**:  
   - Condition: Filter out items where full_name is empty or missing.

9. **Create a Google Sheets node ("Get position JD")**:  
   - Connect your Google Sheets OAuth2 credentials.  
   - Configure to filter rows by job role selected in the Application form.  
   - Access your Positions sheet which maps job roles to JD file links.

10. **Add a Google Drive node ("Download file")**:  
    - Download the JD PDF using file ID from the previous node.

11. **Add an Extract From File node ("Extract Job Description")**:  
    - Extract text from the JD PDF.

12. **Add a Merge node ("Merge")**:  
    - Configure to wait for all inputs.  
    - Merge the filtered candidate profiles and the extracted job description text.

13. **Add a Code node ("Code1")**:  
    - Prepare an array pairing each candidate profile with the job description and selected job role for AI evaluation.

14. **Add a LangChain Chain LLM node ("HR Expert Agent")**:  
    - Prompt the AI to evaluate candidate fit against job description.  
    - Use GPT-4.1-mini with OpenAI credentials.

15. **Add a LangChain Output Parser Structured node ("json parser 2")**:  
    - Define JSON schema for evaluation results (fit score, recommendation, strengths, gaps, notes).

16. **Add a Code node ("Map Columns")**:  
    - Map evaluation results into flat JSON fields matching your Google Sheets evaluation tracking sheet columns.

17. **Add a Google Sheets node ("Update evaluation sheet")**:  
    - Append mapped evaluation data to your evaluation tracking sheet.  
    - Use your Google Sheets OAuth2 credentials.

18. **Add an If node ("Candidate qualified?")**:  
    - Condition: Check if overall_fit_score ≥ 8.

19. **Add a Slack node ("Send message via Slack to the hiring team")**:  
    - Send formatted notification to the hiring team channel for qualified candidates.  
    - Use Slack OAuth2 credentials.

20. **Add a SendGrid node ("Send email to candidate about the result")** (optional and disabled by default):  
    - Send rejection email to candidates who do not qualify.  
    - Configure with SendGrid API credentials.

21. **Connect nodes as per flow:**  
    - Application form → Code → Upload to Google Drive → Extract profile → gpt4-1 model → json parser → Profile Analyzer Agent → Remove bad data CV → Merge (with Extract Job Description output) → Code1 → HR Expert Agent → json parser 2 → Map Columns → Update evaluation sheet → Candidate qualified? → Slack or SendGrid.

22. **Configure credentials:**  
    - OpenAI API for GPT-4 nodes.  
    - Google Sheets OAuth2 for Sheets nodes.  
    - Google Drive OAuth2 for Drive nodes.  
    - Slack OAuth2 for Slack node.  
    - SendGrid API key for email node (optional).

23. **Set up Google Sheets:**  
    - Positions sheet mapping job roles → JD file links.  
    - Evaluation tracking sheet with columns: overall_fit_score, recommendation, required_experience, skill_match, education_certifications, soft_skills_traits, strengths, concerns_or_gaps, final_notes.

24. **Prepare Google Drive folders:**  
    - Folder for uploaded resumes.  
    - Folder for job descriptions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                          | Context or Link                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Workflow automates bulk resume screening and JD matching, designed to reduce bias and save time for HR/TA teams.                                                                                                                                      | Workflow purpose                                                |
| Workflow requires OpenAI GPT-4 API, Google Drive and Sheets access, Slack for notifications, and optionally SendGrid for emails.                                                                                                                     | Integration requirements                                        |
| Customizable fit score threshold (default 8) controls candidate qualification criteria.                                                                                                                                                               | Customization tip                                               |
| Slack messages and email templates can be customized for tone and content.                                                                                                                                                                            | Communication customization                                     |
| Workflow UI form styled with Open Sans font and corporate colors; CSS can be modified in Application form node parameters.                                                                                                                            | UI customization                                               |
| Google Sheets must have a Positions sheet with columns "Job Role" and "Job Description" (file link or ID), and an Evaluation sheet with columns matching mapped output fields.                                                                       | Google Sheets setup                                            |
| Google Drive folders must be pre-created for CV uploads and job descriptions, with proper access permissions.                                                                                                                                          | Drive folder setup                                             |
| Sticky notes within the workflow provide detailed step descriptions and visual aids for clarity.                                                                                                                                                      | Embedded documentation                                         |
| AI prompt templates are designed for objective, professional language and structured markdown outputs, ensuring consistency and auditability.                                                                                                         | AI prompt design                                                |
| Optionally integrate with ATS or job boards by replacing the form trigger with webhook triggers for automated candidate ingestion.                                                                                                                  | Workflow extensibility                                         |
| Links to n8n official site for automation platform details: https://n8n.io                                                                                                                                                                            | Platform information                                           |
| Workflow created with best practices for error handling: filtering invalid data, handling API errors with retries (recommended), and conditional branching based on fit score.                                                                       | Error handling advice                                          |

---

**Disclaimer:**  
This documentation is based exclusively on an automated workflow designed and implemented in n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected materials. All data processed is lawful and publicly accessible.