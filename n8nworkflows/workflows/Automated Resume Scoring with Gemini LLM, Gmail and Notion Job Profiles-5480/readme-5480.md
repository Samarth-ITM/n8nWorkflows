Automated Resume Scoring with Gemini LLM, Gmail and Notion Job Profiles

https://n8nworkflows.xyz/workflows/automated-resume-scoring-with-gemini-llm--gmail-and-notion-job-profiles-5480


# Automated Resume Scoring with Gemini LLM, Gmail and Notion Job Profiles

### 1. Workflow Overview

This workflow automates the process of scoring resumes against a job description with the help of advanced AI models and integrates email, parsing, and data storage services. It targets recruitment teams aiming to efficiently evaluate incoming job applications by extracting relevant candidate information, matching it against job profiles, and updating records accordingly.

The workflow is structured into the following logical blocks:

1.1 **Input Reception:** Monitors a Gmail inbox for job application emails with attachments to trigger the workflow.  
1.2 **Resume Parsing:** Uploads candidate resumes to an external parsing API (LlamaParse), polls for job completion status, and retrieves parsed resume data in markdown format.  
1.3 **Data Extraction:** Uses AI-driven information extractors to pull personal, professional, and educational details from the parsed resume content.  
1.4 **Job Profile Retrieval:** Fetches the job description profile from a Notion database page for comparison.  
1.5 **Candidate Scoring:** Employs an HR expert AI model to score candidate fitment against the job profile based on extracted data.  
1.6 **Data Update and Labeling:** Updates a Google Sheet with candidate and job information and labels the processed email to prevent duplicate processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming Gmail messages with resumes attached, filtering by specific subject and attachment presence to initiate the workflow.

**Nodes Involved:**  
- Gmail Trigger  
- Sticky Note (Step 1 Explanation)

**Node Details:**  

- **Gmail Trigger**  
  - Type: Trigger node, listens for new emails.  
  - Configuration:  
    - Filters for emails with subject containing "Job application for SDR" and having attachments.  
    - Downloads attachments automatically.  
    - Polls every minute.  
  - Inputs: External Gmail inbox.  
  - Outputs: Email metadata and attachments to "Upload to LlamaParse".  
  - Edge cases: Auth token expiration, email filter mismatch, attachment download failures.

- **Sticky Note**  
  - Purpose: Explains step 1 – waiting for resumes via email.  
  - No functional role beyond documentation.

---

#### 2.2 Resume Parsing

**Overview:**  
Uploads the resume file to LlamaParse API for structured parsing, polls for processing status, and retrieves the parsed resume in markdown format once ready.

**Nodes Involved:**  
- Upload to LlamaParse  
- Get Processing Status  
- Wait to stay within service limits  
- Is Job Ready? (Switch node)  
- Get Parsed Resume  
- Sticky Note (Parsing Explanation)

**Node Details:**  

- **Upload to LlamaParse**  
  - Type: HTTP Request node.  
  - Configuration:  
    - POST to LlamaParse upload endpoint with multipart form-data including the resume attachment.  
    - Uses bearer and header authentication.  
    - Sends attachment under form field "file".  
  - Inputs: Attachment from Gmail Trigger.  
  - Outputs: Job ID for processing status check.  
  - Edge cases: API auth failure, network timeout, malformed attachments.

- **Get Processing Status**  
  - Type: HTTP Request node.  
  - Configuration:  
    - GET request to LlamaParse job status endpoint, URL dynamically built with job ID.  
    - Authenticated with bearer token.  
  - Inputs: Job ID from Upload node or retries.  
  - Outputs: JSON with job status (SUCCESS, ERROR, CANCELED, PENDING).  
  - Edge cases: API throttling, job failure, inconsistent job ID.

- **Wait to stay within service limits**  
  - Type: Wait node.  
  - Configuration: Pauses workflow for 1 second before retrying status check to respect API rate limits.  
  - Inputs: From "Is Job Ready?" on PENDING status.  
  - Outputs: Back to "Get Processing Status".  
  - Edge cases: Delays causing timeout in upstream nodes.

- **Is Job Ready?**  
  - Type: Switch node.  
  - Configuration:  
    - Checks the "status" field of the job status JSON.  
    - Routes based on status: SUCCESS proceeds, ERROR/CANCELED can terminate or log, PENDING loops back after wait.  
  - Inputs: Job status JSON.  
  - Outputs: To "Get Parsed Resume" on SUCCESS, or to wait and retry on PENDING.  
  - Edge cases: Unexpected status values, infinite loops if status never changes.

- **Get Parsed Resume**  
  - Type: HTTP Request node.  
  - Configuration:  
    - GET request to fetch parsed resume result in markdown format using job ID.  
    - Authenticated with bearer token.  
  - Inputs: Job ID from "Is Job Ready?".  
  - Outputs: Parsed resume text for data extraction.  
  - Edge cases: Incomplete parse data, API failures.

- **Sticky Note1**  
  - Purpose: Indicates step 2, parsing the resume.

---

#### 2.3 Data Extraction

**Overview:**  
Extracts structured personal, professional, and educational information from the parsed resume markdown using specialized AI extractors.

**Nodes Involved:**  
- Professional Information Extractor  
- Educational Information Extractor  
- Personal Information Extractor  
- Merge  
- Merge1  
- Sticky Note2

**Node Details:**  

- **Professional Information Extractor**  
  - Type: LangChain Information Extractor node.  
  - Configuration:  
    - Input text: parsed resume markdown from "Get Parsed Resume".  
    - Attributes extracted: Job History, Skills (bulleted list), Titles & Employers with durations, Work Experience summary by role, Promotion count.  
  - Inputs: Parsed resume markdown.  
  - Outputs: JSON with professional info.  
  - Edge cases: Ambiguous text, missing data fields.

- **Educational Information Extractor**  
  - Type: LangChain Information Extractor.  
  - Configuration:  
    - Extracts tabular education data: schools, degrees, passing year, achievements.  
    - Input text from parsed resume markdown.  
  - Inputs: Parsed resume markdown.  
  - Outputs: JSON with educational info.  
  - Edge cases: Unstructured education details, missing graduation years.

- **Personal Information Extractor**  
  - Type: LangChain Information Extractor.  
  - Configuration:  
    - Extracts Mobile No., City, Full Name, Email.  
    - Input text from parsed resume markdown.  
  - Inputs: Parsed resume markdown.  
  - Outputs: JSON with personal info.  
  - Edge cases: Privacy concerns, incorrect extraction if text format varies.

- **Merge**  
  - Type: Merge node.  
  - Configuration: Combines outputs of Professional and Educational extractors into a single JSON object.  
  - Inputs: Outputs from Professional and Educational Information Extractors.  
  - Outputs: Combined data to "Merge1".  
  - Edge cases: Data collision or overwrites if keys overlap.

- **Merge1**  
  - Type: Merge node.  
  - Configuration: Combines output of "Personal Information Extractor" with merged professional and educational info.  
  - Inputs: Output from Merge and Personal Information Extractor.  
  - Outputs: Consolidated candidate profile data.  
  - Edge cases: Same as Merge node.

- **Sticky Note2**  
  - Purpose: Documents data extraction step identifying candidate info categories.

---

#### 2.4 Job Profile Retrieval

**Overview:**  
Fetches the job profile data from a configured Notion database page to provide the target profile for candidate matching.

**Nodes Involved:**  
- Get Job Profile  
- Sticky Note5

**Node Details:**  

- **Get Job Profile**  
  - Type: Notion node.  
  - Configuration:  
    - Operation: Get database page by URL.  
    - Page URL corresponds to the Sales Development Representative job description.  
  - Inputs: Merged candidate data (via Merge1).  
  - Outputs: Job profile data for the scoring AI.  
  - Edge cases: Notion API limits, invalid or inaccessible page URL, credential expiration.

- **Sticky Note5**  
  - Purpose: Indicates job profile retrieval step.

---

#### 2.5 Candidate Scoring

**Overview:**  
Uses an HR expert LLM chain to analyze the candidate profile against the job profile, providing detailed fitment scores and explanations.

**Nodes Involved:**  
- HR Expert LLM  
- Sticky Note3

**Node Details:**  

- **HR Expert LLM**  
  - Type: LangChain Chain LLM node.  
  - Configuration:  
    - Input template includes Job Profile URL and candidate attributes: Job History, Skills, Titles, Work Experience, Education, Name, City.  
    - Prompt instructs the model to score overall fit and specific categories (education, skills, role match) from 1 to 10, with a 100-word reasoning explanation.  
    - Uses a defined prompt type for structured output.  
  - Inputs: Job profile data and consolidated candidate info from "Get Job Profile" and "Merge1".  
  - Outputs: Scoring results for downstream processing.  
  - Edge cases: LLM API limits, prompt construction errors, ambiguous inputs.

- **Sticky Note3**  
  - Purpose: Marks HR expert scoring step.

---

#### 2.6 Data Update and Labeling

**Overview:**  
Updates candidate and job application data into a Google Sheet and applies a Gmail label to the processed email to prevent reprocessing.

**Nodes Involved:**  
- Update Gsheet  
- Add label to message  
- Sticky Note4

**Node Details:**  

- **Update Gsheet**  
  - Type: Google Sheets node.  
  - Configuration:  
    - Operation: Append or update data in a specified Google Sheet (LinkedIn Jobs Data).  
    - Auto-maps input data to sheet columns including id, title, company info, location, application count, description, and timestamps.  
  - Inputs: Scored candidate and job data from "HR Expert LLM".  
  - Outputs: Confirmation of update, passes data to labeling step.  
  - Edge cases: Google API quota limits, data type mismatches, sheet access permissions.

- **Add label to message**  
  - Type: Gmail node (not trigger).  
  - Configuration:  
    - Adds a specific Gmail label ("Label_9") to the processed email message.  
    - Uses the message ID from the initial Gmail Trigger.  
  - Inputs: Email metadata and update confirmation.  
  - Outputs: Confirmation of label addition.  
  - Edge cases: Label not found, Gmail API limits, auth expiration.

- **Sticky Note4**  
  - Purpose: Describes updating Google Sheet and labeling email to avoid duplication.

---

### 3. Summary Table

| Node Name                     | Node Type                                  | Functional Role                             | Input Node(s)                     | Output Node(s)                | Sticky Note                                            |
|-------------------------------|--------------------------------------------|---------------------------------------------|----------------------------------|------------------------------|--------------------------------------------------------|
| Gmail Trigger                 | gmailTrigger (trigger)                      | Monitor incoming job application emails     | -                                | Upload to LlamaParse          | ## Step 1: Waiting for Resume. This node is fetching resumes from Emails basis the specific Subject every hour. |
| Upload to LlamaParse          | httpRequest                                | Upload resume attachment for parsing        | Gmail Trigger                   | Get Processing Status         | ## Step 2: Parsing the Resume                             |
| Get Processing Status         | httpRequest                                | Poll parsing job status                       | Upload to LlamaParse            | Is Job Ready?                 | ## Step 2: Parsing the Resume                             |
| Is Job Ready?                 | switch                                    | Evaluate job status and route workflow       | Get Processing Status           | Get Parsed Resume, Wait to stay within service limits | ## Step 2: Parsing the Resume                             |
| Wait to stay within service limits | wait                                  | Pause to respect API rate limits             | Is Job Ready? (PENDING path)    | Get Processing Status         | ## Step 2: Parsing the Resume                             |
| Get Parsed Resume             | httpRequest                               | Retrieve parsed resume markdown               | Is Job Ready? (SUCCESS path)    | Professional Info Extractor, Educational Info Extractor, Personal Info Extractor | ## Step 2: Parsing the Resume                             |
| Professional Information Extractor | informationExtractor (LangChain)       | Extract professional data from resume        | Get Parsed Resume               | Merge                        | ## Data Extraction from Resume                            |
| Educational Information Extractor | informationExtractor (LangChain)       | Extract educational data from resume         | Get Parsed Resume               | Merge                        | ## Data Extraction from Resume                            |
| Personal Information Extractor | informationExtractor (LangChain)         | Extract personal data from resume             | Get Parsed Resume               | Merge1                       | ## Data Extraction from Resume                            |
| Merge                        | merge                                     | Combine professional and educational info    | Professional Info Extractor, Educational Info Extractor | Merge1                       | ## Data Extraction from Resume                            |
| Merge1                       | merge                                     | Combine all candidate info                     | Merge, Personal Info Extractor  | Get Job Profile              | ## Data Extraction from Resume                            |
| Get Job Profile              | notion                                    | Fetch job description profile                  | Merge1                        | HR Expert LLM                | ## Getting Job Profile                                   |
| HR Expert LLM                | chainLlm (LangChain)                       | Score candidate fit against job profile       | Get Job Profile                | Update Gsheet                | ## HR Expert to score the Candidate fitment              |
| Update Gsheet                | googleSheets                              | Append or update candidate/job data in sheet | HR Expert LLM                  | Add label to message         | ## Updating the Gsheet and Label to avoid duplication    |
| Add label to message         | gmail                                     | Add Gmail label to processed message          | Update Gsheet                  | -                            | ## Updating the Gsheet and Label to avoid duplication    |
| Sticky Note                  | stickyNote                                | Documentation / explanation                    | -                              | -                            | ## Step 1: Waiting for Resume. This node is fetching resumes from Emails basis the specific Subject every hour. |
| Sticky Note1                 | stickyNote                                | Documentation / explanation                    | -                              | -                            | ## Step 2: Parsing the Resume                             |
| Sticky Note2                 | stickyNote                                | Documentation / explanation                    | -                              | -                            | ## Data Extraction from Resume                            |
| Sticky Note3                 | stickyNote                                | Documentation / explanation                    | -                              | -                            | ## HR Expert to score the Candidate fitment              |
| Sticky Note4                 | stickyNote                                | Documentation / explanation                    | -                              | -                            | ## Updating the Gsheet and Label to avoid duplication    |
| Sticky Note5                 | stickyNote                                | Documentation / explanation                    | -                              | -                            | ## Getting Job Profile                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Node Type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail access.  
   - Set filter query: `subject: Job application for SDR, has:attachment`  
   - Enable attachment download.  
   - Set polling interval to every 1 minute.

2. **Add HTTP Request Node to Upload Resume:**  
   - Node Name: Upload to LlamaParse  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.cloud.llamaindex.ai/api/parsing/upload`  
   - Authentication: HTTP Bearer and Header Auth (set with provided credentials).  
   - Body Type: Multipart form-data, parameter "file" set to attachment from Gmail Trigger.  
   - Headers: Accept: application/json.

3. **Add HTTP Request Node to Check Job Status:**  
   - Node Name: Get Processing Status  
   - Node Type: HTTP Request  
   - Method: GET  
   - URL template: `https://api.cloud.llamaindex.ai/api/parsing/job/{{ $json.id }}` (dynamic job ID).  
   - Authentication: HTTP Bearer and Header Auth.

4. **Add Switch Node to Check Job Status:**  
   - Node Name: Is Job Ready?  
   - Node Type: Switch  
   - Condition: Evaluate `$json.status` equals SUCCESS, ERROR, CANCELED, or PENDING.  
   - Routes: SUCCESS to next step; PENDING to wait/retry; ERROR/CANCELED for error handling.

5. **Add Wait Node:**  
   - Node Name: Wait to stay within service limits  
   - Node Type: Wait  
   - Duration: 1 second  
   - Connect PENDING output of Switch node back to Get Processing Status node.

6. **Add HTTP Request Node to Get Parsed Resume:**  
   - Node Name: Get Parsed Resume  
   - Node Type: HTTP Request  
   - Method: GET  
   - URL template: `https://api.cloud.llamaindex.ai/api/parsing/job/{{ $json.id }}/result/markdown`  
   - Authentication: HTTP Bearer and Header Auth.

7. **Add Three LangChain Information Extractor Nodes:**  
   - **Professional Information Extractor:**  
     - Input: `{{$json.markdown}}` from Get Parsed Resume.  
     - Attributes: Job History, Skills, Titles & Employers, Work Experience, Promotions Checker.  
   - **Educational Information Extractor:**  
     - Input: `{{$json.markdown}}` from Get Parsed Resume.  
     - Attribute: Education Information (tabular).  
   - **Personal Information Extractor:**  
     - Input: `{{$json.markdown}}` from Get Parsed Resume.  
     - Attributes: Mobile No., City, Full Name, Email.

8. **Add Merge Node:**  
   - Node Name: Merge  
   - Mode: Combine Inputs  
   - Inputs: Professional Information Extractor and Educational Information Extractor outputs.

9. **Add Merge Node:**  
   - Node Name: Merge1  
   - Mode: Combine Inputs  
   - Inputs: Merge node output and Personal Information Extractor output.

10. **Add Notion Node:**  
    - Node Name: Get Job Profile  
    - Operation: Get database page.  
    - Page URL: Use the Notion page URL for the job description.  
    - Configure Notion API credentials.

11. **Add LangChain Chain LLM Node:**  
    - Node Name: HR Expert LLM  
    - Input Template: Include Job Profile URL and candidate profile data from Merge1.  
    - Prompt: Request scoring from 1 to 10 on overall fit, education, skills, roles, plus 100 words reasoning.  
    - Configure appropriate LangChain/OpenAI/Gemini credentials.

12. **Add Google Sheets Node:**  
    - Node Name: Update Gsheet  
    - Operation: Append or update rows.  
    - Document ID and Sheet name: Set to target Google Sheet for storing candidate/job data.  
    - Map inputs to sheet columns, enable auto-mapping.  
    - Configure Google OAuth2 credentials.

13. **Add Gmail Node to Add Label:**  
    - Node Name: Add label to message  
    - Operation: Add label to Gmail message.  
    - Label ID: "Label_9" (or existing label ID to mark processed emails).  
    - Message ID: From Gmail Trigger node output.  
    - Configure Gmail OAuth2 credentials.

14. **Connect Nodes Sequentially:**  
    - Gmail Trigger → Upload to LlamaParse → Get Processing Status → Is Job Ready?  
    - From Is Job Ready? SUCCESS → Get Parsed Resume → Professional, Educational, Personal Extractors  
    - Merge Professional + Educational → Merge1 with Personal  
    - Merge1 → Get Job Profile → HR Expert LLM → Update Gsheet → Add label to message  
    - Is Job Ready? PENDING → Wait → Get Processing Status (loop)

15. **Add Sticky Notes (Optional):**  
    - Add sticky notes to explain key steps for clarity and documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| The workflow uses LlamaParse API for resume parsing: https://www.llamaindex.ai                | External parsing API for extracting structured resume data.                                                     |
| Google Gemini (PaLM) API powers the AI model nodes for extraction and scoring.               | Requires valid Google Cloud credentials with PaLM API enabled.                                                  |
| Gmail OAuth2 integration requires appropriate scopes to read emails, download attachments, and modify labels. | Gmail API scopes: `https://www.googleapis.com/auth/gmail.readonly`, `https://www.googleapis.com/auth/gmail.modify` |
| Notion API is used to fetch job descriptions stored as database pages.                       | Requires Notion integration with read access to specified pages.                                                |
| Google Sheets OAuth2 credentials must have write access to the target spreadsheet.           | Google Sheets API enabled in Google Cloud Console.                                                              |
| The workflow applies a Gmail label “Label_9” to processed emails to prevent duplicate processing. | Ensure the label exists in Gmail settings.                                                                       |
| The HR Expert LLM prompt is designed for detailed scoring and explanation, providing actionable insights. | Prompt design is customizable for different roles or scoring criteria.                                           |
| Workflow respects API rate limits by polling with a 1-second wait between LlamaParse status checks. | Prevents hitting rate limits and ensures timely processing.                                                     |

---

**Disclaimer:** The text is derived exclusively from an automated n8n workflow, strictly adhering to content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public.